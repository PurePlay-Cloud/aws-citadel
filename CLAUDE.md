# CLAUDE.md

Citadel — a multi-agent AI platform for enterprise application development on AWS Bedrock AgentCore. Four-layer monorepo, deployed as 5 AWS CDK stacks.

Read **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** first for the big picture (layers, data flows, DynamoDB tables, patterns). This file captures the operational detail and gotchas that the docs don't.

## Layout

| Dir | Layer | Stack | Lang |
|-----|-------|-------|------|
| `backend/` | AppSync GraphQL API, Cognito, DynamoDB, EventBridge, 50+ Lambda resolvers, **all CDK stacks live here** | `citadel-backend-{env}` | CDK TypeScript (Node 24) |
| `service/` | Bedrock AgentCore Runtime (`agent_intake_single`), OpenSearch KB, AgentCore Gateway | `citadel-services-{env}` | Python 3.13 (no package.json) |
| `arbiter/` | Multi-agent orchestration: Supervisor, Fabricator, Worker Wrapper, Step Runner | `citadel-arbiter-{env}` | Python 3.14 Lambdas |
| `frontend/` | React 18 + Vite + Amplify SPA | `citadel-frontend-{env}` | TypeScript |
| (in backend) | Per-app API Gateway publishing | `citadel-gateway-{env}` | TypeScript |
| `docs/` | 13 architecture + how-to guides | — | — |

**The CDK app and all stack definitions live in `backend/bin/app.ts` + `backend/lib/*.ts`.** The arbiter and service layers are *deployed by* the backend CDK (as Python/container Lambda assets) — they have no deploy tooling of their own. `service/` is in npm workspaces but has no `package.json`.

## Deploy

```bash
aws sso login --profile ppc-sandpit          # REQUIRED first — SSO profile, else "no credentials configured"
./deploy.sh --profile ppc-sandpit            # canonical full deploy (builds frontend+backend, deploys 5 stacks)
```

First deploy only — bootstrap once:
```bash
cd backend && npx cdk bootstrap aws://041073296931/ap-southeast-2 --profile ppc-sandpit
```

`./deploy.sh` flags: `--backend-only` (backend+services+arbiter+gateway, no frontend), `--frontend-only`, `--skip-frontend`/`--skip-backend` (skip a build phase), `--dry-run` (build + `cdk diff` only), `--no-verify` (skip health check), `--admin-email <addr>`. A bare positional arg (e.g. `./deploy.sh citadel-backend-dev`) does a single-stack deploy. Env (`dev`/`staging`/`prod`) comes from `ENVIRONMENT` in `backend/.env` — **there is no `--env` flag**.

Teardown: `./clean.sh` (local artifacts only) or `./clean.sh --aws --profile ppc-sandpit` (destroys stacks too — note `citadel-agent-*` IAM roles are orphaned and must be deleted by hand).

**Deploy prerequisites that will silently break a build:**
- **Container runtime must be running.** CDK bundles Python/container Lambdas. `brew install finch && finch vm init`, set `CDK_DOCKER=finch` (Docker Desktop is intentionally avoided for licensing). `deploy.sh` auto-starts the Finch VM but aborts if no runtime is found.
- **Node 24+ and Python 3.14+** are required. (Root `package.json` says `node>=18` and the README mentions Node 18 — both are stale.)
- Admin password is **auto-generated into Secrets Manager** (`citadel/admin-password-{env}`) on first deploy — do not set it in `.env`; change it on first login.

## Build / test / lint (per layer)

Backend (`cd backend`):
```bash
npm run build           # tsc → dist/  (REQUIRED before any cdk command — cdk.json runs `node dist/bin/app.js`)
npm run build:lambda    # esbuild-bundle Lambda handlers → dist/lambda/  (SEPARATE step; see gotchas)
npm test                # jest unit (ts-jest); -- --testPathPattern=<name> for one file
npm run test:integration
npm run cdk:diff / cdk:synth / nag   # nag = cdk synth --all (cdk-nag AwsSolutions runs on EVERY synth and fails on violations)
```

Frontend (`cd frontend`):
```bash
npm install             # NOT `npm ci` — no package-lock.json is committed
npm run dev             # Vite dev server on http://localhost:3000 (not 5173)
npm run build           # → build/  (not dist/; backend FrontendStack consumes frontend/build/)
npm test                # jest + jsdom
```

Arbiter (Python — run pytest **directly from repo root**, there is no npm wrapper):
```bash
pip install pytest hypothesis boto3      # test deps are NOT pinned anywhere
pytest arbiter/ -v --tb=short            # exactly how CI runs them (Hypothesis property-based tests)
```

Root: `npm run dev` / `build` / `test` / `lint` fan out to service+backend+frontend (NOT arbiter). `npm run install:all` installs every workspace.

## Critical gotchas

- **`npm run build` before any `cdk` command.** `cdk.json` runs `node dist/bin/app.js`; a stale `dist/` silently deploys old infra code.
- **`build:lambda` is separate from `build`.** `cdk deploy` and `npm run deploy:quick` do **not** re-bundle Lambda handlers — Lambda code changes need `npm run build:lambda` first. Only full `npm run deploy` and `./deploy.sh` chain both.
- **`build:lambda` is a two-pass esbuild**, not a plain glob. Pass 1 bundles `src/lambda/*.ts` with `--external:@aws-sdk/*` (runtime provides the SDK). Pass 2 re-bundles 6 specific handlers (`registry-provisioner`, `agent-config-resolver`, `tool-config-resolver`, `registry-sync`, `integration-resolver`, `gateway-registration-handler`) *with* the SDK bundled in. `snowflake-sdk`/`@databricks/sql` stay external in both. A new resolver needing the bundled SDK must be added to pass 2.
- **Only 5 stacks deploy.** `KnowledgeBaseStack` is commented out and `PipelineStack` is never instantiated in `bin/app.ts` — ignore them.
- **GatewayStack↔BackendStack circular dep is broken with a hardcoded ARN** (`arn:aws:lambda:{region}:{account}:function:citadel-app-publish-handler-{env}` passed to `addPublishHandlerResolvers`). Renaming the gateway publish handler breaks the AppSync data-source wiring.
- **Region is ap-southeast-2** (account `041073296931`, profile `ppc-sandpit`, deployed dev at https://d1gx54o89mc4l4.cloudfront.net). But `buildspec-deploy.yml` and `.github/workflows/deploy.yml` default to `us-east-1`, and some doc examples say `us-east-1`. Don't assume the CI default.
- **cdk-nag suppressions are centralized in `bin/app.ts`** with `AAF-NAG-*` tags and reasons — add new ones there, scoped to the exact resource path.
- **Two execution engines coexist:** the legacy SQS-driven **Supervisor** (`task.request`/`task.completion`) and the newer DAG **Step Runner** (`citadel-step-runner`, `workflow.*` EventBridge events). Know which one a change targets.
- **Agent credential isolation is security-critical.** Worker Wrapper runs user-uploaded Python agent code in an isolated subprocess (`agent_runner.py`); scoped STS creds go **only** into the child env, never `os.environ` of the parent Lambda. Don't "simplify" this — an env-var approach was abandoned for credential-leakage risk. See [docs/AGENT_PERMISSIONS.md](docs/AGENT_PERMISSIONS.md).

### Known-broken / orphaned (don't rely on these)
- `backend` npm scripts `deploy:backend` / `deploy:all` point at `backend/scripts/` which **doesn't exist**. Use root `./deploy.sh`.
- `frontend` npm script `generate-aws-exports` points at a missing `scripts/generate-aws-exports.js`.
- **No ESLint config** exists in `backend/` or `frontend/` (lint relies on `tsconfig` strict mode); no `.prettierrc` (defaults). `npm run lint` may no-op or fail.
- **CI mismatch:** `ci.yml`/`deploy.yml` still run `npm ci` for the frontend (and cache `frontend/package-lock.json`), but no lock file exists — frontend CI breaks until a lock is committed or CI switches to `npm install`. (The working-tree edit to `deploy.sh` already switched the local deploy to `npm install`.)
- `backend/.schema-validate.tmp.js` and `arbiter/temp_emailTool.py` are orphaned scratch files, not wired into anything.

## Conventions

- **Resource naming is uniformly `citadel-*-{env}`.** Stacks `citadel-{layer}-{env}`; tables `citadel-{resource}-{env}`; EventBridge bus `citadel-agents-{env}`. **Scoped IAM role prefixes are load-bearing** (CDK grants wildcard IAM on them; tests assert them): datastore `citadel-ds-{id}`, integration `citadel-int-{id}`, agent/app `citadel-agent-{id}`.
- **Unified adapter pattern:** all 27 datastores + 13 integrations implement `ConnectorAdapter` (`backend/src/adapters/base.ts`). Datastore adapters are concrete classes in `backend/src/lambda/adapters/` registered in `registry.ts`; integrations are a `ConnectorSpec` in `backend/src/utils/connector-registry.ts` handled by `BaseIntegrationAdapter`. `requiredPolicies()` must return resource-scoped ARNs for `connect` (wildcards only for `provision`).
- **Every resolver:** `sanitizeForLogging()` before any log, route by `event.info.fieldName`, org-scoped access via `getCallerOrgId()` (Cognito `custom:organization`), optimistic locking via `version` condition expression, validate inputs with `backend/src/utils/validation.ts`.
- **EventBridge:** one bus, at-least-once delivery — handlers wrap logic in `IdempotencyGuard` (`backend/src/utils/idempotency.ts`). Sources are dot-namespaced (`citadel.backend`, `citadel.workflows`, `citadel.apps`); every event detail includes `correlationId` + `timestamp`.
- **GraphQL:** enum type names PascalCase, enum values UPPER_SNAKE_CASE; operation IDs snake_case; runtime statuses lowercase. Schema: `backend/src/schema/schema.graphql`.
- **Tests colocated in `__tests__/`**; unit tests use `aws-sdk-client-mock`, transformation/policy logic uses `fast-check` / Hypothesis property tests. Path alias `@/*` → `src/*`.

## Common recipes (full steps in docs/)

- **Add a resolver** → [docs/RESOLVER_GUIDE.md](docs/RESOLVER_GUIDE.md): new `backend/src/lambda/<name>-resolver.ts` (auto-bundled), wire Lambda + `addLambdaDataSource` + `createResolver` in `backend/lib/backend-stack.ts`, extend `schema.graphql`, add colocated tests.
- **Add a datastore/integration adapter** → [docs/ADAPTER_GUIDE.md](docs/ADAPTER_GUIDE.md) (8-step checklist incl. GraphQL enum, `operations-registry.ts`, frontend `connectorRegistry.ts`, `docs/INTEGRATION_SETUP.md` section).
- **Add an EventBridge event** → [docs/EVENTBRIDGE_CATALOG.md](docs/EVENTBRIDGE_CATALOG.md): constant in `backend/src/utils/events.ts`, publish via `publishEvent()`, CDK rule if it triggers a Lambda, Fan-out + AppSync subscription if the frontend needs it.
- **Add a service-layer agent tool** → `@tool` fn in `service/agent_intake_single/tools/`, register in `agent.py`'s tool list, and describe it in the `SYSTEM_PROMPT` phase instructions.

Deeper references: [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md), [docs/POLICY_MANAGER.md](docs/POLICY_MANAGER.md), [docs/AGENT_APPS.md](docs/AGENT_APPS.md), [docs/BLUEPRINTS_WORKFLOWS.md](docs/BLUEPRINTS_WORKFLOWS.md), [docs/DATASTORES_INTEGRATIONS.md](docs/DATASTORES_INTEGRATIONS.md).
