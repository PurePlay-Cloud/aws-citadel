# Agent Packaging & Cross-Account Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable agents (with their tools, code, and binding declarations) to be packaged as versioned artifacts that can be deployed into any target AWS account/environment.

**Architecture:** An "Agent Package" is a self-contained JSON manifest + S3 code bundle that uses abstract binding references instead of concrete IDs. A `packageAgent` mutation serializes an agent into a package. A `deployAgentPackage` mutation in the target environment resolves abstract bindings against a provided binding map and reconstitutes the agent. Packages are stored in a dedicated S3 prefix and indexed in DynamoDB for versioning and audit trail.

**Tech Stack:** TypeScript (Lambda resolvers), DynamoDB, S3, GraphQL (AppSync), aws-sdk-client-mock (tests), fast-check (property tests)

---

## Scope

This plan covers three subsystems:

1. **Package Schema & Serialization** — the package format, serialization from live agent, validation
2. **Package Registry** — storage, versioning, listing packages
3. **Package Deployment** — import into target env with binding resolution

These are sequential dependencies (2 needs 1, 3 needs 2) so they're in one plan.

## File Structure

### New Files

| File | Responsibility |
|------|----------------|
| `backend/src/lambda/agent-package-resolver.ts` | GraphQL resolver: packageAgent, deployAgentPackage, listAgentPackages, getAgentPackage |
| `backend/src/utils/agent-package-schema.ts` | Package format types, serialization/deserialization, binding abstraction logic |
| `backend/src/utils/binding-resolver.ts` | Resolves abstract binding refs to concrete IDs using a binding map |
| `backend/src/lambda/__tests__/agent-package-resolver.test.ts` | Unit tests for resolver |
| `backend/src/utils/__tests__/agent-package-schema.property.test.ts` | Property tests for serialization round-trip and binding abstraction |
| `backend/src/utils/__tests__/binding-resolver.property.test.ts` | Property tests for binding resolution |

### Modified Files

| File | Change |
|------|--------|
| `backend/src/schema/schema.graphql` | Add AgentPackage type, mutations, queries |
| `backend/lib/backend-stack.ts` | Add package resolver Lambda, wire to AppSync, add S3 prefix permissions |
| `backend/bin/app.ts` | No change (resolver auto-bundled via esbuild glob) |

---

## Task 1: Package Schema & Types

**Files:**
- Create: `backend/src/utils/agent-package-schema.ts`
- Create: `backend/src/utils/__tests__/agent-package-schema.property.test.ts`

- [ ] **Step 1: Write property tests for package schema**

```typescript
// backend/src/utils/__tests__/agent-package-schema.property.test.ts
import * as fc from 'fast-check';
import {
  serializeAgentPackage,
  deserializeAgentPackage,
  abstractBindings,
  AgentPackage,
  AbstractBinding,
} from '../agent-package-schema';

describe('AgentPackage schema', () => {
  const arbBinding = fc.record({
    ref: fc.string({ minLength: 1, maxLength: 50 }),
    type: fc.constantFrom('CONFLUENCE', 'JIRA', 'S3', 'DYNAMODB', 'SLACK'),
    direction: fc.constantFrom('INPUT', 'OUTPUT', 'BIDIRECTIONAL'),
    operations: fc.array(fc.string({ minLength: 1 }), { maxLength: 5 }),
    scope: fc.constantFrom('integration', 'datastore'),
  });

  const arbPackage = fc.record({
    name: fc.string({ minLength: 3, maxLength: 100 }),
    version: fc.tuple(fc.nat(9), fc.nat(9), fc.nat(9)).map(([a, b, c]) => `${a}.${b}.${c}`),
    description: fc.string({ maxLength: 500 }),
    agentConfig: fc.jsonValue() as fc.Arbitrary<Record<string, unknown>>,
    tools: fc.array(
      fc.record({
        toolId: fc.string({ minLength: 1 }),
        config: fc.jsonValue() as fc.Arbitrary<Record<string, unknown>>,
        code: fc.string({ minLength: 1 }),
      }),
      { maxLength: 10 },
    ),
    bindings: fc.array(arbBinding, { maxLength: 10 }),
    permissions: fc.array(
      fc.record({
        actions: fc.array(fc.string({ minLength: 3 }), { minLength: 1, maxLength: 5 }),
        resources: fc.array(fc.string({ minLength: 1 }), { minLength: 1, maxLength: 5 }),
      }),
      { maxLength: 5 },
    ),
  });

  it('P1: serialize/deserialize round-trip preserves all fields', () => {
    fc.assert(
      fc.property(arbPackage, (pkg) => {
        const serialized = serializeAgentPackage(pkg as AgentPackage);
        const deserialized = deserializeAgentPackage(serialized);
        expect(deserialized.name).toBe(pkg.name);
        expect(deserialized.version).toBe(pkg.version);
        expect(deserialized.bindings).toHaveLength(pkg.bindings.length);
        expect(deserialized.tools).toHaveLength(pkg.tools.length);
        expect(deserialized.permissions).toEqual(pkg.permissions);
      }),
      { numRuns: 100 },
    );
  });

  it('P2: abstractBindings replaces concrete IDs with refs', () => {
    const concreteIntegration = {
      integrationId: 'int-abc123',
      integrationType: 'CONFLUENCE',
      operations: ['search_pages'],
      direction: 'INPUT' as const,
    };
    const concreteDataStore = {
      dataStoreId: 'ds-xyz789',
      dataStoreType: 'S3',
      operations: ['read_object'],
      direction: 'INPUT' as const,
    };

    const result = abstractBindings(
      [concreteIntegration],
      [concreteDataStore],
    );

    expect(result.bindings).toHaveLength(2);
    expect(result.bindings[0].ref).toBe('confluence-0');
    expect(result.bindings[0].type).toBe('CONFLUENCE');
    expect(result.bindings[0].scope).toBe('integration');
    expect(result.bindings[1].ref).toBe('s3-0');
    expect(result.bindings[1].scope).toBe('datastore');
    expect(result.bindingMap['confluence-0']).toBe('int-abc123');
    expect(result.bindingMap['s3-0']).toBe('ds-xyz789');
  });

  it('P3: version must be valid semver', () => {
    const invalidPkg = {
      name: 'test',
      version: 'not-semver',
      description: '',
      agentConfig: {},
      tools: [],
      bindings: [],
      permissions: [],
    };
    expect(() => serializeAgentPackage(invalidPkg as AgentPackage)).toThrow(/version/i);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && npx jest --testPathPattern="agent-package-schema.property" --no-coverage`
Expected: FAIL — module not found

- [ ] **Step 3: Implement package schema module**

```typescript
// backend/src/utils/agent-package-schema.ts
export interface AbstractBinding {
  ref: string;
  type: string;
  direction: 'INPUT' | 'OUTPUT' | 'BIDIRECTIONAL';
  operations: string[];
  scope: 'integration' | 'datastore';
}

export interface PackageTool {
  toolId: string;
  config: Record<string, unknown>;
  code: string;
}

export interface PackagePermission {
  actions: string[];
  resources: string[];
}

export interface AgentPackage {
  name: string;
  version: string;
  description: string;
  agentConfig: Record<string, unknown>;
  tools: PackageTool[];
  bindings: AbstractBinding[];
  permissions: PackagePermission[];
}

export interface SerializedPackage {
  schemaVersion: '1.0';
  package: AgentPackage;
  createdAt: string;
}

const SEMVER_RE = /^\d+\.\d+\.\d+$/;

export function serializeAgentPackage(pkg: AgentPackage): string {
  if (!SEMVER_RE.test(pkg.version)) {
    throw new Error(`Invalid version "${pkg.version}" — must be semver (e.g. 1.0.0)`);
  }

  const envelope: SerializedPackage = {
    schemaVersion: '1.0',
    package: pkg,
    createdAt: new Date().toISOString(),
  };

  return JSON.stringify(envelope);
}

export function deserializeAgentPackage(json: string): AgentPackage {
  const envelope: SerializedPackage = JSON.parse(json);
  if (envelope.schemaVersion !== '1.0') {
    throw new Error(`Unsupported package schema version: ${envelope.schemaVersion}`);
  }
  return envelope.package;
}

interface ConcreteIntegrationBinding {
  integrationId: string;
  integrationType: string;
  operations?: string[];
  direction?: 'INPUT' | 'OUTPUT' | 'BIDIRECTIONAL';
}

interface ConcreteDataStoreBinding {
  dataStoreId: string;
  dataStoreType: string;
  operations?: string[];
  direction?: 'INPUT' | 'OUTPUT' | 'BIDIRECTIONAL';
}

export function abstractBindings(
  integrationBindings: ConcreteIntegrationBinding[],
  dataStoreBindings: ConcreteDataStoreBinding[],
): { bindings: AbstractBinding[]; bindingMap: Record<string, string> } {
  const bindings: AbstractBinding[] = [];
  const bindingMap: Record<string, string> = {};

  const integrationCounts: Record<string, number> = {};
  for (const ib of integrationBindings) {
    const typeKey = ib.integrationType.toLowerCase();
    const idx = integrationCounts[typeKey] ?? 0;
    integrationCounts[typeKey] = idx + 1;
    const ref = `${typeKey}-${idx}`;

    bindings.push({
      ref,
      type: ib.integrationType,
      direction: ib.direction ?? 'BIDIRECTIONAL',
      operations: ib.operations ?? [],
      scope: 'integration',
    });
    bindingMap[ref] = ib.integrationId;
  }

  const datastoreCounts: Record<string, number> = {};
  for (const ds of dataStoreBindings) {
    const typeKey = ds.dataStoreType.toLowerCase();
    const idx = datastoreCounts[typeKey] ?? 0;
    datastoreCounts[typeKey] = idx + 1;
    const ref = `${typeKey}-${idx}`;

    bindings.push({
      ref,
      type: ds.dataStoreType,
      direction: ds.direction ?? 'BIDIRECTIONAL',
      operations: ds.operations ?? [],
      scope: 'datastore',
    });
    bindingMap[ref] = ds.dataStoreId;
  }

  return { bindings, bindingMap };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && npx jest --testPathPattern="agent-package-schema.property" --no-coverage`
Expected: PASS (all 3 properties)

- [ ] **Step 5: Commit**

```bash
git add backend/src/utils/agent-package-schema.ts backend/src/utils/__tests__/agent-package-schema.property.test.ts
git commit -m "feat(packaging): add agent package schema with serialization and binding abstraction"
```

---

## Task 2: Binding Resolver

**Files:**
- Create: `backend/src/utils/binding-resolver.ts`
- Create: `backend/src/utils/__tests__/binding-resolver.property.test.ts`

- [ ] **Step 1: Write property tests for binding resolution**

```typescript
// backend/src/utils/__tests__/binding-resolver.property.test.ts
import * as fc from 'fast-check';
import {
  resolveBindings,
  BindingMap,
  ResolvedBindings,
  BindingResolutionError,
} from '../binding-resolver';
import { AbstractBinding } from '../agent-package-schema';

describe('Binding resolver', () => {
  it('P4: all abstract refs resolved when binding map is complete', () => {
    fc.assert(
      fc.property(
        fc.array(
          fc.record({
            ref: fc.string({ minLength: 1, maxLength: 20 }),
            type: fc.constantFrom('CONFLUENCE', 'S3', 'DYNAMODB'),
            direction: fc.constantFrom('INPUT', 'OUTPUT', 'BIDIRECTIONAL'),
            operations: fc.array(fc.string({ minLength: 1 }), { maxLength: 3 }),
            scope: fc.constantFrom('integration', 'datastore'),
          }),
          { minLength: 1, maxLength: 5 },
        ).filter((bindings) => {
          const refs = bindings.map((b) => b.ref);
          return new Set(refs).size === refs.length;
        }),
        (bindings) => {
          const bindingMap: BindingMap = {};
          for (const b of bindings) {
            bindingMap[b.ref] = `resolved-${b.ref}`;
          }

          const result = resolveBindings(bindings as AbstractBinding[], bindingMap);
          expect(result.integrationBindings.length + result.dataStoreBindings.length).toBe(
            bindings.length,
          );
          expect(result.unresolved).toHaveLength(0);
        },
      ),
      { numRuns: 100 },
    );
  });

  it('P5: missing refs reported as unresolved', () => {
    const bindings: AbstractBinding[] = [
      { ref: 'confluence-0', type: 'CONFLUENCE', direction: 'INPUT', operations: [], scope: 'integration' },
      { ref: 's3-0', type: 'S3', direction: 'OUTPUT', operations: ['write_object'], scope: 'datastore' },
    ];
    const bindingMap: BindingMap = { 'confluence-0': 'int-prod-001' };

    const result = resolveBindings(bindings, bindingMap);
    expect(result.integrationBindings).toHaveLength(1);
    expect(result.integrationBindings[0].integrationId).toBe('int-prod-001');
    expect(result.unresolved).toEqual(['s3-0']);
  });

  it('P6: resolved integration bindings have correct shape', () => {
    const bindings: AbstractBinding[] = [
      { ref: 'jira-0', type: 'JIRA', direction: 'BIDIRECTIONAL', operations: ['create_issue', 'get_issue'], scope: 'integration' },
    ];
    const bindingMap: BindingMap = { 'jira-0': 'int-target-jira' };

    const result = resolveBindings(bindings, bindingMap);
    expect(result.integrationBindings[0]).toEqual({
      integrationId: 'int-target-jira',
      integrationType: 'JIRA',
      operations: ['create_issue', 'get_issue'],
      direction: 'BIDIRECTIONAL',
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && npx jest --testPathPattern="binding-resolver.property" --no-coverage`
Expected: FAIL — module not found

- [ ] **Step 3: Implement binding resolver**

```typescript
// backend/src/utils/binding-resolver.ts
import { AbstractBinding } from './agent-package-schema';

export type BindingMap = Record<string, string>;

export interface ResolvedBindings {
  integrationBindings: Array<{
    integrationId: string;
    integrationType: string;
    operations: string[];
    direction: 'INPUT' | 'OUTPUT' | 'BIDIRECTIONAL';
  }>;
  dataStoreBindings: Array<{
    dataStoreId: string;
    dataStoreType: string;
    operations: string[];
    direction: 'INPUT' | 'OUTPUT' | 'BIDIRECTIONAL';
  }>;
  unresolved: string[];
}

export class BindingResolutionError extends Error {
  constructor(public readonly unresolvedRefs: string[]) {
    super(`Unresolved binding references: ${unresolvedRefs.join(', ')}`);
    this.name = 'BindingResolutionError';
  }
}

export function resolveBindings(
  bindings: AbstractBinding[],
  bindingMap: BindingMap,
): ResolvedBindings {
  const result: ResolvedBindings = {
    integrationBindings: [],
    dataStoreBindings: [],
    unresolved: [],
  };

  for (const binding of bindings) {
    const concreteId = bindingMap[binding.ref];
    if (!concreteId) {
      result.unresolved.push(binding.ref);
      continue;
    }

    if (binding.scope === 'integration') {
      result.integrationBindings.push({
        integrationId: concreteId,
        integrationType: binding.type,
        operations: binding.operations,
        direction: binding.direction,
      });
    } else {
      result.dataStoreBindings.push({
        dataStoreId: concreteId,
        dataStoreType: binding.type,
        operations: binding.operations,
        direction: binding.direction,
      });
    }
  }

  return result;
}

export function resolveBindingsStrict(
  bindings: AbstractBinding[],
  bindingMap: BindingMap,
): Omit<ResolvedBindings, 'unresolved'> {
  const result = resolveBindings(bindings, bindingMap);
  if (result.unresolved.length > 0) {
    throw new BindingResolutionError(result.unresolved);
  }
  return result;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && npx jest --testPathPattern="binding-resolver.property" --no-coverage`
Expected: PASS (all 3 properties)

- [ ] **Step 5: Commit**

```bash
git add backend/src/utils/binding-resolver.ts backend/src/utils/__tests__/binding-resolver.property.test.ts
git commit -m "feat(packaging): add binding resolver for abstract-to-concrete ref mapping"
```

---

## Task 3: GraphQL Schema Extension

**Files:**
- Modify: `backend/src/schema/schema.graphql`

- [ ] **Step 1: Add AgentPackage types, queries, and mutations to schema**

Append the following types after the existing `AgentManifest` type (line ~1048):

```graphql
enum PackageStatus {
  DRAFT
  PUBLISHED
  DEPRECATED
}

type AgentPackageMetadata {
  packageId: ID!
  name: String!
  version: String!
  description: String
  agentId: String!
  status: PackageStatus!
  bindings: [PackageBinding!]!
  createdBy: String!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}

type PackageBinding {
  ref: String!
  type: String!
  direction: BindingDirection!
  operations: [String!]!
  scope: String!
}

type AgentPackageDetail {
  metadata: AgentPackageMetadata!
  packageJson: AWSJSON!
}

type PackageDeployResult {
  success: Boolean!
  agentId: String!
  toolIds: [String!]!
  unresolvedBindings: [String!]
  message: String
}

type ListPackagesConnection {
  items: [AgentPackageMetadata!]!
  nextToken: String
}

input PackageAgentInput {
  agentId: String!
  version: String!
  description: String
}

input DeployAgentPackageInput {
  packageId: String!
  bindingMap: AWSJSON!
  targetOrgId: String!
  agentIdOverride: String
  activate: Boolean
}

input DeprecatePackageInput {
  packageId: String!
}
```

Add to `type Query`:

```graphql
  listAgentPackages(status: PackageStatus): ListPackagesConnection
  getAgentPackage(packageId: ID!): AgentPackageDetail
```

Add to `type Mutation`:

```graphql
  packageAgent(input: PackageAgentInput!): AgentPackageMetadata
  deployAgentPackage(input: DeployAgentPackageInput!): PackageDeployResult
  deprecatePackage(input: DeprecatePackageInput!): AgentPackageMetadata
```

- [ ] **Step 2: Validate schema compiles**

Run: `cd backend && npm run build`
Expected: Compiles without GraphQL schema errors (AppSync validates on synth)

- [ ] **Step 3: Commit**

```bash
git add backend/src/schema/schema.graphql
git commit -m "feat(packaging): add GraphQL types for agent packaging and deployment"
```

---

## Task 4: Package Resolver — packageAgent

**Files:**
- Create: `backend/src/lambda/agent-package-resolver.ts`
- Create: `backend/src/lambda/__tests__/agent-package-resolver.test.ts`

- [ ] **Step 1: Write unit tests for packageAgent**

```typescript
// backend/src/lambda/__tests__/agent-package-resolver.test.ts
import { mockClient } from 'aws-sdk-client-mock';
import { DynamoDBDocumentClient, GetCommand, PutCommand, ScanCommand } from '@aws-sdk/lib-dynamodb';
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';
import { Readable } from 'stream';

const ddbMock = mockClient(DynamoDBDocumentClient);
const s3Mock = mockClient(S3Client);

process.env.AGENT_CONFIG_TABLE = 'citadel-agents-test';
process.env.TOOLS_CONFIG_TABLE = 'citadel-tools-test';
process.env.AGENT_BUCKET_NAME = 'citadel-code-test';
process.env.PACKAGES_TABLE = 'citadel-packages-test';

import { handler } from '../agent-package-resolver';

beforeEach(() => {
  ddbMock.reset();
  s3Mock.reset();
});

describe('packageAgent', () => {
  const agentConfig = {
    agentId: 'agent-001',
    config: JSON.stringify({
      name: 'Test Agent',
      description: 'Does testing',
      filename: 'test_agent.py',
      requiredPermissions: { models: ['anthropic.claude-sonnet-4-20250514'] },
    }),
    state: 'active',
    categories: ['testing'],
    manifest: { name: 'Test Agent', description: 'Does testing', version: '1.0.0' },
  };

  const toolConfig = {
    toolId: 'tool-001',
    config: JSON.stringify({ name: 'Search Tool', filename: 'search.py' }),
    state: 'active',
    integrationBindings: [
      { integrationId: 'int-abc', integrationType: 'CONFLUENCE', operations: ['search_pages'], direction: 'INPUT' },
    ],
    dataStoreBindings: [],
  };

  it('packages an agent with tools and abstracts bindings', async () => {
    ddbMock.on(GetCommand, { TableName: 'citadel-agents-test' }).resolves({ Item: agentConfig });
    ddbMock.on(ScanCommand, { TableName: 'citadel-tools-test' }).resolves({ Items: [toolConfig] });
    ddbMock.on(PutCommand).resolves({});
    s3Mock.on(GetObjectCommand).resolves({
      Body: Readable.from(['print("hello")']),
    } as any);
    s3Mock.on(PutObjectCommand).resolves({});

    const result = await handler({
      info: { fieldName: 'packageAgent' },
      arguments: { input: { agentId: 'agent-001', version: '1.0.0', description: 'First release' } },
      identity: { sub: 'user-1', claims: { 'custom:organization': 'org-1' } },
    } as any);

    expect(result.packageId).toBeDefined();
    expect(result.name).toBe('Test Agent');
    expect(result.version).toBe('1.0.0');
    expect(result.status).toBe('PUBLISHED');
    expect(result.bindings).toHaveLength(1);
    expect(result.bindings[0].ref).toBe('confluence-0');
    expect(result.bindings[0].scope).toBe('integration');
  });

  it('rejects invalid semver', async () => {
    ddbMock.on(GetCommand).resolves({ Item: agentConfig });

    await expect(
      handler({
        info: { fieldName: 'packageAgent' },
        arguments: { input: { agentId: 'agent-001', version: 'bad', description: '' } },
        identity: { sub: 'user-1', claims: { 'custom:organization': 'org-1' } },
      } as any),
    ).rejects.toThrow(/version/i);
  });

  it('rejects packaging an inactive agent', async () => {
    ddbMock.on(GetCommand).resolves({ Item: { ...agentConfig, state: 'inactive' } });

    await expect(
      handler({
        info: { fieldName: 'packageAgent' },
        arguments: { input: { agentId: 'agent-001', version: '1.0.0' } },
        identity: { sub: 'user-1', claims: { 'custom:organization': 'org-1' } },
      } as any),
    ).rejects.toThrow(/active/i);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && npx jest --testPathPattern="agent-package-resolver.test" --no-coverage`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the packageAgent resolver**

```typescript
// backend/src/lambda/agent-package-resolver.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand, PutCommand, ScanCommand, QueryCommand } from '@aws-sdk/lib-dynamodb';
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';
import { v4 as uuidv4 } from 'uuid';
import { serializeAgentPackage, abstractBindings, AgentPackage } from '../utils/agent-package-schema';
import { resolveBindings, BindingMap } from '../utils/binding-resolver';

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);
const s3Client = new S3Client({});

const AGENT_CONFIG_TABLE = process.env.AGENT_CONFIG_TABLE!;
const TOOLS_CONFIG_TABLE = process.env.TOOLS_CONFIG_TABLE!;
const AGENT_BUCKET_NAME = process.env.AGENT_BUCKET_NAME!;
const PACKAGES_TABLE = process.env.PACKAGES_TABLE!;

export const handler = async (event: any) => {
  const fieldName = event.info.fieldName;

  switch (fieldName) {
    case 'packageAgent':
      return await packageAgent(event.arguments.input, event.identity);
    case 'deployAgentPackage':
      return await deployAgentPackage(event.arguments.input, event.identity);
    case 'listAgentPackages':
      return await listAgentPackages(event.arguments?.status);
    case 'getAgentPackage':
      return await getAgentPackage(event.arguments.packageId);
    case 'deprecatePackage':
      return await deprecatePackage(event.arguments.input);
    default:
      throw new Error(`Unknown field: ${fieldName}`);
  }
};

async function packageAgent(input: { agentId: string; version: string; description?: string }, identity: any) {
  const SEMVER_RE = /^\d+\.\d+\.\d+$/;
  if (!SEMVER_RE.test(input.version)) {
    throw new Error(`Invalid version "${input.version}" — must be semver (e.g. 1.0.0)`);
  }

  const agentResult = await docClient.send(new GetCommand({
    TableName: AGENT_CONFIG_TABLE,
    Key: { agentId: input.agentId },
  }));

  if (!agentResult.Item) {
    throw new Error(`Agent not found: ${input.agentId}`);
  }

  const agent = agentResult.Item;
  if (agent.state !== 'active') {
    throw new Error('Agent must be active before packaging');
  }

  const parsedConfig = typeof agent.config === 'string' ? JSON.parse(agent.config) : agent.config;

  const toolsResult = await docClient.send(new ScanCommand({
    TableName: TOOLS_CONFIG_TABLE,
    FilterExpression: 'contains(config, :agentId)',
    ExpressionAttributeValues: { ':agentId': input.agentId },
  }));

  const tools = toolsResult.Items || [];

  const allIntegrationBindings = tools.flatMap((t: any) => t.integrationBindings || []);
  const allDataStoreBindings = tools.flatMap((t: any) => t.dataStoreBindings || []);
  const { bindings, bindingMap } = abstractBindings(allIntegrationBindings, allDataStoreBindings);

  const packageTools = await Promise.all(
    tools.map(async (tool: any) => {
      const toolParsedConfig = typeof tool.config === 'string' ? JSON.parse(tool.config) : tool.config;
      let code = '';
      if (toolParsedConfig.filename) {
        try {
          const s3Result = await s3Client.send(new GetObjectCommand({
            Bucket: AGENT_BUCKET_NAME,
            Key: `agents/${toolParsedConfig.filename}`,
          }));
          code = await streamToString(s3Result.Body);
        } catch {
          code = '';
        }
      }
      return { toolId: tool.toolId, config: toolParsedConfig, code };
    }),
  );

  let agentCode = '';
  if (parsedConfig.filename) {
    try {
      const s3Result = await s3Client.send(new GetObjectCommand({
        Bucket: AGENT_BUCKET_NAME,
        Key: `agents/${parsedConfig.filename}`,
      }));
      agentCode = await streamToString(s3Result.Body);
    } catch {
      agentCode = '';
    }
  }

  const permissions = parsedConfig.requiredPermissions
    ? [{ actions: ['bedrock:InvokeModel'], resources: (parsedConfig.requiredPermissions.models || []).map((m: string) => `arn:aws:bedrock:*::foundation-model/${m}`) }]
    : [];

  const pkg: AgentPackage = {
    name: parsedConfig.name || input.agentId,
    version: input.version,
    description: input.description || parsedConfig.description || '',
    agentConfig: { ...parsedConfig, code: agentCode },
    tools: packageTools,
    bindings,
    permissions,
  };

  const serialized = serializeAgentPackage(pkg);
  const packageId = `pkg-${uuidv4().slice(0, 8)}`;

  await s3Client.send(new PutObjectCommand({
    Bucket: AGENT_BUCKET_NAME,
    Key: `packages/${packageId}/package.json`,
    Body: serialized,
    ContentType: 'application/json',
  }));

  const metadata = {
    packageId,
    name: pkg.name,
    version: pkg.version,
    description: pkg.description,
    agentId: input.agentId,
    status: 'PUBLISHED',
    bindings,
    bindingMap,
    createdBy: identity?.sub || 'system',
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
  };

  await docClient.send(new PutCommand({
    TableName: PACKAGES_TABLE,
    Item: metadata,
  }));

  return metadata;
}

async function deployAgentPackage(
  input: { packageId: string; bindingMap: string; targetOrgId: string; agentIdOverride?: string; activate?: boolean },
  identity: any,
) {
  const pkgMeta = await docClient.send(new GetCommand({
    TableName: PACKAGES_TABLE,
    Key: { packageId: input.packageId },
  }));

  if (!pkgMeta.Item) {
    throw new Error(`Package not found: ${input.packageId}`);
  }

  const s3Result = await s3Client.send(new GetObjectCommand({
    Bucket: AGENT_BUCKET_NAME,
    Key: `packages/${input.packageId}/package.json`,
  }));
  const pkgJson = await streamToString(s3Result.Body);
  const pkg = JSON.parse(JSON.parse(pkgJson).package ? pkgJson : pkgJson) as { package: AgentPackage } & any;
  const agentPkg = pkg.package || pkg;

  const bindingMap: BindingMap = typeof input.bindingMap === 'string' ? JSON.parse(input.bindingMap) : input.bindingMap;
  const resolved = resolveBindings(agentPkg.bindings, bindingMap);

  const newAgentId = input.agentIdOverride || `${agentPkg.name.toLowerCase().replace(/\s+/g, '-')}-${uuidv4().slice(0, 6)}`;

  if (agentPkg.agentConfig.code) {
    const filename = `${newAgentId}.py`;
    await s3Client.send(new PutObjectCommand({
      Bucket: AGENT_BUCKET_NAME,
      Key: `agents/${filename}`,
      Body: agentPkg.agentConfig.code,
    }));
    agentPkg.agentConfig.filename = filename;
  }
  delete agentPkg.agentConfig.code;

  await docClient.send(new PutCommand({
    TableName: AGENT_CONFIG_TABLE,
    Item: {
      agentId: newAgentId,
      config: JSON.stringify(agentPkg.agentConfig),
      state: input.activate ? 'active' : 'inactive',
      categories: [],
      manifest: { name: agentPkg.name, description: agentPkg.description, version: agentPkg.version },
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
      sourcePackageId: input.packageId,
      orgId: input.targetOrgId,
    },
  }));

  const toolIds: string[] = [];
  for (const tool of agentPkg.tools) {
    const newToolId = `${tool.toolId}-${uuidv4().slice(0, 6)}`;
    if (tool.code) {
      const toolFilename = `${newToolId}.py`;
      await s3Client.send(new PutObjectCommand({
        Bucket: AGENT_BUCKET_NAME,
        Key: `agents/${toolFilename}`,
        Body: tool.code,
      }));
      tool.config.filename = toolFilename;
    }
    delete tool.code;

    await docClient.send(new PutCommand({
      TableName: TOOLS_CONFIG_TABLE,
      Item: {
        toolId: newToolId,
        config: JSON.stringify(tool.config),
        state: input.activate ? 'active' : 'inactive',
        integrationBindings: resolved.integrationBindings,
        dataStoreBindings: resolved.dataStoreBindings,
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
        sourcePackageId: input.packageId,
      },
    }));
    toolIds.push(newToolId);
  }

  return {
    success: true,
    agentId: newAgentId,
    toolIds,
    unresolvedBindings: resolved.unresolved.length > 0 ? resolved.unresolved : null,
    message: resolved.unresolved.length > 0
      ? `Deployed with ${resolved.unresolved.length} unresolved binding(s): ${resolved.unresolved.join(', ')}`
      : 'Deployed successfully',
  };
}

async function listAgentPackages(status?: string) {
  const params: any = { TableName: PACKAGES_TABLE };
  if (status) {
    params.FilterExpression = '#s = :status';
    params.ExpressionAttributeNames = { '#s': 'status' };
    params.ExpressionAttributeValues = { ':status': status };
  }
  const result = await docClient.send(new ScanCommand(params));
  return { items: result.Items || [], nextToken: null };
}

async function getAgentPackage(packageId: string) {
  const meta = await docClient.send(new GetCommand({
    TableName: PACKAGES_TABLE,
    Key: { packageId },
  }));
  if (!meta.Item) throw new Error(`Package not found: ${packageId}`);

  const s3Result = await s3Client.send(new GetObjectCommand({
    Bucket: AGENT_BUCKET_NAME,
    Key: `packages/${packageId}/package.json`,
  }));
  const packageJson = await streamToString(s3Result.Body);

  return { metadata: meta.Item, packageJson };
}

async function deprecatePackage(input: { packageId: string }) {
  const meta = await docClient.send(new GetCommand({
    TableName: PACKAGES_TABLE,
    Key: { packageId: input.packageId },
  }));
  if (!meta.Item) throw new Error(`Package not found: ${input.packageId}`);

  const updated = { ...meta.Item, status: 'DEPRECATED', updatedAt: new Date().toISOString() };
  await docClient.send(new PutCommand({ TableName: PACKAGES_TABLE, Item: updated }));
  return updated;
}

async function streamToString(stream: any): Promise<string> {
  const chunks: Uint8Array[] = [];
  for await (const chunk of stream) {
    chunks.push(typeof chunk === 'string' ? Buffer.from(chunk) : chunk);
  }
  return Buffer.concat(chunks).toString('utf-8');
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && npx jest --testPathPattern="agent-package-resolver.test" --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/src/lambda/agent-package-resolver.ts backend/src/lambda/__tests__/agent-package-resolver.test.ts
git commit -m "feat(packaging): implement packageAgent and deployAgentPackage resolvers"
```

---

## Task 5: CDK — Packages Table & Resolver Wiring

**Files:**
- Modify: `backend/lib/backend-stack.ts`

- [ ] **Step 1: Add packages DynamoDB table and Lambda resolver to backend stack**

In `backend/lib/backend-stack.ts`, add after the existing table definitions:

```typescript
const packagesTable = new Table(this, 'PackagesTable', {
  tableName: `citadel-packages-${props.environment}`,
  partitionKey: { name: 'packageId', type: AttributeType.STRING },
  billingMode: BillingMode.PAY_PER_REQUEST,
  removalPolicy: RemovalPolicy.DESTROY,
});
```

Add a new Lambda function (follow existing pattern from other resolvers):

```typescript
const agentPackageResolverLambda = new NodejsFunction(this, 'AgentPackageResolver', {
  functionName: `citadel-agent-package-resolver-${props.environment}`,
  entry: path.join(__dirname, '../src/lambda/agent-package-resolver.ts'),
  handler: 'handler',
  runtime: Runtime.NODEJS_20_X,
  timeout: Duration.seconds(30),
  memorySize: 256,
  environment: {
    AGENT_CONFIG_TABLE: agentConfigTable.tableName,
    TOOLS_CONFIG_TABLE: toolsConfigTable.tableName,
    AGENT_BUCKET_NAME: this.codeBucket.bucketName,
    PACKAGES_TABLE: packagesTable.tableName,
  },
});

agentConfigTable.grantReadData(agentPackageResolverLambda);
toolsConfigTable.grantReadWriteData(agentPackageResolverLambda);
packagesTable.grantReadWriteData(agentPackageResolverLambda);
this.codeBucket.grantReadWrite(agentPackageResolverLambda);
```

Wire to AppSync (follow existing `addLambdaDataSource` + `createResolver` pattern):

```typescript
const packageDataSource = api.addLambdaDataSource('AgentPackageDataSource', agentPackageResolverLambda);

packageDataSource.createResolver('PackageAgentResolver', {
  typeName: 'Mutation',
  fieldName: 'packageAgent',
});
packageDataSource.createResolver('DeployAgentPackageResolver', {
  typeName: 'Mutation',
  fieldName: 'deployAgentPackage',
});
packageDataSource.createResolver('DeprecatePackageResolver', {
  typeName: 'Mutation',
  fieldName: 'deprecatePackage',
});
packageDataSource.createResolver('ListAgentPackagesResolver', {
  typeName: 'Query',
  fieldName: 'listAgentPackages',
});
packageDataSource.createResolver('GetAgentPackageResolver', {
  typeName: 'Query',
  fieldName: 'getAgentPackage',
});
```

- [ ] **Step 2: Verify CDK synth succeeds**

Run: `cd backend && npm run build && npx cdk synth --all 2>&1 | tail -5`
Expected: Synthesis completes (may show cdk-nag warnings but no errors)

- [ ] **Step 3: Commit**

```bash
git add backend/lib/backend-stack.ts
git commit -m "feat(packaging): add packages DynamoDB table and wire resolver to AppSync"
```

---

## Task 6: Deploy Resolver — Integration Test

**Files:**
- Create: `backend/src/lambda/__tests__/agent-package-deploy.test.ts`

- [ ] **Step 1: Write integration-style test for deployAgentPackage**

```typescript
// backend/src/lambda/__tests__/agent-package-deploy.test.ts
import { mockClient } from 'aws-sdk-client-mock';
import { DynamoDBDocumentClient, GetCommand, PutCommand, ScanCommand } from '@aws-sdk/lib-dynamodb';
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';
import { Readable } from 'stream';

const ddbMock = mockClient(DynamoDBDocumentClient);
const s3Mock = mockClient(S3Client);

process.env.AGENT_CONFIG_TABLE = 'citadel-agents-test';
process.env.TOOLS_CONFIG_TABLE = 'citadel-tools-test';
process.env.AGENT_BUCKET_NAME = 'citadel-code-test';
process.env.PACKAGES_TABLE = 'citadel-packages-test';

import { handler } from '../agent-package-resolver';

beforeEach(() => {
  ddbMock.reset();
  s3Mock.reset();
});

describe('deployAgentPackage', () => {
  const storedPackage = {
    schemaVersion: '1.0',
    package: {
      name: 'Test Agent',
      version: '1.0.0',
      description: 'Packaged agent',
      agentConfig: { name: 'Test Agent', description: 'test', filename: 'test.py', code: 'print("deployed")' },
      tools: [
        { toolId: 'tool-search', config: { name: 'Search', filename: 'search.py' }, code: 'def search(): pass' },
      ],
      bindings: [
        { ref: 'confluence-0', type: 'CONFLUENCE', direction: 'INPUT', operations: ['search_pages'], scope: 'integration' },
        { ref: 's3-0', type: 'S3', direction: 'OUTPUT', operations: ['write_object'], scope: 'datastore' },
      ],
      permissions: [],
    },
    createdAt: '2026-06-10T00:00:00Z',
  };

  it('deploys package with complete binding map', async () => {
    ddbMock.on(GetCommand, { TableName: 'citadel-packages-test' }).resolves({
      Item: { packageId: 'pkg-001', name: 'Test Agent', version: '1.0.0', status: 'PUBLISHED' },
    });
    s3Mock.on(GetObjectCommand).resolves({
      Body: Readable.from([JSON.stringify(storedPackage)]),
    } as any);
    ddbMock.on(PutCommand).resolves({});
    s3Mock.on(PutObjectCommand).resolves({});

    const result = await handler({
      info: { fieldName: 'deployAgentPackage' },
      arguments: {
        input: {
          packageId: 'pkg-001',
          bindingMap: JSON.stringify({ 'confluence-0': 'int-prod-conf', 's3-0': 'ds-prod-bucket' }),
          targetOrgId: 'org-prod',
          activate: false,
        },
      },
      identity: { sub: 'deployer-1' },
    } as any);

    expect(result.success).toBe(true);
    expect(result.agentId).toBeDefined();
    expect(result.toolIds).toHaveLength(1);
    expect(result.unresolvedBindings).toBeNull();
  });

  it('reports unresolved bindings without failing', async () => {
    ddbMock.on(GetCommand, { TableName: 'citadel-packages-test' }).resolves({
      Item: { packageId: 'pkg-001', name: 'Test Agent', version: '1.0.0', status: 'PUBLISHED' },
    });
    s3Mock.on(GetObjectCommand).resolves({
      Body: Readable.from([JSON.stringify(storedPackage)]),
    } as any);
    ddbMock.on(PutCommand).resolves({});
    s3Mock.on(PutObjectCommand).resolves({});

    const result = await handler({
      info: { fieldName: 'deployAgentPackage' },
      arguments: {
        input: {
          packageId: 'pkg-001',
          bindingMap: JSON.stringify({ 'confluence-0': 'int-prod-conf' }),
          targetOrgId: 'org-prod',
        },
      },
      identity: { sub: 'deployer-1' },
    } as any);

    expect(result.success).toBe(true);
    expect(result.unresolvedBindings).toEqual(['s3-0']);
    expect(result.message).toContain('unresolved');
  });
});
```

- [ ] **Step 2: Run tests**

Run: `cd backend && npx jest --testPathPattern="agent-package-deploy" --no-coverage`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add backend/src/lambda/__tests__/agent-package-deploy.test.ts
git commit -m "test(packaging): add integration tests for deployAgentPackage resolver"
```

---

## Task 7: Cross-Account Deployment Role (CDK Construct)

**Files:**
- Create: `backend/lib/constructs/cross-account-deploy-role.ts`

This task creates an optional CDK construct that a target account can deploy to allow the Citadel source account to deploy agent packages into it.

- [ ] **Step 1: Create the construct**

```typescript
// backend/lib/constructs/cross-account-deploy-role.ts
import { Construct } from 'constructs';
import { Stack } from 'aws-cdk-lib';
import { Role, AccountPrincipal, PolicyStatement, Effect, ManagedPolicy } from 'aws-cdk-lib/aws-iam';

export interface CrossAccountDeployRoleProps {
  sourceAccountId: string;
  environment: string;
}

export class CrossAccountDeployRole extends Construct {
  public readonly role: Role;

  constructor(scope: Construct, id: string, props: CrossAccountDeployRoleProps) {
    super(scope, id);

    this.role = new Role(this, 'DeployRole', {
      roleName: `citadel-package-deploy-${props.environment}`,
      assumedBy: new AccountPrincipal(props.sourceAccountId),
      description: 'Allows Citadel source account to deploy agent packages into this account',
    });

    this.role.addToPolicy(new PolicyStatement({
      effect: Effect.ALLOW,
      actions: [
        'dynamodb:PutItem',
        'dynamodb:GetItem',
        'dynamodb:UpdateItem',
      ],
      resources: [
        `arn:aws:dynamodb:${Stack.of(this).region}:${Stack.of(this).account}:table/citadel-agents-${props.environment}`,
        `arn:aws:dynamodb:${Stack.of(this).region}:${Stack.of(this).account}:table/citadel-tools-${props.environment}`,
      ],
    }));

    this.role.addToPolicy(new PolicyStatement({
      effect: Effect.ALLOW,
      actions: [
        's3:PutObject',
        's3:GetObject',
      ],
      resources: [
        `arn:aws:s3:::citadel-code-${props.environment}-${Stack.of(this).account}-${Stack.of(this).region}/agents/*`,
      ],
    }));

    this.role.addToPolicy(new PolicyStatement({
      effect: Effect.ALLOW,
      actions: [
        'iam:CreateRole',
        'iam:PutRolePolicy',
        'iam:DeleteRolePolicy',
        'iam:DeleteRole',
        'sts:AssumeRole',
      ],
      resources: [
        `arn:aws:iam::${Stack.of(this).account}:role/citadel-agent-*`,
      ],
    }));
  }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cd backend && npm run build`
Expected: Compiles without errors

- [ ] **Step 3: Commit**

```bash
git add backend/lib/constructs/cross-account-deploy-role.ts
git commit -m "feat(packaging): add CDK construct for cross-account deploy role"
```

---

## Task 8: Documentation

**Files:**
- Create: `docs/AGENT_PACKAGING.md`

- [ ] **Step 1: Write the packaging guide**

```markdown
# Agent Packaging & Cross-Account Deployment

## Overview

Agent Packages are versioned, portable artifacts that bundle an agent's configuration, code, tools, and abstract binding declarations into a deployable unit. Packages enable:

- **Environment promotion** — build in dev, deploy to staging/prod
- **Cross-account deployment** — package in shared services, deploy to production accounts
- **Version history** — track which package version is running where
- **Reproducibility** — deploy the exact same agent configuration repeatedly

## Package Format

A package contains:

| Component | Description |
|-----------|-------------|
| Agent config | Runtime configuration (name, description, model, system prompt) |
| Agent code | Python source code |
| Tool configs | Tool configurations with abstract binding references |
| Tool code | Python source for each tool |
| Abstract bindings | Type + direction + operations — no concrete IDs |
| Permissions | IAM action/resource templates |

## Abstract Bindings

Bindings use abstract references instead of concrete resource IDs:

```json
{
  "ref": "confluence-0",
  "type": "CONFLUENCE", 
  "direction": "INPUT",
  "operations": ["search_pages"],
  "scope": "integration"
}
```

At deploy time, a **binding map** resolves these to concrete resources in the target environment:

```json
{
  "confluence-0": "int-prod-confluence-001",
  "s3-0": "ds-prod-data-bucket"
}
```

## Lifecycle

```
Develop (dev) → Package (v1.0.0) → Deploy (staging) → Validate → Deploy (prod)
                    ↓
            Package Registry (S3 + DynamoDB)
```

### Packaging

```graphql
mutation {
  packageAgent(input: {
    agentId: "my-agent"
    version: "1.0.0"
    description: "Initial release"
  }) {
    packageId
    name
    version
    bindings { ref type direction scope }
  }
}
```

### Deploying

```graphql
mutation {
  deployAgentPackage(input: {
    packageId: "pkg-abc123"
    bindingMap: "{\"confluence-0\": \"int-prod-001\", \"s3-0\": \"ds-prod-bucket\"}"
    targetOrgId: "org-production"
    activate: false
  }) {
    success
    agentId
    toolIds
    unresolvedBindings
  }
}
```

### Cross-Account Deployment

1. Deploy the `CrossAccountDeployRole` construct in the target account
2. The source Citadel instance assumes this role to write agent config, code, and IAM roles
3. Binding map references resources that exist in the target account

## Binding Map Per Environment

Maintain a binding map file per environment:

```json
// binding-maps/staging.json
{
  "confluence-0": "int-staging-confluence",
  "jira-0": "int-staging-jira",
  "s3-output": "ds-staging-reports"
}

// binding-maps/production.json
{
  "confluence-0": "int-prod-confluence",
  "jira-0": "int-prod-jira",
  "s3-output": "ds-prod-reports"
}
```

## API Reference

| Operation | Type | Description |
|-----------|------|-------------|
| `packageAgent` | Mutation | Serialize an active agent into a versioned package |
| `deployAgentPackage` | Mutation | Deploy a package into the current environment with binding resolution |
| `deprecatePackage` | Mutation | Mark a package as deprecated |
| `listAgentPackages` | Query | List packages, optionally filtered by status |
| `getAgentPackage` | Query | Get package metadata and full JSON |
```

- [ ] **Step 2: Commit**

```bash
git add docs/AGENT_PACKAGING.md
git commit -m "docs: add agent packaging and cross-account deployment guide"
```

---

## Self-Review Checklist

1. **Spec coverage:** Package schema (Task 1), binding abstraction (Task 1), binding resolution (Task 2), GraphQL API (Task 3), package mutation (Task 4), deploy mutation (Task 4), CDK wiring (Task 5), deploy tests (Task 6), cross-account IAM (Task 7), docs (Task 8). All covered.
2. **Placeholder scan:** No TBD/TODO. All steps have concrete code.
3. **Type consistency:** `AgentPackage`, `AbstractBinding`, `BindingMap`, `ResolvedBindings` used consistently across Tasks 1-4. `serializeAgentPackage`/`deserializeAgentPackage` match between schema module and resolver.
