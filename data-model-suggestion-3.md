# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Embedded Analytics SDK · Created: 2026-05-11

## Philosophy

This model uses a pragmatic hybrid approach: core structural fields that are queried frequently (IDs, names, foreign keys, timestamps, status flags) live in typed relational columns, while variable, domain-specific, or rapidly evolving fields are stored in JSONB columns. PostgreSQL's JSONB support — including GIN indexes, containment queries, and JSON path expressions — makes this a first-class data type, not a workaround.

The hybrid pattern is widely used in modern SaaS platforms. Stripe stores payment method details in JSONB alongside relational order/customer fields. Shopify uses JSONB metafields for merchant-customizable product attributes. For an embedded analytics SDK, this approach is especially powerful because different SaaS vendors embedding the SDK will have wildly different requirements: one might need custom widget properties for healthcare compliance, another might need financial-specific chart annotations, and a third might need jurisdiction-specific RLS metadata. JSONB columns absorb this variability without schema migrations.

This architecture reduces the table count by ~40% compared to fully normalized designs, accelerates MVP development by eliminating the need to predict every field upfront, and provides a natural evolution path: fields that prove critical can be promoted from JSONB to typed columns when query patterns stabilize.

**Best for:** Teams prioritizing rapid MVP development and multi-customer flexibility, where different SaaS vendors embedding the SDK will have varying requirements that cannot all be predicted at design time.

**Trade-offs:**
- (+) Dramatically fewer tables (~20 vs ~35) — lower cognitive load and simpler migrations
- (+) New features often require zero schema changes — just add fields to JSONB
- (+) Absorbs per-customer/per-jurisdiction variability without schema-per-tenant complexity
- (+) PostgreSQL JSONB has mature indexing (GIN, btree on expressions) and query support
- (+) Faster MVP: ship with JSONB flexibility, promote to columns when patterns stabilize
- (-) JSONB fields lack database-enforced constraints (CHECK, NOT NULL, FOREIGN KEY)
- (-) Application-layer validation is required for JSONB contents — more room for inconsistent data
- (-) JSONB queries can be slower than typed column queries for complex filtering
- (-) Schema documentation must be maintained separately (JSONB contents aren't self-describing in `\d`)
- (-) Type coercion in JSONB queries requires explicit casting (`->>'field')::integer`)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OAuth 2.0 / OIDC | SSO provider configuration stored as JSONB in `organizations.sso_config` |
| JWT (RFC 7519) | Embed token scopes and RLS context stored as JSONB for flexible claim structures |
| ISO 3166-1/2 | Jurisdiction codes in `workspaces.settings` JSONB for per-workspace regional config |
| ISO 4217 | Currency codes in widget `visual_config` JSONB for financial chart formatting |
| Cube.js Semantic Layer | Cube definitions stored as JSONB matching the Cube YAML schema structure |
| OCSF | Audit event payloads stored as JSONB with OCSF-inspired field naming |
| JSON Schema (RFC draft) | JSONB column schemas documented as JSON Schema definitions in code |
| CloudEvents | Event metadata follows CloudEvents attribute naming in JSONB |

---

## Organizations, Workspaces & Users

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'starter', 'pro', 'enterprise')),
    -- JSONB: billing, feature flags, SSO config, custom branding defaults
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "billing_email": "billing@acme.com",
    --   "sso": {
    --     "provider_type": "oidc",
    --     "issuer_url": "https://auth.acme.com",
    --     "client_id": "embedded-analytics",
    --     "client_secret_encrypted": "enc:...",
    --     "metadata_url": "https://auth.acme.com/.well-known/openid-configuration"
    --   },
    --   "feature_flags": {
    --     "nl_query_enabled": true,
    --     "self_serve_canvas": false,
    --     "max_workspaces": 50,
    --     "max_data_sources": 10
    --   },
    --   "default_theme": {
    --     "primary_color": "#4F46E5",
    --     "font_family": "Inter, sans-serif"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: jurisdiction, locale, theme overrides, custom RLS defaults
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "country_code": "US",
    --   "region_code": "US-CA",
    --   "locale": "en-US",
    --   "timezone": "America/Los_Angeles",
    --   "theme_overrides": {
    --     "primary_color": "#1D4ED8",
    --     "logo_url": "https://cdn.tenant.com/logo.svg"
    --   },
    --   "default_rls_context": {
    --     "tenant_id": "tenant-123",
    --     "region": "west"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_workspaces_org ON workspaces(organization_id);
-- GIN index on settings for JSONB containment queries
CREATE INDEX idx_workspaces_settings ON workspaces USING gin(settings);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: profile preferences, notification settings
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "theme": "dark",
    --   "default_dashboard_id": "uuid-here",
    --   "notification_channels": ["email", "slack"],
    --   "timezone": "America/New_York"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Unified membership table (org + workspace memberships in one table)
CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    scope_type      TEXT NOT NULL CHECK (scope_type IN ('organization', 'workspace')),
    scope_id        UUID NOT NULL,                 -- org or workspace UUID
    role            TEXT NOT NULL DEFAULT 'viewer',
    -- JSONB: custom permissions, metadata
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- permissions example:
    -- {
    --   "dashboards": ["read", "write"],
    --   "data_sources": ["read"],
    --   "semantic_layer": ["read", "write"],
    --   "rls_policies": ["read"],
    --   "custom": {
    --     "can_export_pdf": true,
    --     "max_query_rows": 10000
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, scope_type, scope_id)
);

CREATE INDEX idx_memberships_user ON memberships(user_id);
CREATE INDEX idx_memberships_scope ON memberships(scope_type, scope_id);
```

---

## Data Sources & Schema Cache

```sql
CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    engine          TEXT NOT NULL
                    CHECK (engine IN ('postgresql', 'mysql', 'snowflake', 'bigquery',
                                      'redshift', 'clickhouse', 'databricks', 'mongodb')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: connection details (encrypted sensitive fields)
    connection      JSONB NOT NULL,
    -- connection example:
    -- {
    --   "host": "db.example.com",
    --   "port": 5432,
    --   "database": "analytics",
    --   "ssl_mode": "require",
    --   "max_pool_size": 10,
    --   "credentials_encrypted": "enc:...",
    --   "engine_specific": {
    --     "warehouse": "COMPUTE_WH",  -- Snowflake
    --     "project_id": "my-project", -- BigQuery
    --     "region": "us-east-1"       -- Redshift
    --   }
    -- }
    -- JSONB: cached schema metadata from sync
    schema_cache    JSONB DEFAULT '{}',
    -- schema_cache example:
    -- {
    --   "tables": [
    --     {
    --       "schema": "public",
    --       "name": "orders",
    --       "type": "table",
    --       "row_count_estimate": 150000,
    --       "columns": [
    --         {"name": "id", "type": "uuid", "nullable": false, "primary_key": true},
    --         {"name": "customer_id", "type": "uuid", "nullable": false},
    --         {"name": "total", "type": "numeric(10,2)", "nullable": false},
    --         {"name": "created_at", "type": "timestamptz", "nullable": false}
    --       ]
    --     }
    --   ],
    --   "synced_at": "2026-05-10T14:30:00Z"
    -- }
    workspace_ids   UUID[] NOT NULL DEFAULT '{}', -- which workspaces can access this source
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_org ON data_sources(organization_id);
-- GIN index for workspace access array lookups
CREATE INDEX idx_data_sources_ws ON data_sources USING gin(workspace_ids);
```

---

## Semantic Layer

```sql
-- Semantic cube definitions — structured JSONB matching Cube.js YAML format
CREATE TABLE semantic_cubes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    sql_table       TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    version         INTEGER NOT NULL DEFAULT 1,
    -- JSONB: full cube definition (measures, dimensions, joins, segments)
    definition      JSONB NOT NULL,
    -- definition example:
    -- {
    --   "measures": [
    --     {
    --       "name": "total_revenue",
    --       "sql": "SUM(amount)",
    --       "type": "sum",
    --       "format": "currency",
    --       "description": "Total revenue from all orders"
    --     },
    --     {
    --       "name": "order_count",
    --       "sql": "id",
    --       "type": "count"
    --     }
    --   ],
    --   "dimensions": [
    --     {
    --       "name": "status",
    --       "sql": "status",
    --       "type": "string"
    --     },
    --     {
    --       "name": "created_date",
    --       "sql": "created_at",
    --       "type": "time",
    --       "is_primary_key": false
    --     }
    --   ],
    --   "joins": [
    --     {
    --       "target": "customers",
    --       "relationship": "many_to_one",
    --       "sql_on": "{orders}.customer_id = {customers}.id"
    --     }
    --   ],
    --   "segments": [
    --     {
    --       "name": "completed_orders",
    --       "sql": "status = 'completed'"
    --     }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE INDEX idx_semantic_cubes_org ON semantic_cubes(organization_id);
-- GIN index on definition for searching measures/dimensions by name
CREATE INDEX idx_semantic_cubes_def ON semantic_cubes USING gin(definition);
```

---

## Dashboards, Widgets & Collections

```sql
CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES collections(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    -- JSONB: metadata, sharing settings, access restrictions
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, parent_id, slug)
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
    -- JSONB: layout config, global filters, refresh settings
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "layout_version": 1,
    --   "grid_columns": 12,
    --   "auto_refresh_seconds": 300,
    --   "global_filters": [
    --     {
    --       "dimension": "created_date",
    --       "type": "date_range",
    --       "default": "last_30_days"
    --     }
    --   ],
    --   "sharing": {
    --     "public_link_enabled": false,
    --     "embed_allowed": true
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dashboards_ws ON dashboards(workspace_id);
CREATE INDEX idx_dashboards_collection ON dashboards(collection_id);

-- Widgets: core position is relational, everything else is JSONB
CREATE TABLE widgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dashboard_id    UUID NOT NULL REFERENCES dashboards(id) ON DELETE CASCADE,
    widget_type     TEXT NOT NULL
                    CHECK (widget_type IN ('bar_chart', 'line_chart', 'pie_chart', 'table',
                                           'kpi', 'scatter', 'area_chart', 'heatmap',
                                           'map', 'text', 'filter', 'custom')),
    title           TEXT,
    -- Grid position (always queried for layout rendering)
    grid_x          INTEGER NOT NULL DEFAULT 0,
    grid_y          INTEGER NOT NULL DEFAULT 0,
    grid_w          INTEGER NOT NULL DEFAULT 6,
    grid_h          INTEGER NOT NULL DEFAULT 4,
    -- JSONB: query definition + visual configuration combined
    query_config    JSONB NOT NULL DEFAULT '{}',
    -- query_config example:
    -- {
    --   "cube": "orders",
    --   "measures": ["total_revenue", "order_count"],
    --   "dimensions": ["status"],
    --   "time_dimension": {
    --     "dimension": "created_date",
    --     "granularity": "month"
    --   },
    --   "filters": [
    --     {"dimension": "status", "operator": "equals", "values": ["completed"]}
    --   ],
    --   "order_by": [{"name": "total_revenue", "direction": "desc"}],
    --   "limit": 100
    -- }
    visual_config   JSONB NOT NULL DEFAULT '{}',
    -- visual_config example:
    -- {
    --   "color_palette": ["#4F46E5", "#10B981", "#F59E0B"],
    --   "x_axis": {"label": "Month", "show_grid": true},
    --   "y_axis": {"label": "Revenue ($)", "format": "currency", "currency": "USD"},
    --   "legend": {"show": true, "position": "bottom"},
    --   "tooltip": {"enabled": true},
    --   "annotations": [
    --     {"type": "threshold", "value": 100000, "label": "Target", "color": "#EF4444"}
    --   ],
    --   "custom_css": ".widget-title { font-weight: 700; }"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_widgets_dashboard ON widgets(dashboard_id);
```

---

## Row-Level Security

```sql
CREATE TABLE rls_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: full RLS policy definition + workspace bindings in one place
    policy          JSONB NOT NULL,
    -- policy example:
    -- {
    --   "description": "Filter orders by tenant_id",
    --   "table_name": "public.orders",
    --   "filter_column": "tenant_id",
    --   "filter_type": "equals",
    --   "bindings": [
    --     {"workspace_id": "ws-uuid-1", "filter_value": "tenant-abc"},
    --     {"workspace_id": "ws-uuid-2", "filter_value": "tenant-xyz"}
    --   ],
    --   "custom_sql": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rls_policies_org ON rls_policies(organization_id);
CREATE INDEX idx_rls_policies_ds ON rls_policies(data_source_id);
-- GIN index for finding policies by workspace binding
CREATE INDEX idx_rls_policies_bindings ON rls_policies USING gin(policy);
```

---

## Embed Tokens

```sql
CREATE TABLE embed_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    token_hash      TEXT NOT NULL UNIQUE,
    expires_at      TIMESTAMPTZ NOT NULL,
    -- JSONB: scopes, RLS context, custom claims
    claims          JSONB NOT NULL DEFAULT '{}',
    -- claims example:
    -- {
    --   "scopes": ["dashboard:read", "query:execute"],
    --   "rls_context": {"tenant_id": "tenant-abc", "region": "us-west"},
    --   "dashboard_ids": ["dash-uuid-1", "dash-uuid-2"],
    --   "max_query_rows": 10000,
    --   "custom": {
    --     "department": "engineering",
    --     "cost_center": "CC-100"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embed_tokens_ws ON embed_tokens(workspace_id);
CREATE INDEX idx_embed_tokens_expires ON embed_tokens(expires_at);
```

---

## Query Execution & Cache

```sql
CREATE TABLE query_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    widget_id       UUID REFERENCES widgets(id),
    cube_name       TEXT,
    duration_ms     INTEGER NOT NULL,
    row_count       INTEGER,
    cache_hit       BOOLEAN NOT NULL DEFAULT false,
    error_message   TEXT,
    -- JSONB: full query details for debugging and analytics
    query_details   JSONB NOT NULL DEFAULT '{}',
    -- query_details example:
    -- {
    --   "measures": ["total_revenue"],
    --   "dimensions": ["status"],
    --   "filters": [{"dimension": "status", "operator": "equals", "values": ["completed"]}],
    --   "generated_sql": "SELECT status, SUM(amount) FROM orders WHERE ...",
    --   "data_source_id": "ds-uuid",
    --   "rls_applied": true,
    --   "rls_filter": "tenant_id = 'tenant-abc'"
    -- }
    executed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_query_exec_ws ON query_executions(workspace_id);
CREATE INDEX idx_query_exec_time ON query_executions(executed_at);

CREATE TABLE query_cache (
    cache_key       TEXT PRIMARY KEY,
    result_data     BYTEA,
    row_count       INTEGER,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_query_cache_expires ON query_cache(expires_at);
```

---

## Audit Trail

```sql
CREATE TABLE audit_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    workspace_id    UUID,
    actor_id        UUID,
    action          TEXT NOT NULL,              -- e.g., 'dashboard.created'
    resource_type   TEXT NOT NULL,              -- e.g., 'dashboard'
    resource_id     UUID,
    outcome         TEXT NOT NULL DEFAULT 'success',
    -- JSONB: flexible event details (OCSF-inspired)
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "actor_type": "user",
    --   "ip_address": "192.168.1.100",
    --   "user_agent": "Mozilla/5.0...",
    --   "severity": "info",
    --   "changes": {
    --     "title": {"from": "Old Title", "to": "New Title"}
    --   },
    --   "rls_context": {"tenant_id": "tenant-abc"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_events(organization_id);
CREATE INDEX idx_audit_time ON audit_events(created_at);
CREATE INDEX idx_audit_action ON audit_events(action);
-- GIN index for searching audit details
CREATE INDEX idx_audit_details ON audit_events USING gin(details);
```

---

## Natural Language Query

```sql
CREATE TABLE nl_query_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    -- JSONB: session context (active cube, filters, etc.)
    context         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE nl_query_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES nl_query_sessions(id) ON DELETE CASCADE,
    role            TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content         TEXT NOT NULL,
    -- JSONB: generated SQL, execution results, error details
    result          JSONB DEFAULT NULL,
    -- result example:
    -- {
    --   "generated_sql": "SELECT status, COUNT(*) FROM orders GROUP BY status",
    --   "cube_used": "orders",
    --   "measures": ["order_count"],
    --   "dimensions": ["status"],
    --   "execution_id": "exec-uuid",
    --   "row_count": 5,
    --   "suggested_chart_type": "pie_chart"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_nl_sessions_ws ON nl_query_sessions(workspace_id);
CREATE INDEX idx_nl_messages_session ON nl_query_messages(session_id);
```

---

## Example: JSONB Containment Query — Find Workspaces with Specific RLS

```sql
-- Find all RLS policies that have a binding for workspace ws-uuid-1
SELECT id, name, policy->>'table_name' AS table_name
FROM rls_policies
WHERE policy @> '{"bindings": [{"workspace_id": "ws-uuid-1"}]}'
  AND is_active = true;
```

## Example: JSONB Path Query — Search Semantic Layer

```sql
-- Find all cubes that have a measure of type 'sum'
SELECT id, name
FROM semantic_cubes
WHERE definition @? '$.measures[*] ? (@.type == "sum")'
  AND is_published = true;
```

## Example: Promoting a JSONB Field to a Typed Column

```sql
-- When analytics show that 'country_code' is queried in 80%+ of workspace lookups,
-- promote it from JSONB settings to a typed column:

ALTER TABLE workspaces ADD COLUMN country_code CHAR(2);

UPDATE workspaces
SET country_code = settings->>'country_code'
WHERE settings->>'country_code' IS NOT NULL;

CREATE INDEX idx_workspaces_country ON workspaces(country_code);

-- The JSONB field can remain for backward compatibility or be removed
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Workspaces | 2 | Settings/config in JSONB |
| Users & Memberships | 2 | Unified membership table with JSONB permissions |
| Data Sources | 1 | Connection + schema cache in JSONB |
| Semantic Layer | 1 | Full cube definition in JSONB |
| Collections & Dashboards | 3 | Dashboard config in JSONB |
| Widgets | 1 | Query + visual config in JSONB |
| RLS Policies | 1 | Policy + bindings in JSONB |
| Embed Tokens | 1 | Claims in JSONB |
| Query Engine | 2 | Execution details in JSONB + cache |
| Audit Trail | 1 | Event details in JSONB |
| NL Query | 2 | Session context + message results in JSONB |
| **Total** | **17** | ~50% fewer tables than normalized |

---

## Key Design Decisions

1. **JSONB for anything that varies per customer or evolves rapidly** — widget visual config, data source connection options, SSO settings, RLS bindings, and semantic layer definitions all live in JSONB. These are the areas where different SaaS vendors will have the most divergent requirements.

2. **Relational columns for everything that is queried for filtering, joining, or ordering** — IDs, foreign keys, names, slugs, timestamps, boolean flags, and status fields remain as typed columns with proper indexes. The rule: if it appears in a WHERE, JOIN, or ORDER BY, it should be a column.

3. **Unified memberships table** — instead of separate tables for org memberships and workspace memberships, a single `memberships` table with `scope_type` + `scope_id` reduces table count and simplifies permission queries.

4. **Schema cache as JSONB in data_sources** — rather than separate `data_source_tables` and `data_source_columns` tables (6 total in the normalized model), the cached schema metadata from database sync is stored as JSONB. This is appropriate because the schema cache is a snapshot that is fully replaced on each sync, not incrementally updated.

5. **Semantic cube definition as a single JSONB column** — the entire cube definition (measures, dimensions, joins, segments) is stored as one JSONB document matching the Cube.js YAML structure. This makes import/export trivial and allows the semantic layer to evolve without migrations.

6. **Widget query_config and visual_config as separate JSONB columns** — while both are JSONB, separating query definition from visual configuration allows re-querying without touching visual config and vice versa.

7. **GIN indexes on all major JSONB columns** — enables containment queries (`@>`) and JSON path queries (`@?`) with index support, keeping JSONB queries performant.

8. **workspace_ids as UUID array in data_sources** — instead of a junction table, which workspaces can access a data source is stored as a UUID array with GIN index. This works well when the array is small (typical: 1-20 workspaces per data source) and avoids a join for the most common query ("can this workspace use this data source?").

9. **Promotion path from JSONB to columns** — the design explicitly anticipates that high-frequency JSONB fields will be promoted to typed columns as query patterns mature. This is an evolutionary architecture, not a permanent state.

10. **Audit event details in JSONB** — audit events capture the "what changed" details in JSONB, allowing the schema to accommodate new resource types and change formats without migrations. The action and resource_type columns provide relational filtering while details provides the payload.
