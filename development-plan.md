# Embedded Analytics SDK — Phased Development Plan

> Project: Embedded Analytics SDK (Candidate #033)
> Created: 2026-05-25
> Based on: research.md, features.md, README.md, data-model-suggestion-1 through 4

---

## Technology Decisions & Rationale

### Data Model: Hybrid Relational + JSONB (Suggestion 3) with ClickHouse Analytics Path (from Suggestion 4)

**Decision:** Start with the Hybrid Relational + JSONB model (Suggestion 3) for the MVP, with a planned migration to a dual-engine PostgreSQL + ClickHouse architecture (Suggestion 4) in Phase 7.

**Rationale:**
- The fully normalized model (Suggestion 1, ~33 tables) imposes excessive schema rigidity for an SDK whose embedding customers will have wildly different requirements. Adding a new widget property or RLS metadata field should not require a database migration.
- The event-sourced model (Suggestion 2) delivers excellent audit and temporal query capabilities but imposes disproportionate initial development complexity (event replay, projection management, snapshot strategy) for an MVP whose first priority is shipping a working embedded SDK.
- The hybrid model (Suggestion 3, ~17 tables) provides the fastest path to MVP: JSONB absorbs per-customer variability (widget configs, RLS policies, semantic layer definitions, SSO settings), while typed relational columns enforce integrity on core structural data (IDs, foreign keys, tenant hierarchy, timestamps). Fields that prove critical can be promoted from JSONB to typed columns as query patterns stabilize.
- ClickHouse (from Suggestion 4) is deferred to Phase 7 because query execution logging and usage analytics do not need columnar storage until the platform handles >1,000 queries/minute. The MVP uses PostgreSQL for query logs; the architecture is designed so the ClickHouse migration is additive (new tables, CDC pipeline) rather than disruptive.

### Language & Runtime: TypeScript / Node.js

**Rationale:** The SDK's primary consumers are React/Vue/Angular frontend developers. A TypeScript backend ensures type safety across the full stack, enables shared type definitions between the SDK client and server, and aligns with the Cube.js ecosystem (TypeScript-native). Node.js provides non-blocking I/O suitable for the query proxy workload (forwarding queries to customer data warehouses).

### Frontend SDK: React (primary) + Web Components (framework-agnostic)

**Rationale:** React dominates the SaaS frontend market. A React SDK is the highest-impact first delivery. Web Components (via Lit or Stencil) provide framework-agnostic embedding for Vue, Angular, Svelte, and vanilla JS customers. The Web Component layer wraps the React component library, ensuring a single source of truth for rendering logic.

### Semantic Layer: Custom YAML/JSON definitions inspired by Cube.js

**Rationale:** Building on Cube.js's MIT-licensed semantic layer pattern (cubes, measures, dimensions, joins) avoids reinventing the wheel while keeping the SDK self-contained. Storing definitions as JSONB in PostgreSQL (per the hybrid data model) enables API-driven management, version control export, and LLM-powered auto-generation.

### Query Engine: Direct warehouse query with caching layer

**Rationale:** The SDK proxies queries to customer data warehouses (Snowflake, BigQuery, Redshift, PostgreSQL) with a caching layer (Redis for hot cache, PostgreSQL for warm cache). This avoids ingesting customer data into the SDK's own storage (a significant security and compliance simplification).

### Authentication: JWT embed tokens + OIDC/OAuth 2.0 SSO

**Rationale:** Per-session JWT embed tokens (short-lived, scoped, with RLS context) are the industry standard for embedded analytics. OIDC/OAuth 2.0 SSO integration delegates identity management to the host application's identity provider, avoiding user management complexity.

### Licensing: MIT (core SDK) + Proprietary (managed cloud)

**Rationale:** MIT is the most permissive open-source license, matching Cube.js and maximizing adoption. Commercial revenue comes from the managed cloud service, enterprise support, and premium features (NL query, AI suggestions).

---

## Project Structure

```
embedded-analytics-sdk/
├── packages/
│   ├── core/                    # Shared types, interfaces, constants
│   │   ├── src/
│   │   │   ├── types/           # TypeScript type definitions
│   │   │   ├── schemas/         # Zod validation schemas
│   │   │   └── constants/       # Shared constants
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── server/                  # Backend API server
│   │   ├── src/
│   │   │   ├── api/             # REST API routes (Express/Fastify)
│   │   │   ├── auth/            # JWT, OIDC, embed token logic
│   │   │   ├── db/              # PostgreSQL repositories + migrations
│   │   │   ├── query-engine/    # Warehouse query proxy + cache
│   │   │   ├── semantic/        # Semantic layer parser + validator
│   │   │   ├── rls/             # Row-level security engine
│   │   │   ├── themes/          # Theme resolution + CSS generation
│   │   │   ├── ws/              # WebSocket handlers (live refresh)
│   │   │   └── ai/              # NL query + dashboard suggestion (Phase 6+)
│   │   ├── migrations/          # PostgreSQL migration files
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── react-sdk/               # React component library
│   │   ├── src/
│   │   │   ├── components/      # Dashboard, Widget, Filter, Chart components
│   │   │   ├── hooks/           # useQuery, useDashboard, useTheme, etc.
│   │   │   ├── providers/       # EmbedProvider (context + token management)
│   │   │   └── charts/          # Chart renderers (recharts wrappers)
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── web-components/          # Framework-agnostic Web Component wrappers
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── cli/                     # Developer CLI (schema sync, deploy, etc.)
│       ├── src/
│       ├── package.json
│       └── tsconfig.json
├── apps/
│   ├── demo/                    # Demo SaaS app embedding the SDK
│   ├── docs/                    # Documentation site (Docusaurus or similar)
│   └── playground/              # Interactive semantic layer playground
├── docker/                      # Docker Compose for local development
├── turbo.json                   # Turborepo monorepo config
├── package.json                 # Root workspace config
└── tsconfig.base.json           # Shared TypeScript config
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    │
    ├──► Phase 2: Query Engine & Semantic Layer
    │       │
    │       ├──► Phase 3: Dashboard & Widget System
    │       │       │
    │       │       ├──► Phase 5: Self-Serve Canvas
    │       │       │
    │       │       └──► Phase 6: AI & NL Query
    │       │
    │       └──► Phase 4: Multi-Tenancy & RLS
    │               │
    │               └──► Phase 8: SOC 2 & Enterprise Governance
    │
    ├──► Phase 7: ClickHouse Analytics Engine
    │       │
    │       └──► Phase 9: Agentic Analytics & Advanced AI
    │
    └──► Phase 10: Managed Cloud & Monetization

Phase 11: Accessibility & Benchmarking (parallel, no hard dependencies)
```

**Critical path:** Phase 1 -> Phase 2 -> Phase 3 -> Phase 5 (MVP to self-serve)

**Parallel tracks after Phase 2:**
- Track A (User-Facing): Phases 3, 5, 6 (dashboards, self-serve, AI)
- Track B (Platform): Phases 4, 7, 8 (security, scale, compliance)
- Track C (Business): Phase 10 (cloud, billing)

---

## Phase 1: Foundation & Core Infrastructure

**Duration:** 4 weeks
**Dependencies:** None (starting point)
**Goal:** Monorepo scaffolding, database schema, authentication, basic REST API, and CI/CD pipeline.

### Task 1.1: Monorepo Setup & Toolchain

**What:** Initialize the Turborepo monorepo with all packages (`core`, `server`, `react-sdk`, `web-components`, `cli`). Configure TypeScript, ESLint, Prettier, Vitest, and Docker Compose for local PostgreSQL.

**Design:**
```typescript
// turbo.json
{
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] }
  }
}

// packages/core/src/types/organization.ts
export interface Organization {
  id: string;          // UUID
  name: string;
  slug: string;
  plan: 'free' | 'starter' | 'pro' | 'enterprise';
  settings: OrganizationSettings;
  createdAt: Date;
  updatedAt: Date;
}

export interface OrganizationSettings {
  billingEmail?: string;
  sso?: SSOConfig;
  featureFlags?: Record<string, boolean | number>;
  defaultTheme?: ThemeOverrides;
}
```

**Testing:**
- `turbo run build` completes without errors across all packages
- `turbo run typecheck` passes for all packages
- `docker compose up -d` starts PostgreSQL 16 and Redis on default ports
- Vitest configuration runs a sample test in each package
- ESLint/Prettier pass on the entire monorepo

### Task 1.2: PostgreSQL Schema & Migrations

**What:** Implement the Hybrid Relational + JSONB schema (Data Model Suggestion 3) using a migration tool (node-pg-migrate or Drizzle ORM migrations). Create the 17 core tables: organizations, workspaces, users, memberships, data_sources, semantic_cubes, collections, dashboards, widgets, rls_policies, embed_tokens, query_executions, query_cache, audit_events, nl_query_sessions, nl_query_messages, sso_configurations.

**Design:**
```typescript
// packages/server/src/db/migrate.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { migrate } from 'drizzle-orm/node-postgres/migrator';

export async function runMigrations(connectionString: string) {
  const db = drizzle(connectionString);
  await migrate(db, { migrationsFolder: './migrations' });
}

// packages/server/src/db/schema/organizations.ts (Drizzle ORM schema)
import { pgTable, uuid, text, jsonb, timestamp, pgEnum } from 'drizzle-orm/pg-core';

export const planEnum = pgEnum('plan', ['free', 'starter', 'pro', 'enterprise']);

export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  plan: planEnum('plan').notNull().default('free'),
  settings: jsonb('settings').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing:**
- Migration runs forward and backward (up/down) without errors on a clean database
- All 17 tables are created with correct columns, types, indexes, and constraints
- Foreign key constraints enforce referential integrity (e.g., workspace cannot reference non-existent organization)
- UNIQUE constraints prevent duplicate slugs, duplicate memberships
- JSONB columns accept arbitrary JSON and reject non-JSON input
- GIN indexes exist on `workspaces.settings`, `rls_policies.policy`, `audit_events.details`

### Task 1.3: Authentication — JWT Embed Tokens

**What:** Implement JWT embed token issuance and validation. The host SaaS application calls a server-side API to generate a short-lived JWT for a specific workspace, with scopes and RLS context embedded in the token claims.

**Design:**
```typescript
// packages/server/src/auth/embed-token.ts
import jwt from 'jsonwebtoken';

export interface EmbedTokenPayload {
  workspaceId: string;
  userId?: string;
  scopes: string[];         // e.g., ['dashboard:read', 'query:execute']
  rlsContext: Record<string, string>;  // e.g., { tenant_id: 'abc' }
}

export function issueEmbedToken(
  payload: EmbedTokenPayload,
  secret: string,
  expiresInSeconds: number = 3600
): string {
  return jwt.sign(
    {
      wid: payload.workspaceId,
      uid: payload.userId,
      scp: payload.scopes,
      rls: payload.rlsContext,
    },
    secret,
    { expiresIn: expiresInSeconds, algorithm: 'HS256', issuer: 'embedded-analytics' }
  );
}

export function verifyEmbedToken(token: string, secret: string): EmbedTokenPayload {
  const decoded = jwt.verify(token, secret, { issuer: 'embedded-analytics' }) as any;
  return {
    workspaceId: decoded.wid,
    userId: decoded.uid,
    scopes: decoded.scp,
    rlsContext: decoded.rls,
  };
}

// packages/server/src/auth/middleware.ts
export function requireEmbedToken(requiredScope?: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) return res.status(401).json({ error: 'Missing embed token' });
    try {
      const payload = verifyEmbedToken(token, process.env.EMBED_TOKEN_SECRET!);
      if (requiredScope && !payload.scopes.includes(requiredScope)) {
        return res.status(403).json({ error: 'Insufficient scope' });
      }
      req.embedContext = payload;
      next();
    } catch (err) {
      return res.status(401).json({ error: 'Invalid or expired token' });
    }
  };
}
```

**Testing:**
- Token issuance produces a valid JWT with correct claims (workspaceId, scopes, rlsContext)
- Token verification succeeds for a valid, non-expired token
- Token verification rejects expired tokens (test with 1-second expiry + wait)
- Token verification rejects tokens signed with a different secret
- Token verification rejects malformed tokens (truncated, random strings)
- Scope enforcement: middleware rejects tokens missing a required scope
- Token hash is stored in `embed_tokens` table; raw token is never stored
- RLS context is correctly round-tripped through token issuance and verification

### Task 1.4: REST API Scaffold & CRUD Endpoints

**What:** Implement CRUD REST API endpoints for organizations, workspaces, and users using Express or Fastify. Include input validation (Zod), error handling, and request logging.

**Design:**
```typescript
// packages/server/src/api/routes/workspaces.ts
import { z } from 'zod';

const CreateWorkspaceSchema = z.object({
  name: z.string().min(1).max(255),
  slug: z.string().regex(/^[a-z0-9-]+$/).min(1).max(100),
  settings: z.record(z.unknown()).optional().default({}),
});

router.post('/organizations/:orgId/workspaces', requireAuth('admin'), async (req, res) => {
  const orgId = z.string().uuid().parse(req.params.orgId);
  const body = CreateWorkspaceSchema.parse(req.body);
  const workspace = await workspaceRepo.create({ organizationId: orgId, ...body });
  await auditLog({ action: 'workspace.created', resourceType: 'workspace', resourceId: workspace.id, ...req.auditContext });
  res.status(201).json(workspace);
});

router.get('/organizations/:orgId/workspaces', requireAuth('viewer'), async (req, res) => {
  const orgId = z.string().uuid().parse(req.params.orgId);
  const workspaces = await workspaceRepo.listByOrganization(orgId);
  res.json({ data: workspaces });
});
```

**Testing:**
- POST /organizations creates an organization and returns 201 with UUID
- POST /organizations with duplicate slug returns 409 Conflict
- POST /organizations/:orgId/workspaces creates a workspace scoped to the org
- GET /organizations/:orgId/workspaces returns only workspaces belonging to that org
- Invalid input (missing name, invalid slug format, non-UUID orgId) returns 400 with Zod error details
- Unauthenticated requests return 401
- All mutations create audit_events records
- Response bodies match the TypeScript interfaces defined in `@embedded-analytics/core`

### Task 1.5: CI/CD Pipeline

**What:** Configure GitHub Actions for continuous integration: lint, typecheck, unit tests, integration tests (against Docker PostgreSQL), and package publishing.

**Design:**
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: analytics_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx turbo run lint typecheck
      - run: npx turbo run test
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/analytics_test
```

**Testing:**
- Pipeline runs on every push and PR
- Lint and typecheck jobs fail the build on violations
- Unit tests run without database dependency
- Integration tests run against the CI PostgreSQL service
- Build artifacts are produced for all packages

### Definition of Done — Phase 1
- [ ] Monorepo builds and typechecks across all packages
- [ ] PostgreSQL schema deployed via migrations (17 tables, all indexes, all constraints)
- [ ] JWT embed token issuance and verification working with scope enforcement
- [ ] CRUD endpoints for organizations, workspaces, users with Zod validation
- [ ] Audit events logged for all mutations
- [ ] Docker Compose runs the full stack locally (server + PostgreSQL + Redis)
- [ ] CI pipeline passes on GitHub Actions
- [ ] 80%+ unit test coverage for auth and API modules

---

## Phase 2: Query Engine & Semantic Layer

**Duration:** 5 weeks
**Dependencies:** Phase 1 (database schema, auth, API scaffold)
**Goal:** Connect to customer data warehouses, define semantic layer cubes, execute queries with caching, and return results via API.

### Task 2.1: Data Source Connection Manager

**What:** Implement a connection manager that establishes and pools connections to customer data warehouses (PostgreSQL, Snowflake, BigQuery, Redshift). Credentials are encrypted at rest. Connection health is monitored.

**Design:**
```typescript
// packages/server/src/query-engine/connection-manager.ts
import { Pool } from 'pg';

export interface DataSourceConnection {
  engine: 'postgresql' | 'snowflake' | 'bigquery' | 'redshift' | 'clickhouse' | 'mysql';
  execute(sql: string, params?: unknown[]): Promise<QueryResult>;
  testConnection(): Promise<{ ok: boolean; latencyMs: number; error?: string }>;
  close(): Promise<void>;
}

export class ConnectionManager {
  private pools: Map<string, DataSourceConnection> = new Map();

  async getConnection(dataSourceId: string): Promise<DataSourceConnection> {
    if (this.pools.has(dataSourceId)) return this.pools.get(dataSourceId)!;
    const config = await this.loadDecryptedConfig(dataSourceId);
    const conn = await this.createConnection(config);
    this.pools.set(dataSourceId, conn);
    return conn;
  }

  private async createConnection(config: DecryptedDataSourceConfig): Promise<DataSourceConnection> {
    switch (config.engine) {
      case 'postgresql': return new PostgreSQLConnection(config);
      case 'snowflake':  return new SnowflakeConnection(config);
      case 'bigquery':   return new BigQueryConnection(config);
      case 'redshift':   return new RedshiftConnection(config);
      default: throw new Error(`Unsupported engine: ${config.engine}`);
    }
  }
}
```

**Testing:**
- PostgreSQL connection: execute a simple query, verify results match expected data
- Connection pooling: multiple concurrent requests reuse the same pool
- testConnection returns latency and success/failure status
- Invalid credentials return a clear error, not a crash
- Connection is closed gracefully on shutdown
- Encrypted credentials are decrypted only in memory; plaintext never written to disk or logs
- Schema sync: `INFORMATION_SCHEMA` query retrieves table/column metadata and stores as JSONB in `data_sources.schema_cache`

### Task 2.2: Semantic Layer — Cube Definitions

**What:** Implement the semantic layer: YAML/JSON cube definition parser, validator, and storage. Cubes define measures (aggregations), dimensions (group-by fields), and joins between cubes.

**Design:**
```typescript
// packages/server/src/semantic/cube-parser.ts
import { z } from 'zod';

const MeasureSchema = z.object({
  name: z.string().regex(/^[a-z_][a-z0-9_]*$/),
  sql: z.string().min(1),
  type: z.enum(['count', 'sum', 'avg', 'min', 'max', 'count_distinct', 'custom']),
  format: z.enum(['number', 'currency', 'percent']).optional(),
  description: z.string().optional(),
});

const DimensionSchema = z.object({
  name: z.string().regex(/^[a-z_][a-z0-9_]*$/),
  sql: z.string().min(1),
  type: z.enum(['string', 'number', 'time', 'boolean', 'geo']),
  isPrimaryKey: z.boolean().optional().default(false),
  description: z.string().optional(),
});

const JoinSchema = z.object({
  target: z.string(),
  relationship: z.enum(['one_to_one', 'one_to_many', 'many_to_one', 'many_to_many']),
  sqlOn: z.string(),
});

const CubeDefinitionSchema = z.object({
  measures: z.array(MeasureSchema).min(1),
  dimensions: z.array(DimensionSchema),
  joins: z.array(JoinSchema).optional().default([]),
  segments: z.array(z.object({ name: z.string(), sql: z.string() })).optional().default([]),
});

export function parseCubeDefinition(input: unknown): CubeDefinition {
  return CubeDefinitionSchema.parse(input);
}
```

**Testing:**
- Valid cube definition (YAML and JSON) parses without errors
- Invalid cube definition (missing measures, invalid measure type, malformed SQL reference) returns descriptive validation errors
- Measure names enforce snake_case convention
- Duplicate measure/dimension names within a cube are rejected
- Join references to non-existent cubes are flagged as warnings (validated at query time)
- CRUD API: POST /semantic-cubes creates a cube; GET returns the definition; PUT updates it; DELETE removes it
- Version field increments on each update
- Published cubes are immutable (updates create a new version)

### Task 2.3: Query Compiler

**What:** Compile a query request (measures, dimensions, filters, time dimension, order, limit) against the semantic layer into SQL for the target data warehouse dialect.

**Design:**
```typescript
// packages/server/src/query-engine/query-compiler.ts
export interface QueryRequest {
  cube: string;
  measures: string[];
  dimensions?: string[];
  filters?: Filter[];
  timeDimension?: { dimension: string; granularity: TimeGranularity; dateRange?: [string, string] };
  orderBy?: { name: string; direction: 'asc' | 'desc' }[];
  limit?: number;
}

export interface CompiledQuery {
  sql: string;
  params: unknown[];
  dialect: 'postgresql' | 'snowflake' | 'bigquery' | 'redshift';
}

export class QueryCompiler {
  compile(request: QueryRequest, cube: CubeDefinition, dialect: Dialect): CompiledQuery {
    const selectCols = [
      ...request.measures.map(m => this.resolveMeasure(cube, m)),
      ...this.resolveDimensions(cube, request.dimensions),
    ];
    const fromClause = this.resolveFrom(cube);
    const whereClause = this.resolveFilters(cube, request.filters, dialect);
    const groupBy = request.dimensions?.length ? this.resolveGroupBy(request.dimensions) : '';
    const orderBy = this.resolveOrderBy(request.orderBy);
    const limit = request.limit ? `LIMIT ${request.limit}` : '';

    return {
      sql: `SELECT ${selectCols.join(', ')} FROM ${fromClause} ${whereClause} ${groupBy} ${orderBy} ${limit}`,
      params: [],
      dialect,
    };
  }
}
```

**Testing:**
- Simple query (1 measure, 1 dimension) compiles to correct SQL for PostgreSQL
- Time dimension with granularity generates correct DATE_TRUNC for PostgreSQL, DATEADD for Snowflake, TIMESTAMP_TRUNC for BigQuery
- Filters compile to parameterized WHERE clauses (no SQL injection)
- Join queries across two cubes produce correct JOIN...ON clauses
- RLS filter is appended to WHERE clause (tested in Task 2.4)
- ORDER BY and LIMIT are correctly applied
- Measure references to non-existent cubes produce a clear error
- Compiled SQL is logged with parameters redacted for debugging

### Task 2.4: RLS Enforcement

**What:** Implement row-level security enforcement. When a query is executed in the context of an embed token, the RLS context from the token is combined with the RLS policies for the workspace to inject filter conditions into the compiled SQL.

**Design:**
```typescript
// packages/server/src/rls/rls-engine.ts
export class RLSEngine {
  async applyRLS(
    compiledQuery: CompiledQuery,
    workspaceId: string,
    rlsContext: Record<string, string>,
    dataSourceId: string
  ): Promise<CompiledQuery> {
    const policies = await this.policyRepo.findActive(dataSourceId, workspaceId);
    const rlsFilters = policies.map(policy => {
      const binding = policy.policy.bindings.find(b => b.workspace_id === workspaceId);
      if (!binding) return null;
      return `${policy.policy.filter_column} = $${compiledQuery.params.length + 1}`;
    }).filter(Boolean);

    if (rlsFilters.length === 0) return compiledQuery;

    const rlsWhereClause = rlsFilters.join(' AND ');
    return {
      ...compiledQuery,
      sql: compiledQuery.sql.includes('WHERE')
        ? compiledQuery.sql.replace('WHERE', `WHERE (${rlsWhereClause}) AND`)
        : `${compiledQuery.sql} WHERE ${rlsWhereClause}`,
      params: [...compiledQuery.params, ...Object.values(rlsContext)],
    };
  }
}
```

**Testing:**
- Query with one RLS policy appends the correct WHERE filter
- Query with multiple RLS policies appends all filters with AND
- Query without any RLS policies passes through unchanged
- RLS context values are parameterized (not string-interpolated) — SQL injection test
- Missing RLS binding for a workspace rejects the query with 403
- RLS filters are applied to JOIN queries on the correct table alias
- Audit event records which RLS policies were applied to each query

### Task 2.5: Query Cache

**What:** Implement a two-tier query cache: Redis for hot results (<10 seconds), PostgreSQL for warm results (<5 minutes). Cache keys are computed from the hash of (cube + measures + dimensions + filters + RLS context).

**Design:**
```typescript
// packages/server/src/query-engine/query-cache.ts
import { createHash } from 'crypto';

export class QueryCache {
  constructor(private redis: Redis, private db: Database) {}

  computeCacheKey(request: QueryRequest, rlsContext: Record<string, string>): string {
    const input = JSON.stringify({ ...request, rls: rlsContext });
    return createHash('sha256').update(input).digest('hex');
  }

  async get(key: string): Promise<CachedResult | null> {
    // Try Redis first (hot cache)
    const redisResult = await this.redis.get(`qcache:${key}`);
    if (redisResult) return { data: JSON.parse(redisResult), source: 'redis' };

    // Fall back to PostgreSQL (warm cache)
    const pgResult = await this.db.queryCache.findByKey(key);
    if (pgResult && pgResult.expiresAt > new Date()) {
      // Promote back to Redis
      await this.redis.setex(`qcache:${key}`, 10, JSON.stringify(pgResult.resultData));
      return { data: pgResult.resultData, source: 'postgres' };
    }
    return null;
  }

  async set(key: string, data: unknown, ttlSeconds: number): Promise<void> {
    await Promise.all([
      this.redis.setex(`qcache:${key}`, Math.min(ttlSeconds, 10), JSON.stringify(data)),
      this.db.queryCache.upsert({ cacheKey: key, resultData: data, expiresAt: new Date(Date.now() + ttlSeconds * 1000) }),
    ]);
  }
}
```

**Testing:**
- Same query request + RLS context produces the same cache key
- Different RLS contexts produce different cache keys (tenant isolation)
- Cache hit returns data without executing a warehouse query
- Cache miss executes the query and stores the result
- Expired cache entries are not returned
- Redis failure gracefully falls back to PostgreSQL cache
- Cache invalidation: data source sync clears all cache entries for that source
- Cache key collision test: two semantically different queries never produce the same key

### Task 2.6: Query Execution API

**What:** Expose a REST API endpoint that accepts a query request, compiles it against the semantic layer, applies RLS, checks the cache, executes against the warehouse, and returns results with metadata.

**Design:**
```typescript
// packages/server/src/api/routes/query.ts
router.post('/query', requireEmbedToken('query:execute'), async (req, res) => {
  const queryRequest = QueryRequestSchema.parse(req.body);
  const { workspaceId, rlsContext } = req.embedContext;

  const cube = await cubeRepo.findByName(queryRequest.cube, workspaceId);
  if (!cube) return res.status(404).json({ error: `Cube '${queryRequest.cube}' not found` });

  const compiled = queryCompiler.compile(queryRequest, cube.definition, dataSource.engine);
  const secured = await rlsEngine.applyRLS(compiled, workspaceId, rlsContext, cube.dataSourceId);

  const cacheKey = queryCache.computeCacheKey(queryRequest, rlsContext);
  const cached = await queryCache.get(cacheKey);

  if (cached) {
    await queryExecLog.record({ ...logBase, cacheHit: true, durationMs: 0 });
    return res.json({ data: cached.data, meta: { cacheHit: true, source: cached.source } });
  }

  const startTime = performance.now();
  const conn = await connectionManager.getConnection(cube.dataSourceId);
  const result = await conn.execute(secured.sql, secured.params);
  const durationMs = Math.round(performance.now() - startTime);

  await queryCache.set(cacheKey, result.rows, cube.cacheTtlSeconds ?? 300);
  await queryExecLog.record({ ...logBase, cacheHit: false, durationMs, rowCount: result.rows.length });

  res.json({ data: result.rows, meta: { cacheHit: false, durationMs, rowCount: result.rows.length } });
});
```

**Testing:**
- Valid query returns 200 with rows and metadata
- Second identical query returns cache hit (meta.cacheHit = true)
- Query referencing non-existent cube returns 404
- Query without required scope returns 403
- Query with invalid filters returns 400 with validation errors
- Query execution is logged in `query_executions` table with duration, cache hit, and row count
- Warehouse connection timeout returns 504 Gateway Timeout
- Result set respects the LIMIT parameter

### Definition of Done — Phase 2
- [ ] Data source CRUD API with encrypted credential storage
- [ ] Schema sync populates `data_sources.schema_cache` from warehouse introspection
- [ ] Semantic layer CRUD API with Zod validation for cube definitions
- [ ] Query compiler generates correct SQL for PostgreSQL and at least one cloud warehouse (Snowflake or BigQuery)
- [ ] RLS enforcement injects per-workspace filters into compiled SQL
- [ ] Two-tier query cache (Redis + PostgreSQL) with correct key isolation per tenant
- [ ] POST /query endpoint executes the full pipeline (compile -> RLS -> cache -> execute -> return)
- [ ] Query execution log records every query with performance metrics
- [ ] Integration tests against a real PostgreSQL data warehouse (Docker)
- [ ] 80%+ test coverage for query compiler and RLS engine

---

## Phase 3: Dashboard & Widget System (React SDK)

**Duration:** 5 weeks
**Dependencies:** Phase 2 (query engine, semantic layer)
**Goal:** React component library that renders embedded dashboards with charts, filters, and theming. Publishable to npm.

### Task 3.1: EmbedProvider & React Context

**What:** Create the `<EmbedProvider>` context component that initializes the SDK with an embed token, establishes a connection to the API server, and provides query/theme context to child components.

**Design:**
```tsx
// packages/react-sdk/src/providers/EmbedProvider.tsx
import React, { createContext, useContext, useEffect, useState } from 'react';

interface EmbedConfig {
  apiUrl: string;
  token: string;
  theme?: ThemeOverrides;
}

interface EmbedContextValue {
  apiUrl: string;
  token: string;
  theme: ResolvedTheme;
  query: (request: QueryRequest) => Promise<QueryResult>;
  isReady: boolean;
}

const EmbedContext = createContext<EmbedContextValue | null>(null);

export function EmbedProvider({ config, children }: { config: EmbedConfig; children: React.ReactNode }) {
  const [isReady, setIsReady] = useState(false);
  const [theme, setTheme] = useState<ResolvedTheme>(defaultTheme);

  useEffect(() => {
    // Validate token, fetch workspace theme, merge overrides
    initializeEmbed(config).then(({ resolvedTheme }) => {
      setTheme(resolvedTheme);
      setIsReady(true);
    });
  }, [config.token]);

  const query = async (request: QueryRequest) => {
    const res = await fetch(`${config.apiUrl}/query`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${config.token}` },
      body: JSON.stringify(request),
    });
    if (!res.ok) throw new QueryError(await res.json());
    return res.json();
  };

  return (
    <EmbedContext.Provider value={{ apiUrl: config.apiUrl, token: config.token, theme, query, isReady }}>
      {isReady ? children : <LoadingSkeleton />}
    </EmbedContext.Provider>
  );
}

export function useEmbed() {
  const ctx = useContext(EmbedContext);
  if (!ctx) throw new Error('useEmbed must be used within <EmbedProvider>');
  return ctx;
}
```

**Testing:**
- `<EmbedProvider>` renders children only after initialization completes
- `useEmbed()` throws outside of provider context
- Invalid token shows error state, not loading spinner
- Theme overrides merge correctly with workspace defaults (override wins)
- `query()` function correctly passes token in Authorization header
- Token expiry triggers a re-authentication callback

### Task 3.2: Chart Components

**What:** Build chart components (BarChart, LineChart, PieChart, KPI, DataTable) using Recharts. Each component accepts a query request, executes it via the EmbedProvider context, and renders the result.

**Design:**
```tsx
// packages/react-sdk/src/components/BarChart.tsx
import { BarChart as RechartsBar, Bar, XAxis, YAxis, Tooltip, CartesianGrid } from 'recharts';

interface BarChartProps {
  cube: string;
  measures: string[];
  dimensions: string[];
  filters?: Filter[];
  colorPalette?: string[];
  xAxisLabel?: string;
  yAxisLabel?: string;
  height?: number;
}

export function BarChart(props: BarChartProps) {
  const { query, theme } = useEmbed();
  const { data, isLoading, error } = useQuery(query, {
    cube: props.cube,
    measures: props.measures,
    dimensions: props.dimensions,
    filters: props.filters,
  });

  if (isLoading) return <ChartSkeleton type="bar" height={props.height} />;
  if (error) return <ChartError error={error} />;

  const colors = props.colorPalette ?? theme.colorPalette;
  return (
    <RechartsBar data={data} width="100%" height={props.height ?? 300}>
      <CartesianGrid strokeDasharray="3 3" />
      <XAxis dataKey={props.dimensions[0]} label={props.xAxisLabel} />
      <YAxis label={props.yAxisLabel} />
      <Tooltip />
      {props.measures.map((measure, i) => (
        <Bar key={measure} dataKey={measure} fill={colors[i % colors.length]} />
      ))}
    </RechartsBar>
  );
}
```

**Testing:**
- BarChart renders correct number of bars matching data rows
- LineChart renders correct number of data points
- PieChart renders correct number of segments with correct proportions
- KPI component displays formatted number with label
- DataTable renders rows and columns with correct headers
- Empty data set renders an "empty state" component, not a crash
- Loading state renders skeleton with correct dimensions
- Error state renders error message with retry button
- Color palette from theme is applied when no override is provided
- Custom color palette override takes precedence

### Task 3.3: Dashboard Layout Component

**What:** Build the `<Dashboard>` component that loads a dashboard definition (widget list + grid layout) from the API and renders widgets in a responsive CSS grid.

**Design:**
```tsx
// packages/react-sdk/src/components/Dashboard.tsx
import { Responsive, WidthProvider } from 'react-grid-layout';

const ResponsiveGrid = WidthProvider(Responsive);

interface DashboardProps {
  dashboardId: string;
  className?: string;
  onWidgetClick?: (widgetId: string) => void;
}

export function Dashboard({ dashboardId, className, onWidgetClick }: DashboardProps) {
  const { apiUrl, token } = useEmbed();
  const { dashboard, isLoading } = useDashboard(dashboardId);

  if (isLoading) return <DashboardSkeleton />;

  const layout = dashboard.widgets.map(w => ({
    i: w.id, x: w.gridX, y: w.gridY, w: w.gridW, h: w.gridH,
  }));

  return (
    <ResponsiveGrid className={className} layouts={{ lg: layout }} cols={{ lg: 12, md: 8, sm: 4 }}>
      {dashboard.widgets.map(widget => (
        <div key={widget.id} onClick={() => onWidgetClick?.(widget.id)}>
          <WidgetRenderer widget={widget} />
        </div>
      ))}
    </ResponsiveGrid>
  );
}

function WidgetRenderer({ widget }: { widget: Widget }) {
  const ChartComponent = chartComponentMap[widget.widgetType];
  if (!ChartComponent) return <UnsupportedWidget type={widget.widgetType} />;
  return <ChartComponent {...widget.queryConfig} {...widget.visualConfig} />;
}
```

**Testing:**
- Dashboard loads and renders correct number of widgets
- Widgets are positioned according to grid layout (x, y, w, h)
- Responsive breakpoints adjust column count (12 -> 8 -> 4)
- Unsupported widget types render a graceful fallback, not a crash
- onWidgetClick callback fires with correct widget ID
- Dashboard with zero widgets renders an empty state
- Dashboard loading skeleton matches the expected layout

### Task 3.4: Theme Engine & White-Labelling

**What:** Implement theme resolution: organization default -> workspace override -> embed token override. Generate CSS custom properties from the resolved theme and inject them into the SDK's component tree.

**Design:**
```typescript
// packages/react-sdk/src/themes/theme-engine.ts
export interface ThemeConfig {
  primaryColor: string;
  secondaryColor: string;
  backgroundColor: string;
  textColor: string;
  fontFamily: string;
  borderRadius: string;
  logoUrl?: string;
  accessibilityMode: boolean;
}

export function resolveTheme(
  orgDefault: Partial<ThemeConfig>,
  workspaceOverride: Partial<ThemeConfig>,
  embedOverride: Partial<ThemeConfig>
): ThemeConfig {
  return {
    ...defaultTheme,
    ...orgDefault,
    ...workspaceOverride,
    ...embedOverride,
  };
}

export function themeToCSSVars(theme: ThemeConfig): Record<string, string> {
  return {
    '--ea-primary': theme.primaryColor,
    '--ea-secondary': theme.secondaryColor,
    '--ea-bg': theme.backgroundColor,
    '--ea-text': theme.textColor,
    '--ea-font': theme.fontFamily,
    '--ea-radius': theme.borderRadius,
  };
}
```

**Testing:**
- Theme resolution cascade: embed override > workspace override > org default > system default
- CSS custom properties are injected into the SDK root element
- Components use CSS vars (not hardcoded colors) for all visual properties
- Logo URL renders the customer's logo in dashboard headers
- Accessibility mode enables high-contrast color palette
- No SDK vendor branding is visible in any theme configuration
- Theme change triggers re-render of all affected components

### Task 3.5: npm Package Publishing

**What:** Configure package.json for publishing `@embedded-analytics/react-sdk` to npm. Include TypeScript declarations, tree-shaking support (ESM + CJS), and a README with usage examples.

**Design:**
```json
// packages/react-sdk/package.json
{
  "name": "@embedded-analytics/react-sdk",
  "version": "0.1.0",
  "main": "./dist/cjs/index.js",
  "module": "./dist/esm/index.js",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.js",
      "types": "./dist/types/index.d.ts"
    }
  },
  "sideEffects": false,
  "peerDependencies": {
    "react": "^18.0.0 || ^19.0.0",
    "react-dom": "^18.0.0 || ^19.0.0"
  }
}
```

**Testing:**
- `npm pack` produces a tarball with correct file structure
- ESM import works: `import { Dashboard } from '@embedded-analytics/react-sdk'`
- CJS require works: `const { Dashboard } = require('@embedded-analytics/react-sdk')`
- TypeScript declarations are included and provide correct types
- Tree shaking: importing a single component does not bundle the entire library
- Peer dependency warnings appear when React is not installed
- Bundle size < 100KB gzipped (excluding peer dependencies)

### Definition of Done — Phase 3
- [ ] EmbedProvider initializes, authenticates, and provides query context
- [ ] 5 chart types (Bar, Line, Pie, KPI, Table) render correctly from query results
- [ ] Dashboard component renders widget grid with responsive layout
- [ ] Theme engine resolves cascading themes and injects CSS vars
- [ ] No vendor branding visible in any rendered output
- [ ] React SDK published to npm with TypeScript declarations and tree-shaking
- [ ] Demo app demonstrates a fully embedded dashboard in a host application
- [ ] Storybook or equivalent component documentation with live examples
- [ ] Visual regression tests for chart components

---

## Phase 4: Multi-Tenancy & Row-Level Security Hardening

**Duration:** 4 weeks
**Dependencies:** Phase 2 (query engine, RLS engine)
**Goal:** Production-grade multi-tenancy with workspace isolation, RLS policy management UI, OIDC/SAML SSO integration, and security audit.

### Task 4.1: Workspace Isolation Verification

**What:** Implement and test strict workspace isolation: every API call, query execution, and data access is scoped to the workspace identified by the embed token. Add middleware that rejects cross-workspace access attempts.

**Design:**
```typescript
// packages/server/src/middleware/workspace-isolation.ts
export function enforceWorkspaceIsolation() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const { workspaceId } = req.embedContext;
    // Verify workspace exists and belongs to the token's organization
    const workspace = await workspaceRepo.findById(workspaceId);
    if (!workspace || !workspace.isActive) {
      return res.status(403).json({ error: 'Workspace not found or inactive' });
    }
    // Attach workspace to request context for downstream use
    req.workspace = workspace;
    req.organizationId = workspace.organizationId;
    next();
  };
}
```

**Testing:**
- Token for workspace A cannot query data sources assigned to workspace B
- Token for workspace A cannot load dashboards belonging to workspace B
- Token for an inactive workspace is rejected with 403
- Token for a deleted workspace is rejected with 403
- Cross-workspace data leakage test: 10,000 random query combinations across 50 workspaces; zero cross-tenant results
- API responses never include data from other workspaces (check all list endpoints)

### Task 4.2: RLS Policy Management API

**What:** Build CRUD endpoints for RLS policies: create, update, deactivate, test. Include a policy testing endpoint that shows what filters would be applied for a given workspace without executing a query.

**Design:**
```typescript
// packages/server/src/api/routes/rls-policies.ts
router.post('/rls-policies/:policyId/test', requireAuth('admin'), async (req, res) => {
  const { policyId } = req.params;
  const { workspaceId, sampleQuery } = req.body;

  const policy = await rlsPolicyRepo.findById(policyId);
  const binding = policy.policy.bindings.find(b => b.workspace_id === workspaceId);

  const sampleCompiled = queryCompiler.compile(sampleQuery, cube.definition, 'postgresql');
  const withRLS = await rlsEngine.applyRLS(sampleCompiled, workspaceId, { [policy.policy.filter_column]: binding.filter_value }, cube.dataSourceId);

  res.json({
    policy: { name: policy.name, table: policy.policy.table_name, column: policy.policy.filter_column },
    workspace: { id: workspaceId, filterValue: binding.filter_value },
    resultingSQL: withRLS.sql,  // with parameterized RLS filters
  });
});
```

**Testing:**
- Create RLS policy with workspace bindings
- Test endpoint shows correct SQL with filters applied
- Policy deactivation stops RLS enforcement for that policy
- Policy without any workspace bindings returns a clear warning
- Multiple policies on the same table combine with AND
- Policy on a non-existent table column is rejected at creation time
- Audit trail logs all policy changes

### Task 4.3: OIDC/SAML SSO Integration

**What:** Implement SSO configuration for organizations: OIDC discovery, SAML metadata parsing, user provisioning on first login, and JWT-to-embed-token exchange.

**Design:**
```typescript
// packages/server/src/auth/oidc.ts
import { Issuer, Client } from 'openid-client';

export class OIDCProvider {
  async discover(issuerUrl: string): Promise<{ authorizationUrl: string; tokenUrl: string }> {
    const issuer = await Issuer.discover(issuerUrl);
    return {
      authorizationUrl: issuer.metadata.authorization_endpoint!,
      tokenUrl: issuer.metadata.token_endpoint!,
    };
  }

  async exchangeCode(code: string, client: Client): Promise<{ idToken: string; userInfo: UserInfo }> {
    const tokenSet = await client.callback(redirectUri, { code });
    const userInfo = await client.userinfo(tokenSet.access_token!);
    return { idToken: tokenSet.id_token!, userInfo };
  }
}
```

**Testing:**
- OIDC discovery fetches provider metadata from `.well-known/openid-configuration`
- Authorization code exchange produces valid user info
- First-time user is provisioned (created in users table + membership)
- Returning user is matched by email, not re-created
- SAML assertion parsing extracts user attributes correctly
- SSO configuration update takes effect on next login
- Invalid issuer URL returns clear error at configuration time

### Task 4.4: Security Audit & Penetration Test Preparation

**What:** Conduct internal security review: check for SQL injection in the query compiler, validate RLS enforcement under adversarial inputs, verify token security, and document the threat model.

**Testing:**
- SQL injection test suite: 50+ injection payloads against query filters, dimension names, measure names
- RLS bypass test: attempt to access data outside the workspace by manipulating filter values, token claims, and API parameters
- Token forgery test: attempt to use a token from a different organization
- Rate limiting: verify that the API rate-limits excessive requests (e.g., 1,000 req/min per token)
- CORS: verify that the API rejects requests from non-allowlisted origins
- HTTPS enforcement: verify that the API rejects HTTP connections
- Dependency audit: `npm audit` with zero high/critical vulnerabilities

### Definition of Done — Phase 4
- [ ] Workspace isolation verified with cross-tenant data leakage tests
- [ ] RLS policy CRUD API with testing endpoint
- [ ] OIDC SSO integration working with at least one provider (Auth0 or Okta)
- [ ] Security audit completed with all findings resolved
- [ ] SQL injection test suite passes (zero vulnerabilities)
- [ ] Rate limiting configured and tested
- [ ] Threat model documented

---

## Phase 5: Self-Serve Canvas

**Duration:** 4 weeks
**Dependencies:** Phase 3 (React SDK, dashboard system)
**Goal:** End-customers of the SaaS vendor can compose and save their own dashboard views within guardrails set by the vendor.

### Task 5.1: Canvas Editor Component

**What:** Build a `<CanvasEditor>` React component that provides a drag-and-drop interface for end-users to add widgets to a personal dashboard. The available widgets and data sources are constrained by the vendor's configuration.

**Design:**
```tsx
// packages/react-sdk/src/components/CanvasEditor.tsx
interface CanvasEditorProps {
  allowedWidgetTypes?: WidgetType[];
  allowedCubes?: string[];
  maxWidgets?: number;
  onSave: (layout: CanvasLayout) => Promise<void>;
}

export function CanvasEditor({ allowedWidgetTypes, allowedCubes, maxWidgets, onSave }: CanvasEditorProps) {
  const [layout, dispatch] = useReducer(canvasReducer, initialCanvasState);

  return (
    <div className="ea-canvas-editor">
      <WidgetPalette
        types={allowedWidgetTypes ?? allWidgetTypes}
        cubes={allowedCubes}
        onAdd={(widget) => dispatch({ type: 'ADD_WIDGET', widget })}
        disabled={layout.widgets.length >= (maxWidgets ?? 20)}
      />
      <ResponsiveGrid
        layout={layout.grid}
        onLayoutChange={(newGrid) => dispatch({ type: 'UPDATE_GRID', grid: newGrid })}
      >
        {layout.widgets.map(w => (
          <div key={w.id}>
            <WidgetRenderer widget={w} isEditing />
            <WidgetToolbar
              onConfigure={() => dispatch({ type: 'CONFIGURE_WIDGET', widgetId: w.id })}
              onRemove={() => dispatch({ type: 'REMOVE_WIDGET', widgetId: w.id })}
            />
          </div>
        ))}
      </ResponsiveGrid>
      <SaveBar onSave={() => onSave(layout)} />
    </div>
  );
}
```

**Testing:**
- Widget palette displays only allowed widget types
- Dragging a widget from the palette onto the canvas adds it to the grid
- Widget can be resized and repositioned within the grid
- Widget configuration panel allows changing measures, dimensions, and visual settings
- maxWidgets constraint prevents adding beyond the limit
- allowedCubes constraint limits data source selection
- Save button serializes the canvas layout and calls onSave
- Undo/redo supports at least 20 steps

### Task 5.2: Canvas Persistence API

**What:** API endpoints for saving, loading, and listing user-created canvas layouts. Canvases are scoped to a workspace + user.

**Design:**
```typescript
// packages/server/src/api/routes/canvas.ts
router.post('/canvases', requireEmbedToken('canvas:write'), async (req, res) => {
  const { workspaceId, userId } = req.embedContext;
  const body = CanvasCreateSchema.parse(req.body);
  const canvas = await canvasRepo.create({ workspaceId, userId, ...body });
  res.status(201).json(canvas);
});

router.get('/canvases', requireEmbedToken('canvas:read'), async (req, res) => {
  const { workspaceId, userId } = req.embedContext;
  const canvases = await canvasRepo.listByUser(workspaceId, userId);
  res.json({ data: canvases });
});
```

**Testing:**
- User A's canvases are not visible to User B (within the same workspace)
- Canvas layout round-trips correctly (save -> load -> identical layout)
- Canvas title and description are editable
- Canvas deletion removes the canvas and all associated widget configurations
- Maximum canvas count per user is enforced (configurable per organization)

### Task 5.3: Guardrails & Sandbox Enforcement

**What:** Ensure that self-serve canvas respects vendor-defined guardrails: allowed cubes, allowed widget types, maximum row limits, restricted dimensions, and query rate limits per user.

**Testing:**
- Canvas widget referencing a non-allowed cube is rejected at save time
- Canvas widget referencing a restricted dimension is rejected
- Canvas query exceeding the row limit is truncated with a warning
- Canvas creation beyond the per-user limit returns 429
- Guardrails are configured per organization, not per workspace (vendor controls them)

### Definition of Done — Phase 5
- [ ] Canvas editor component with drag-and-drop widget placement
- [ ] Widget configuration panel for measures, dimensions, and visual settings
- [ ] Canvas persistence API with user scoping
- [ ] Vendor guardrails enforced (allowed cubes, widget types, row limits)
- [ ] Undo/redo for canvas editing
- [ ] Canvas responsive rendering on mobile viewports
- [ ] Demo: end-user creates a custom dashboard within a host SaaS application

---

## Phase 6: AI & Natural Language Query

**Duration:** 5 weeks
**Dependencies:** Phase 2 (semantic layer, query engine), Phase 3 (React SDK)
**Goal:** LLM-powered "chat with your data" embedded component. End-users type natural language questions; the system translates them to semantic layer queries and renders results.

### Task 6.1: NL-to-Semantic Query Translator

**What:** Build a service that takes a natural language question, the available semantic layer definitions (cubes, measures, dimensions), and translates the question into a structured QueryRequest using an LLM (Claude API).

**Design:**
```typescript
// packages/server/src/ai/nl-query-translator.ts
export class NLQueryTranslator {
  constructor(private anthropic: Anthropic) {}

  async translate(
    question: string,
    availableCubes: CubeDefinition[],
    conversationHistory: NLMessage[]
  ): Promise<{ queryRequest: QueryRequest; explanation: string }> {
    const systemPrompt = this.buildSystemPrompt(availableCubes);
    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      system: systemPrompt,
      messages: [
        ...conversationHistory.map(m => ({ role: m.role, content: m.content })),
        { role: 'user', content: question },
      ],
      // Structured output via tool_use
      tools: [{
        name: 'execute_query',
        description: 'Execute an analytics query',
        input_schema: QueryRequestToolSchema,
      }],
    });

    const toolUse = response.content.find(c => c.type === 'tool_use');
    return {
      queryRequest: toolUse.input as QueryRequest,
      explanation: response.content.find(c => c.type === 'text')?.text ?? '',
    };
  }

  private buildSystemPrompt(cubes: CubeDefinition[]): string {
    return `You are an analytics assistant. The user's data is organized into the following cubes:\n\n${
      cubes.map(c => `Cube "${c.name}": measures=[${c.measures.map(m => m.name).join(', ')}], dimensions=[${c.dimensions.map(d => d.name).join(', ')}]`).join('\n')
    }\n\nTranslate the user's question into a query using the execute_query tool. Always explain what you're doing.`;
  }
}
```

**Testing:**
- "How many orders last month?" translates to `{ cube: 'orders', measures: ['order_count'], timeDimension: { dimension: 'created_date', granularity: 'month' } }`
- "Show revenue by product category" translates to `{ cube: 'orders', measures: ['total_revenue'], dimensions: ['product_category'] }`
- Ambiguous questions prompt the LLM to ask for clarification (not guess)
- Questions referencing non-existent measures/dimensions return a clear explanation
- Multi-turn conversation maintains context (follow-up questions work)
- Conversation history is stored in `nl_query_sessions` and `nl_query_messages`
- Token usage is tracked per organization for billing purposes

### Task 6.2: NL Query Chat Component

**What:** Build a `<NLQueryChat>` React component: a chat interface where users type questions, see the generated query explanation, and view chart results inline.

**Design:**
```tsx
// packages/react-sdk/src/components/NLQueryChat.tsx
export function NLQueryChat({ cubes }: { cubes?: string[] }) {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const { apiUrl, token } = useEmbed();

  const handleSubmit = async (question: string) => {
    setMessages(prev => [...prev, { role: 'user', content: question }]);
    const response = await fetch(`${apiUrl}/nl-query`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({ question, sessionId, cubes }),
    });
    const { explanation, queryResult, suggestedChartType } = await response.json();
    setMessages(prev => [...prev, {
      role: 'assistant',
      content: explanation,
      chart: { type: suggestedChartType, data: queryResult.data },
    }]);
  };

  return (
    <div className="ea-nl-chat">
      <MessageList messages={messages} />
      <ChatInput onSubmit={handleSubmit} placeholder="Ask a question about your data..." />
    </div>
  );
}
```

**Testing:**
- User types a question and receives a text explanation + chart
- Chart type is auto-selected based on query shape (time series -> line chart, categories -> bar chart)
- Follow-up questions maintain session context
- Error from LLM displays a user-friendly message, not a stack trace
- Rate limiting prevents excessive LLM API calls (configurable per org)
- Chat history persists across page refreshes (session-based)

### Task 6.3: AI Dashboard Suggestions

**What:** Given a customer's data schema (from data source sync), use an LLM to suggest an initial dashboard layout with relevant widgets, measures, and dimensions.

**Design:**
```typescript
// packages/server/src/ai/dashboard-suggester.ts
export class DashboardSuggester {
  async suggest(
    schemaCache: DataSourceSchemaCache,
    existingCubes: CubeDefinition[]
  ): Promise<DashboardSuggestion> {
    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      system: 'You are an analytics dashboard designer. Given a database schema and semantic layer definitions, suggest a dashboard layout with the most useful widgets for a business user.',
      messages: [{ role: 'user', content: JSON.stringify({ schema: schemaCache, cubes: existingCubes }) }],
      tools: [{ name: 'suggest_dashboard', input_schema: DashboardSuggestionSchema }],
    });
    return parseSuggestion(response);
  }
}
```

**Testing:**
- E-commerce schema (orders, products, customers) produces relevant dashboard (revenue KPI, orders over time, top products)
- SaaS schema (users, subscriptions, events) produces relevant dashboard (MRR, churn rate, active users)
- Suggestion includes widget types, grid positions, measures, and dimensions
- Suggestion respects available cubes (does not invent non-existent measures)
- User can accept, modify, or reject the suggestion

### Definition of Done — Phase 6
- [ ] NL-to-semantic query translation using Claude API with structured output
- [ ] Multi-turn conversation with session persistence
- [ ] Chat component renders inline charts with auto-selected chart types
- [ ] AI dashboard suggestion generates relevant layouts from schema
- [ ] LLM token usage tracked per organization
- [ ] Rate limiting on AI endpoints
- [ ] Feedback mechanism (thumbs up/down) stored for model improvement

---

## Phase 7: ClickHouse Analytics Engine

**Duration:** 4 weeks
**Dependencies:** Phase 1 (PostgreSQL schema), Phase 2 (query execution logging)
**Goal:** Offload high-volume query execution logs, audit events, and usage analytics to ClickHouse. Enable sub-second performance dashboards and cross-tenant benchmarking.

### Task 7.1: ClickHouse Schema Deployment

**What:** Deploy the ClickHouse analytics schema (from Data Model Suggestion 4): `query_executions`, `dashboard_views`, `audit_events`, `nl_query_events` tables with MergeTree engines, monthly partitioning, TTL, and materialized views.

**Design:**
```sql
-- ClickHouse: query_executions with MergeTree and 365-day TTL
CREATE TABLE query_executions (
    event_id UUID,
    organization_id UUID,
    workspace_id UUID,
    cube_name LowCardinality(String),
    duration_ms UInt32,
    cache_hit UInt8,
    executed_at DateTime64(3, 'UTC'),
    event_date Date DEFAULT toDate(executed_at)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (organization_id, workspace_id, executed_at)
TTL event_date + INTERVAL 365 DAY;
```

**Testing:**
- All 4 ClickHouse tables create successfully
- 3 materialized views (hourly query stats, daily dashboard popularity, NL query success) create successfully
- TTL: insert a record with an old date; verify it is dropped after merge
- Partition pruning: query for a specific month touches only that partition
- LowCardinality columns reduce storage by >50% for enum-like fields

### Task 7.2: CDC Pipeline (PostgreSQL -> ClickHouse)

**What:** Implement a CDC (Change Data Capture) pipeline that reads query execution events from PostgreSQL and inserts them into ClickHouse. Start with the outbox pattern (polling `ch_sync_outbox`); document the Debezium upgrade path.

**Design:**
```typescript
// packages/server/src/sync/clickhouse-sync-worker.ts
export class ClickHouseSyncWorker {
  async processBatch(batchSize: number = 1000): Promise<number> {
    const events = await this.db.chSyncOutbox.findUnsynced(batchSize);
    if (events.length === 0) return 0;

    const grouped = groupBy(events, 'targetTable');
    for (const [table, rows] of Object.entries(grouped)) {
      await this.clickhouse.insert({
        table,
        values: rows.map(r => r.payload),
        format: 'JSONEachRow',
      });
    }

    await this.db.chSyncOutbox.markSynced(events.map(e => e.id));
    return events.length;
  }
}
```

**Testing:**
- Query execution in PostgreSQL is written to outbox; sync worker inserts into ClickHouse
- Batch processing: 10,000 events are synced in <5 seconds
- Idempotency: duplicate events do not create duplicate ClickHouse rows
- Worker crash recovery: unsynced events are picked up on restart
- ClickHouse downtime: outbox grows without data loss; catch-up syncs on recovery
- Materialized views update correctly as new data arrives

### Task 7.3: Performance Dashboard API

**What:** Expose API endpoints that query ClickHouse materialized views for performance dashboards: query latency P95, cache hit rates, error rates, dashboard popularity.

**Testing:**
- /analytics/query-performance returns hourly P95 latency for the last 7 days
- /analytics/dashboard-popularity returns top 10 dashboards by view count
- /analytics/cache-hit-rate returns cache hit percentage over time
- Queries against materialized views complete in <100ms for 1M+ events
- Endpoints are scoped to the authenticated organization (no cross-tenant data)

### Definition of Done — Phase 7
- [ ] ClickHouse schema deployed with 4 tables and 3 materialized views
- [ ] CDC pipeline syncs events from PostgreSQL outbox to ClickHouse
- [ ] Performance dashboard API endpoints return correct analytics
- [ ] Materialized views provide sub-100ms query performance
- [ ] TTL auto-expires old data (verified)
- [ ] Debezium upgrade path documented

---

## Phase 8: SOC 2 & Enterprise Governance

**Duration:** 4 weeks
**Dependencies:** Phase 4 (security audit, RLS hardening)
**Goal:** Achieve SOC 2 Type II readiness. Implement audit log export, data retention policies, and enterprise governance features.

### Task 8.1: Audit Log Export API

**What:** API endpoint for exporting audit events in OCSF-compatible JSON format. Support filtering by date range, action, resource type, and actor.

**Testing:**
- Export returns OCSF-compatible JSON with correct field mapping
- Date range filter returns only events within the range
- Action filter (e.g., `dashboard.created`) returns only matching events
- Export pagination handles >100,000 events without timeout
- Export is scoped to the authenticated organization

### Task 8.2: Data Retention Policies

**What:** Configurable per-organization data retention: auto-delete query execution logs, NL query history, and audit events after a configurable period (default: 365 days for queries, 730 days for audit).

**Testing:**
- Retention policy deletes expired records on schedule
- Retention policy respects per-organization configuration
- Audit events with `severity: critical` are exempt from retention (legal hold)
- Retention job runs without impacting API performance

### Task 8.3: Analytics-as-Code Export/Import

**What:** Export and import semantic layer definitions, dashboard layouts, and RLS policies as version-controlled YAML/JSON files. Enables GitOps workflows for analytics configuration.

**Design:**
```typescript
// packages/cli/src/commands/export.ts
export async function exportAnalyticsConfig(orgId: string, outputDir: string) {
  const cubes = await api.semanticCubes.list(orgId);
  const dashboards = await api.dashboards.list(orgId);
  const rlsPolicies = await api.rlsPolicies.list(orgId);

  await writeYAML(`${outputDir}/cubes/`, cubes);
  await writeYAML(`${outputDir}/dashboards/`, dashboards);
  await writeYAML(`${outputDir}/rls-policies/`, rlsPolicies);
}
```

**Testing:**
- Export produces valid YAML files for cubes, dashboards, and RLS policies
- Import creates or updates entities from YAML files
- Import is idempotent: running import twice produces the same state
- Import validates YAML against schemas before applying changes
- Diff command shows what would change without applying

### Definition of Done — Phase 8
- [ ] Audit log export API in OCSF-compatible format
- [ ] Data retention policies configurable per organization
- [ ] Analytics-as-Code CLI export/import with YAML
- [ ] SOC 2 Type II evidence collection documented (access controls, audit trails, encryption)
- [ ] Enterprise governance: API key management, IP allowlisting
- [ ] Penetration test completed by external firm (or preparation checklist)

---

## Phase 9: Agentic Analytics & Advanced AI

**Duration:** 5 weeks
**Dependencies:** Phase 6 (AI/NL query), Phase 7 (ClickHouse analytics)
**Goal:** AI agents that proactively surface insights, detect anomalies, and trigger actions in the host SaaS application.

### Task 9.1: Anomaly Detection Engine

**What:** Analyze query result trends using statistical methods (z-score, IQR) and/or LLM reasoning to detect anomalies in time-series data. Surface anomalies as alerts in embedded dashboards.

**Testing:**
- Revenue drop of >20% from moving average triggers an anomaly alert
- Anomaly alerts include a natural language explanation
- False positive rate < 5% on synthetic test data with known anomalies
- Anomaly detection runs asynchronously without impacting dashboard load time

### Task 9.2: Proactive Insight Surfacing

**What:** Based on user role, recent activity, and data patterns, proactively push relevant insights to users (e.g., "Your conversion rate increased 15% this week" as a notification in the embedded dashboard).

**Testing:**
- Insights are relevant to the user's most-viewed dashboards
- Insights refresh daily based on latest data
- Users can dismiss insights (dismissed insights don't reappear)
- Insights are scoped to the user's RLS context (no data leakage)

### Task 9.3: Webhook Actions

**What:** Allow SaaS vendors to configure webhook actions triggered by analytics events (anomalies, thresholds, scheduled reports). The SDK sends HTTP POST requests to vendor-configured URLs with event payloads.

**Testing:**
- Threshold breach triggers webhook with correct payload
- Webhook retries on failure (3 attempts with exponential backoff)
- Webhook payload includes the analytics data that triggered the action
- Webhook signatures (HMAC-SHA256) are verified by the receiving endpoint
- Webhook configuration CRUD API with URL validation

### Definition of Done — Phase 9
- [ ] Anomaly detection identifies meaningful deviations in time-series data
- [ ] Proactive insights surface role-relevant information to embedded dashboard users
- [ ] Webhook actions trigger on configurable analytics events
- [ ] All AI features respect RLS and workspace isolation
- [ ] AI features are toggleable per organization (feature flags)

---

## Phase 10: Managed Cloud & Monetization

**Duration:** 5 weeks
**Dependencies:** Phase 1 (core infrastructure), can run parallel to Phases 6-9
**Goal:** Hosted managed cloud service with self-service onboarding, usage-based billing, and customer portal.

### Task 10.1: Multi-Tenant Cloud Infrastructure

**What:** Deploy the SDK server as a multi-tenant cloud service (AWS/GCP). Each organization shares the infrastructure but has isolated data. Terraform/Pulumi for infrastructure-as-code.

**Testing:**
- New organization provisioning completes in <30 seconds
- Organization data is isolated (database row-level, not schema-level)
- Auto-scaling handles 10x traffic spike within 5 minutes
- Health check endpoints return correct status for all dependencies

### Task 10.2: Usage-Based Billing Integration

**What:** Integrate with Stripe for usage-based billing. Track Monthly Active Viewers (MAV), query volume, and AI token usage. Generate invoices based on plan + overages.

**Testing:**
- MAV count accurately reflects unique embed token users per month
- Query volume count matches `query_executions` table count
- Stripe webhook processes subscription events (upgrade, downgrade, cancel)
- Invoice line items match usage metrics
- Free tier enforces limits (e.g., 1,000 queries/month, 1 data source)

### Task 10.3: Customer Portal & Documentation Site

**What:** Self-service portal for organization admins: API key management, usage dashboards, billing history, and documentation. Documentation site with guides, API reference, and interactive examples.

**Testing:**
- API key creation, rotation, and revocation
- Usage dashboard shows MAV, queries, and costs
- Documentation site builds and deploys from markdown
- Interactive API examples execute against a sandbox environment
- Onboarding wizard guides new users through: create org -> add data source -> create first cube -> embed first dashboard

### Definition of Done — Phase 10
- [ ] Managed cloud deployed with multi-tenant isolation
- [ ] Self-service onboarding: sign up -> embed first dashboard in <30 minutes
- [ ] Usage-based billing via Stripe with MAV and query tracking
- [ ] Customer portal with API key management and usage dashboards
- [ ] Documentation site with guides, API reference, and interactive examples
- [ ] Free tier with enforced limits
- [ ] 99.9% uptime SLA target with monitoring and alerting

---

## Phase 11: Accessibility, Web Components & Benchmarking

**Duration:** 4 weeks
**Dependencies:** Phase 3 (React SDK), Phase 7 (ClickHouse analytics)
**Goal:** WCAG 2.1 AA compliance, framework-agnostic Web Component SDK, and cross-tenant anonymized benchmark analytics.

### Task 11.1: WCAG 2.1 AA Compliance

**What:** Audit all React SDK components for accessibility. Implement keyboard navigation, screen reader support, ARIA labels, color contrast compliance, and focus management.

**Testing:**
- axe-core automated audit passes with zero violations on all chart components
- Keyboard-only navigation through dashboard (Tab, Enter, Escape, Arrow keys)
- Screen reader announces chart data, widget titles, and filter values
- Color contrast ratio >= 4.5:1 for all text elements
- Focus trap in modal dialogs (widget configuration, NL query chat)

### Task 11.2: Web Component SDK

**What:** Wrap the React SDK components as Web Components using Lit or Stencil. Enable embedding in Vue, Angular, Svelte, and vanilla JS applications.

**Testing:**
- `<ea-dashboard dashboard-id="..." token="...">` renders in plain HTML
- Web Component works in Vue 3 application
- Web Component works in Angular 17 application
- Shadow DOM encapsulates styles (no CSS leakage in/out)
- Attributes and properties are correctly mapped
- Events bubble correctly to the host application

### Task 11.3: Cross-Tenant Anonymized Benchmarks

**What:** Generate anonymized peer comparison metrics using ClickHouse. SaaS vendors can show customers how their metrics compare to anonymized cohort averages (e.g., "Your query latency is better than 80% of similar workspaces").

**Testing:**
- Benchmark data is aggregated and anonymized (no individual workspace data exposed)
- Minimum cohort size (e.g., 10 workspaces) required before benchmarks are shown
- Benchmark metrics are correctly calculated (percentile ranking)
- Benchmarks are opt-in per organization
- Benchmark API endpoint returns data in <200ms

### Definition of Done — Phase 11
- [ ] WCAG 2.1 AA compliance verified by automated audit and manual testing
- [ ] Web Component SDK published to npm with framework integration examples
- [ ] Cross-tenant benchmarks with anonymization and minimum cohort enforcement
- [ ] Accessibility documentation for end-user-facing components
- [ ] Web Component bundle size < 150KB gzipped

---

## Summary Timeline

| Phase | Name | Duration | Cumulative | Key Deliverable |
|-------|------|----------|------------|-----------------|
| 1 | Foundation & Core Infrastructure | 4 weeks | Month 1 | Monorepo, DB schema, auth, CRUD API |
| 2 | Query Engine & Semantic Layer | 5 weeks | Month 2-3 | Warehouse queries, caching, RLS |
| 3 | Dashboard & Widget System | 5 weeks | Month 3-4 | React SDK on npm, chart components |
| 4 | Multi-Tenancy & RLS Hardening | 4 weeks | Month 4-5 | Production security, SSO |
| 5 | Self-Serve Canvas | 4 weeks | Month 5-6 | End-user dashboard builder |
| 6 | AI & Natural Language Query | 5 weeks | Month 6-7 | Chat with your data, AI suggestions |
| 7 | ClickHouse Analytics Engine | 4 weeks | Month 7-8 | Performance dashboards, CDC |
| 8 | SOC 2 & Enterprise Governance | 4 weeks | Month 8-9 | Audit export, retention, GitOps |
| 9 | Agentic Analytics & Advanced AI | 5 weeks | Month 9-10 | Anomaly detection, webhooks |
| 10 | Managed Cloud & Monetization | 5 weeks | Month 10-12 | SaaS billing, customer portal |
| 11 | Accessibility & Benchmarking | 4 weeks | Month 12-13 | WCAG 2.1, Web Components, benchmarks |

**MVP (Phases 1-3):** ~14 weeks / 3.5 months — a working embedded analytics React SDK with multi-tenant query execution, caching, and white-label theming.

**v1.0 (Phases 1-5):** ~22 weeks / 5.5 months — adds production security, SSO, and self-serve canvas.

**Full Platform (Phases 1-11):** ~49 weeks / 12-13 months — complete platform with AI, scale, compliance, and monetization.
