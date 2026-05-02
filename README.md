# Embedded Analytics SDK

**Candidate #33** — A headless embedded analytics platform enabling SaaS products to white-label customer-facing dashboards without iframe lock-in, requiring minimal engineering overhead.

## Market Opportunity

The embedded analytics market is **$23–50 billion in 2025** and is growing at **15–18% CAGR through 2035**. SaaS companies increasingly recognize analytics as a value driver—differentiation rather than commodity feature.

**Market context:**
- 68% of SaaS products plan to expand analytics capabilities in 2025–2026
- Average time-to-market for custom-built embedded analytics: **6–18 months of engineering**
- Build vs. buy calculation: embedding Sisense ($50K+/year), Luzmo ($800–5K/month), or Embeddable ($2K+/month) is faster than building
- Opportunities: no open-source tool offers production-grade embedded analytics; Cube.js and Superset are low-level, requiring significant UI work

## What This Platform Solves

A **developer-friendly, headless embedded analytics SDK** that lets SaaS vendors embed interactive dashboards in customer products with zero iframe, full UI control, and multi-tenant isolation built-in.

**Core value proposition:**
- **Headless architecture**: No iframes, no vendor branding, full React component control
- **Multi-tenancy**: Workspace-based isolation, row-level security, per-tenant JWT tokens
- **White-labelling**: Full CSS theming and brand customization
- **Self-serve analytics**: End-customers compose custom dashboard views within guardrails
- **Natural language queries**: Embedded "chat with your data" for end-users
- **Affordable**: $800–$2K/month entry point (vs. $50K+/year for Sisense)

## Competitive Differentiation

| Aspect | This Platform | Luzmo | Embeddable | Sisense | Metabase |
|--------|---------------|--------|-----------|---------|----------|
| **Headless** | Yes | Partial | Yes | Yes | No (iframe) |
| **Self-Serve** | Yes (Canvas) | Yes | Yes | Partial | Yes |
| **White-Label** | Yes | Yes | Yes | Yes | Limited |
| **Multi-Tenancy** | Workspace | Workspace | Per-token | Per-org | Per-user |
| **Self-Hostable** | Yes (MIT) | No | No | No | Yes (AGPL) |
| **Price** | $99+/mo | $800+/mo | $2K+/mo | $50K+/yr | Free/custom |

## Key Features

### Must-Have (MVP)
- Iframe-free embedding via Web Component or React SDK
- Row-level security with per-tenant JWT token issuance
- White-labelling: full removal of SDK vendor branding
- Multi-tenancy: workspace or schema-based isolation
- Connection to 3+ major data warehouses (Snowflake, BigQuery, Redshift)
- OIDC/OAuth 2.0 SSO integration
- REST API for programmatic dashboard management

### Should-Have (v1.1)
- Semantic layer: metric definitions, join logic, business logic
- Natural language query for end-customers (LLM-powered "chat with your data")
- Self-serve analytics: end-customers compose and save custom views
- SOC 2 Type II certification
- Analytics as Code: version-controlled semantic layer definitions
- AI-generated dashboard suggestions from customer data schema

### Nice-to-Have (Backlog)
- Automated RLS policy generation and validation from schema inspection
- Semantic layer auto-generation from SQL schema
- Agentic analytics: AI agents surface insights and trigger actions
- WCAG 2.1 AA accessibility compliance
- Cross-tenant anonymised benchmark analytics
- Usage-driven proactive insight surfacing

## Technology Stack

**Backend**: TypeScript/Node.js or Python, ClickHouse or PostgreSQL  
**Frontend**: React, Web Components (framework-agnostic)  
**Semantic Layer**: dbt or custom YAML definitions  
**Data Access**: SQL API, GraphQL API  
**Licensing**: MIT or Apache 2.0 (fully permissive)

## Market Entry Strategy

1. **MVP Launch** (months 1–5): Headless React SDK with multi-tenancy and basic semantic layer
2. **Feature Expansion** (months 6–10): Self-serve Canvas, natural language query, semantic layer depth
3. **Enterprise Push** (months 11–18): SOC 2 certification, governance features, analytics-as-code
4. **Monetization**: Open-source core + managed cloud tier ($99–$2K/month), enterprise support ($5K+/month)

## Why This Matters

- **SaaS differentiation**: Analytics is no longer a nice-to-have; it's an expected feature. Vendors need embedded solutions that ship fast.
- **Build vs. buy bottleneck**: Custom embedded analytics take 6–18 months. No open-source alternative exists that spans from SDK to semantic layer to UI.
- **White-label demand**: Luzmo ($800+/mo), Embeddable ($2K+/mo), and Sisense ($50K+/yr) are all proprietary. An affordable, self-hostable open-source alternative captures SMB/mid-market.
- **AI opportunity**: No tool auto-suggests dashboards from customer data schemas or generates semantic layers from SQL inspection. LLM-powered suggestions would accelerate onboarding from weeks to days.

## Success Metrics

- **Year 1**: 200+ active deployments, $150K ARR from managed cloud + support
- **Year 2**: 1,000+ active deployments, $1M+ ARR; featured in G2 Leaders category
- **Year 3**: 5,000+ active deployments, $5M+ ARR; adopted by 10+ Series B+ SaaS companies
