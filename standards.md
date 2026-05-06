# Standards & API Reference

> Project: Wiki and Knowledge Base · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management; governs access control, encryption, and audit logging for knowledge bases containing confidential company information, intellectual property, and internal processes; Annex A A.5.10 (Acceptable use of information), A.8.3 (Information access restriction). URL: https://www.iso.org/standard/82875.html

- **ISO 9001:2015 — Quality Management Systems** — Requires organisations to maintain documented information as knowledge evidence; knowledge bases serve as the primary repository for documented procedures, work instructions, and quality records referenced in Clause 7.5. URL: https://www.iso.org/standard/62085.html

- **ISO 30401:2018 — Knowledge Management Systems** — International standard for knowledge management systems in organisations; defines requirements for creating, capturing, organising, retaining, sharing, and applying knowledge; directly motivates enterprise knowledge base implementation requirements. URL: https://www.iso.org/standard/68683.html

### W3C & IETF Standards

- **RFC 7763 / RFC 7764 — MIME Type for Markdown** — IETF RFCs registering the `text/markdown` MIME type and documenting Markdown variants (CommonMark, GitHub Flavored Markdown, MultiMarkdown, Pandoc, Markdown Extra); the standard MIME type for Markdown content in knowledge base APIs. URL: https://datatracker.ietf.org/doc/html/rfc7763

- **CommonMark Specification** — Unambiguous specification and test suite for Markdown (spec.commonmark.org); adopted by GitHub, GitLab, Discourse, Stack Exchange, Codeberg, and major wiki platforms (Wiki.js, Outline) as the canonical Markdown rendering standard. URL: https://spec.commonmark.org/

- **RFC 4287 — Atom Syndication Format** — IETF standard for web feed syndication; used by knowledge bases and wikis for publishing recent changes, new articles, and content updates as Atom feeds consumable by RSS readers and content aggregators. URL: https://datatracker.ietf.org/doc/html/rfc4287

- **RFC 5023 — Atom Publishing Protocol (AtomPub)** — IETF standard for publishing and editing web resources using HTTP and Atom; historically used by wiki platforms for programmatic content creation; now largely superseded by REST/JSON APIs. URL: https://datatracker.ietf.org/doc/html/rfc5023

- **RFC 4685 — Sitemaps Protocol** — XML sitemap format (adopted as a de facto standard by Google, Bing, Yahoo) for declaring crawlable pages in knowledge bases; essential for search engine discoverability of public knowledge bases. URL: https://www.sitemaps.org/protocol.html

- **RFC 6749 — OAuth 2.0** — Authorization framework used by Confluence, Notion, Outline, and Wiki.js for third-party app integrations, SSO provider connections, and API client authentication. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — Standard token format used in knowledge base API authentication flows and user session management. URL: https://datatracker.ietf.org/doc/html/rfc7519

- **W3C WCAG 2.2 — Web Content Accessibility Guidelines** — Governs accessibility of knowledge base web interfaces; public-facing knowledge bases must meet WCAG 2.1 AA minimum (WCAG 2.2 AA recommended); requires keyboard navigation, sufficient contrast, and screen-reader compatibility. URL: https://www.w3.org/TR/WCAG22/

- **Schema.org Article / TechArticle** — Structured data vocabulary for marking up knowledge base articles to improve search engine understanding and rich snippet display; used by public-facing documentation sites. URL: https://schema.org/TechArticle

### Data Model & API Specifications

- **OpenAPI 3.1** — Used by Confluence Cloud v2 API, Outline, and Wiki.js to document their REST management APIs; enables SDK generation, Postman collection import, and automated integration testing. URL: https://spec.openapis.org/oas/latest.html

- **GraphQL** — Used by Outline as its primary query API for pages, collections, and search; enables flexible document tree traversal and full-text search result queries. URL: https://graphql.org/

- **MediaWiki Action API** — MediaWiki's legacy JSON/PHP API (action=query, action=parse, action=edit) used by Wikipedia and thousands of enterprise wikis; extensive but complex; gradually being superseded by the newer MediaWiki REST API (v1). URL: https://www.mediawiki.org/wiki/API

- **MediaWiki REST API v1** — Modern REST/JSON API for MediaWiki (released 2020+); simpler endpoint design for page reading, revision history, searching, and editing; available at `/w/rest.php/v1/`. URL: https://www.mediawiki.org/wiki/API:REST_API

- **WebDAV (RFC 4918)** — Web Distributed Authoring and Versioning; used by some wiki and knowledge base platforms for file attachment management and remote document authoring over HTTP; Confluence supports WebDAV for space export. URL: https://datatracker.ietf.org/doc/html/rfc4918

- **OpenSearch 1.1** — Simple format for sharing search results; used by MediaWiki and other wikis to expose search as a machine-readable description document; enables browser search bar integration and federated knowledge base search. URL: https://github.com/dewitt/opensearch

### Security & Authentication Standards

- **SCIM 2.0 (RFC 7643/7644)** — System for Cross-domain Identity Management; used for automated user provisioning and deprovisioning in enterprise knowledge base deployments (Confluence, Notion Enterprise, Outline) with identity providers (Okta, Azure AD). URL: https://datatracker.ietf.org/doc/html/rfc7643

- **SAML 2.0** — Security Assertion Markup Language; required for SSO integration in enterprise knowledge base deployments; Confluence, Notion, Wiki.js, and Outline all support SAML 2.0 for enterprise tier SSO. URL: https://docs.oasis-open.org/security/saml/v2.0/

- **GDPR Article 32 — Security of Processing** — Knowledge bases containing employee information, personal data, or client-related documentation must implement appropriate access controls and retention policies; internal knowledge bases must comply with data subject access request (DSAR) obligations. URL: https://gdpr-info.eu/art-32-gdpr/

- **OWASP Top 10 (2021) — A05: Security Misconfiguration** — Public-facing knowledge bases are frequently targeted through misconfigured access controls exposing internal pages; A01 (Broken Access Control) directly applies to wikis with space/page permission models. URL: https://owasp.org/Top10/

- **SOC 2 Type II** — De facto enterprise compliance certification required for SaaS knowledge base procurement by large enterprise customers; Notion, Confluence Cloud, and Outline maintain SOC 2 Type II reports. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

### MCP Server Specifications

Knowledge bases are a primary target for MCP integration, enabling AI agents to search and query institutional knowledge:

- **MediaWiki MCP Server (Professional Wiki)** — Open-source MCP server enabling LLM clients to interact with any MediaWiki wiki; supports page reading, editing, summarisation, search, and history retrieval; allows AI tools to create and update pages. URL: https://github.com/ProfessionalWiki/MediaWiki-MCP-Server

- **Wiki.js MCP Server** — Community MCP server (wikijs-mcp) enabling AI systems to access and interact with Wiki.js knowledge bases through MCP; supports article search and content retrieval. URL: https://skywork.ai/skypage/en/wiki-js-mcp-servers-ai-knowledge-base/1980466980427780096

- **Notion MCP Server (Official)** — makenotion/notion-mcp-server v2.2.1 (Notion API 2026-03-11); enables AI agents to search, read, and write Notion pages and databases including knowledge base articles; 4,257+ GitHub stars. URL: https://github.com/makenotion/notion-mcp-server

- **Atlassian MCP Server (Official Remote)** — Atlassian's official remote MCP server covers both Jira and Confluence; enables AI agents to search, read, and create Confluence pages, spaces, and comments. URL: https://developer.atlassian.com/

---

## Similar Products — Developer Documentation & APIs

### Confluence Cloud (Atlassian)

- **Description:** Leading enterprise knowledge base and wiki platform integrated with Atlassian suite (Jira, Bitbucket); space-based organisation with rich text pages; v2 REST API (cursor-based pagination) released 2023; Forge serverless app framework for extensions.
- **API Documentation:** https://developer.atlassian.com/cloud/confluence/rest/v2/intro/
- **SDKs/Libraries:** atlassian-python-api (Python); @atlaskit/api-client (JavaScript); Forge SDK; atlassian-connect (web app framework)
- **Developer Guide:** https://developer.atlassian.com/cloud/confluence/
- **Standards:** REST/JSON (OpenAPI v3, v2 current), OAuth 2.0 (3LO), API tokens, SAML 2.0, SCIM 2.0
- **Authentication:** OAuth 2.0 (3-legged for user context); API token + Basic Auth; Forge JWT for apps

### Notion

- **Description:** Flexible all-in-one workspace with block-based page model; wikis, databases, docs, and AI agents in one platform; REST API for pages, databases, blocks, and users; official MCP server (September 2025 AI Agents with autonomous 20-minute task runs).
- **API Documentation:** https://developers.notion.com/reference/intro
- **SDKs/Libraries:** @notionhq/client (official JavaScript/TypeScript); notion-client (Python); notion-sdk-go
- **Developer Guide:** https://developers.notion.com/
- **Standards:** REST/JSON (OpenAPI 3.1), OAuth 2.0, webhooks (beta), SAML 2.0 (Enterprise)
- **Authentication:** Internal integration tokens; OAuth 2.0 for public integrations; Bearer token

### MediaWiki (Open Source)

- **Description:** Open-source (GPL v2+) wiki engine powering Wikipedia and thousands of enterprise wikis; mature Action API (legacy JSON/PHP) and modern REST API v1; extensible via hooks and extensions; world's most deployed wiki software.
- **API Documentation:** https://www.mediawiki.org/wiki/API and https://www.mediawiki.org/wiki/API:REST_API
- **SDKs/Libraries:** mwclient (Python); mediawiki-api (JavaScript/npm); mwbot (Node.js); pywikibot (Python, Wikipedia ecosystem)
- **Developer Guide:** https://www.mediawiki.org/wiki/Developer_hub
- **Standards:** REST/JSON (REST API v1), Action API (JSON/XML), OpenSearch 1.1, Atom (RFC 4287), SAML 2.0 (via extensions)
- **Authentication:** Session cookies (legacy); OAuth 2.0 (MediaWiki OAuth extension); Bot passwords; API tokens (CSRF tokens for writes)

### Outline (Open Source)

- **Description:** Open-source (BSL 1.1 / self-hosted) team knowledge base with a clean Notion-like editor; real-time collaborative editing; Markdown-based storage; REST and GraphQL APIs; S3-compatible storage for attachments; strong SSO support.
- **API Documentation:** https://www.getoutline.com/developers
- **SDKs/Libraries:** outline (npm community SDK); REST/GraphQL API; Postman collection available
- **Developer Guide:** https://www.getoutline.com/developers
- **Standards:** REST/JSON, GraphQL, OpenAPI, OAuth 2.0, SAML 2.0, OIDC
- **Authentication:** API key (personal access token); OAuth 2.0 for app integrations; SAML 2.0 / OIDC for SSO

### Wiki.js (Open Source)

- **Description:** Open-source (AGPL-3.0) wiki platform with a modern UI; supports Markdown (CommonMark + GFM), WYSIWYG, AsciiDoc, and HTML editors; Git-backed storage option; REST and GraphQL APIs; Kubernetes-ready; active MCP server ecosystem.
- **API Documentation:** https://docs.requarks.io/dev/api
- **SDKs/Libraries:** GraphQL API (primary); REST API endpoints; community MCP server (wikijs-mcp)
- **Developer Guide:** https://docs.requarks.io/
- **Standards:** GraphQL, REST/JSON, OpenAPI (partial), OAuth 2.0, SAML 2.0, LDAP, Git (storage backend)
- **Authentication:** JWT access tokens; OAuth 2.0 / OIDC / SAML 2.0 for SSO; API keys for programmatic access

### Obsidian (Local-first / Plugin API)

- **Description:** Local-first Markdown knowledge base with a rich plugin ecosystem; note graph visualisation; not a team wiki but widely used for personal knowledge management (PKM); extensible via TypeScript plugin API; Obsidian Publish for sharing; Obsidian Sync for cross-device.
- **API Documentation:** https://docs.obsidian.md/
- **SDKs/Libraries:** obsidian.d.ts (TypeScript plugin API); Community plugins (dataview, templater, tasks); obsidian-mcp (community MCP server)
- **Developer Guide:** https://docs.obsidian.md/Home
- **Standards:** CommonMark + extensions (custom properties in YAML frontmatter), Local REST API (via community plugin), Markdown (RFC 7763)
- **Authentication:** Plugin API (no auth for local; Obsidian Sync uses account tokens)

### Bookstack (Open Source)

- **Description:** Open-source (MIT licence) wiki and documentation platform with a hierarchical structure (Shelves → Books → Chapters → Pages); REST API for full CRUD; LDAP/SAML/OAuth2 SSO; strongly typed content model compared to free-form wikis.
- **API Documentation:** https://demo.bookstackapp.com/api/docs
- **SDKs/Libraries:** REST API (JSON); bookstack-api (Python community SDK); Postman collection
- **Developer Guide:** https://www.bookstackapp.com/docs/
- **Standards:** REST/JSON, OpenAPI (partial), OAuth 2.0, SAML 2.0, LDAP
- **Authentication:** API token + token ID (Authorization header); SAML 2.0 / OIDC for SSO

### GitBook

- **Description:** Commercial documentation platform targeting developer-facing public documentation and internal knowledge bases; Git-backed (GitHub/GitLab sync); REST API and webhooks; Markdown-based with WYSIWYG editing; AI-assisted writing and search.
- **API Documentation:** https://developer.gitbook.com/
- **SDKs/Libraries:** REST API (JSON); @gitbook/api (TypeScript); Webhooks
- **Developer Guide:** https://developer.gitbook.com/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, Git (content versioning)
- **Authentication:** API token; OAuth 2.0 for integrations; SAML 2.0 (Enterprise)

---

## Notes

- **CommonMark as the universal Markdown standard**: CommonMark (spec.commonmark.org) has become the de facto canonical Markdown specification; all modern knowledge base platforms (Outline, Wiki.js, GitBook, Obsidian) implement CommonMark + GFM extensions. New platforms should adopt CommonMark as the baseline.

- **MediaWiki MCP Server (2025-2026)**: The Professional Wiki MediaWiki MCP Server is the most mature open-source knowledge base MCP integration; enables AI agents to perform full read/write operations on any MediaWiki instance including Wikipedia.

- **ISO 30401:2018 knowledge management standard**: The existence of an ISO standard specifically for knowledge management systems (30401) provides a framework for enterprise knowledge base requirements that goes beyond technical API considerations to include knowledge creation, capture, and application lifecycle.

- **Git-backed wikis**: Wiki.js and Outline's Git storage backends mean knowledge base content can be version-controlled, diffed, and merged using standard Git workflows; aligns with developer-first knowledge management approaches and enables GitOps-style documentation pipelines.

- **GDPR and public wikis**: Public-facing knowledge bases that include user-contributed content (MediaWiki-style) must implement GDPR-compliant revision deletion, personal data erasure from revision history, and IP anonymisation for anonymous contributors.

- **Open-source landscape**: MediaWiki (GPL v2+), Wiki.js (AGPL-3.0), Outline (BSL 1.1, self-hosted free), Bookstack (MIT), and BookWiki are the primary self-hosted open-source options; Outline and Wiki.js are the modern developer-friendly choices; Bookstack is preferred for structured hierarchical documentation.
