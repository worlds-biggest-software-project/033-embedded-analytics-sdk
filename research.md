# Embedded Analytics SDK

> Candidate #033 · Researched: 2026-05-03

## Existing Products and Software Packages

- **Reveal** (Commercial) - True embedded analytics SDK with deep UI control; native integration (.NET, Angular, React, Blazor). Non-iframe, no vendor branding.
- **ThoughtSpot Visual Embed SDK** (Commercial) - Cloud-based BI with embedded SDKs; search-driven analytics; headless option available.
- **Sisense Compose SDK** (Commercial) - Code-first approach to embedding; headless BI architecture. Advanced white-label capabilities.
- **Looker (Google Cloud)** - Embedded dashboards via iFrame and SDK; Google ecosystem integration; commercial with open source components.
- **Tableau Developer Embed** (Commercial) - Embedding via JavaScript SDK; limited vs. commercial Tableau.
- **Power BI Embedded** (Commercial, Microsoft) - Azure-native embedding; tight Office 365 integration; costly licensing model.
- **Explo** (Commercial) - Purpose-built embedded analytics; SDKs for React/Vue; white-label with customization.
- **Metabase** (open source + commercial) - Dashboards embeddable via iFrame; Metabase Enterprise for more control.

## Relevant Industry Standards or Protocols

- **iFrame Embedding** - Common but limited; sandboxed content, vendor branding visible, limited customization.
- **JavaScript SDK Standards** - Modern approach for headless embedding; control over UI/UX, better performance, multi-framework support (React, Vue, Angular, Svelte).
- **REST API / GraphQL APIs** - Headless BI approach; maximum flexibility; developers build custom UI.
- **Multi-Tenancy Standards** - Row-level security (RLS), customer isolation, per-customer branding and permissions.
- **Authentication Standards** - OAuth 2.0, SAML, JWT for secure embedding; identity provider integration.
- **Component Library Standards** - Composable analytics allowing embedding of individual charts vs. full dashboards.

## Available Research Materials

- **"White Label Embedded Analytics: Complete Guide for SaaS 2025"** (Knowi) - Comprehensive guide covering technical and business considerations.
- **"The Best White Label Analytics Platform Of 2026"** (Reveal BI) - Platform comparison and selection guide.
- **"A Pocket Guide to Embedded Business Intelligence 2026"** (Holistics) - Architecture patterns and integration strategies.
- **"Top Embedded Analytics Tools and Platforms 2026"** (Embeddable.com) - Comparative review of major vendors.
- **"8 Best Embedded BI Solutions of 2025"** (Explo) - Feature and pricing comparison.
- **"20 Best White Label Business Intelligence Software 2025"** (DotnetReport) - Broad market survey.
- **Sisense vs. ThoughtSpot White-Labeling** - Multiple detailed comparisons available.

## Market Research

- **Market Size**: Embedded analytics market estimated at $1.5-2B globally; growing 18-20% CAGR (2024-2030).
- **Growth Drivers**: SaaS companies adding analytics to products (competitive necessity), customer data visibility demands, white-label requirements, reduced analytics infrastructure costs.
- **Key Buyer Personas**: SaaS product teams, CTOs integrating analytics, product managers, data teams at SaaS companies, ISVs building analytics-heavy applications.
- **Pain Points**: Performance and latency (end-user experience), licensing costs (per-embedding user), customization limitations, vendor lock-in, managing customer data governance.
- **Pricing Models**: Per-embedding user ($1-$50+ per user/month), usage-based (queries, data volume), enterprise annual contracts ($50K-$500K+).
- **Market Events**: Composable analytics trend (component-level vs. full-dashboard embedding) gaining traction; vendors moving to headless architectures (2024-2025); SaaS buyer preferences shifting to code-first SDKs vs. iFrame.

## AI-Native Opportunity

- **Intelligent Dashboard Recommendation**: LLM understanding user role, company size, and data schema to automatically generate optimal embedded dashboard configurations.
- **Self-Tuning Performance**: ML system monitoring embedded analytics performance and auto-optimizing query caching, data refresh intervals, and chart rendering based on usage patterns.
- **Natural Language Querying**: LLM-powered natural language interface for embedded analytics, allowing end-users to ask questions without SQL or BI tool knowledge.
- **Predictive Insights Engine**: AI-powered anomaly detection and predictive indicators automatically surfaced in embedded dashboards (e.g., "Revenue on track for 105% of target").
- **Dynamic Theming & UX**: LLM-informed system automatically adjusting embedded analytics UI based on end-user preferences, accessibility needs, and context.
