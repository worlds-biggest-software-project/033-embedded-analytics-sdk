# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Embedded Analytics SDK · Created: 2026-05-11

## Philosophy

This model treats every state change in the platform as an immutable event. Instead of mutating rows in-place (UPDATE dashboard SET title = 'New Title'), the system appends an event (`dashboard.title_changed`) to an append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimized projections (materialized views) are built from events for fast querying, following the CQRS (Command Query Responsibility Segregation) pattern.

This architecture is used by financial ledger systems, event-driven microservices (e.g., Kafka-based architectures), and compliance-heavy platforms where "what happened and when" is as important as "what is the current state." PostgreSQL's JSONB columns and sequences provide a solid foundation for event sourcing without requiring a dedicated event store like EventStoreDB.

For an embedded analytics SDK, the event-sourced approach is particularly compelling because the platform already deals in data and queries — an event store becomes a powerful source of usage analytics, audit trails, and AI training data. Every dashboard edit, query execution, RLS policy change, and embed token issuance is captured as a first-class event, enabling temporal queries ("what did this dashboard look like last Tuesday?"), complete audit compliance, and ML-powered insight generation from usage patterns.

**Best for:** Teams building a compliance-first platform where full audit trails, temporal queries, and AI-powered usage analytics are core differentiators — not afterthoughts.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — every change is captured forever
- (+) Temporal queries are trivial: "show me this dashboard as of date X" is just event replay up to that point
- (+) Event stream is a goldmine for AI/ML: usage patterns, anomaly detection, and dashboard recommendation engines
- (+) Easy to add new read models (projections) without changing the write path
- (+) Natural fit for undo/redo and versioning features
- (-) Higher storage requirements — events are never deleted, only compacted via snapshots
- (-) Read-after-write consistency requires careful projection management
- (-) More complex than CRUD for simple operations; higher initial development effort
- (-) Debugging requires understanding event replay, not just "SELECT * FROM table"
- (-) Snapshot management adds operational complexity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OCSF Event Classes | Event types follow OCSF naming conventions (actor, action, target, outcome) |
| CloudEvents (CNCF) | Event envelope format aligns with CloudEvents spec (id, source, type, time, data) |
| OAuth 2.0 / JWT | Authentication events and token lifecycle tracked as first-class events |
| ISO 8601 | All timestamps in events use ISO 8601 with timezone (TIMESTAMPTZ) |
| Cube.js Semantic Layer | Semantic layer definitions stored as event streams; changes tracked as versioned events |
| GDPR Article 17 | Right to erasure handled via crypto-shredding: per-tenant encryption keys deleted to render events unreadable |

---

## Event Store (Core Write Model)

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    sequence_num    BIGSERIAL PRIMARY KEY,            -- global ordering
    event_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                     -- aggregate/entity ID
    stream_type     TEXT NOT NULL,                      -- e.g., 'workspace', 'dashboard', 'data_source'
    event_type      TEXT NOT NULL,                      -- e.g., 'dashboard.created', 'widget.moved'
    event_version   INTEGER NOT NULL,                   -- per-stream version for optimistic concurrency
    data            JSONB NOT NULL,                     -- event payload
    metadata        JSONB NOT NULL DEFAULT '{}',        -- actor, ip, user_agent, correlation_id
    organization_id UUID NOT NULL,                      -- partition key for tenant isolation
    workspace_id    UUID,                               -- nullable (org-level events have no workspace)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)                   -- optimistic concurrency control
);

-- Fast lookups by stream (entity history)
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
-- Fast lookups by type (projection rebuilds)
CREATE INDEX idx_events_type ON events(event_type);
-- Fast lookups by organization (tenant isolation)
CREATE INDEX idx_events_org ON events(organization_id);
-- Time-based queries
CREATE INDEX idx_events_created ON events(created_at);
-- Global ordering for projections
CREATE INDEX idx_events_sequence ON events(sequence_num);

-- Example event payload for 'dashboard.created':
-- {
--   "title": "Revenue Overview",
--   "description": "Monthly revenue metrics for Q1 2026",
--   "collection_id": "550e8400-e29b-41d4-a716-446655440000",
--   "is_template": false,
--   "layout_version": 1
-- }

-- Example metadata:
-- {
--   "actor_id": "user-uuid-here",
--   "actor_type": "user",
--   "ip_address": "192.168.1.100",
--   "user_agent": "Mozilla/5.0...",
--   "correlation_id": "req-uuid-here",
--   "source": "api"
-- }
```

### Snapshots (Performance Optimization)

```sql
-- Periodic snapshots to avoid replaying entire event streams
CREATE TABLE snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INTEGER NOT NULL,             -- event_version at time of snapshot
    state           JSONB NOT NULL,                -- full aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshots_stream ON snapshots(stream_id, snapshot_version DESC);
```

### Event Type Registry

```sql
-- Schema registry for event types (self-documenting, enables validation)
CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,              -- e.g., 'dashboard.created'
    stream_type     TEXT NOT NULL,                  -- e.g., 'dashboard'
    description     TEXT,
    schema_json     JSONB,                          -- JSON Schema for the event data payload
    introduced_in   TEXT,                           -- version where this event type was added
    is_deprecated   BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- 'organization.created', 'organization.updated', 'organization.plan_changed'
-- 'workspace.created', 'workspace.deactivated', 'workspace.theme_applied'
-- 'data_source.connected', 'data_source.synced', 'data_source.connection_failed'
-- 'semantic_cube.defined', 'semantic_cube.published', 'semantic_measure.added'
-- 'dashboard.created', 'dashboard.published', 'dashboard.title_changed'
-- 'widget.added', 'widget.moved', 'widget.resized', 'widget.query_changed'
-- 'rls_policy.created', 'rls_policy.bound_to_workspace', 'rls_policy.deactivated'
-- 'query.executed', 'query.cache_hit', 'query.failed'
-- 'embed_token.issued', 'embed_token.expired', 'embed_token.revoked'
-- 'nl_query.asked', 'nl_query.sql_generated', 'nl_query.executed'
-- 'user.joined_organization', 'user.role_changed', 'user.removed'
```

---

## Read Model Projections (Materialized Views)

These tables are **derived** from the event store. They can be rebuilt from scratch by replaying events. They exist purely for query performance.

### Projection: Organizations & Workspaces

```sql
CREATE TABLE proj_organizations (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL,
    billing_email   TEXT,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_workspaces (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES proj_organizations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    country_code    CHAR(2),
    region_code     TEXT,
    is_active       BOOLEAN NOT NULL,
    theme_id        UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_proj_ws_org ON proj_workspaces(organization_id);
```

### Projection: Users & Memberships

```sql
CREATE TABLE proj_users (
    id              UUID PRIMARY KEY,
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    is_active       BOOLEAN NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_user_memberships (
    user_id         UUID NOT NULL,
    scope_type      TEXT NOT NULL CHECK (scope_type IN ('organization', 'workspace')),
    scope_id        UUID NOT NULL,
    role            TEXT NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, scope_type, scope_id)
);

CREATE INDEX idx_proj_memberships_scope ON proj_user_memberships(scope_type, scope_id);
```

### Projection: Dashboards & Widgets

```sql
CREATE TABLE proj_dashboards (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    collection_id   UUID,
    title           TEXT NOT NULL,
    description     TEXT,
    is_template     BOOLEAN NOT NULL,
    is_published    BOOLEAN NOT NULL,
    layout_version  INTEGER NOT NULL,
    widget_count    INTEGER NOT NULL DEFAULT 0,      -- denormalized for listing performance
    created_by      UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    -- Version tracking from events
    current_version INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_proj_dashboards_ws ON proj_dashboards(workspace_id);

CREATE TABLE proj_widgets (
    id              UUID PRIMARY KEY,
    dashboard_id    UUID NOT NULL,
    widget_type     TEXT NOT NULL,
    title           TEXT,
    grid_x          INTEGER NOT NULL,
    grid_y          INTEGER NOT NULL,
    grid_w          INTEGER NOT NULL,
    grid_h          INTEGER NOT NULL,
    -- Query definition (denormalized)
    cube_name       TEXT,
    measures        TEXT[],
    dimensions      TEXT[],
    filters_sql     TEXT,
    time_dimension  TEXT,
    time_granularity TEXT,
    -- Visual config (denormalized)
    color_palette   TEXT[],
    show_legend     BOOLEAN,
    number_format   TEXT,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_widgets_dashboard ON proj_widgets(dashboard_id);
```

### Projection: Data Sources & Semantic Layer

```sql
CREATE TABLE proj_data_sources (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            TEXT NOT NULL,
    engine          TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL,
    last_synced_at  TIMESTAMPTZ,
    table_count     INTEGER DEFAULT 0,              -- denormalized
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_ds_org ON proj_data_sources(organization_id);

CREATE TABLE proj_semantic_cubes (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    data_source_id  UUID NOT NULL,
    name            TEXT NOT NULL,
    sql_table       TEXT,
    is_published    BOOLEAN NOT NULL,
    measure_count   INTEGER NOT NULL DEFAULT 0,     -- denormalized
    dimension_count INTEGER NOT NULL DEFAULT 0,     -- denormalized
    version         INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (organization_id, name)
);
```

### Projection: Query Analytics (Time-Series Optimized)

```sql
-- Aggregated query performance metrics (updated by projection processor)
CREATE TABLE proj_query_stats_hourly (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL,
    data_source_id  UUID NOT NULL,
    hour            TIMESTAMPTZ NOT NULL,            -- truncated to hour
    total_queries   INTEGER NOT NULL DEFAULT 0,
    cache_hits      INTEGER NOT NULL DEFAULT 0,
    avg_duration_ms INTEGER NOT NULL DEFAULT 0,
    p95_duration_ms INTEGER NOT NULL DEFAULT 0,
    error_count     INTEGER NOT NULL DEFAULT 0,
    UNIQUE (workspace_id, data_source_id, hour)
);

CREATE INDEX idx_query_stats_ws_time ON proj_query_stats_hourly(workspace_id, hour);
```

---

## Projection Tracking

```sql
-- Tracks which events each projection has processed (checkpoint)
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,              -- e.g., 'dashboards', 'query_stats_hourly'
    last_sequence   BIGINT NOT NULL DEFAULT 0,     -- last processed event sequence_num
    last_processed  TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message   TEXT
);

-- Example projection names:
-- 'organizations', 'workspaces', 'users', 'dashboards', 'widgets',
-- 'data_sources', 'semantic_layer', 'query_stats_hourly', 'rls_policies'
```

---

## Transactional Outbox (for External Event Publishing)

```sql
-- Outbox pattern: events staged for publishing to external message broker
CREATE TABLE event_outbox (
    id              BIGSERIAL PRIMARY KEY,
    event_id        UUID NOT NULL,
    event_type      TEXT NOT NULL,
    stream_type     TEXT NOT NULL,
    organization_id UUID NOT NULL,
    payload         JSONB NOT NULL,                -- full event data + metadata
    published       BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_unpublished ON event_outbox(published, created_at)
    WHERE published = false;
```

---

## Supporting Tables (Not Event-Sourced)

Some data is operational and doesn't benefit from event sourcing:

```sql
-- Encrypted data source credentials (operational, not event-sourced)
CREATE TABLE data_source_credentials (
    data_source_id  UUID PRIMARY KEY,              -- matches stream_id for data source events
    connection_options_encrypted TEXT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Query result cache (ephemeral, not event-sourced)
CREATE TABLE query_cache (
    cache_key       TEXT PRIMARY KEY,
    result_data     BYTEA,
    row_count       INTEGER,
    data_source_id  UUID NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_query_cache_expires ON query_cache(expires_at);

-- SSO configuration (operational, rarely changes)
CREATE TABLE sso_configurations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL UNIQUE,
    provider_type   TEXT NOT NULL CHECK (provider_type IN ('oidc', 'saml')),
    issuer_url      TEXT NOT NULL,
    client_id       TEXT NOT NULL,
    client_secret_encrypted TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Embed tokens (short-lived, not worth event-sourcing)
CREATE TABLE embed_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL,
    user_id         UUID,
    token_hash      TEXT NOT NULL UNIQUE,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    rls_context     JSONB,
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embed_tokens_expires ON embed_tokens(expires_at);
```

---

## Example: Temporal Query — Dashboard State at a Point in Time

```sql
-- Reconstruct the state of a dashboard as it existed on 2026-03-15
-- by replaying all events up to that timestamp

WITH dashboard_events AS (
    SELECT
        event_type,
        data,
        event_version,
        created_at
    FROM events
    WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'  -- dashboard UUID
      AND stream_type = 'dashboard'
      AND created_at <= '2026-03-15T23:59:59Z'
    ORDER BY event_version ASC
)
SELECT
    -- Initial state from 'dashboard.created' event
    (SELECT data->>'title' FROM dashboard_events WHERE event_type = 'dashboard.created') AS original_title,
    -- Latest title (may have been changed)
    COALESCE(
        (SELECT data->>'title' FROM dashboard_events
         WHERE event_type = 'dashboard.title_changed'
         ORDER BY event_version DESC LIMIT 1),
        (SELECT data->>'title' FROM dashboard_events WHERE event_type = 'dashboard.created')
    ) AS title_at_date,
    (SELECT COUNT(*) FROM dashboard_events WHERE event_type = 'widget.added') -
    (SELECT COUNT(*) FROM dashboard_events WHERE event_type = 'widget.removed') AS widget_count_at_date,
    (SELECT MAX(created_at) FROM dashboard_events) AS last_modified_at_date;
```

## Example: Usage Analytics from Events

```sql
-- Top 10 most-queried dashboards in the last 30 days
SELECT
    e.data->>'dashboard_id' AS dashboard_id,
    pd.title AS dashboard_title,
    COUNT(*) AS query_count,
    AVG((e.data->>'duration_ms')::int) AS avg_duration_ms,
    COUNT(*) FILTER (WHERE (e.data->>'cache_hit')::boolean) AS cache_hits
FROM events e
LEFT JOIN proj_dashboards pd ON pd.id = (e.data->>'dashboard_id')::uuid
WHERE e.event_type = 'query.executed'
  AND e.organization_id = 'org-uuid-here'
  AND e.created_at >= now() - INTERVAL '30 days'
GROUP BY e.data->>'dashboard_id', pd.title
ORDER BY query_count DESC
LIMIT 10;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only events table |
| Snapshots | 1 | Periodic aggregate state snapshots |
| Event Registry | 1 | Self-documenting event type catalog |
| Projection: Orgs & Workspaces | 2 | Derived read models |
| Projection: Users | 2 | Derived read models |
| Projection: Dashboards & Widgets | 2 | Derived read models |
| Projection: Data Sources & Semantic | 2 | Derived read models |
| Projection: Query Analytics | 1 | Time-series aggregation |
| Projection Tracking | 1 | Checkpoint management |
| Outbox | 1 | External event publishing |
| Operational (not event-sourced) | 4 | Credentials, cache, SSO, tokens |
| **Total** | **18** | ~50% fewer tables than normalized |

---

## Key Design Decisions

1. **Single events table with JSONB payload** — all event types stored in one table, discriminated by `stream_type` and `event_type`. The `data` JSONB column carries the type-specific payload. This keeps the write path simple and allows new event types to be added without schema migrations.

2. **Optimistic concurrency via (stream_id, event_version) uniqueness** — prevents concurrent writes from corrupting an entity's event stream. Writers must specify the expected version when appending events.

3. **Projections are disposable** — all `proj_*` tables can be dropped and rebuilt from events. This is a key advantage: if a projection schema needs to change, rebuild it rather than migrate it.

4. **Event type registry for self-documentation** — the `event_type_registry` table documents every event type with a JSON Schema for its payload, making the system self-describing and enabling runtime event validation.

5. **Transactional outbox for reliable external publishing** — events destined for external consumers (webhooks, message brokers) are staged in the `event_outbox` table within the same transaction as the event write, ensuring at-least-once delivery.

6. **Credentials and cache are NOT event-sourced** — sensitive credentials and ephemeral cache data don't benefit from immutable event history. These are stored in traditional mutable tables.

7. **Projection checkpoints for exactly-once processing** — each projection tracks the last event sequence number it processed, enabling crash recovery and ordered replay.

8. **GDPR compliance via crypto-shredding** — rather than deleting events (which would break the append-only guarantee), per-tenant encryption keys can be destroyed to render events for a deleted workspace/organization permanently unreadable.

9. **Hourly query stats projection** — rather than querying the raw event stream for performance dashboards, a time-series projection pre-aggregates query metrics hourly, making observability queries fast.

10. **Snapshot strategy** — aggregates with more than ~500 events get periodic snapshots stored as JSONB. Replay starts from the latest snapshot rather than from event zero, keeping read latency bounded.
