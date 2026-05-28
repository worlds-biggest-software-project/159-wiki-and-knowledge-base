# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Wiki & Knowledge Base · Created: 2026-05-20

## Philosophy

This model treats every change to every entity as an immutable event appended to a central event store. The event store is the single source of truth; all read-optimised tables (projections) are materialised views derived from replaying events. This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from purpose-built projections.

This approach is inspired by enterprise content management systems where full audit trails are non-negotiable — financial document repositories, regulated industries, and platforms where "who changed what, when, and why" must be answerable instantly. Wikipedia's revision history is conceptually event-sourced (every edit is an immutable revision); this model generalises that principle to all entities, not just page content.

The key insight for a wiki is that content versioning is already a form of event sourcing. This model extends that philosophy to collections, permissions, user profiles, and integrations — every state change is captured, enabling temporal queries ("show me the permission state on March 15th"), full audit compliance, and the ability to add new read models (projections) without modifying the write path.

**Best for:** Enterprise deployments in regulated industries (finance, healthcare, government) where complete audit trails, temporal queries, and compliance reporting are hard requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail of every change ever made
- (+) Temporal queries: reconstruct any entity's state at any point in time
- (+) New read models can be added by replaying events, no schema migration needed
- (+) Natural fit for document versioning (wikis already version content)
- (-) Higher storage requirements (events are never deleted)
- (-) Read model consistency is eventual, not immediate
- (-) More complex to implement and operate than a direct CRUD model
- (-) Debugging requires understanding both events and projections
- (-) Snapshotting needed for entities with many events to avoid slow replays

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CommonMark | Document content captured in `document.updated` events as CommonMark body |
| ISO 30401:2018 | Knowledge lifecycle transitions modelled as explicit events (published, archived, verified) |
| ISO/IEC 27001:2022 | Immutable event store satisfies A.8.15 (Logging) and A.8.17 (Clock synchronisation) |
| GDPR Article 17 | Right to erasure handled via crypto-shredding — user PII encrypted with per-user key; key deletion renders events unreadable |
| SOC 2 Type II | Event store provides the audit evidence required for SOC 2 controls |
| OWASP A09:2021 | Security monitoring events (login, permission changes) logged immutably |
| pgvector | Embeddings stored in a dedicated projection table, rebuilt from content events |

---

## Event Store (Write Side)

```sql
-- The event store is the single source of truth.
-- Events are immutable and append-only.
CREATE TABLE event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(50) NOT NULL,   -- 'document', 'collection', 'user', 'permission'
    stream_id       UUID NOT NULL,          -- ID of the aggregate (document ID, collection ID, etc.)
    version         INTEGER NOT NULL,       -- monotonically increasing per stream
    event_type      VARCHAR(100) NOT NULL,  -- e.g. 'document.created', 'document.content_updated'
    payload         JSONB NOT NULL,         -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"actor_id": "uuid", "ip": "1.2.3.4", "user_agent": "...", "correlation_id": "uuid"}
    workspace_id    UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, version)
);

-- Payload examples for key event types:
--
-- document.created:
-- {
--   "title": "Getting Started Guide",
--   "collection_id": "uuid",
--   "parent_id": null,
--   "body": "# Getting Started\n\nWelcome to...",
--   "status": "draft"
-- }
--
-- document.content_updated:
-- {
--   "title": "Getting Started Guide",
--   "body": "# Getting Started\n\nUpdated welcome...",
--   "change_summary": "Added troubleshooting section"
-- }
--
-- document.published:
-- {
--   "published_at": "2026-05-20T10:00:00Z"
-- }
--
-- permission.granted:
-- {
--   "entity_type": "collection",
--   "entity_id": "uuid",
--   "grantee_type": "user",
--   "grantee_id": "uuid",
--   "permission": "read_write"
-- }

CREATE INDEX idx_event_stream ON event(stream_id, version);
CREATE INDEX idx_event_type ON event(event_type);
CREATE INDEX idx_event_workspace ON event(workspace_id);
CREATE INDEX idx_event_created ON event(created_at);
CREATE INDEX idx_event_stream_type ON event(stream_type, stream_id);

-- Snapshots to avoid replaying long event chains
CREATE TABLE snapshot (
    stream_type     VARCHAR(50) NOT NULL,
    stream_id       UUID NOT NULL,
    version         INTEGER NOT NULL,       -- event version this snapshot was taken at
    state           JSONB NOT NULL,         -- full aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, version)
);
```

## Read-Side Projections

### Workspace & User Projections

```sql
-- Projected from user.created, user.updated, user.suspended events
CREATE TABLE p_workspace (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subdomain       VARCHAR(100) UNIQUE,
    settings        JSONB DEFAULT '{}',
    event_version   INTEGER NOT NULL,       -- last event version applied
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE p_user (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    email           VARCHAR(320) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    role            VARCHAR(20) NOT NULL DEFAULT 'member',
    is_suspended    BOOLEAN NOT NULL DEFAULT FALSE,
    last_active_at  TIMESTAMPTZ,
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_p_user_workspace ON p_user(workspace_id);
CREATE INDEX idx_p_user_email ON p_user(email);

CREATE TABLE p_team (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE p_team_member (
    team_id         UUID NOT NULL,
    user_id         UUID NOT NULL,
    role            VARCHAR(20) NOT NULL DEFAULT 'member',
    PRIMARY KEY (team_id, user_id)
);
```

### Document & Collection Projections

```sql
-- Projected from collection.created, collection.updated, collection.deleted events
CREATE TABLE p_collection (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    description     TEXT,
    icon            VARCHAR(50),
    color           VARCHAR(7),
    permission      VARCHAR(20) NOT NULL DEFAULT 'read_write',
    created_by_id   UUID,
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_p_collection_workspace ON p_collection(workspace_id);

-- Projected from document.* events
-- This is the main read model for document listing and detail views
CREATE TABLE p_document (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    collection_id   UUID,
    parent_id       UUID,
    title           VARCHAR(1000) NOT NULL DEFAULT '',
    slug            VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL DEFAULT '',
    emoji           VARCHAR(10),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    template        BOOLEAN NOT NULL DEFAULT FALSE,
    revision_count  INTEGER NOT NULL DEFAULT 1,
    published_at    TIMESTAMPTZ,
    created_by_id   UUID NOT NULL,
    last_edited_by_id UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    deleted_at      TIMESTAMPTZ,
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    search_vector   TSVECTOR
);

CREATE INDEX idx_p_document_workspace ON p_document(workspace_id);
CREATE INDEX idx_p_document_collection ON p_document(collection_id);
CREATE INDEX idx_p_document_parent ON p_document(parent_id);
CREATE INDEX idx_p_document_search ON p_document USING GIN(search_vector);
CREATE INDEX idx_p_document_status ON p_document(workspace_id, status);

-- Revision history projection (derived from document.content_updated events)
CREATE TABLE p_revision (
    id              UUID PRIMARY KEY,
    document_id     UUID NOT NULL,
    version         INTEGER NOT NULL,
    title           VARCHAR(1000) NOT NULL,
    body            TEXT NOT NULL,
    editor_id       UUID NOT NULL,
    change_summary  TEXT,
    created_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (document_id, version)
);

CREATE INDEX idx_p_revision_document ON p_revision(document_id);
```

### Permission Projection

```sql
-- Materialised from permission.granted and permission.revoked events
CREATE TABLE p_entity_permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(30) NOT NULL,  -- 'collection' or 'document'
    entity_id       UUID NOT NULL,
    grantee_type    VARCHAR(10) NOT NULL,  -- 'user' or 'team'
    grantee_id      UUID NOT NULL,
    permission      VARCHAR(20) NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL,
    event_version   INTEGER NOT NULL
);

CREATE INDEX idx_p_perm_entity ON p_entity_permission(entity_type, entity_id);
CREATE INDEX idx_p_perm_grantee ON p_entity_permission(grantee_type, grantee_id);
```

### Governance Projections

```sql
CREATE TABLE p_verification (
    id              UUID PRIMARY KEY,
    document_id     UUID NOT NULL,
    verified_by_id  UUID NOT NULL,
    status          VARCHAR(20) NOT NULL,
    verified_at     TIMESTAMPTZ NOT NULL,
    next_review_at  TIMESTAMPTZ,
    event_version   INTEGER NOT NULL
);

CREATE INDEX idx_p_verification_document ON p_verification(document_id);
CREATE INDEX idx_p_verification_next ON p_verification(next_review_at);

CREATE TABLE p_document_owner (
    document_id     UUID NOT NULL,
    user_id         UUID NOT NULL,
    assigned_at     TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (document_id, user_id)
);
```

### AI & Search Projections

```sql
CREATE EXTENSION IF NOT EXISTS vector;

-- Rebuilt whenever document.content_updated events are processed
CREATE TABLE p_document_embedding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL,
    chunk_index     INTEGER NOT NULL DEFAULT 0,
    chunk_text      TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,
    model           VARCHAR(100) NOT NULL,
    event_version   INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, chunk_index)
);

CREATE INDEX idx_p_embedding_document ON p_document_embedding(document_id);
CREATE INDEX idx_p_embedding_vector ON p_document_embedding
    USING hnsw (embedding vector_cosine_ops);

-- AI interaction log (also event-sourced but projected for analytics)
CREATE TABLE p_ai_answer (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    user_id         UUID NOT NULL,
    question        TEXT NOT NULL,
    answer          TEXT NOT NULL,
    confidence      FLOAT,
    source_ids      UUID[],
    feedback        VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL
);
```

### Links & Engagement Projections

```sql
CREATE TABLE p_document_link (
    source_id       UUID NOT NULL,
    target_id       UUID NOT NULL,
    PRIMARY KEY (source_id, target_id)
);

CREATE INDEX idx_p_doclink_target ON p_document_link(target_id);

CREATE TABLE p_document_star (
    user_id         UUID NOT NULL,
    document_id     UUID NOT NULL,
    starred_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, document_id)
);

CREATE TABLE p_document_view (
    document_id     UUID NOT NULL,
    user_id         UUID NOT NULL,
    view_count      INTEGER NOT NULL DEFAULT 1,
    last_viewed_at  TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (document_id, user_id)
);
```

### Comments Projection

```sql
CREATE TABLE p_comment (
    id              UUID PRIMARY KEY,
    document_id     UUID NOT NULL,
    parent_id       UUID,
    author_id       UUID NOT NULL,
    body            TEXT NOT NULL,
    resolved_at     TIMESTAMPTZ,
    resolved_by_id  UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_p_comment_document ON p_comment(document_id);
```

### Integration Projections

```sql
CREATE TABLE p_integration (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    type            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    event_version   INTEGER NOT NULL
);

CREATE TABLE p_webhook (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    event_version   INTEGER NOT NULL
);

CREATE TABLE p_api_key (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    user_id         UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    key_hash        TEXT NOT NULL,
    expires_at      TIMESTAMPTZ,
    event_version   INTEGER NOT NULL
);
```

### Attachments Projection

```sql
CREATE TABLE p_attachment (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    document_id     UUID,
    uploaded_by_id  UUID NOT NULL,
    filename        VARCHAR(500) NOT NULL,
    mime_type       VARCHAR(200) NOT NULL,
    byte_size       BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_p_attachment_document ON p_attachment(document_id);
```

---

## Projection Checkpoint Tracking

```sql
-- Tracks which event each projector has processed up to
CREATE TABLE projection_checkpoint (
    projector_name  VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example: Temporal Query

Reconstruct a document's state at a specific point in time by replaying events:

```sql
-- Find all events for a document up to a given timestamp
SELECT event_type, payload, created_at
FROM event
WHERE stream_type = 'document'
  AND stream_id = :document_id
  AND created_at <= :as_of_timestamp
ORDER BY version ASC;

-- The application replays these events against the Document aggregate
-- to reconstruct the exact state at that moment.
```

## Example: Permission Audit

Who had access to a document on a specific date?

```sql
-- Find all permission events for a collection up to a date
SELECT event_type, payload, metadata, created_at
FROM event
WHERE stream_type = 'permission'
  AND (payload->>'entity_id')::UUID = :collection_id
  AND created_at <= :audit_date
ORDER BY version ASC;

-- Replay grants and revocations to compute the permission set at that time.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write) | 2 | event, snapshot |
| Workspace & User Projections | 4 | p_workspace, p_user, p_team, p_team_member |
| Document & Collection Projections | 3 | p_collection, p_document, p_revision |
| Permission Projection | 1 | p_entity_permission |
| Governance Projections | 2 | p_verification, p_document_owner |
| AI & Search Projections | 2 | p_document_embedding, p_ai_answer |
| Links & Engagement Projections | 3 | p_document_link, p_document_star, p_document_view |
| Comments Projection | 1 | p_comment |
| Integration Projections | 3 | p_integration, p_webhook, p_api_key |
| Attachments Projection | 1 | p_attachment |
| Infrastructure | 1 | projection_checkpoint |
| **Total** | **23** | 2 write-side + 21 read-side |

---

## Key Design Decisions

1. **Single event table with stream partitioning** — All events live in one table, partitioned logically by `stream_type` and `stream_id`. This simplifies the write path and enables cross-stream queries (e.g., "all events in workspace X in the last hour"). For very high-volume deployments, PostgreSQL table partitioning by `created_at` range can be added.

2. **JSONB payloads for event flexibility** — Event payloads are JSONB rather than typed columns. This allows new event types to be added without schema migrations. Event schema validation happens at the application layer.

3. **Projection tables mirror the normalized model** — The read-side projections intentionally resemble a traditional relational schema. This means existing SQL queries, ORMs, and reporting tools work against projections exactly as they would against a CRUD database.

4. **Projection checkpointing** — Each projector tracks the last event it processed. If a projector fails, it resumes from the checkpoint rather than replaying all events from the beginning.

5. **Snapshots for long-lived aggregates** — Documents with hundreds of edits benefit from periodic snapshots. Instead of replaying 500 events, load the snapshot at version 450 and replay only the last 50 events.

6. **GDPR compliance via crypto-shredding** — Rather than deleting events (which would violate immutability), user PII in event payloads is encrypted with a per-user key stored in a separate key management table. When a user exercises their right to erasure, the key is destroyed, rendering their PII in events unreadable while preserving the event stream structure.

7. **Eventual consistency trade-off** — Projections may lag behind the event store by milliseconds to seconds. For wiki use cases this is acceptable — a user who edits a document and immediately views it will see their own changes (read-your-own-writes can be guaranteed via the write path returning the new state directly).
