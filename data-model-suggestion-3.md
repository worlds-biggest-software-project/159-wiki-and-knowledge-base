# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Wiki & Knowledge Base · Created: 2026-05-20

## Philosophy

This model uses a compact relational core for identity, hierarchy, and relationships, while pushing variable and domain-specific data into JSONB columns. The principle is: if a field is queried, filtered, or joined on, it gets a proper column; if it varies by context, is display-only, or changes shape over time, it lives in JSONB.

This approach is inspired by Notion's flexible database model (where every page is a block and properties are schema-free) and by Outline's practical design (Sequelize models with PostgreSQL, modest table count, selective JSONB for settings and metadata). It is the fastest path to an MVP that can evolve without constant schema migrations, while still providing the relational integrity needed for permissions, hierarchy, and search.

The JSONB columns in PostgreSQL are not schema-less blobs — they support GIN indexing, containment queries (`@>`), path queries (`->>`, `#>>`), and partial updates (`jsonb_set`). This gives near-relational query performance on structured JSON fields while allowing each workspace, document type, or integration to carry its own shape of metadata.

**Best for:** Startups and small teams building an MVP rapidly, or products that need to support multi-tenant customisation (custom fields, workspace-specific settings) without per-tenant schema changes.

**Trade-offs:**
- (+) Fewest tables — fastest to implement and migrate
- (+) Custom fields and metadata without schema migrations
- (+) JSONB GIN indexes provide strong query performance on structured JSON
- (+) Natural fit for settings, preferences, and integration configs
- (+) Easy to add new document types or content blocks
- (-) JSONB fields lack foreign key enforcement — referential integrity is application-level
- (-) Complex JSONB queries can be harder to optimise than flat column queries
- (-) Reporting tools may struggle with JSONB structure
- (-) Risk of JSONB columns becoming dumping grounds without discipline
- (-) Type safety relies on application validation, not database constraints

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CommonMark | Document body stored as CommonMark Markdown text |
| ISO 30401:2018 | Document lifecycle managed via `status` column with JSONB `governance` metadata |
| OAuth 2.0 / SAML 2.0 | Auth provider config stored in `auth_providers` JSONB array on workspace |
| SCIM 2.0 | User profile attributes stored in `profile` JSONB, aligned with SCIM User schema |
| WCAG 2.2 | Accessibility preferences in user `preferences` JSONB |
| pgvector | Embeddings in dedicated table (vectors cannot live in JSONB) |
| OpenAPI 3.1 | API documentation pages can carry OpenAPI spec metadata in `properties` JSONB |

---

## Core Tables

```sql
-- Workspace: the top-level tenant
CREATE TABLE workspace (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subdomain       VARCHAR(100) UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "logo_url": "https://...",
    --   "default_locale": "en",
    --   "allowed_domains": ["company.com"],
    --   "auth_providers": [
    --     {"type": "saml", "name": "Okta", "entity_id": "...", "sso_url": "..."},
    --     {"type": "google", "name": "Google", "client_id": "..."}
    --   ],
    --   "features": {"ai_answer_bot": true, "stale_detection_days": 90}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_workspace_settings ON workspace USING GIN(settings);

-- User: workspace member
CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(20) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('admin', 'member', 'viewer', 'guest')),
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "avatar_url": "https://...",
    --   "timezone": "America/New_York",
    --   "locale": "en",
    --   "title": "Senior Engineer",
    --   "department": "Engineering",
    --   "scim_external_id": "okta-12345"
    -- }
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "theme": "dark",
    --   "notifications": {"email": true, "slack": true},
    --   "default_collection_id": "uuid"
    -- }
    is_suspended    BOOLEAN NOT NULL DEFAULT FALSE,
    last_active_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, email)
);

CREATE INDEX idx_user_workspace ON "user"(workspace_id);
CREATE INDEX idx_user_email ON "user"(email);
CREATE INDEX idx_user_profile ON "user" USING GIN(profile);

-- Team: group of users for permission assignment
CREATE TABLE team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_member (
    team_id         UUID NOT NULL REFERENCES team(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);
```

## Content

```sql
-- Collection: a grouping of documents (equivalent to Confluence space, Outline collection)
CREATE TABLE collection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    description     TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "icon": "📚",
    --   "color": "#4A90D9",
    --   "default_permission": "read_write",
    --   "sort_order": 0,
    --   "custom_fields_schema": [
    --     {"name": "priority", "type": "select", "options": ["high", "medium", "low"]},
    --     {"name": "owner_team", "type": "text"}
    --   ]
    -- }
    created_by_id   UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, slug)
);

CREATE INDEX idx_collection_workspace ON collection(workspace_id);

-- Document: the primary content entity
CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    collection_id   UUID REFERENCES collection(id) ON DELETE SET NULL,
    parent_id       UUID REFERENCES document(id) ON DELETE SET NULL,
    title           VARCHAR(1000) NOT NULL DEFAULT '',
    slug            VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL DEFAULT '',      -- CommonMark Markdown content
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'archived')),
    template        BOOLEAN NOT NULL DEFAULT FALSE,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "emoji": "🚀",
    --   "full_width": true,
    --   "reading_time_minutes": 5,
    --   "tags": ["onboarding", "engineering"],
    --   "custom_fields": {
    --     "priority": "high",
    --     "owner_team": "Platform"
    --   },
    --   "seo": {
    --     "meta_description": "...",
    --     "canonical_url": "..."
    --   }
    -- }
    governance      JSONB NOT NULL DEFAULT '{}',
    -- governance example:
    -- {
    --   "owners": ["user-uuid-1", "user-uuid-2"],
    --   "last_verified_at": "2026-03-15T10:00:00Z",
    --   "last_verified_by": "user-uuid-1",
    --   "next_review_at": "2026-06-15T10:00:00Z",
    --   "verification_status": "verified",
    --   "stale_detected_at": null,
    --   "quality_score": 0.85
    -- }
    revision_count  INTEGER NOT NULL DEFAULT 1,
    published_at    TIMESTAMPTZ,
    created_by_id   UUID NOT NULL REFERENCES "user"(id),
    last_edited_by_id UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    search_vector   TSVECTOR
);

CREATE INDEX idx_document_workspace ON document(workspace_id);
CREATE INDEX idx_document_collection ON document(collection_id);
CREATE INDEX idx_document_parent ON document(parent_id);
CREATE INDEX idx_document_status ON document(workspace_id, status);
CREATE INDEX idx_document_search ON document USING GIN(search_vector);
CREATE INDEX idx_document_properties ON document USING GIN(properties);
CREATE INDEX idx_document_governance ON document USING GIN(governance);
CREATE INDEX idx_document_tags ON document USING GIN((properties->'tags'));
CREATE INDEX idx_document_deleted ON document(deleted_at) WHERE deleted_at IS NOT NULL;

-- Example: find all documents tagged 'onboarding'
-- SELECT * FROM document WHERE properties->'tags' @> '"onboarding"';

-- Example: find all stale documents
-- SELECT * FROM document WHERE governance->>'verification_status' = 'stale';
```

## Versioning

```sql
-- Revision: stores each version of a document's content
CREATE TABLE revision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    version         INTEGER NOT NULL,
    title           VARCHAR(1000) NOT NULL,
    body            TEXT NOT NULL,
    editor_id       UUID NOT NULL REFERENCES "user"(id),
    change_summary  TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"word_count": 1234, "diff_stats": {"added": 50, "removed": 12}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, version)
);

CREATE INDEX idx_revision_document ON revision(document_id);
```

## Permissions

```sql
-- Unified permission table for both collections and documents
CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    entity_type     VARCHAR(20) NOT NULL CHECK (entity_type IN ('collection', 'document')),
    entity_id       UUID NOT NULL,
    grantee_type    VARCHAR(10) NOT NULL CHECK (grantee_type IN ('user', 'team')),
    grantee_id      UUID NOT NULL,
    level           VARCHAR(20) NOT NULL CHECK (level IN ('read', 'read_write', 'admin')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (entity_type, entity_id, grantee_type, grantee_id)
);

CREATE INDEX idx_permission_entity ON permission(entity_type, entity_id);
CREATE INDEX idx_permission_grantee ON permission(grantee_type, grantee_id);
```

## Comments

```sql
CREATE TABLE comment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES comment(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES "user"(id),
    body            TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"resolved_at": "...", "resolved_by": "uuid", "block_ref": "block-id"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comment_document ON comment(document_id);
```

## AI & Search

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_embedding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL DEFAULT 0,
    chunk_text      TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"model": "text-embedding-3-small", "token_count": 256}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, chunk_index)
);

CREATE INDEX idx_embedding_document ON document_embedding(document_id);
CREATE INDEX idx_embedding_vector ON document_embedding
    USING hnsw (embedding vector_cosine_ops);

CREATE TABLE ai_interaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id),
    interaction     JSONB NOT NULL,
    -- interaction example:
    -- {
    --   "type": "answer_bot",
    --   "question": "How do I set up staging?",
    --   "answer": "To set up staging, follow...",
    --   "confidence": 0.87,
    --   "sources": [{"document_id": "uuid", "title": "Staging Guide", "relevance": 0.92}],
    --   "feedback": "helpful"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_workspace ON ai_interaction(workspace_id);
CREATE INDEX idx_ai_created ON ai_interaction(created_at);
```

## Links, Engagement & Attachments

```sql
CREATE TABLE document_link (
    source_id       UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    PRIMARY KEY (source_id, target_id)
);

CREATE INDEX idx_doclink_target ON document_link(target_id);

CREATE TABLE user_document_state (
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    state           JSONB NOT NULL DEFAULT '{}',
    -- state example:
    -- {
    --   "starred": true,
    --   "starred_at": "2026-05-10T...",
    --   "view_count": 12,
    --   "last_viewed_at": "2026-05-20T...",
    --   "reading_progress": 0.75,
    --   "bookmarks": [{"block_id": "...", "label": "Important section"}]
    -- }
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, document_id)
);

CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    document_id     UUID REFERENCES document(id) ON DELETE SET NULL,
    uploaded_by_id  UUID NOT NULL REFERENCES "user"(id),
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(200) NOT NULL,
    byte_size       BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"width": 800, "height": 600, "alt_text": "Architecture diagram"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_document ON attachment(document_id);
```

## Integrations & Activity

```sql
CREATE TABLE integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    type            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    -- config is entirely integration-specific:
    -- Slack: {"webhook_url": "...", "channel": "#wiki-updates", "events": ["document.published"]}
    -- Jira: {"base_url": "...", "project_key": "WIKI", "api_token_encrypted": "..."}
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_key (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    key_hash        TEXT NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    actor_id        UUID REFERENCES "user"(id),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {"ip": "1.2.3.4", "old_title": "Setup", "new_title": "Setup Guide", "collection_name": "Engineering"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_workspace ON activity(workspace_id, created_at);
CREATE INDEX idx_activity_entity ON activity(entity_type, entity_id);
CREATE INDEX idx_activity_actor ON activity(actor_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity | 4 | workspace, user, team, team_member |
| Content | 2 | collection, document |
| Versioning | 1 | revision |
| Permissions | 1 | permission (unified, polymorphic) |
| Comments | 1 | comment |
| AI & Search | 2 | document_embedding, ai_interaction |
| Links & Engagement | 2 | document_link, user_document_state |
| Attachments | 1 | attachment |
| Integrations & API | 2 | integration, api_key |
| Activity | 1 | activity |
| **Total** | **17** | |

---

## Key Design Decisions

1. **JSONB for variable-shape data, columns for queryable data** — `status`, `workspace_id`, `collection_id`, and `parent_id` are columns because they appear in WHERE clauses and JOINs. Tags, custom fields, owner lists, and verification metadata go in JSONB because their shape varies per workspace and document type. GIN indexes on JSONB columns ensure these remain queryable:
   ```sql
   -- Find documents by tag
   SELECT id, title FROM document
   WHERE properties->'tags' @> '"onboarding"'
     AND workspace_id = :ws_id;

   -- Find documents owned by a user (from governance JSONB)
   SELECT id, title FROM document
   WHERE governance->'owners' @> to_jsonb(:user_id::text);
   ```

2. **Unified permission table** — Instead of separate `collection_permission` and `document_permission` tables, a single `permission` table with `entity_type` polymorphism reduces table count and simplifies permission queries.

3. **Governance in JSONB, not separate tables** — Document ownership, verification, and staleness metadata live inside the `document.governance` JSONB column. This avoids three extra tables while keeping the data queryable via GIN index. The trade-off is that ownership history requires the activity log rather than a dedicated audit trail.

4. **User-document state consolidation** — Stars, view counts, reading progress, and bookmarks for a user-document pair are stored in a single `user_document_state` row with JSONB, rather than separate `star`, `view`, and `bookmark` tables.

5. **17 tables total** — Roughly 40% fewer tables than the fully normalized model. This reduces migration complexity, simplifies the ORM layer, and makes the codebase easier to onboard new developers into.

6. **Integration config as pure JSONB** — Each integration type (Slack, Jira, GitHub) has entirely different configuration fields. JSONB is the natural fit here; a relational approach would require either a sparse wide table or an EAV pattern, both worse than JSONB.

7. **AI interactions as JSONB documents** — The `ai_interaction` table stores the full question/answer/sources/feedback payload as a single JSONB document. This allows the AI feature to evolve its response structure (adding citations, reasoning traces, etc.) without schema changes.
