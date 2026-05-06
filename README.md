# Wiki & Knowledge Base

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source wiki and knowledge base offering collaborative docs, semantic search, an auto-constructed knowledge graph, and full version history.

A team knowledge platform for engineering, support, HR, and operations teams who need a single, trustworthy place to capture and retrieve organisational knowledge. It combines a modern collaborative editor with a permission-aware AI answer bot and automated content governance, addressing the gap between flexible-but-disorganised wikis like Confluence and Notion and specialised AI knowledge layers like Guru.

---

## Why Wiki & Knowledge Base?

- **Confluence** is dominant in the enterprise but is widely criticised for a dated editor, drift toward disorganisation at scale, and limited semantic search relative to AI-native competitors.
- **Notion** is highly flexible but gates AI Q&A and Agents to its USD 20/user/month Business tier, and performance degrades on large databases.
- **Guru** delivers strong AI search but is a specialised layer rather than a complete wiki, and pricing climbs from USD 10 to USD 14+/user/month for full functionality.
- **Obsidian** offers full data ownership but lacks real-time collaboration and enterprise admin features.
- AI search and answer-bot capabilities currently command a significant pricing premium across the market, often doubling base-plan cost; an open-source alternative removes that toll.

---

## Key Features

### Authoring & Organisation

- Rich-text and Markdown editing with templates for standardised pages
- Hierarchical organisation (spaces, collections, or database-style structures)
- Real-time collaboration with comments and @mentions
- Version history and activity log
- Bulk import and export

### Search, Q&A & Knowledge Graph

- AI semantic search with vector embeddings
- Answer bot that responds to natural-language questions with cited sources and confidence scores
- Auto-constructed knowledge graph inferring relationships between documents, teams, and concepts
- Graph view for visualising document relationships
- Cross-tool federation across Slack, Confluence, GitHub wikis, and Notion

### Governance & Content Health

- Document verification workflows so owners confirm accuracy
- Automated stale-content detection with routing to documented owners
- Document quality scoring for freshness, completeness, and accuracy
- Auto-suggestion of updates when related code or systems change
- Role-based access control and permission-aware retrieval

### Collaboration & Delivery

- Onboarding knowledge pathways: AI-generated reading lists and checkpoints by role
- In-workflow knowledge delivery (e.g. browser extension, chat surfaces)
- Slack and Microsoft Teams integration for answer delivery
- Mobile app or responsive web client
- API and webhooks for custom integrations

---

## AI-Native Advantage

Unlike incumbents where AI is a paid add-on bolted onto a legacy editor, this project treats semantic search, the answer bot, and the knowledge graph as core primitives. AI infers document relationships without manual tagging, surfaces stale content automatically, generates personalised onboarding pathways, and federates answers across Slack threads, Confluence pages, GitHub wikis, and Notion in a single response. Studies cited in the research show keyword-only tools achieve 52–58% accuracy on complex queries versus 73%+ for tools with semantic and AI-verification layers.

---

## Tech Stack & Deployment

Expected deployment modes include self-hosted (for enterprises requiring on-premises control comparable to Confluence Data Center) and managed cloud. The platform aligns with established standards: Markdown / CommonMark for authoring, OpenAPI for documenting APIs, and OAuth 2.0, SAML 2.0, and SCIM for enterprise SSO and user provisioning. WCAG 2.2 accessibility compliance is targeted to meet enterprise procurement requirements. DITA support is on the backlog for structured-authoring teams.

---

## Market Context

The knowledge management software market is estimated at USD 13.70 billion in 2025, growing to USD 16.22 billion in 2026, with the narrower knowledge base segment at USD 390.93 million in 2026 (Mordor Intelligence; OpenPR). Incumbent pricing ranges from USD 4–9/user/month at the SMB end (Tettra, Nuclino) to USD 20+/user/month for AI-enabled enterprise tiers (Notion Business, Guru Enterprise). Primary buyers are engineering teams documenting architecture and runbooks, customer-support teams managing FAQs, HR teams maintaining policies, and knowledge managers at consulting and professional-services firms.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
