# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Embedded Analytics SDK · Created: 2026-05-11

## Philosophy

This model follows traditional normalized relational design where every distinct concept in the domain gets its own table with explicit foreign key relationships. The schema enforces referential integrity at the database level, making it impossible to create orphaned records (e.g., a widget referencing a non-existent dashboard, or a permission grant for a deleted workspace).

This approach mirrors how enterprise BI platforms like Metabase and Apache Superset structure their internal application databases — separate tables for dashboards, cards/widgets, collections, data sources, users, and permissions, all connected through foreign keys. It is the most familiar pattern for teams with relational database experience and provides the strongest guarantees for data consistency.

The normalized model is best suited for teams that prioritize data integrity and auditability over development velocity, and for deployments where the schema is relatively stable and well-understood before implementation begins.

**Best for:** Teams building a compliance-conscious, enterprise-grade embedded analytics platform where data integrity and complex cross-entity queries are paramount.

**Trade-offs:**
- (+) Strongest referential integrity — the database itself prevents invalid states
- (+) Standard SQL tooling works out of the box — any developer can query, debug, and report
- (+) Well-understood migration patterns (ALTER TABLE, new junction tables)
- (+) Excellent for complex JOIN queries across entities (e.g., "all dashboards in workspace X that use data source Y")
- (-) High table count (~35-40 tables) increases cognitive load and migration complexity
- (-) Schema changes require migrations even for minor additions (new permission type, new widget property)
- (-) Junction tables for many-to-many relationships add write overhead
- (-) Multi-jurisdiction variability (different field requirements per region) is awkward without JSONB escape hatches

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OAuth 2.0 / OIDC | `oauth_providers` and `sso_configurations` tables model identity provider integration per workspace |
| JWT (RFC 7519) | `embed_tokens` table stores token metadata for per-session, per-tenant access control |
| ISO 3166-1 | `workspaces.country_code` and `workspaces.region_code` for jurisdiction-aware deployments |
| RBAC (NIST SP 800-162) | Normalized `roles`, `permissions`, `role_permissions`, `user_workspace_roles` tables |
| Cube.js Semantic Layer | `semantic_cubes`, `semantic_measures`, `semantic_dimensions`, `semantic_joins` tables mirror Cube YAML structure |
| OCSF Event Classes | `audit_events` table structure aligns with OCSF metadata fields (actor, action, target, outcome, severity) |
| WCAG 2.1 AA | `themes.accessibility_mode` flag for high-contrast/accessible theme variants |

---

## Core Platform Tables

### Organizations & Workspaces

```sql
-- Top-level organization (the SaaS vendor embedding analytics)
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'starter', 'pro', 'enterprise')),
    billing_email   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tenant workspace (one per end-customer of the SaaS vendor)
CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    country_code    CHAR(2),         -- ISO 3166-1 alpha-2
    region_code     TEXT,            -- ISO 3166-2 subdivision
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_workspaces_org ON workspaces(organization_id);
```

### Users & Authentication

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Many-to-many: users belong to organizations with a role
CREATE TABLE user_organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, organization_id)
);

-- Many-to-many: users can access specific workspaces with granular roles
CREATE TABLE user_workspace_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'viewer'
                    CHECK (role IN ('admin', 'editor', 'viewer')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, workspace_id)
);

CREATE INDEX idx_user_org_memberships_user ON user_organization_memberships(user_id);
CREATE INDEX idx_user_ws_memberships_user ON user_workspace_memberships(user_id);
CREATE INDEX idx_user_ws_memberships_ws ON user_workspace_memberships(workspace_id);
```

### RBAC — Roles & Permissions

```sql
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,  -- built-in vs custom
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   TEXT NOT NULL,  -- 'dashboard', 'data_source', 'workspace', 'semantic_layer'
    action          TEXT NOT NULL,  -- 'read', 'write', 'delete', 'share', 'embed'
    description     TEXT,
    UNIQUE (resource_type, action)
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_workspace_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_by      UUID REFERENCES users(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, workspace_id, role_id)
);
```

### SSO & OAuth Configuration

```sql
CREATE TABLE sso_configurations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    provider_type   TEXT NOT NULL CHECK (provider_type IN ('oidc', 'saml')),
    issuer_url      TEXT NOT NULL,
    client_id       TEXT NOT NULL,
    client_secret_encrypted TEXT NOT NULL,  -- encrypted at rest
    metadata_url    TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, provider_type)
);

-- Short-lived embed tokens for per-session access
CREATE TABLE embed_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),          -- nullable for anonymous embeds
    token_hash      TEXT NOT NULL UNIQUE,                -- SHA-256 hash; raw token never stored
    scopes          TEXT[] NOT NULL DEFAULT '{}',        -- e.g., {'dashboard:read', 'query:execute'}
    rls_context     JSONB,                               -- row-level security filter context
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embed_tokens_ws ON embed_tokens(workspace_id);
CREATE INDEX idx_embed_tokens_expires ON embed_tokens(expires_at);
```

---

## Data Source Management

```sql
CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    engine          TEXT NOT NULL
                    CHECK (engine IN ('postgresql', 'mysql', 'snowflake', 'bigquery',
                                      'redshift', 'clickhouse', 'databricks', 'mongodb')),
    host            TEXT,
    port            INTEGER,
    database_name   TEXT,
    connection_options_encrypted TEXT,  -- encrypted JSON with credentials
    ssl_mode        TEXT DEFAULT 'require',
    max_pool_size   INTEGER DEFAULT 10,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_org ON data_sources(organization_id);

-- Cached schema metadata from data source sync
CREATE TABLE data_source_tables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    schema_name     TEXT NOT NULL DEFAULT 'public',
    table_name      TEXT NOT NULL,
    table_type      TEXT DEFAULT 'table' CHECK (table_type IN ('table', 'view', 'materialized_view')),
    row_count_estimate BIGINT,
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (data_source_id, schema_name, table_name)
);

CREATE TABLE data_source_columns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_id        UUID NOT NULL REFERENCES data_source_tables(id) ON DELETE CASCADE,
    column_name     TEXT NOT NULL,
    data_type       TEXT NOT NULL,
    is_nullable     BOOLEAN NOT NULL DEFAULT true,
    is_primary_key  BOOLEAN NOT NULL DEFAULT false,
    ordinal_position INTEGER NOT NULL,
    UNIQUE (table_id, column_name)
);

CREATE INDEX idx_ds_tables_source ON data_source_tables(data_source_id);
CREATE INDEX idx_ds_columns_table ON data_source_columns(table_id);

-- Per-workspace data source access (which workspaces can use which data sources)
CREATE TABLE workspace_data_sources (
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    PRIMARY KEY (workspace_id, data_source_id)
);
```

---

## Semantic Layer

```sql
-- Mirrors Cube.js cube definitions
CREATE TABLE semantic_cubes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,               -- e.g., 'orders'
    sql_table       TEXT,                        -- e.g., 'public.orders'
    sql_query       TEXT,                        -- alternative: custom SQL
    description     TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE TABLE semantic_measures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cube_id         UUID NOT NULL REFERENCES semantic_cubes(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,               -- e.g., 'total_revenue'
    sql_expression  TEXT NOT NULL,               -- e.g., 'SUM(amount)'
    measure_type    TEXT NOT NULL
                    CHECK (measure_type IN ('count', 'sum', 'avg', 'min', 'max',
                                            'count_distinct', 'running_total', 'custom')),
    format          TEXT,                        -- e.g., 'currency', 'percent', 'number'
    description     TEXT,
    UNIQUE (cube_id, name)
);

CREATE TABLE semantic_dimensions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cube_id         UUID NOT NULL REFERENCES semantic_cubes(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,               -- e.g., 'created_date'
    sql_expression  TEXT NOT NULL,               -- e.g., 'created_at'
    dimension_type  TEXT NOT NULL
                    CHECK (dimension_type IN ('string', 'number', 'time', 'boolean', 'geo')),
    is_primary_key  BOOLEAN NOT NULL DEFAULT false,
    description     TEXT,
    UNIQUE (cube_id, name)
);

CREATE TABLE semantic_joins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_cube_id  UUID NOT NULL REFERENCES semantic_cubes(id) ON DELETE CASCADE,
    target_cube_id  UUID NOT NULL REFERENCES semantic_cubes(id) ON DELETE CASCADE,
    relationship    TEXT NOT NULL
                    CHECK (relationship IN ('one_to_one', 'one_to_many', 'many_to_one', 'many_to_many')),
    sql_on          TEXT NOT NULL,               -- e.g., '{orders}.customer_id = {customers}.id'
    UNIQUE (source_cube_id, target_cube_id)
);

CREATE INDEX idx_semantic_cubes_org ON semantic_cubes(organization_id);
CREATE INDEX idx_semantic_measures_cube ON semantic_measures(cube_id);
CREATE INDEX idx_semantic_dimensions_cube ON semantic_dimensions(cube_id);
```

---

## Dashboards & Widgets

```sql
-- Collections organize dashboards (like folders)
CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES collections(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, parent_id, slug)
);

CREATE INDEX idx_collections_ws ON collections(workspace_id);
CREATE INDEX idx_collections_parent ON collections(parent_id);

CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    collection_id   UUID REFERENCES collections(id) ON DELETE SET NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    is_template     BOOLEAN NOT NULL DEFAULT false,  -- reusable template across workspaces
    is_published    BOOLEAN NOT NULL DEFAULT false,
    layout_version  INTEGER NOT NULL DEFAULT 1,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dashboards_ws ON dashboards(workspace_id);
CREATE INDEX idx_dashboards_collection ON dashboards(collection_id);

-- Individual chart/widget on a dashboard
CREATE TABLE widgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dashboard_id    UUID NOT NULL REFERENCES dashboards(id) ON DELETE CASCADE,
    widget_type     TEXT NOT NULL
                    CHECK (widget_type IN ('bar_chart', 'line_chart', 'pie_chart', 'table',
                                           'kpi', 'scatter', 'area_chart', 'heatmap',
                                           'map', 'text', 'filter', 'custom')),
    title           TEXT,
    -- Grid layout position (CSS Grid / react-grid-layout compatible)
    grid_x          INTEGER NOT NULL DEFAULT 0,
    grid_y          INTEGER NOT NULL DEFAULT 0,
    grid_w          INTEGER NOT NULL DEFAULT 6,
    grid_h          INTEGER NOT NULL DEFAULT 4,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_widgets_dashboard ON widgets(dashboard_id);

-- Query definition backing a widget
CREATE TABLE widget_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    widget_id       UUID NOT NULL REFERENCES widgets(id) ON DELETE CASCADE,
    cube_id         UUID REFERENCES semantic_cubes(id),
    measures        TEXT[] NOT NULL DEFAULT '{}',     -- measure names from semantic_measures
    dimensions      TEXT[] NOT NULL DEFAULT '{}',     -- dimension names from semantic_dimensions
    filters_sql     TEXT,                             -- WHERE clause fragment
    time_dimension  TEXT,                             -- time dimension name for time-series
    time_granularity TEXT CHECK (time_granularity IN ('second', 'minute', 'hour', 'day', 'week', 'month', 'quarter', 'year')),
    order_by        TEXT,
    row_limit       INTEGER DEFAULT 1000,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Visual configuration for a widget (colors, labels, axes)
CREATE TABLE widget_configurations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    widget_id       UUID NOT NULL REFERENCES widgets(id) ON DELETE CASCADE,
    color_palette   TEXT[] DEFAULT '{#4F46E5, #10B981, #F59E0B, #EF4444, #8B5CF6}',
    x_axis_label    TEXT,
    y_axis_label    TEXT,
    show_legend     BOOLEAN DEFAULT true,
    show_grid       BOOLEAN DEFAULT true,
    number_format   TEXT DEFAULT '#,##0',
    currency_code   CHAR(3),                         -- ISO 4217
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (widget_id)
);
```

---

## Themes & White-Labelling

```sql
CREATE TABLE themes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    primary_color   TEXT NOT NULL DEFAULT '#4F46E5',
    secondary_color TEXT NOT NULL DEFAULT '#10B981',
    background_color TEXT NOT NULL DEFAULT '#FFFFFF',
    text_color      TEXT NOT NULL DEFAULT '#111827',
    font_family     TEXT DEFAULT 'Inter, sans-serif',
    border_radius   TEXT DEFAULT '8px',
    logo_url        TEXT,
    favicon_url     TEXT,
    accessibility_mode BOOLEAN NOT NULL DEFAULT false,  -- WCAG 2.1 AA high-contrast
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

-- Per-workspace theme override (inherits from org theme if not set)
CREATE TABLE workspace_themes (
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    theme_id        UUID NOT NULL REFERENCES themes(id) ON DELETE CASCADE,
    PRIMARY KEY (workspace_id)
);
```

---

## Row-Level Security Policies

```sql
-- RLS policy definitions (what filters to apply per workspace)
CREATE TABLE rls_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    table_name      TEXT NOT NULL,               -- source table this policy applies to
    filter_column   TEXT NOT NULL,               -- column to filter on
    filter_type     TEXT NOT NULL
                    CHECK (filter_type IN ('equals', 'in', 'range', 'custom_sql')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- RLS policy bindings: which workspaces get which filter values
CREATE TABLE rls_policy_bindings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES rls_policies(id) ON DELETE CASCADE,
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    filter_value    TEXT NOT NULL,                -- the value to filter by for this workspace
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (policy_id, workspace_id)
);

CREATE INDEX idx_rls_policies_org ON rls_policies(organization_id);
CREATE INDEX idx_rls_bindings_policy ON rls_policy_bindings(policy_id);
CREATE INDEX idx_rls_bindings_ws ON rls_policy_bindings(workspace_id);
```

---

## Query Cache & Execution

```sql
CREATE TABLE query_cache (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cache_key       TEXT NOT NULL UNIQUE,         -- hash of (cube + measures + dimensions + filters + rls_context)
    result_data     BYTEA,                        -- compressed query result
    row_count       INTEGER,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_query_cache_key ON query_cache(cache_key);
CREATE INDEX idx_query_cache_expires ON query_cache(expires_at);

-- Query execution log for performance monitoring
CREATE TABLE query_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    widget_id       UUID REFERENCES widgets(id),
    cube_id         UUID REFERENCES semantic_cubes(id),
    query_sql       TEXT,
    duration_ms     INTEGER NOT NULL,
    row_count       INTEGER,
    cache_hit       BOOLEAN NOT NULL DEFAULT false,
    error_message   TEXT,
    executed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_query_exec_ws ON query_executions(workspace_id);
CREATE INDEX idx_query_exec_time ON query_executions(executed_at);
```

---

## Audit Trail

```sql
-- OCSF-aligned audit event log
CREATE TABLE audit_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    workspace_id    UUID REFERENCES workspaces(id),
    actor_id        UUID REFERENCES users(id),
    actor_type      TEXT NOT NULL DEFAULT 'user'
                    CHECK (actor_type IN ('user', 'system', 'api_key')),
    action          TEXT NOT NULL,                -- e.g., 'dashboard.created', 'query.executed', 'rls.updated'
    resource_type   TEXT NOT NULL,                -- e.g., 'dashboard', 'data_source', 'workspace'
    resource_id     UUID,
    outcome         TEXT NOT NULL DEFAULT 'success'
                    CHECK (outcome IN ('success', 'failure', 'error')),
    severity        TEXT NOT NULL DEFAULT 'info'
                    CHECK (severity IN ('info', 'low', 'medium', 'high', 'critical')),
    ip_address      INET,
    user_agent      TEXT,
    details         TEXT,                         -- human-readable description
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_events_org ON audit_events(organization_id);
CREATE INDEX idx_audit_events_ws ON audit_events(workspace_id);
CREATE INDEX idx_audit_events_time ON audit_events(created_at);
CREATE INDEX idx_audit_events_action ON audit_events(action);
```

---

## Natural Language Query (AI)

```sql
CREATE TABLE nl_query_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE nl_query_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES nl_query_sessions(id) ON DELETE CASCADE,
    role            TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content         TEXT NOT NULL,
    generated_sql   TEXT,                         -- SQL generated by LLM (if applicable)
    cube_id         UUID REFERENCES semantic_cubes(id),
    execution_id    UUID REFERENCES query_executions(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_nl_sessions_ws ON nl_query_sessions(workspace_id);
CREATE INDEX idx_nl_messages_session ON nl_query_messages(session_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Workspaces | 2 | Core tenant hierarchy |
| Users & Authentication | 3 | User accounts + org/workspace memberships |
| RBAC | 4 | Roles, permissions, grants |
| SSO & Tokens | 2 | OAuth/SAML config + embed tokens |
| Data Sources | 4 | Connections, schema cache, workspace access |
| Semantic Layer | 4 | Cubes, measures, dimensions, joins |
| Dashboards & Widgets | 5 | Collections, dashboards, widgets, queries, configs |
| Themes | 2 | Organization themes + workspace overrides |
| RLS | 2 | Policy definitions + workspace bindings |
| Query Engine | 2 | Cache + execution log |
| Audit | 1 | OCSF-aligned event log |
| AI / NL Query | 2 | Chat sessions + messages |
| **Total** | **33** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — standard for distributed SaaS; avoids integer sequence conflicts across services and enables client-side ID generation.

2. **Two-level tenant hierarchy (Organization > Workspace)** — the SaaS vendor is the Organization; their end-customers are Workspaces. This mirrors the Luzmo/Metabase workspace model and allows per-workspace RLS, theming, and permissions.

3. **Separate semantic layer tables** — rather than storing Cube.js YAML files as blobs, the semantic layer is fully normalized into `semantic_cubes`, `semantic_measures`, `semantic_dimensions`, and `semantic_joins`. This enables the platform to validate, version, and query the semantic layer programmatically.

4. **RLS as a first-class entity** — row-level security policies are modeled explicitly with per-workspace bindings, not embedded in query logic. This makes RLS auditable, testable, and manageable via API.

5. **Encrypted credential storage** — data source credentials and SSO client secrets are stored as encrypted text, never plaintext. The encryption key is managed outside the database.

6. **OCSF-aligned audit trail** — the `audit_events` table uses OCSF-inspired fields (actor, action, resource, outcome, severity) for structured, queryable audit logging.

7. **Widget queries separated from widget layout** — `widget_queries` and `widget_configurations` are separate tables from `widgets`, allowing the same widget to be re-queried or restyled independently.

8. **Explicit junction tables for all many-to-many relationships** — `user_organization_memberships`, `user_workspace_memberships`, `workspace_data_sources`, `role_permissions` avoid implicit or JSONB-based associations.

9. **Query execution log for performance observability** — every query execution is logged with duration, cache hit status, and error information, enabling self-tuning performance optimization.

10. **Natural language query as a conversation model** — NL query sessions and messages are stored as a chat history, allowing multi-turn conversations and tracking which SQL was generated and executed.
