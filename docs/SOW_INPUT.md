# Citadel — Statement of Work Input

Points for SOW generation. We (the AWS Partner) deploy this platform into the customer's AWS account.

---

## 1. Project Overview

**Platform:** Citadel — a multi-agent AI platform for enterprise application development.

**Purpose:** Enables organisations to build, configure, publish, and operate multi-agent AI applications that orchestrate work across datastores, integrations, and LLMs using AWS Bedrock AgentCore.

**Core value prop:**
- Visual agent app builder with drag-and-drop workflow canvas
- 27 datastore adapters + 13 integration connectors (SaaS, AWS-native, MCP)
- Automated credential scoping — each agent gets least-privilege IAM via subprocess isolation
- Per-app API Gateway publishing with key-based auth for consuming systems
- Real-time execution monitoring via AppSync WebSocket subscriptions

---

## 2. Architecture Summary

Four-layer monorepo deployed as 5 AWS CDK stacks:

| Layer | Stack | Purpose | Language |
|-------|-------|---------|----------|
| Backend | `citadel-backend-{env}` | AppSync GraphQL API, Cognito, DynamoDB (13 tables), EventBridge, 50+ Lambda resolvers | CDK TypeScript (Node 24) |
| Services | `citadel-services-{env}` | Bedrock AgentCore Runtime, OpenSearch Knowledge Base, AgentCore Gateway (MCP) | Python 3.13 |
| Arbiter | `citadel-arbiter-{env}` | Multi-agent orchestration: Supervisor, Fabricator, Worker Wrapper, Step Runner | Python 3.14 |
| Gateway | `citadel-gateway-{env}` | Per-app API Gateway HTTP APIs, Lambda authorizer, usage metrics | TypeScript |
| Frontend | `citadel-frontend-{env}` | React 18 + Vite SPA on S3/CloudFront | TypeScript |

Two additional stacks exist but are not deployed in standard delivery:
- `citadel-knowledge-base-{env}` — standalone Bedrock Knowledge Base (commented out)
- `citadel-pipeline-{env}` — CI/CD CodePipeline (self-mutating, multi-env)

---

## 3. AWS Services Consumed

### Compute
- AWS Lambda (50+ functions — Node.js 24 and Python 3.14)
- Lambda@Edge (frontend routing)

### AI/ML
- Amazon Bedrock (Claude Sonnet 4 via Converse API)
- Amazon Bedrock AgentCore Runtime (4 agents: Assessment, Design, Planning, Implementation)
- Amazon Bedrock AgentCore Gateway (MCP protocol)
- Amazon Bedrock Knowledge Base (OpenSearch Serverless backing)

### API & Networking
- AWS AppSync (GraphQL + real-time WebSocket subscriptions)
- Amazon API Gateway HTTP APIs (per-app published endpoints)
- Amazon CloudFront (SPA CDN)

### Storage & Data
- Amazon DynamoDB (13 tables, on-demand billing, single-table design with GSIs)
- Amazon S3 (document storage, code artifacts, frontend hosting)
- Amazon OpenSearch Serverless (knowledge base vector store)

### Security & Identity
- Amazon Cognito User Pools (4 RBAC groups: admin, project_manager, architect, developer)
- AWS IAM (scoped roles per agent/datastore/integration: `citadel-agent-*`, `citadel-ds-*`, `citadel-int-*`)
- AWS STS (temporary credential vending with 1hr sessions)
- AWS Secrets Manager (integration credentials, admin password)
- AWS SSM Parameter Store (integration configuration)

### Integration & Orchestration
- Amazon EventBridge (single bus `citadel-agents-{env}`, at-least-once delivery)
- Amazon SQS (worker queue, fabricator queue)

### Deployment & Operations
- AWS CDK 2.100+ (all IaC)
- AWS CloudFormation (stack lifecycle)
- Amazon ECR (container images for agents)
- AWS CloudWatch (structured logging, metrics)

---

## 4. Scope of Work — Deliverables

### Phase 1: Environment Setup & Foundation
- AWS account configuration and SSO profile setup
- CDK bootstrap in target region
- Container runtime provisioning (Finch — Docker Desktop licensing avoided)
- Environment variable configuration (`.env` file with account, region, admin email)
- Network and security group validation
- Bedrock model access enablement (Claude Sonnet 4)

### Phase 2: Platform Deployment
- Full 5-stack CDK deployment via `./deploy.sh`
- Stack deployment order: Backend → Services → Arbiter → Gateway → Frontend
- Admin user seeding (auto-generated password via Secrets Manager)
- Default organisation provisioning (Default, Engineering, Product, Operations)
- Cognito group creation (admin, project_manager, architect, developer)
- CloudFront distribution provisioning
- AgentCore Gateway configuration and verification

### Phase 3: Integration & Connector Configuration
- SaaS connector setup (Confluence, Jira, ServiceNow, Slack, Microsoft, PagerDuty, Zendesk)
- AWS-native connector validation (Lambda, Smithy/DynamoDB/S3/SQS/SNS, MCP Server)
- Datastore adapter provisioning (customer-specific subset of 27 supported types)
- Health monitoring validation (15-minute EventBridge schedule)
- Credential vending verification (scoped IAM roles created/assumed correctly)

### Phase 4: Agent Platform Validation
- Agent App creation end-to-end test (DRAFT → ACTIVE → PUBLISHED lifecycle)
- Workflow Builder validation (DAG execution, parallel branches, convergence nodes)
- Fabricator verification (dynamic agent/tool creation)
- Per-app API Gateway publishing (key-based auth, EventBridge routing)
- Supervisor orchestration test (Bedrock Converse API with circuit breaker)
- Worker subprocess isolation verification (scoped credentials in child env only)
- Intent-based routing validation (multi-workflow apps)

### Phase 5: Operational Readiness
- CloudWatch dashboard and alarm configuration
- Structured logging validation (all components emit JSON with entity IDs)
- EventBridge dead-letter queue configuration
- Idempotency verification (IdempotencyGuard across all event handlers)
- Backup and disaster recovery documentation
- Runbook creation (credential rotation, stuck execution recovery, IAM role cleanup)
- Cost alerting and budget configuration

### Phase 6: Knowledge Transfer & Handover
- Platform walkthrough with customer technical team
- Architecture documentation review
- Operational procedures handover
- Deployment automation walkthrough (deploy.sh, CI/CD pipeline if activated)
- Security model briefing (credential isolation, RBAC, org-scoped access)

---

## 5. Key Technical Details for Estimating

### Deployment Complexity
- First deploy: ~15-20 minutes (automated)
- CDK bootstrap required per account/region
- Container runtime required for Python/container Lambda bundling
- Two-pass esbuild for Lambda handlers (SDK externalization vs bundling)
- 5 stacks with cross-stack dependencies (outputs/imports)
- GatewayStack↔BackendStack circular dependency broken with hardcoded ARN

### Infrastructure Scale (dev environment)
- 50+ Lambda functions
- 13 DynamoDB tables with 4+ GSIs
- 2 SQS queues
- 1 EventBridge bus with multiple rules
- 1 AppSync API with 40+ resolvers
- 1 Cognito User Pool
- 2+ S3 buckets
- 1 CloudFront distribution
- 1 OpenSearch Serverless collection
- Per-app API Gateway instances (variable)
- Per-agent/datastore/integration IAM roles (variable)

### Security-Critical Components
- Agent credential isolation via subprocess (NOT env vars — previous approach abandoned for security)
- Scoped IAM role lifecycle (create on provision, delete on archive)
- Org-scoped data access on every resolver operation
- API key validation on published app endpoints
- Secrets Manager for all third-party credentials
- No wildcard IAM actions permitted (validation enforced)

### Multi-Environment Support
- Environment suffix on all resources: `citadel-*-{env}`
- Driven by `ENVIRONMENT` variable in `.env` (dev/staging/prod)
- Same CDK app supports parallel environments in same account
- CI/CD pipeline stack available but optional

---

## 6. Estimated Monthly Cost (Dev Environment)

| Service | Estimate |
|---------|----------|
| Lambda | $50–100 |
| DynamoDB (on-demand) | $25–50 |
| S3 | $5–10 |
| CloudFront | $10–20 |
| AppSync | $20–40 |
| Bedrock AgentCore | $100–200 |
| OpenSearch Serverless | $50–100 |
| **Total** | **$260–520/mo** |

Production with higher volume would scale linearly on Lambda/DynamoDB/Bedrock; OpenSearch Serverless has a floor cost.

---

## 7. Prerequisites & Assumptions

### Customer Provides
- AWS account with admin-level IAM access
- AWS SSO or IAM credentials for deployment
- Bedrock model access approved (Claude Sonnet 4 in target region)
- Target region confirmed (default: ap-southeast-2)
- Admin email address for initial user
- Network connectivity requirements for SaaS integrations
- List of required datastores and integrations for Phase 3

### Partner Assumes
- Customer account is not under AWS Organizations restrictive SCPs that block required services
- Bedrock AgentCore is available in the target region
- No existing resource naming conflicts with `citadel-*` prefix
- Customer will change auto-generated admin password on first login
- Node.js 24+ and Python 3.14+ available in build environment

---

## 8. Security & Compliance Points

- All data encrypted at rest (DynamoDB, S3, Secrets Manager default encryption)
- All data encrypted in transit (TLS everywhere)
- Cognito-based authentication with RBAC groups
- Organisation-scoped data isolation (multi-tenant within single deployment)
- Least-privilege credential vending (per-agent, per-datastore, per-integration, per-app)
- Subprocess isolation for user-uploaded agent code (no ambient credential access)
- API key authentication for published app endpoints with expiry/rotation support
- Structured audit logging on all operations
- No secrets in environment variables or source code
- cdk-nag AwsSolutions pack runs on every synth (compliance enforcement)

---

## 9. Exclusions / Out of Scope (default — adjust per engagement)

- Custom SaaS integration development beyond the 13 supported types
- Custom datastore adapter development beyond the 27 supported types
- Application logic / agent code development (customer responsibility post-handover)
- DNS / custom domain configuration (CloudFront supports it but not included by default)
- WAF / DDoS protection configuration
- Multi-region / DR deployment
- Load testing at production scale
- SOC2 / ISO27001 / HIPAA compliance gap analysis
- Ongoing managed services / SLA-backed support

---

## 10. Success Criteria

1. All 5 CDK stacks deployed successfully with no CloudFormation errors
2. Admin user can log in via CloudFront URL and access all platform pages
3. At least one agent app created, published, and accessible via API Gateway endpoint
4. At least one datastore and one integration connected with health check passing
5. Workflow execution completes end-to-end with real-time progress visible in frontend
6. Credential isolation verified (agent subprocess has scoped creds, parent Lambda unaffected)
7. EventBridge events flowing correctly (idempotency guard preventing duplicates)
8. CloudWatch logs emitting structured JSON from all components
9. Customer technical team can independently deploy changes via `./deploy.sh`
10. Operational runbook delivered and walkthrough completed

---

## 11. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Bedrock AgentCore not available in target region | Blocks Services stack | Confirm regional availability before engagement start |
| Restrictive SCPs blocking service creation | Blocks deployment | Pre-flight SCP audit in Phase 1 |
| Container runtime issues (Finch VM) | Blocks CDK bundling | Fallback to Docker; test build env early |
| OpenSearch Serverless cold-start latency | Slow first queries | Pre-warm during Phase 4 validation |
| IAM role limits (default 1000/account) | Blocks at scale | Request limit increase if >200 agents expected |
| Bedrock throttling under load | Circuit breaker opens | Configure appropriate concurrency limits |
| Frontend CI expects package-lock.json (not committed) | CI fails | Use `npm install` not `npm ci` in pipelines |
| `us-east-1` defaults in CI configs vs actual `ap-southeast-2` deployment | Wrong region | Override in all CI/deploy configs |

---

## 12. Reference Architecture Diagram

```
Users → CloudFront → S3 (React SPA)
         ↕ GraphQL + WebSocket
       AppSync → Lambda Resolvers → DynamoDB (13 tables)
         ↕ EventBridge
       Supervisor → Worker Wrapper → [Subprocess: Agent Code + Scoped Creds]
         ↕ SQS                         ↕ Bedrock (Claude)
       Fabricator → S3 (code)          ↕ Datastores/Integrations (scoped IAM)
         ↕
       Step Runner → DAG Execution → Real-time Subscriptions
         
Published Apps:
  External Client → API Gateway → Lambda Authorizer → EventBridge → Supervisor
```

---

## 13. Technology Versions

| Technology | Version |
|------------|---------|
| Node.js | 24+ |
| Python | 3.14+ (arbiter), 3.13 (service) |
| AWS CDK | 2.100+ |
| React | 18 |
| Vite | Latest |
| TypeScript | 5.x |
| Container runtime | Finch (recommended) or Docker |
| Bedrock model | Claude Sonnet 4 (anthropic.claude-sonnet-4-20250514) |
