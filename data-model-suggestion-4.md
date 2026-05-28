# Data Model Suggestion 4: Dual-Engine (PostgreSQL + ClickHouse)

> Project: Embedded Analytics SDK · Created: 2026-05-11

## Philosophy

This model recognizes that an embedded analytics platform has two fundamentally different data access patterns that a single database engine cannot optimally serve: (1) **OLTP operations** — CRUD management of workspaces, dashboards, users, permissions, and configuration with strong consistency and transactional guarantees; and (2) **OLAP operations** — high-throughput analytical query execution, query performance monitoring, usage analytics, and time-series aggregation where columnar storage and vectorized execution dominate.

Rather than forcing PostgreSQL to serve both roles (acceptable for small-to-medium scale) or building complex materialized view pipelines, this architecture uses **PostgreSQL for the application/operational layer** and **ClickHouse for the analytics/observability layer**. Data flows from PostgreSQL to ClickHouse via CDC (Change Data Capture) or batch ETL for operational metrics, while ClickHouse handles the heavy analytical workloads directly.

This pattern mirrors how production analytics platforms like Qrvey (Elasticsearch-backed native data lake), Metabase (application DB + customer warehouse), and Cube.js (semantic layer + pre-aggregation cache) separate operational state from analytical query execution. ClickHouse is the preferred choice for the analytics engine because of its columnar storage, vectorized query execution, and native support for approximate aggregation functions — all critical for sub-second dashboard rendering.

**Best for:** Teams building for scale from day one, where query performance SLAs (sub-second dashboard loads), usage analytics, and high-volume query execution logging are core requirements — not just MVP features.

**Trade-offs:**
- (+) Each engine optimized for its workload — OLTP transactions in PostgreSQL, OLAP analytics in ClickHouse
- (+) ClickHouse delivers 10-100x faster analytical queries than PostgreSQL for large datasets
- (+) Query execution logging scales to millions of events per day without impacting application performance
- (+) Built-in time-series capabilities in ClickHouse (TTL, partitioning, materialized views)
- (+) Usage analytics and performance monitoring are first-class capabilities, not afterthoughts
- (-) Two databases to deploy, monitor, backup, and maintain — higher operational complexity
- (-) Data synchronization between PostgreSQL and ClickHouse introduces latency and potential consistency issues
- (-) Team needs expertise in both PostgreSQL and ClickHouse
- (-) Higher infrastructure cost than a single PostgreSQL instance
- (-) CDC pipeline adds another component to the system (Debezium, pg_logical, or custom)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OAuth 2.0 / OIDC | SSO configuration in PostgreSQL operational layer |
| JWT (RFC 7519) | Embed tokens managed in PostgreSQL with RBAC enforcement |
| ISO 3166-1/2 | Jurisdiction codes in workspace settings |
| Cube.js Semantic Layer | Semantic definitions in PostgreSQL; pre-aggregations materialized in ClickHouse |
| OCSF | Audit events and query logs in ClickHouse follow OCSF event structure |
| OpenTelemetry | Query execution spans exported to ClickHouse for observability |
| ClickHouse MergeTree | Time-series data uses ReplacingMergeTree and AggregatingMergeTree engines |

---

## PostgreSQL: Operational Layer

The PostgreSQL schema is a streamlined hybrid (relational + JSONB) focused purely on OLTP operations. No analytical query logs, no usage metrics — those go to ClickHouse.

### Core Platform

```sql
-- Organizations (SaaS vendors)
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'starter', 'pro', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Workspaces (end-customer tenants)
CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_workspaces_org ON workspaces(organization_id);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    preferences     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Memberships (org + workspace)
CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    scope_type      TEXT NOT NULL CHECK (scope_type IN ('organization', 'workspace')),
    scope_id        UUID NOT NULL,
    role            TEXT NOT NULL DEFAULT 'viewer',
    permissions     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, scope_type, scope_id)
);

CREATE INDEX idx_memberships_user ON memberships(user_id);
CREATE INDEX idx_memberships_scope ON memberships(scope_type, scope_id);
```

### Data Sources & Semantic Layer

```sql
CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    engine          TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    connection      JSONB NOT NULL,              -- encrypted credentials
    schema_cache    JSONB DEFAULT '{}',          -- cached table/column metadata
    workspace_ids   UUID[] NOT NULL DEFAULT '{}',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_org ON data_sources(organization_id);

CREATE TABLE semantic_cubes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    sql_table       TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    version         INTEGER NOT NULL DEFAULT 1,
    definition      JSONB NOT NULL,              -- measures, dimensions, joins
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE INDEX idx_semantic_cubes_org ON semantic_cubes(organization_id);
```

### Dashboards & Widgets

```sql
CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES collections(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collections_ws ON collections(workspace_id);

CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    collection_id   UUID REFERENCES collections(id) ON DELETE SET NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    is_template     BOOLEAN NOT NULL DEFAULT false,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID NOT NULL REFERENCES users(id),
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dashboards_ws ON dashboards(workspace_id);

CREATE TABLE widgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dashboard_id    UUID NOT NULL REFERENCES dashboards(id) ON DELETE CASCADE,
    widget_type     TEXT NOT NULL,
    title           TEXT,
    grid_x          INTEGER NOT NULL DEFAULT 0,
    grid_y          INTEGER NOT NULL DEFAULT 0,
    grid_w          INTEGER NOT NULL DEFAULT 6,
    grid_h          INTEGER NOT NULL DEFAULT 4,
    query_config    JSONB NOT NULL DEFAULT '{}',
    visual_config   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_widgets_dashboard ON widgets(dashboard_id);
```

### RLS, SSO & Embed Tokens

```sql
CREATE TABLE rls_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    policy          JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rls_org ON rls_policies(organization_id);

CREATE TABLE sso_configurations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    provider_type   TEXT NOT NULL CHECK (provider_type IN ('oidc', 'saml')),
    issuer_url      TEXT NOT NULL,
    client_id       TEXT NOT NULL,
    client_secret_encrypted TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, provider_type)
);

CREATE TABLE embed_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    token_hash      TEXT NOT NULL UNIQUE,
    claims          JSONB NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embed_tokens_ws ON embed_tokens(workspace_id);
CREATE INDEX idx_embed_tokens_expires ON embed_tokens(expires_at);
```

---

## ClickHouse: Analytics & Observability Layer

All high-volume, time-series, and analytical data lives in ClickHouse. These tables are written to by the application and by CDC from PostgreSQL.

### Query Execution Log

```sql
-- ClickHouse: Every query executed through the SDK
-- Engine: MergeTree partitioned by month for time-based queries
CREATE TABLE query_executions (
    event_id        UUID,
    organization_id UUID,
    workspace_id    UUID,
    user_id         Nullable(UUID),
    widget_id       Nullable(UUID),
    dashboard_id    Nullable(UUID),
    cube_name       LowCardinality(String),
    data_source_id  UUID,

    -- Query details
    measures        Array(String),
    dimensions      Array(String),
    filters         String,                        -- JSON string of filter conditions
    generated_sql   String,
    rls_applied     UInt8,                         -- 0/1 boolean
    rls_filter      String,

    -- Performance metrics
    duration_ms     UInt32,
    row_count       UInt32,
    bytes_scanned   UInt64,
    cache_hit       UInt8,                         -- 0/1 boolean
    error_code      LowCardinality(String),
    error_message   String,

    -- Timestamp
    executed_at     DateTime64(3, 'UTC'),

    -- Partitioning
    event_date      Date DEFAULT toDate(executed_at)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (organization_id, workspace_id, executed_at)
TTL event_date + INTERVAL 365 DAY;

-- Fast lookups by workspace + time range
-- (ORDER BY already covers organization_id, workspace_id, executed_at)
```

### Dashboard View Events

```sql
-- ClickHouse: Tracks every dashboard load/view by end-users
CREATE TABLE dashboard_views (
    event_id        UUID,
    organization_id UUID,
    workspace_id    UUID,
    user_id         Nullable(UUID),
    dashboard_id    UUID,
    embed_token_id  Nullable(UUID),

    -- View metadata
    view_duration_ms UInt32,                       -- how long the user viewed the dashboard
    widget_count    UInt8,
    interaction_count UInt16,                      -- clicks, filters, drill-downs
    source          LowCardinality(String),        -- 'embed', 'direct', 'api'
    user_agent      String,
    ip_country      LowCardinality(String),        -- ISO 3166-1 alpha-2

    -- Timestamp
    viewed_at       DateTime64(3, 'UTC'),
    event_date      Date DEFAULT toDate(viewed_at)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (organization_id, workspace_id, dashboard_id, viewed_at)
TTL event_date + INTERVAL 365 DAY;
```

### Audit Events

```sql
-- ClickHouse: OCSF-aligned audit trail (high-volume, append-only)
CREATE TABLE audit_events (
    event_id        UUID,
    organization_id UUID,
    workspace_id    Nullable(UUID),
    actor_id        Nullable(UUID),
    actor_type      LowCardinality(String),        -- 'user', 'system', 'api_key'

    -- OCSF-inspired fields
    action          LowCardinality(String),        -- 'dashboard.created', 'query.executed'
    resource_type   LowCardinality(String),        -- 'dashboard', 'data_source'
    resource_id     Nullable(UUID),
    outcome         LowCardinality(String),        -- 'success', 'failure', 'error'
    severity        LowCardinality(String),        -- 'info', 'low', 'medium', 'high', 'critical'

    -- Context
    ip_address      IPv4,
    user_agent      String,
    details         String,                        -- JSON string with event-specific details

    -- Timestamp
    created_at      DateTime64(3, 'UTC'),
    event_date      Date DEFAULT toDate(created_at)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (organization_id, action, created_at)
TTL event_date + INTERVAL 730 DAY;                 -- 2-year retention for audit
```

### NL Query Analytics

```sql
-- ClickHouse: Natural language query tracking for AI improvement
CREATE TABLE nl_query_events (
    event_id        UUID,
    organization_id UUID,
    workspace_id    UUID,
    user_id         UUID,
    session_id      UUID,

    -- Query details
    user_question   String,
    generated_sql   String,
    cube_used       LowCardinality(String),
    measures_used   Array(String),
    dimensions_used Array(String),

    -- Outcome
    execution_success UInt8,
    duration_ms     UInt32,
    row_count       UInt32,
    user_feedback   Nullable(Int8),                -- -1, 0, +1 thumbs down/neutral/up

    -- Timestamp
    asked_at        DateTime64(3, 'UTC'),
    event_date      Date DEFAULT toDate(asked_at)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (organization_id, workspace_id, asked_at)
TTL event_date + INTERVAL 365 DAY;
```

### Pre-Aggregated Materialized Views (ClickHouse)

```sql
-- ClickHouse Materialized View: Hourly query stats per workspace
CREATE MATERIALIZED VIEW query_stats_hourly_mv
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (organization_id, workspace_id, data_source_id, hour)
AS
SELECT
    organization_id,
    workspace_id,
    data_source_id,
    toStartOfHour(executed_at) AS hour,
    countState() AS total_queries,
    countIfState(cache_hit = 1) AS cache_hits,
    avgState(duration_ms) AS avg_duration_ms,
    quantileState(0.95)(duration_ms) AS p95_duration_ms,
    countIfState(error_code != '') AS error_count
FROM query_executions
GROUP BY organization_id, workspace_id, data_source_id, hour;

-- ClickHouse Materialized View: Daily dashboard popularity
CREATE MATERIALIZED VIEW dashboard_popularity_daily_mv
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(day)
ORDER BY (organization_id, workspace_id, dashboard_id, day)
AS
SELECT
    organization_id,
    workspace_id,
    dashboard_id,
    toDate(viewed_at) AS day,
    countState() AS view_count,
    uniqState(user_id) AS unique_viewers,
    avgState(view_duration_ms) AS avg_view_duration_ms,
    sumState(interaction_count) AS total_interactions
FROM dashboard_views
GROUP BY organization_id, workspace_id, dashboard_id, day;

-- ClickHouse Materialized View: NL query success rate
CREATE MATERIALIZED VIEW nl_query_success_daily_mv
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(day)
ORDER BY (organization_id, day)
AS
SELECT
    organization_id,
    toDate(asked_at) AS day,
    countState() AS total_questions,
    countIfState(execution_success = 1) AS successful_queries,
    avgState(duration_ms) AS avg_duration_ms,
    countIfState(user_feedback = 1) AS positive_feedback,
    countIfState(user_feedback = -1) AS negative_feedback
FROM nl_query_events
GROUP BY organization_id, day;
```

---

## Data Flow: PostgreSQL to ClickHouse

```sql
-- PostgreSQL: CDC outbox table for ClickHouse sync
-- Application writes here; a CDC worker reads and forwards to ClickHouse
CREATE TABLE ch_sync_outbox (
    id              BIGSERIAL PRIMARY KEY,
    target_table    TEXT NOT NULL,                  -- ClickHouse destination table
    payload         JSONB NOT NULL,                 -- row data to insert
    synced          BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ch_sync_unsynced ON ch_sync_outbox(synced, created_at)
    WHERE synced = false;

-- Alternative: Use Debezium CDC connector to stream PostgreSQL WAL
-- directly to ClickHouse via Kafka. The outbox approach is simpler
-- for MVP; Debezium is recommended for production scale.
```

---

## Example: ClickHouse Query — Dashboard Performance Report

```sql
-- Query ClickHouse: Average query performance by dashboard for the last 7 days
SELECT
    dashboard_id,
    count() AS total_queries,
    avg(duration_ms) AS avg_duration_ms,
    quantile(0.95)(duration_ms) AS p95_duration_ms,
    sum(cache_hit) / count() * 100 AS cache_hit_rate_pct,
    countIf(error_code != '') AS error_count
FROM query_executions
WHERE organization_id = 'org-uuid-here'
  AND executed_at >= now() - INTERVAL 7 DAY
GROUP BY dashboard_id
ORDER BY total_queries DESC
LIMIT 20;
```

## Example: ClickHouse Query — Cross-Tenant Benchmark Analytics

```sql
-- Anonymized peer comparison: how does workspace X compare to peers?
WITH workspace_stats AS (
    SELECT
        workspace_id,
        count() AS query_count,
        avg(duration_ms) AS avg_duration,
        sum(cache_hit) / count() * 100 AS cache_rate
    FROM query_executions
    WHERE organization_id = 'org-uuid-here'
      AND executed_at >= now() - INTERVAL 30 DAY
    GROUP BY workspace_id
)
SELECT
    CASE WHEN workspace_id = 'target-ws-uuid' THEN 'You' ELSE 'Peer Average' END AS label,
    CASE
        WHEN workspace_id = 'target-ws-uuid' THEN query_count
        ELSE avg(query_count)
    END AS query_count,
    CASE
        WHEN workspace_id = 'target-ws-uuid' THEN avg_duration
        ELSE avg(avg_duration)
    END AS avg_query_ms,
    CASE
        WHEN workspace_id = 'target-ws-uuid' THEN cache_rate
        ELSE avg(cache_rate)
    END AS cache_hit_pct
FROM workspace_stats
GROUP BY label, workspace_id, query_count, avg_duration, cache_rate
HAVING workspace_id = 'target-ws-uuid'
   OR workspace_id != 'target-ws-uuid';
```

---

## Table Count Summary

### PostgreSQL (Operational)

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Workspaces | 2 | Core tenant hierarchy |
| Users & Memberships | 2 | Accounts + unified membership |
| Data Sources & Semantic Layer | 2 | Connections + cube definitions |
| Dashboards & Widgets | 3 | Collections, dashboards, widgets |
| RLS & Security | 3 | RLS policies, SSO, embed tokens |
| CDC Outbox | 1 | Sync bridge to ClickHouse |
| **PostgreSQL Total** | **13** | Lean operational layer |

### ClickHouse (Analytics)

| Category | Tables | Notes |
|----------|--------|-------|
| Query Execution Log | 1 | High-volume query events |
| Dashboard Views | 1 | End-user view tracking |
| Audit Events | 1 | OCSF-aligned audit trail |
| NL Query Analytics | 1 | AI improvement data |
| Materialized Views | 3 | Pre-aggregated hourly/daily stats |
| **ClickHouse Total** | **7** | (4 base + 3 materialized views) |

### Combined Total

| Engine | Tables | Materialized Views | Total Objects |
|--------|--------|--------------------|---------------|
| PostgreSQL | 13 | 0 | 13 |
| ClickHouse | 4 | 3 | 7 |
| **Combined** | **17** | **3** | **20** |

---

## Key Design Decisions

1. **Clear separation of concerns: PostgreSQL for state, ClickHouse for events** — any data that is read-modify-write (CRUD) lives in PostgreSQL. Any data that is append-only and queried analytically lives in ClickHouse. This eliminates the tension between transactional consistency and analytical performance.

2. **ClickHouse MergeTree with monthly partitioning** — all ClickHouse tables are partitioned by `toYYYYMM(event_date)`, enabling efficient time-range queries and automatic data expiration via TTL.

3. **LowCardinality for string enums in ClickHouse** — fields like `action`, `resource_type`, `outcome`, `severity`, and `engine` use `LowCardinality(String)` for dictionary encoding, reducing storage by 80-90% for these columns.

4. **AggregatingMergeTree materialized views for real-time dashboards** — rather than running expensive GROUP BY queries on raw event data, materialized views pre-aggregate hourly and daily metrics incrementally as new data arrives.

5. **TTL-based data lifecycle** — query execution logs retain 365 days; audit events retain 730 days (2 years). ClickHouse TTL automatically drops expired partitions, eliminating manual cleanup.

6. **CDC outbox for MVP, Debezium for production** — the `ch_sync_outbox` table in PostgreSQL provides a simple transactional outbox pattern for syncing data to ClickHouse. At scale, this should be replaced with a Debezium CDC connector reading the PostgreSQL WAL directly.

7. **Query cache stays in PostgreSQL (or Redis)** — query result caching is an operational concern with strong consistency requirements (invalidate on data change). It does not belong in ClickHouse.

8. **NL query analytics in ClickHouse, not PostgreSQL** — natural language query events generate high-volume data that is primarily used for AI model improvement and usage reporting. Storing these in ClickHouse keeps PostgreSQL lean and provides powerful analytical query capabilities for training data analysis.

9. **Dashboard view tracking as a first-class ClickHouse table** — this enables the "cross-tenant anonymized benchmarks" feature identified as an underserved opportunity in the features analysis. ClickHouse's approximate aggregation functions (uniq, quantile) make peer comparisons fast even across millions of view events.

10. **PostgreSQL schema is a streamlined hybrid** — the PostgreSQL layer uses the same hybrid relational + JSONB approach as Data Model Suggestion 3, but with fewer tables because all analytical/observability data has been offloaded to ClickHouse. This keeps the operational database small, fast, and easy to back up.
