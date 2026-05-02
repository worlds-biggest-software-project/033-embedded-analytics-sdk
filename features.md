# Embedded Analytics SDK — Feature & Functionality Survey

> Candidate #33 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Sisense (Compose SDK) | Commercial | Proprietary SaaS; custom pricing ($50K–$200K+/year) | https://www.sisense.com |
| Metabase | Open Source / Commercial | AGPL-3.0 (community); Pro $15K+/year; Enterprise $30K+/year | https://www.metabase.com |
| Luzmo (formerly Cumul.io) | Commercial | Proprietary SaaS; from ~$800/mo | https://www.luzmo.com |
| Embeddable | Commercial | Proprietary SaaS; from ~$2,000/mo | https://embeddable.com |
| Qrvey | Commercial | Proprietary SaaS; flat-rate annual subscription | https://qrvey.com |
| Toucan (ToucanToco) | Commercial | Proprietary SaaS; from $1,500/mo | https://www.toucantoco.com |
| GoodData | Commercial | Proprietary SaaS; custom usage-based pricing | https://www.gooddata.com |
| Cube (Cube.js) | Open Source / Commercial | MIT (open-source core); Cube Cloud from $99/mo | https://cube.dev |
| Apache Superset / Preset | Open Source / Commercial | Apache 2.0 (Superset); Preset from ~$20/user/mo | https://superset.apache.org / https://preset.io |
| Explo (now Omni Analytics) | Commercial (acquired) | Proprietary; acquired by Omni Analytics December 2025 | https://www.omni.co |

---

## Feature Analysis by Solution

### Sisense (Compose SDK)

**Core features**
- Compose SDK: headless, code-first embedded analytics — developers build custom UI components that query the Sisense engine, eliminating iframe dependency
- React component library with full TypeScript support
- Multi-tenancy with per-tenant data isolation and row-level security
- AI-powered analytics (GenAI integration with Compose SDK announced 2025): LLM-powered query assistance within developer-built interfaces
- White-labelling: full visual customisation with no Sisense branding constraints
- Single-tenant or multi-tenant SaaS deployment models
- SOC 2 Type II certified

**Differentiating features**
- Compose SDK pioneered the "headless embedded analytics" pattern: full control over presentation layer with Sisense powering the backend query engine
- GenAI integration within Compose SDK (2025) allows developers to embed natural language query capabilities into their applications using Sisense as the data layer
- Most flexible API surface for custom embedded experiences among the commercial tools in this survey

**UX patterns**
- Developer-first: no drag-and-drop dashboard builder for end-users by default; all UI is custom-built using the SDK
- Code examples and a developer portal focus on engineer onboarding
- Complexity scales with customisation: maximum flexibility requires significant engineering effort

**Integration points**
- REST API and React SDK
- Connects to major cloud data warehouses (Snowflake, BigQuery, Redshift, Databricks)
- OIDC/OAuth 2.0 for per-tenant JWT-based authentication
- Anthropic/OpenAI LLM integration for GenAI features

**Known gaps**
- Very high cost ($50K–$200K+/year) limits accessible market to large enterprises
- Company has had ongoing ownership changes and reported financial difficulties under Vista Equity Partners; buyer caution is warranted
- Maximum flexibility requires substantial ongoing engineering investment for maintenance
- Pricing unpredictability with usage-based add-ons noted by multiple user reviews

**Licence / IP notes**
- Proprietary SaaS; no open-source components
- No known patent encumbrances

---

### Metabase

**Core features**
- Open-source BI tool with iframe-based and React SDK-based embedding options
- Interactive Embedding: full Metabase functionality (questions, dashboards, exploration) inside a host application via the React SDK (non-iframe)
- White-labelling: custom branding, navigation, and UX available on Pro and Enterprise plans
- Row-level security via sandbox questions and user attributes for multi-tenant data isolation
- SSO/OIDC support for user provisioning and authentication
- Metabot: AI chat interface allowing end-users to ask questions in plain English
- Self-hosting on any Docker-compatible infrastructure (AGPL-3.0 community edition free)
- Releases: Metabase 56 (August 2025) introduced Embedded Analytics JS and improved theming support

**Differentiating features**
- Largest open-source BI community in this survey; most accessible UI for non-technical end-users
- Metabot (AI query assistant) embedded within customer-facing analytics — allows SaaS products to offer natural language data queries without building a custom NLP layer
- AGPL-3.0 self-hosted free tier means any SaaS company can evaluate full capabilities before paying
- DISABLE_EMBEDDED_SUPERSET_LOGOUT-style feature: authentication managed by host app, no logout button confusion

**UX patterns**
- Non-technical UX: questions built with a GUI query builder rather than SQL by default
- Interactive Embedding provides a fully managed BI experience inside the host app; customers can explore, filter, and drill without the SaaS vendor pre-building every view
- White-label theming via CSS and branding controls on Pro/Enterprise

**Integration points**
- JavaScript SDK for embedding; iframe option for simpler use cases
- 50+ database connectors (PostgreSQL, MySQL, BigQuery, Snowflake, Redshift, MongoDB, etc.)
- OIDC/SAML for SSO
- REST API for programmatic management (creating dashboards, users, permissions)
- Slack and email for scheduled reports and alerts

**Known gaps**
- Iframe-based embedding (the default on free/community tier) limits UI control and inter-frame communication
- White-labelling and multi-tenancy configuration is complex; requires significant setup for production SaaS use cases
- Pro ($15K+/year) and Enterprise ($30K+/year) pricing is high for white-label embedding relative to purpose-built tools like Luzmo
- AGPL-3.0 licence: any network-accessible service built on modified Metabase code must release modifications under AGPL

**Licence / IP notes**
- AGPL-3.0 for community edition: network copyleft; building a competing embedded analytics SaaS on modified Metabase code requires AGPL compliance
- Pro and Enterprise editions are proprietary
- No known patent encumbrances

---

### Luzmo (formerly Cumul.io)

**Core features**
- Purpose-built SaaS embedded analytics; not adapted from a general BI tool
- CSS theming and white-labelling with granular component-level styling
- Workspace-based multi-tenancy: each customer organisation gets its own isolated workspace
- Row-level security via API token configuration; automatic per-tenant data filtering
- React SDK, REST API, and iFrame embedding paths
- Luzmo IQ: natural language querying for end-users (conversational questions about dashboard data)
- Alerts, comments, dashboard sharing, and PDF/scheduled exports
- Usage-based viewer pricing: scales with Monthly Active Viewers (MAV)
- AWS Marketplace listing; Series B funded

**Differentiating features**
- Most purpose-built commercial SaaS option for embedded analytics: designed from the ground up for the SaaS vendor use case, not adapted from enterprise BI
- Luzmo IQ natural language querying embedded within the product — end-customers can ask questions without the SaaS vendor building a custom AI layer
- Workspace-based multi-tenancy is simpler to configure than many competitors' row-level-security-only approaches

**UX patterns**
- Dashboard builder provides a no-code editing interface for SaaS vendors to design templates
- Drag-and-drop component placement with responsive layouts
- End-user interaction via filtered views, drill-down, and (with Luzmo IQ) natural language queries

**Integration points**
- REST API and React SDK for embedding
- Multi-tenancy configured via API (no per-tenant database required)
- Supports most SQL databases and cloud data warehouses as data sources
- AWS Marketplace integration

**Known gaps**
- Closed source; no self-hosting option
- Performance challenges with dataset linking at large scale (comprehensive solution expected Q1 2026 per vendor)
- Limited US support hours due to Belgian headquarters
- Less flexible than headless tools (Cube, Embeddable) for developers who need full component-level control

**Licence / IP notes**
- Proprietary SaaS; closed source
- No known patent encumbrances

---

### Embeddable

**Core features**
- Truly headless embedded analytics: developers define data models and UI components; no iframes
- Web Component and React/Vue embed targets — native to the host app's DOM
- Lightning-fast data service with sub-second query performance via built-in caching
- No-code dashboard builder for end-users: SaaS vendors can give customers a "Custom Canvas" to self-serve their own dashboards within developer-set guardrails
- Custom Canvas: customers compose charts and data models into fully custom saved views
- Security: secure, lightweight web component with per-session token-based access control
- Launched 2023; focused on developer experience and modern embedding patterns

**Differentiating features**
- Positioned as the first embedded analytics tool explicitly built as "truly headless" — the presentation layer is the host app's own UI, not an embedded BI layer
- Custom Canvas (self-serve by end-customers) within guardrails set by the SaaS vendor: a unique balance between vendor control and customer flexibility
- No iframe dependency is a meaningful technical differentiation for applications requiring deep host-app integration

**UX patterns**
- Developer-first: component definitions in code; data models separate from presentation
- End-user self-serve analytics via Custom Canvas: customers build and save their own dashboard views

**Integration points**
- Web Component and React/Vue SDK for embedding
- Built-in integration with Cube.js semantic layer for organisations already using Cube
- REST API for data source and model management

**Known gaps**
- Newer and less proven at scale than Metabase, Luzmo, or GoodData
- Smaller ecosystem and customer reference base
- Higher base price (~$2,000/mo) than some competitors for comparable embedded viewer counts
- Limited public documentation on specific compliance certifications (SOC 2, ISO 27001)

**Licence / IP notes**
- Proprietary SaaS; closed source
- No known patent encumbrances

---

### Qrvey

**Core features**
- Multi-tenant embedded analytics built specifically for SaaS companies
- Native data lake (Elasticsearch-backed) providing cost-effective multi-tenant query scaling
- Semantic layer for business logic abstraction
- No-code dashboard builder for SaaS vendors and (optionally) their end-customers
- Workflow automation: analytics-triggered actions and workflows across the SaaS ecosystem
- Agentic AI capabilities (2025–2026): AI agents that drive action based on analytics insights
- Flat-rate pricing with no per-user or per-viewer fees (unique among commercial embedded tools)
- AWS-native architecture; AWS Marketplace available
- Recognised as a Technology Innovation Leader in Embedded BI by Dresner Advisory Services (2025)

**Differentiating features**
- Flat-rate pricing model eliminates per-viewer cost escalation that afflicts most competitors — significant advantage for high-volume SaaS products with many end-users
- Native data lake architecture: SaaS vendors ingest customer data into Qrvey's Elasticsearch layer, enabling multi-tenant queries without complex per-tenant database isolation
- Agentic AI layer: AI agents that can take actions (not just surface insights) based on analytics — a capability no other tool in this survey has fully productised
- Workflow automation integrated with analytics: dashboards can trigger actions in the host SaaS application

**UX patterns**
- No-code dashboard builder for SaaS vendor administrators and (with configuration) for end-customers
- AWS-native deployment; SaaS vendors deploy Qrvey within their own AWS account

**Integration points**
- AWS-native: deployed in the SaaS vendor's own AWS environment
- REST API and embedded widget library
- JWT-based per-session authentication tokens
- Connects to S3, RDS, and other AWS data sources natively

**Known gaps**
- AWS-only architecture is a hard constraint; not suitable for GCP- or Azure-native SaaS products
- Less well-known outside US enterprise market; smaller community and fewer public case studies
- Complex initial setup due to AWS-native architecture
- Vendor-hosted data lake (Elasticsearch) adds data residency complexity for EU customers

**Licence / IP notes**
- Proprietary SaaS; closed source
- No known patent encumbrances; AWS Marketplace listing confirms AWS infrastructure dependency

---

### Toucan (ToucanToco)

**Core features**
- Narrative-driven embedded analytics: guided data stories with contextual commentary
- White-label with unlimited users and unlimited data (pricing at $1,500/mo is per-deployment, not per-viewer)
- Purpose-built for embedding with a focus on business user accessibility
- Guided story format: analytics presented as a narrative with annotations and contextual explanation
- Strong white-labelling: full brand customisation with no Toucan branding

**Differentiating features**
- Narrative/storytelling approach is unique in this survey: analytics presented as guided stories rather than free-form dashboards
- Unlimited users in pricing model is competitive for SaaS products with large, distributed end-user bases

**UX patterns**
- Prescriptive story-driven UX: less open-ended than traditional dashboard builders
- Designed for business users who are not data analysts

**Integration points**
- REST API and embed SDK
- Standard database and data warehouse connectors
- CSS white-labelling

**Known gaps**
- Prescriptive narrative UX model may not fit use cases requiring ad-hoc exploration or self-serve analytics
- Smaller ecosystem and less developer documentation than Luzmo, Metabase, or GoodData
- EU-headquartered with limited US presence; US support hours may be constrained

**Licence / IP notes**
- Proprietary SaaS; closed source
- No known patent encumbrances

---

### GoodData

**Core features**
- Enterprise-grade headless BI with a full semantic layer (metric definitions, join logic, business ontologies)
- Analytics as Code: semantic layer and dashboard definitions managed in version control via YAML/JSON
- Context Management (2026): AI-augmented semantic modelling with governance and observability for AI data retrieval
- MCP Server (Q1 2026): agentic analytics via Model Context Protocol, enabling AI agents to query GoodData's semantic layer
- GoodData AI (GA May 2025): AI at every layer of the data stack from ingestion to governed insight delivery
- SOC 2 Type II and ISO 27001 certified
- Multi-tenancy with workspace isolation per customer

**Differentiating features**
- GoodData AI positions the platform as "AI-native" from the data layer up — unique among enterprise embedded analytics tools in having AI integrated into the semantic layer itself, not just the query interface
- MCP Server (Q1 2026) is among the earliest embedded analytics platforms to expose analytics via the Model Context Protocol for agentic AI workflows
- Analytics as Code: version-controlled semantic layer definitions allow teams to manage analytics configuration like software — a DevOps-friendly differentiator
- Identified in the GigaOm 2025 Semantic Layer Radar as a universal semantic layer provider delivering flexibility and preventing vendor lock-in

**UX patterns**
- Headless first: presentation layer is the host application's own UI, driven by GoodData APIs
- Developer-facing semantic layer configuration; business users interact via the embedded components or AI query interface
- Context Management exposes governance and observability tooling for data teams managing AI retrieval quality

**Integration points**
- REST API, GraphQL API, and SQL API for embedding
- React SDK and web components
- OIDC/OAuth 2.0 for per-tenant JWT tokens
- MCP Server for AI agent integration
- Connects to major data warehouses (Snowflake, BigQuery, Redshift, Databricks)

**Known gaps**
- Complex setup: semantic layer configuration requires significant data engineering expertise
- Pricing not transparent; custom usage-based pricing creates budget uncertainty
- UI for end-users is less polished than Metabase or Luzmo for non-technical customers
- Smaller community and publicly available documentation than Metabase or Cube

**Licence / IP notes**
- Proprietary SaaS; closed source
- No known patent encumbrances

---

### Cube (Cube.js)

**Core features**
- Open-source semantic layer: metric definitions, joins, and business logic defined once and consumed by any BI, analytics, or AI tool
- Multiple APIs: REST, GraphQL, and SQL — any downstream tool can query via its preferred protocol
- Cube Cloud managed service from $99/mo; self-hosted community edition MIT-licensed
- AI API (Cube Agentic Analytics, GA 2025): turnkey integration with Anthropic Claude and other LLMs; 200+ companies using it within three months of GA; processed 500,000+ lines of semantic layer code
- Dashboard embedding via iframe with semantic layer governance
- New visualisation components (2025): KPI Progress Indicators, Sparklines, Rich Text Labels, interactive maps
- Sub-second query performance via built-in caching and pre-aggregations
- Partnership with Embeddable: Cube semantic layer + Embeddable frontend for headless embedded analytics

**Differentiating features**
- Visualization-agnostic: works with any charting library (Recharts, D3, Chart.js, Victory, Nivo) — no UI lock-in
- AI API as a first-class semantic layer feature: LLMs query the semantic layer directly, giving every AI integration access to governed, trusted metrics
- MIT-licensed open-source core: the most permissive licence in this survey for the semantic layer component
- Agentic Analytics (2026): AI agents that can query, reason over, and take action based on the semantic layer

**UX patterns**
- Developer-first: semantic layer definitions in YAML/JavaScript; no end-user UI out of the box
- Requires pairing with a visualisation library or an embedded analytics frontend (Embeddable, Metabase, Superset)
- Playground for testing queries and schema during development

**Integration points**
- REST, GraphQL, SQL APIs
- SDKs for JavaScript, React, Vue, Angular
- Native connectors for Snowflake, BigQuery, Databricks, Redshift, PostgreSQL, and 30+ others
- Embeddable integration for full headless embedded analytics stack
- Anthropic Claude and other LLMs via AI API
- dbt semantic layer integration

**Known gaps**
- Not a complete embedded dashboard product: requires additional frontend work or a pairing with an embedded analytics tool
- End-user self-serve analytics requires building a complete UI layer on top
- Cube Cloud pricing can scale unpredictably for high query volumes
- Smaller professional services ecosystem compared to Sisense or GoodData

**Licence / IP notes**
- MIT licence for the open-source core: permissive; no copyleft constraints
- Cube Cloud is proprietary
- No known patent encumbrances

---

### Apache Superset / Preset

**Core features**
- Apache Superset 6.0 (released 2025): complete design system overhaul (Ant Design v5), comprehensive dark mode, and hundreds of quality-of-life improvements
- Embedded analytics via Embedded SDK (iframes with guest token-based access control)
- Row-level security for multi-tenant data isolation
- Full theming support for embedded analytics and brand customisation (v6.0)
- DISABLE_EMBEDDED_SUPERSET_LOGOUT feature flag (February 2026): hides logout button in embedded mode; authentication managed by host app via SSO
- Drill to Detail and Drill By in embedded mode (v6.0)
- 40+ chart types; SQL editor for advanced users
- Preset (managed cloud): AI-native BI features built on Superset; simplifies deployment

**Differentiating features**
- Apache 2.0 licence: most permissive open-source licence in this survey; no copyleft constraints; freely embeddable in proprietary products
- Active Apache Software Foundation stewardship with strong community governance and long-term longevity
- Superset 6.0 represents the most significant UI overhaul since the project's inception; dark mode and Ant Design v5 bring it to modern UI standards

**UX patterns**
- Rich chart builder for data analysts; SQL editor for advanced queries
- Embedded iframe with guest tokens for end-user-facing embedding
- Preset adds simplified onboarding and AI features on top of Superset

**Integration points**
- Embedded SDK (iframe + guest tokens)
- 40+ database connectors
- OIDC/SAML for SSO
- REST API for programmatic management
- Preset (managed cloud) adds CI/CD pipelines and team management

**Known gaps**
- Iframe-based embedding limits UI integration and inter-frame communication for host applications requiring deep integration
- White-labelling in the open-source version is limited: removing Superset branding requires significant custom CSS/configuration work
- Multi-tenancy configuration (row-level security) is complex and error-prone
- Heavy infrastructure requirements for self-hosted production deployment (Celery workers, Redis, database)
- Limited per-viewer pricing controls for SaaS vendors embedding for their own customers

**Licence / IP notes**
- Apache 2.0: highly permissive; building a competing or embedding product on Superset requires only preserving the copyright notice and licence file
- No known patent encumbrances

---

### Explo (now Omni Analytics)

**Core features**
- Pre-acquisition: clean developer API; unlimited white-label dashboards
- No-code dashboard builder for SaaS vendors
- Usage-based pricing (previously $1,995+/mo)
- Acquired by Omni Analytics in December 2025; future product direction under Omni TBD

**Differentiating features**
- Unlimited white-label at a fixed price was a differentiator before acquisition
- Omni Analytics is a modern BI tool with strong semantic layer capabilities; acquisition may result in enhanced embedded analytics within Omni's broader offering

**UX patterns**
- Pre-acquisition: self-service dashboard creation via drag-and-drop
- Post-acquisition: unknown; Omni's UX is SQL-first, targeted at data analysts

**Integration points**
- Pre-acquisition: REST API; standard database connectors
- Post-acquisition: Omni's connector ecosystem

**Known gaps**
- Acquisition uncertainty is the dominant risk: product roadmap, pricing, and customer support commitments are unclear under Omni ownership
- Omni's SQL-first analyst focus may not align with Explo's SaaS vendor embedded use case

**Licence / IP notes**
- Proprietary SaaS; closed source
- No known patent encumbrances

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Row-level security (RLS) for per-tenant data isolation in multi-tenant SaaS deployments
- JWT-based authentication tokens issued at runtime for per-session, per-tenant access control
- White-labelling: ability to remove all vendor branding and apply the host application's visual identity
- At least one iframe-free embedding path (React SDK, Web Component, or headless API) for modern integrations
- Connection to major cloud data warehouses (Snowflake, BigQuery, Redshift, Databricks at minimum)
- OIDC/OAuth 2.0 integration for SSO with the host application's identity provider
- SOC 2 Type II certification for enterprise buyer compliance requirements
- REST API for programmatic management of dashboards, users, and permissions

### Differentiating Features
- Headless / component-level architecture: full presentation-layer control for the SaaS vendor without iframe constraints
- Semantic layer: metric definitions, joins, and business logic defined once and consumed by all analytics surfaces
- Natural language query (NL2SQL or NL2Chart) for end-customers: AI-powered chat with their own data within the embedded product
- Self-serve analytics: end-customers build and save their own dashboard views within vendor-defined guardrails (Custom Canvas / self-serve mode)
- Agentic analytics: AI agents that take action based on analytical insights (Qrvey, GoodData MCP Server, Cube AI API are early examples)
- Flat-rate viewer pricing: no per-user or per-MAV cost escalation (Qrvey's differentiator)
- Analytics as Code: version-controlled semantic layer and dashboard definitions for DevOps-aligned teams

### Underserved Areas / Opportunities
- Automated RLS configuration from schema inspection: setting up correct row-level security is consistently identified as the most error-prone and time-consuming aspect of embedded analytics deployment; no tool provides AI-assisted RLS policy generation or validation
- Semantic layer auto-generation from SQL schemas: building metric definitions and join logic currently requires significant manual data engineering effort; LLM-based schema inspection could dramatically reduce this
- AI-generated dashboard suggestions from usage patterns: SaaS vendors must pre-design all dashboard layouts; no tool proactively suggests the most relevant visualisations for a new customer's data schema
- WCAG 2.1/2.2 AA accessibility compliance: enterprise buyers increasingly require this but most platforms do not guarantee or document it
- Cross-tenant benchmark analytics: SaaS vendors have an opportunity to show customers how their metrics compare to anonymised peer cohorts — a high-value feature no current tool provides natively
- Usage-driven proactive insight surfacing: pushing relevant insights to end-users based on role, activity, and data anomalies (as opposed to waiting for users to open dashboards)

### AI-Augmentation Candidates
- Automated RLS policy generation and validation from schema inspection
- Semantic layer auto-generation from SQL schema and business ontology (reducing data engineering effort by 60–80%)
- AI-generated dashboard layout suggestions from customer data schema and historical usage patterns
- Natural language query for end-customers ("chat with your data") as an embedded SDK feature
- Usage-driven proactive insight surfacing: push relevant anomalies and trends to users based on role and recent activity
- Cross-tenant anonymised benchmark generation for SaaS product insights

---

## Legal & IP Summary

The open-source tools in this survey use distinct licences with different implications for a new entrant. Cube.js uses the MIT licence (permissive; no copyleft), and Apache Superset uses the Apache 2.0 licence (permissive; no copyleft beyond attribution). Metabase uses AGPL-3.0 for its community edition: any network-accessible service built on modified Metabase code must release those modifications under AGPL-3.0. Building a competing embedded analytics SaaS on Cube.js or Superset carries no copyleft obligation; building on Metabase does. All commercial tools (Sisense, Luzmo, Embeddable, Qrvey, Toucan, GoodData, Explo/Omni) are proprietary SaaS products with closed source code; they present no licence compatibility concern for a new entrant but their APIs may not be freely re-implemented. No patent-encumbered techniques were identified: row-level security, JWT-based multi-tenancy, semantic layer, and iframe embedding are published and widely practised techniques. The GoodData MCP Server and Cube AI API represent novel integration patterns as of 2026, but the Model Context Protocol (MCP) is an open standard published by Anthropic. WCAG 2.1/2.2 is a regulatory and procurement requirement, not IP. The Explo acquisition by Omni Analytics (December 2025) introduces customer contract uncertainty; any SaaS vendor currently on Explo should review their agreement terms carefully.

---

## Recommended Feature Scope

**Must-have (MVP)**:
- Iframe-free embedding path: Web Component or React SDK with no iframe dependency for deep host-app integration
- Row-level security (RLS) with per-tenant JWT token issuance at runtime
- White-labelling: full removal of SDK vendor branding; CSS theming to match host application identity
- Multi-tenancy architecture: workspace or schema-based tenant isolation with automatic data filtering
- Connection to at least three major data warehouses (Snowflake, BigQuery, Redshift/PostgreSQL)
- OIDC/OAuth 2.0 SSO integration for host application identity propagation
- REST API for programmatic dashboard, user, and permission management

**Should-have (v1.1)**:
- Semantic layer: metric definitions, join logic, and business logic defined once and reused across all embedded surfaces
- Natural language query for end-customers: LLM-powered "chat with your data" within the embedded component
- Self-serve analytics: end-customers can compose and save custom dashboard views within vendor-defined guardrails
- SOC 2 Type II certification
- Analytics as Code: version-controlled semantic layer and dashboard definitions (YAML/JSON)
- AI-generated dashboard suggestions from customer data schema for new customer onboarding

**Nice-to-have (backlog)**:
- Automated RLS policy generation and validation from schema inspection (AI-assisted)
- Semantic layer auto-generation from SQL schema using LLM inspection
- Agentic analytics: AI agents that surface insights and trigger actions in the host SaaS application based on data anomalies
- WCAG 2.1 AA accessibility compliance with documented audit results
- Cross-tenant anonymised benchmark analytics for SaaS product peer comparison
- Usage-driven proactive insight surfacing: push relevant anomalies and trends to end-users based on role and recent activity
