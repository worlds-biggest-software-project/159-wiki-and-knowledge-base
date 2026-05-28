# Data Model Suggestion 4: Graph-Relational (Knowledge Graph Native)

> Project: Wiki & Knowledge Base · Created: 2026-05-20

## Philosophy

This model combines a conventional relational schema for operational CRUD (users, collections, documents, permissions) with a property graph layer for modelling the knowledge graph — the network of relationships between documents, concepts, people, teams, and external entities. The graph layer uses a `graph_node` / `graph_edge` pattern stored in PostgreSQL, augmented by pgvector for semantic similarity edges.

The insight is that a wiki's most differentiating feature — and the one competitors like Notion and Confluence handle poorly — is the knowledge graph: the web of relationships between documents, the concepts they discuss, the people who own them, and the teams they serve. Obsidian and Nuclino recognise this with their graph views, but their graphs are limited to explicit backlinks between documents. An AI-native knowledge base should automatically discover and surface latent relationships: "this API documentation is semantically related to this architecture decision record, which was written by the same team that owns this runbook."

The graph is stored in PostgreSQL rather than a dedicated graph database (Neo4j) to avoid operational complexity. PostgreSQL's recursive CTEs handle multi-hop traversals for the depths typical of a wiki graph (3-5 hops). For extremely deep traversals, Apache AGE (a PostgreSQL extension providing openCypher support) can be added later without changing the storage layer.

**Best for:** Products where the auto-constructed knowledge graph, relationship visualisation, and AI-powered concept linking are the primary differentiators — the "organisational memory" use case.

**Trade-offs:**
- (+) First-class knowledge graph enables powerful discovery and visualisation
- (+) AI can automatically create concept nodes and semantic edges
- (+) Graph queries surface non-obvious relationships across documents
- (+) Same PostgreSQL deployment — no separate graph database to operate
- (+) Flexible edge types allow new relationship kinds without schema changes
- (-) Graph queries (multi-hop traversals) can be expensive on large graphs
- (-) Dual model (relational + graph) increases conceptual complexity
- (-) Graph data must be kept in sync with relational document changes
- (-) Recursive CTEs have limits compared to native graph databases
- (-) More storage for the graph layer (nodes + edges + embeddings)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CommonMark | Document content stored as CommonMark Markdown |
| ISO 30401:2018 | Knowledge lifecycle modelled as graph edges (document --verified_by--> user, --owned_by--> team) |
| Schema.org | Concept taxonomy aligned with Schema.org Thing hierarchy where applicable |
| SKOS (W3C) | Concept relationships use SKOS vocabulary: broader, narrower, related |
| pgvector | Semantic similarity edges computed from vector cosine distance |
| RDF-inspired | Graph node/edge pattern draws from RDF triple stores (subject, predicate, object) |
| OAuth 2.0 / SAML 2.0 | Standard auth; auth provider config in JSONB |
| WCAG 2.2 | Accessibility compliance for graph visualisation (keyboard-navigable, screen-reader labels) |

---

## Relational Core: Identity & Multi-Tenancy

```sql
CREATE TABLE workspace (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subdomain       VARCHAR(100) UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
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
    profile         JSONB NOT NULL DEFAULT '{}',
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
    role            VARCHAR(20) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);
```

## Relational Core: Content

```sql
CREATE TABLE collection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    description     TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
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
    body            TEXT NOT NULL DEFAULT '',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'archived')),
    template        BOOLEAN NOT NULL DEFAULT FALSE,
    properties      JSONB NOT NULL DEFAULT '{}',
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
CREATE INDEX idx_document_search ON document USING GIN(search_vector);
CREATE INDEX idx_document_status ON document(workspace_id, status);
CREATE INDEX idx_document_deleted ON document(deleted_at) WHERE deleted_at IS NOT NULL;

CREATE TABLE revision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    version         INTEGER NOT NULL,
    title           VARCHAR(1000) NOT NULL,
    body            TEXT NOT NULL,
    editor_id       UUID NOT NULL REFERENCES "user"(id),
    change_summary  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, version)
);

CREATE INDEX idx_revision_document ON revision(document_id);
```

## Relational Core: Permissions, Comments, Attachments

```sql
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

CREATE TABLE comment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES comment(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES "user"(id),
    body            TEXT NOT NULL,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comment_document ON comment(document_id);

CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    document_id     UUID REFERENCES document(id) ON DELETE SET NULL,
    uploaded_by_id  UUID NOT NULL REFERENCES "user"(id),
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(200) NOT NULL,
    byte_size       BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_document ON attachment(document_id);
```

## Knowledge Graph Layer

```sql
-- graph_node represents any entity in the knowledge graph.
-- Some nodes are "shadows" of relational entities (documents, users, teams);
-- others are pure graph entities (concepts, topics, external resources).
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    node_type       VARCHAR(50) NOT NULL,
    -- node_type values:
    --   'document'  — shadow of a document row
    --   'concept'   — AI-extracted or user-defined concept (e.g. "Kubernetes", "CI/CD")
    --   'person'    — shadow of a user row
    --   'team'      — shadow of a team row
    --   'topic'     — broader topic grouping (e.g. "Infrastructure", "Security")
    --   'external'  — external resource (URL, API, external doc)
    --   'tag'       — user-created tag
    source_id       UUID,                  -- FK to the relational entity if this is a shadow node
    label           VARCHAR(500) NOT NULL,  -- display label
    description     TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example for concept node:
    -- {
    --   "aliases": ["k8s", "kube"],
    --   "definition": "Container orchestration platform",
    --   "wikipedia_url": "https://en.wikipedia.org/wiki/Kubernetes",
    --   "confidence": 0.95,
    --   "source": "ai_extracted"
    -- }
    embedding       vector(1536),          -- concept embedding for semantic similarity
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gnode_workspace ON graph_node(workspace_id);
CREATE INDEX idx_gnode_type ON graph_node(node_type);
CREATE INDEX idx_gnode_source ON graph_node(source_id) WHERE source_id IS NOT NULL;
CREATE INDEX idx_gnode_label ON graph_node(workspace_id, label);
CREATE INDEX idx_gnode_properties ON graph_node USING GIN(properties);
CREATE INDEX idx_gnode_embedding ON graph_node
    USING hnsw (embedding vector_cosine_ops) WHERE embedding IS NOT NULL;

-- graph_edge represents a typed, weighted, optionally directional relationship
-- between two nodes in the knowledge graph.
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- edge_type values (inspired by SKOS and domain-specific relations):
    --   'links_to'         — document A contains a link to document B
    --   'mentions'         — document mentions a concept
    --   'broader'          — concept A is a broader term for concept B (SKOS)
    --   'narrower'         — concept A is a narrower term for concept B (SKOS)
    --   'related'          — general semantic relatedness (SKOS)
    --   'authored_by'      — document authored by person
    --   'owned_by'         — document owned by person or team
    --   'verified_by'      — document verified by person
    --   'depends_on'       — document depends on another document
    --   'supersedes'       — document supersedes an older document
    --   'similar_to'       — AI-detected semantic similarity (with weight = cosine similarity)
    weight          FLOAT NOT NULL DEFAULT 1.0,  -- edge weight / strength / confidence
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "source": "ai_extracted",
    --   "confidence": 0.87,
    --   "context": "mentioned in section 'Architecture Overview'",
    --   "detected_at": "2026-05-20T10:00:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_workspace ON graph_edge(workspace_id);
CREATE INDEX idx_gedge_source ON graph_edge(source_node_id);
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id);
CREATE INDEX idx_gedge_type ON graph_edge(edge_type);
CREATE INDEX idx_gedge_weight ON graph_edge(weight DESC);
CREATE INDEX idx_gedge_pair ON graph_edge(source_node_id, target_node_id, edge_type);
```

## AI & Semantic Search

```sql
CREATE EXTENSION IF NOT EXISTS vector;

-- Document chunk embeddings for semantic search (separate from graph node embeddings)
CREATE TABLE document_embedding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL DEFAULT 0,
    chunk_text      TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,
    model           VARCHAR(100) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, chunk_index)
);

CREATE INDEX idx_embedding_document ON document_embedding(document_id);
CREATE INDEX idx_embedding_vector ON document_embedding
    USING hnsw (embedding vector_cosine_ops);

-- AI concept extraction jobs — tracks which documents have been processed
CREATE TABLE concept_extraction_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    revision_version INTEGER NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    concepts_found  INTEGER DEFAULT 0,
    edges_created   INTEGER DEFAULT 0,
    error           TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_concept_job_document ON concept_extraction_job(document_id);
CREATE INDEX idx_concept_job_status ON concept_extraction_job(status);

-- AI answer log
CREATE TABLE ai_answer_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id),
    question        TEXT NOT NULL,
    answer          TEXT NOT NULL,
    confidence      FLOAT,
    source_ids      UUID[],
    graph_paths     JSONB,
    -- graph_paths example: paths the AI traversed to find the answer
    -- [
    --   {"nodes": ["doc-uuid", "concept-uuid", "doc-uuid-2"], "edges": ["mentions", "mentioned_in"]},
    --   {"nodes": ["doc-uuid", "doc-uuid-3"], "edges": ["links_to"]}
    -- ]
    feedback        VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_answer_workspace ON ai_answer_log(workspace_id);
```

## Integrations & Activity

```sql
CREATE TABLE integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
    type            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_workspace ON activity(workspace_id, created_at);
CREATE INDEX idx_activity_entity ON activity(entity_type, entity_id);
```

## Engagement

```sql
CREATE TABLE document_star (
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, document_id)
);

CREATE TABLE document_view (
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    view_count      INTEGER NOT NULL DEFAULT 1,
    last_viewed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (document_id, user_id)
);
```

---

## Example Graph Queries

### Find all concepts mentioned in a document

```sql
SELECT gn.label, gn.description, ge.weight
FROM graph_edge ge
JOIN graph_node gn ON gn.id = ge.target_node_id
WHERE ge.source_node_id = (
    SELECT id FROM graph_node WHERE source_id = :document_id AND node_type = 'document'
)
AND ge.edge_type = 'mentions'
ORDER BY ge.weight DESC;
```

### Find related documents via shared concepts (2-hop traversal)

```sql
-- "Documents that discuss the same concepts as this document"
WITH doc_concepts AS (
    SELECT ge.target_node_id AS concept_id
    FROM graph_edge ge
    JOIN graph_node gn ON gn.id = ge.source_node_id
    WHERE gn.source_id = :document_id
      AND gn.node_type = 'document'
      AND ge.edge_type = 'mentions'
),
related_docs AS (
    SELECT gn2.source_id AS related_document_id,
           COUNT(*) AS shared_concepts,
           AVG(ge2.weight) AS avg_relevance
    FROM doc_concepts dc
    JOIN graph_edge ge2 ON ge2.target_node_id = dc.concept_id
                       AND ge2.edge_type = 'mentions'
    JOIN graph_node gn2 ON gn2.id = ge2.source_node_id
                       AND gn2.node_type = 'document'
                       AND gn2.source_id != :document_id
    GROUP BY gn2.source_id
)
SELECT d.id, d.title, rd.shared_concepts, rd.avg_relevance
FROM related_docs rd
JOIN document d ON d.id = rd.related_document_id
ORDER BY rd.shared_concepts DESC, rd.avg_relevance DESC
LIMIT 10;
```

### Build a concept hierarchy (SKOS broader/narrower)

```sql
WITH RECURSIVE concept_tree AS (
    SELECT gn.id, gn.label, 0 AS depth
    FROM graph_node gn
    WHERE gn.id = :root_concept_id

    UNION ALL

    SELECT child.id, child.label, ct.depth + 1
    FROM concept_tree ct
    JOIN graph_edge ge ON ge.source_node_id = ct.id AND ge.edge_type = 'narrower'
    JOIN graph_node child ON child.id = ge.target_node_id
    WHERE ct.depth < 5  -- limit traversal depth
)
SELECT * FROM concept_tree ORDER BY depth, label;
```

### Find semantic neighbours (vector similarity on graph nodes)

```sql
-- Find concepts semantically similar to a given concept
SELECT gn.label, gn.description,
       1 - (gn.embedding <=> (SELECT embedding FROM graph_node WHERE id = :concept_id)) AS similarity
FROM graph_node gn
WHERE gn.workspace_id = :workspace_id
  AND gn.node_type = 'concept'
  AND gn.id != :concept_id
  AND gn.embedding IS NOT NULL
ORDER BY gn.embedding <=> (SELECT embedding FROM graph_node WHERE id = :concept_id)
LIMIT 10;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | workspace, user, team, team_member |
| Content | 3 | collection, document, revision |
| Permissions | 1 | permission (unified) |
| Comments | 1 | comment |
| Attachments | 1 | attachment |
| Knowledge Graph | 2 | graph_node, graph_edge |
| AI & Search | 3 | document_embedding, concept_extraction_job, ai_answer_log |
| Integrations & API | 2 | integration, api_key |
| Activity | 1 | activity |
| Engagement | 2 | document_star, document_view |
| **Total** | **20** | |

---

## Key Design Decisions

1. **Shadow nodes for relational entities** — Documents, users, and teams exist both in their relational tables (for CRUD operations) and as nodes in `graph_node` (for graph queries). The `source_id` column links them. When a document is created or updated, a background job ensures its shadow node and edges are synchronised.

2. **SKOS vocabulary for concept relationships** — The W3C Simple Knowledge Organization System (SKOS) provides a well-defined vocabulary for concept hierarchies: `broader`, `narrower`, `related`. Using standard edge types means the graph is interoperable with other knowledge systems and can be exported as RDF/SKOS if needed.

3. **AI-extracted vs user-created edges** — The `properties.source` field on edges distinguishes AI-extracted relationships (confidence < 1.0) from user-created ones (confidence = 1.0). This allows the UI to show AI-inferred relationships differently and lets users confirm or dismiss them.

4. **Vector embeddings on graph nodes** — Concept nodes carry their own embeddings, enabling "find similar concepts" queries without joining to a separate embedding table. This powers the knowledge graph's ability to suggest relationships that are semantically similar but not explicitly linked.

5. **Concept extraction job tracking** — The `concept_extraction_job` table ensures each document revision is processed exactly once by the AI concept extractor, preventing duplicate edges and enabling retry on failure.

6. **Graph traversal in PostgreSQL** — Recursive CTEs handle 3-5 hop traversals efficiently for wiki-scale graphs (thousands to low millions of nodes). If the graph grows to tens of millions of nodes, Apache AGE can be added as a PostgreSQL extension for openCypher query support without migrating data.

7. **Graph paths in AI answers** — The `ai_answer_log.graph_paths` field records which graph traversals the AI used to find its answer. This provides explainability: "I found this answer by following: Document A --mentions--> Concept 'Kubernetes' --mentioned_in--> Document B."
