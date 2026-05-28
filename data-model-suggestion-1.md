# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Wiki & Knowledge Base · Created: 2026-05-20

## Philosophy

This model follows the traditional normalized relational approach, inspired by MediaWiki's mature database schema and Confluence's space/page hierarchy. Every concept — workspace, collection, document, revision, comment, permission — has its own dedicated table with strict foreign key relationships. Content versioning follows MediaWiki's proven pattern of separating revision metadata from content storage.

The design prioritises data integrity and query flexibility. Every relationship is explicit, every constraint enforced at the database level. This approach has been battle-tested by Wikipedia (MediaWiki) and thousands of enterprise wikis for over two decades. It excels when the team needs complex cross-entity reporting, regulatory compliance auditing, and the ability to query any dimension of the data without schema changes.

**Best for:** Teams with strong SQL expertise deploying a compliance-sensitive enterprise wiki where data integrity and ad-hoc querying are paramount.

**Trade-offs:**
- (+) Maximum data integrity with foreign keys and constraints
- (+) Easy to reason about; every relationship is visible in the schema
- (+) Excellent for complex reporting and analytics queries
- (+) Well-understood by most database engineers
- (-) More tables means more JOINs for common read paths
- (-) Schema migrations required for every new entity type
- (-) Vector embeddings and knowledge graph features feel bolted on
- (-) Higher write amplification for operations touching many tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CommonMark (spec.commonmark.org) | Document content stored as CommonMark Markdown in the `content` table |
| ISO 30401:2018 | Knowledge lifecycle states (draft, published, archived, stale) modelled as document statuses |
| OAuth 2.0 / SAML 2.0 (RFC 6749) | `auth_provider` table stores SSO provider configurations |
| SCIM 2.0 (RFC 7643/7644) | User provisioning fields aligned with SCIM User schema attributes |
| WCAG 2.2 | Accessibility metadata fields on documents (alt_text support, reading_level) |
| RFC 7763 | Content MIME type stored per content record (`text/markdown` per RFC 7763) |
| Schema.org TechArticle | SEO metadata fields aligned with Schema.org TechArticle vocabulary |
| pgvector | Vector embeddings stored in dedicated `document_embedding` table |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE workspace (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subdomain       VARCHAR(100) UNIQUE,
    logo_url        TEXT,
    default_locale  VARCHAR(10) DEFAULT 'en',
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    role            VARCHAR(20) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('admin', 'member', 'viewer', 'guest')),
    locale          VARCHAR(10),
    timezone        VARCHAR(50),
    is_suspended    BOOLEAN NOT NULL DEFAULT FALSE,
    last_active_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, email)
);

CREATE INDEX idx_user_workspace ON "user"(workspace_id);
CREATE INDEX idx_user_email ON "user"(email);

CREATE TABLE team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_member (
    team_id         UUID NOT NULL REFERENCES team(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('admin', 'member')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);

CREATE TABLE auth_provider (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    provider_type   VARCHAR(30) NOT NULL
                    CHECK (provider_type IN ('saml', 'oidc', 'google', 'github', 'email')),
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,       -- provider-specific config (entity ID, SSO URL, etc.)
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Content Organisation

```sql
CREATE TABLE collection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    description     TEXT,
    icon            VARCHAR(50),
    color           VARCHAR(7),            -- hex color code
    sort_order      INTEGER NOT NULL DEFAULT 0,
    permission      VARCHAR(20) NOT NULL DEFAULT 'read_write'
                    CHECK (permission IN ('read_write', 'read', 'private')),
    created_by_id   UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, slug)
);

CREATE INDEX idx_collection_workspace ON collection(workspace_id);

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    collection_id   UUID REFERENCES collection(id) ON DELETE SET NULL,
    parent_id       UUID REFERENCES document(id) ON DELETE SET NULL,
    title           VARCHAR(1000) NOT NULL DEFAULT '',
    slug            VARCHAR(255) NOT NULL,
    emoji           VARCHAR(10),
    sort_order      FLOAT NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'archived')),
    template        BOOLEAN NOT NULL DEFAULT FALSE,
    full_width      BOOLEAN NOT NULL DEFAULT FALSE,
    published_at    TIMESTAMPTZ,
    created_by_id   UUID NOT NULL REFERENCES "user"(id),
    last_edited_by_id UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ            -- soft delete
);

CREATE INDEX idx_document_workspace ON document(workspace_id);
CREATE INDEX idx_document_collection ON document(collection_id);
CREATE INDEX idx_document_parent ON document(parent_id);
CREATE INDEX idx_document_status ON document(workspace_id, status);
CREATE INDEX idx_document_deleted ON document(deleted_at) WHERE deleted_at IS NOT NULL;
CREATE INDEX idx_document_template ON document(workspace_id, template) WHERE template = TRUE;

-- Full-text search index
ALTER TABLE document ADD COLUMN search_vector TSVECTOR;
CREATE INDEX idx_document_search ON document USING GIN(search_vector);
```

## Content & Versioning (MediaWiki-inspired)

```sql
-- Separating content from document metadata, following MediaWiki's
-- revision -> slot -> content -> text architecture
CREATE TABLE revision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    version         INTEGER NOT NULL,
    editor_id       UUID NOT NULL REFERENCES "user"(id),
    title           VARCHAR(1000) NOT NULL,
    change_summary  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, version)
);

CREATE INDEX idx_revision_document ON revision(document_id);
CREATE INDEX idx_revision_created ON revision(created_at);

CREATE TABLE content (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    revision_id     UUID NOT NULL REFERENCES revision(id) ON DELETE CASCADE,
    body            TEXT NOT NULL,          -- CommonMark Markdown (text/markdown per RFC 7763)
    mime_type       VARCHAR(100) NOT NULL DEFAULT 'text/markdown',
    byte_size       INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_revision ON content(revision_id);
```

## Comments & Reactions

```sql
CREATE TABLE comment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES comment(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES "user"(id),
    body            TEXT NOT NULL,
    resolved_at     TIMESTAMPTZ,
    resolved_by_id  UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comment_document ON comment(document_id);
CREATE INDEX idx_comment_parent ON comment(parent_id);

CREATE TABLE reaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    document_id     UUID REFERENCES document(id) ON DELETE CASCADE,
    comment_id      UUID REFERENCES comment(id) ON DELETE CASCADE,
    emoji           VARCHAR(10) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (document_id IS NOT NULL OR comment_id IS NOT NULL),
    UNIQUE (user_id, document_id, emoji),
    UNIQUE (user_id, comment_id, emoji)
);
```

## Permissions & Access Control

```sql
CREATE TABLE collection_permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collection(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES "user"(id) ON DELETE CASCADE,
    team_id         UUID REFERENCES team(id) ON DELETE CASCADE,
    permission      VARCHAR(20) NOT NULL
                    CHECK (permission IN ('read', 'read_write', 'admin')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (user_id IS NOT NULL OR team_id IS NOT NULL)
);

CREATE INDEX idx_coll_perm_collection ON collection_permission(collection_id);
CREATE INDEX idx_coll_perm_user ON collection_permission(user_id);
CREATE INDEX idx_coll_perm_team ON collection_permission(team_id);

CREATE TABLE document_permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES "user"(id) ON DELETE CASCADE,
    team_id         UUID REFERENCES team(id) ON DELETE CASCADE,
    permission      VARCHAR(20) NOT NULL
                    CHECK (permission IN ('read', 'read_write', 'admin')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (user_id IS NOT NULL OR team_id IS NOT NULL)
);

CREATE INDEX idx_doc_perm_document ON document_permission(document_id);
CREATE INDEX idx_doc_perm_user ON document_permission(user_id);
```

## Templates

```sql
CREATE TABLE template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    collection_id   UUID REFERENCES collection(id) ON DELETE SET NULL,
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,
    icon            VARCHAR(50),
    category        VARCHAR(100),
    created_by_id   UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_template_workspace ON template(workspace_id);
```

## Content Governance

```sql
CREATE TABLE document_owner (
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (document_id, user_id)
);

CREATE TABLE verification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    verified_by_id  UUID NOT NULL REFERENCES "user"(id),
    status          VARCHAR(20) NOT NULL
                    CHECK (status IN ('verified', 'needs_update', 'stale')),
    verified_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    next_review_at  TIMESTAMPTZ,
    notes           TEXT
);

CREATE INDEX idx_verification_document ON verification(document_id);
CREATE INDEX idx_verification_next_review ON verification(next_review_at);

CREATE TABLE stale_content_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    reason          VARCHAR(50) NOT NULL
                    CHECK (reason IN ('age', 'broken_links', 'low_views', 'ai_detected')),
    notified_owner  BOOLEAN NOT NULL DEFAULT FALSE,
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_stale_alert_unresolved ON stale_content_alert(resolved_at)
    WHERE resolved_at IS NULL;
```

## AI & Search

```sql
-- pgvector extension required
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_embedding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL DEFAULT 0,
    chunk_text      TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,   -- OpenAI text-embedding-3-small dimension
    model           VARCHAR(100) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, chunk_index)
);

CREATE INDEX idx_embedding_document ON document_embedding(document_id);
CREATE INDEX idx_embedding_vector ON document_embedding
    USING hnsw (embedding vector_cosine_ops);

CREATE TABLE ai_answer_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id),
    question        TEXT NOT NULL,
    answer          TEXT NOT NULL,
    confidence      FLOAT,
    source_document_ids UUID[] NOT NULL DEFAULT '{}',
    feedback        VARCHAR(20)
                    CHECK (feedback IN ('helpful', 'not_helpful', 'incorrect')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_answer_workspace ON ai_answer_log(workspace_id);
```

## Integrations

```sql
CREATE TABLE integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    type            VARCHAR(50) NOT NULL
                    CHECK (type IN ('slack', 'teams', 'jira', 'github', 'webhook')),
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,        -- integration-specific configuration
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_by_id   UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    secret          TEXT,
    events          TEXT[] NOT NULL,        -- e.g. {'document.created', 'document.updated'}
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_key (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    key_hash        TEXT NOT NULL,          -- bcrypt hash of the API key
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Activity & Audit

```sql
CREATE TABLE activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    actor_id        UUID REFERENCES "user"(id),
    action          VARCHAR(50) NOT NULL,   -- e.g. 'document.created', 'collection.deleted'
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    metadata        JSONB DEFAULT '{}',     -- action-specific details
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_workspace ON activity(workspace_id);
CREATE INDEX idx_activity_entity ON activity(entity_type, entity_id);
CREATE INDEX idx_activity_actor ON activity(actor_id);
CREATE INDEX idx_activity_created ON activity(created_at);
```

## Document Links & Backlinks

```sql
CREATE TABLE document_link (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, target_id)
);

CREATE INDEX idx_doclink_source ON document_link(source_id);
CREATE INDEX idx_doclink_target ON document_link(target_id);

CREATE TABLE document_star (
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, document_id)
);

CREATE TABLE document_view (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    count           INTEGER NOT NULL DEFAULT 1,
    last_viewed_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docview_document ON document_view(document_id);
CREATE INDEX idx_docview_user ON document_view(user_id);
```

## File Attachments

```sql
CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    document_id     UUID REFERENCES document(id) ON DELETE SET NULL,
    uploaded_by_id  UUID NOT NULL REFERENCES "user"(id),
    filename        VARCHAR(500) NOT NULL,
    mime_type       VARCHAR(200) NOT NULL,
    byte_size       BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,          -- S3 / object storage key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_document ON attachment(document_id);
CREATE INDEX idx_attachment_workspace ON attachment(workspace_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 5 | workspace, user, team, team_member, auth_provider |
| Content Organisation | 2 | collection, document |
| Content & Versioning | 2 | revision, content |
| Comments & Reactions | 2 | comment, reaction |
| Permissions | 2 | collection_permission, document_permission |
| Templates | 1 | template |
| Content Governance | 3 | document_owner, verification, stale_content_alert |
| AI & Search | 2 | document_embedding, ai_answer_log |
| Integrations | 3 | integration, webhook, api_key |
| Activity & Audit | 1 | activity |
| Links & Engagement | 3 | document_link, document_star, document_view |
| Attachments | 1 | attachment |
| **Total** | **27** | |

---

## Key Design Decisions

1. **MediaWiki-inspired revision/content split** — Revision metadata (who, when, version number) is separated from content body, allowing efficient revision listing without loading full text.

2. **Adjacency list for document hierarchy** — Documents reference their parent via `parent_id`. While this requires recursive CTEs for tree queries, it is the simplest model and works well with PostgreSQL's `WITH RECURSIVE`:
   ```sql
   WITH RECURSIVE doc_tree AS (
       SELECT id, title, parent_id, 0 AS depth
       FROM document WHERE id = :root_id
       UNION ALL
       SELECT d.id, d.title, d.parent_id, dt.depth + 1
       FROM document d JOIN doc_tree dt ON d.parent_id = dt.id
   )
   SELECT * FROM doc_tree ORDER BY depth;
   ```

3. **Dual permission model** — Permissions are checked at both collection and document level. Collection permissions provide the default, document-level permissions override for specific users/teams.

4. **Soft deletes on documents** — `deleted_at` column enables trash/restore functionality without data loss. Hard deletes happen via a background cleanup job after a retention period.

5. **Dedicated embedding table with chunking** — Documents are chunked and each chunk gets its own embedding vector, enabling paragraph-level semantic search rather than whole-document matching.

6. **Flexible activity log** — The `activity` table uses `entity_type` + `entity_id` polymorphism rather than per-entity audit tables, keeping the schema simple while supporting audit queries across all entity types.

7. **Document links as a first-class table** — Extracted links between documents are stored explicitly, enabling backlink queries and knowledge graph visualisation without parsing content at query time.
