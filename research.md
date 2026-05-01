# Embedded Analytics SDK

> Candidate #33 · Researched: 2026-05-01

## Existing Products and Software Packages

| Tool | Type | Description | Pricing | Strengths / Weaknesses |
|------|------|-------------|---------|------------------------|
| **Sisense (Compose SDK)** | Commercial | Enterprise embedded analytics with a code-first Compose SDK enabling headless, iframe-free embedding; strong multi-tenancy and row-level security | Custom enterprise pricing (historically $50K–$200K+/year) | Strengths: powerful Compose SDK, headless option, developer-friendly, strong SaaS multi-tenancy support. Weaknesses: very expensive, heavyweight deployment, recent financial difficulties and ownership changes |
| **Metabase** | Open Source / Commercial | Popular BI tool with embedded analytics via signed URLs and iframes; white-label requires Pro ($15K+/year) or Enterprise ($30K+/year) | Open source: free; Pro: $15K+/year; Enterprise: $30K+/year | Strengths: easy self-hosting, large community, accessible UI. Weaknesses: iframe-based embedding limits UI control, white-label expensive, multi-tenancy complex to configure |
| **Luzmo (formerly Cumul.io)** | Commercial | Purpose-built SaaS-embedded analytics platform with usage-based pricing; CSS theming, workspace-based multi-tenancy, REST/React SDK | Starter: ~$800–$995/mo; scales with Monthly Active Viewers | Strengths: SaaS-native multi-tenancy, generous viewer pricing, clean SDK. Weaknesses: closed source, Belgian-headquartered (limited US support hours), less flexible than headless tools |
| **Embeddable** | Commercial | Developer-first headless embedded analytics platform launched 2023; first tool positioned as truly "headless" (no iframes, full component control) | White-label embedding from ~$2,000/mo | Strengths: maximum UI control, modern developer experience, no iframe lock-in. Weaknesses: newer/less proven at scale, smaller ecosystem |
| **Qrvey** | Commercial | End-to-end embedded analytics built for multi-tenant SaaS; no-code dashboard management + analytics APIs; AWS-native | Flat-fee annual subscription (custom pricing) | Strengths: purpose-built for multi-tenant SaaS, strong no-code dashboard builder, AWS-native architecture. Weaknesses: less well-known outside US enterprise, AWS-only lock-in |
| **Toucan (ToucanToco)** | Commercial | Narrative-driven embedded analytics with guided data stories; strong white-label | From $1,500/mo (unlimited users/data) | Strengths: unique narrative/storytelling approach, unlimited users, purpose-built for embedding. Weaknesses: prescriptive UX model may not fit all embedding contexts |
| **GoodData** | Commercial | Enterprise embedded analytics with headless option and semantic layer; custom usage-based pricing | Custom (project-specific, scales with client count) | Strengths: semantic layer, strong multi-tenancy, headless/API-first option. Weaknesses: complex setup, pricing not transparent |
| **Cube (Cube.js)** | Open Source / Commercial | Open-source semantic layer and analytical API platform; acts as a headless BI layer that connects to any visualization library | Open source: free (MIT); Cube Cloud: from $99/mo | Strengths: visualization-agnostic (works with Recharts, D3, Chart.js, etc.), strong semantic layer, open source core. Weaknesses: requires custom UI work on top, not a complete embedded dashboard product |
| **Apache Superset / Preset** | Open Source / Commercial | Feature-rich open source BI with iframe embedding; Preset is the managed cloud offering with embedded analytics support | Superset: free (Apache 2.0); Preset: from ~$20/user/mo | Strengths: powerful, active Apache community, extensive chart types. Weaknesses: iframe-based embedding, limited white-label support in open source version, heavy infrastructure requirements |
| **Explo** | Commercial (acquired) | Embedded analytics with unlimited white-label dashboards; acquired by Omni Analytics in December 2025 | Previously $1,995+/mo; future pricing under Omni TBD | Strengths: clean developer API, unlimited white-label. Weaknesses: acquisition uncertainty, future direction unclear |

## Relevant Industry Standards or Protocols

- **OpenID Connect (OIDC) / OAuth 2.0** — Standard authentication and authorization protocols; critical for securely issuing per-tenant embedded tokens (JWT-based) in multi-tenant SaaS contexts.
- **Row-Level Security (RLS)** — Not a formal standard but a near-universal requirement; most embedded analytics platforms implement proprietary RLS mechanisms; lack of standardization is a significant pain point.
- **W3C WCAG 2.1 / 2.2 (Accessibility)** — Web Content Accessibility Guidelines; enterprise buyers increasingly require embedded dashboards to be WCAG AA compliant, which most platforms do not guarantee.
- **ISO/IEC 27001** — Information security management; required by enterprise SaaS buyers integrating analytics into their products; relevant for data residency and tenant isolation guarantees.
- **SOC 2 Type II** — De facto compliance standard for SaaS infrastructure in the US; nearly all commercial embedded analytics vendors must hold this certification.
- **GDPR / CCPA** — Embedded analytics SDKs process end-user data on behalf of the SaaS product's customers; the SaaS vendor is typically a data processor, requiring DPA agreements and data residency controls.
- **OpenAPI 3.x** — REST API specification standard; embedded analytics platforms expose management and query APIs that increasingly follow OpenAPI specifications for SDK generation.
- **Apache Arrow / Parquet** — Columnar data formats enabling efficient data transfer between analytics backends and frontend SDKs; growing adoption as performance standard for embedded analytics query results.

## Available Research Materials

1. Embeddable.com (2026). *Embedded Analytics Tools Pricing Comparison (Spring 2026).* https://embeddable.com/blog/embedded-analytics-pricing-and-benefit-comparison — Industry comparison; vendor-produced but data-rich.

2. Embeddable.com (2026). *Top Embedded Analytics Tools and Platforms for 2026.* https://embeddable.com/blog/top-embedded-analytics-platforms-for-user-facing-analytics — Vendor-produced competitive analysis; useful for landscape mapping.

3. Precedence Research (2025). *Embedded Analytics Market Size to Hit USD 100.98 Billion by 2035.* https://www.precedenceresearch.com/embedded-analytics-market — Market research report; methodology not fully disclosed.

4. Fortune Business Insights (2025). *Embedded Analytics Market Size, Share & Industry Analysis, 2030.* https://www.fortunebusinessinsights.com/embedded-analytics-market-108883 — Market research report; peer-reviewed methodology not confirmed.

5. SNS Insider via GlobeNewswire (2026). *Low-Code Embedded Analytics Market Set to Hit USD 46.45 Billion by 2035.* https://www.globenewswire.com/news-release/2026/03/04/3249044/0/en/Low-Code-Embedded-Analytics-Market-Set-to-Hit-USD-46-45-Billion-by-2035-SNS-Insider.html — Market forecast press release.

6. Toucantoco (2026). *Embedded Analytics: Multi-Tenancy, RLS, Tools & Pricing (2026).* https://www.toucantoco.com/en/blog/embedded-analytics-multi-tenancy-row-level-security-pricing — Practitioner guide; vendor-produced.

7. Holistics.io (2026). *15 Best Embedded Analytics & BI Tools (2026 Edition).* https://www.holistics.io/blog/best-embedded-analytics-tools/ — Comparative vendor review; practitioner audience.

## Market Research

**Market Size:** Market size estimates vary significantly due to differing scope definitions. Key data points from 2025 research:
- Precedence Research: USD 23.41 billion in 2025 → USD 100.98 billion by 2035 at 15.74% CAGR
- Mordor Intelligence: USD 78.45–78.53 billion in 2025 → USD 150.40 billion by 2030 at 13.9% CAGR
- SNS Insider (low-code sub-segment): USD 46.45 billion by 2035

The wide range reflects differing inclusions of BI tooling, OEM licensing, and custom-built embedded dashboards. A conservative mid-range estimate is ~$25–50 billion in 2025 for embedded analytics broadly, growing at 13–16% CAGR through 2030.

**Pricing Landscape:**

| Tier | Representative Tools | Approx. Cost |
|------|---------------------|--------------|
| Open Source / Free | Cube.js (self-hosted), Apache Superset | $0 |
| Developer Entry | Cube Cloud ($99+/mo), Preset (~$20/user/mo) | $99–$500/mo |
| Purpose-Built SaaS Embedded | Luzmo ($800–$995/mo), Toucan ($1,500+/mo), Embeddable ($2,000+/mo), Explo ($1,995+/mo) | $800–$5,000/mo |
| Enterprise | Metabase Enterprise ($30K+/year), GoodData (custom), Sisense (custom, $50K+/year) | $15K–$200K+/year |

**Key Buyer Personas:**
- SaaS product teams needing to embed analytics dashboards for their own customers without building from scratch
- Engineering leaders at B2B SaaS companies seeking to reduce time-to-market for analytics features
- CTOs evaluating build vs. buy for customer-facing reporting, where building custom is estimated at 6–18 months of engineering effort
- Product managers at data-intensive verticals (fintech, HR tech, logistics, healthtech) where analytics is a differentiating feature

**Notable Acquisitions / Funding:**
- Explo acquired by Omni Analytics in December 2025 — signals consolidation in the mid-market embedded analytics space.
- Sisense has had ongoing ownership changes (Vista Equity Partners); financial health concerns have made buyers cautious.
- Luzmo raised Series B funding; remains the most accessible purpose-built option by pricing.
- Cube raised $25M Series B (2022); remains the leading open-source semantic layer for custom embedded analytics builds.

## AI-Native Opportunity

- **AI-generated chart and dashboard suggestions:** Current embedded analytics platforms require SaaS vendors to pre-design all dashboard layouts for their customers. An AI-native SDK could analyze a customer's data schema and usage patterns to proactively suggest the most relevant visualizations and layouts, reducing onboarding time from days to minutes.
- **Natural language query for end-customers:** Existing tools expose fixed dashboards; end-users cannot ask ad-hoc questions. An AI-native embedded SDK with an LLM query layer would allow SaaS products to offer their customers a "chat with your data" experience without the vendor building a custom NLP layer — a feature no current SDK provides out of the box.
- **Automated multi-tenancy RLS configuration:** Setting up correct row-level security across tenants is one of the most error-prone and time-consuming aspects of embedding analytics. AI could validate, suggest, and auto-generate RLS policies from schema inspection, dramatically reducing misconfiguration risk.
- **Semantic layer auto-generation:** Building a semantic layer (metric definitions, joins, business logic) currently requires significant manual effort from data engineers. An LLM trained on SQL schemas and business ontologies could draft semantic layer configurations that human engineers then review, cutting implementation time by 60–80%.
- **Usage-driven insight surfacing:** Rather than waiting for end-users to explore dashboards, an AI-native embedded SDK could proactively push relevant insights to users based on their role, recent activity, and anomalies in their data — turning passive analytics into an active intelligence layer that increases product stickiness.
