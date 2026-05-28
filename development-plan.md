# Wiki & Knowledge Base — Phased Development Plan

> Project: 159-wiki-and-knowledge-base · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript (Node.js 22 LTS) | Full-stack unification with the frontend; mature async I/O for handling concurrent document edits; excellent library ecosystem for LLM SDKs, Markdown processing, and WebSocket-based collaboration |
| API framework | Fastify 5 | High-performance HTTP with built-in OpenAPI 3.1 schema generation via `@fastify/swagger`; standards.md lists OpenAPI as the reference API format used by Confluence, Outline, and Wiki.js |
| Real-time collaboration | Yjs + y-websocket | CRDT-based collaborative editing; battle-tested in Outline and many collaborative editors; enables offline-capable editing that syncs on reconnect |
| Database | PostgreSQL 16 + pgvector | Relational core for identity, permissions, and content; pgvector for semantic search embeddings; all four data model suggestions target PostgreSQL; aligns with Outline's stack |
| ORM / query builder | Drizzle ORM | Type-safe SQL with TypeScript inference; generates migration files; supports pgvector column types; lightweight compared to Prisma |
| Full-text search | PostgreSQL tsvector + GIN indexes | Built-in; avoids external search infra for MVP; upgradeable to Elasticsearch/Meilisearch later |
| Vector embeddings | OpenAI text-embedding-3-small (1536d) | Industry-standard embedding model; switchable via adapter pattern; pgvector HNSW indexes for ANN queries |
| LLM (answer bot) | Claude API (Anthropic) | Best-in-class for knowledge synthesis and citation; long context window for multi-document RAG; switchable via adapter |
| Task queue | BullMQ (Redis-backed) | Async processing for embedding generation, concept extraction, stale-content detection, webhook delivery; Redis also serves as cache and pub/sub layer |
| Cache | Redis 7 | Session cache, rate limiting, real-time presence, BullMQ backend |
| Object storage | S3-compatible (MinIO for self-hosted) | File attachments, document exports; S3 API is the universal standard for object storage |
| Frontend | Next.js 15 (App Router) | Server components for SEO-friendly public pages; client components for the editor; shares TypeScript types with backend; Vercel-deployable for SaaS mode |
| Editor | Tiptap 3 (ProseMirror) | Extensible rich-text editor with Markdown import/export; supports collaborative editing via Yjs; used by Outline; CommonMark-compatible per standards.md |
| Graph visualisation | D3.js (force-directed) | Lightweight, customisable graph rendering for the knowledge graph view; no heavyweight dependency |
| Authentication | NextAuth.js v5 + custom SAML/OIDC | OAuth 2.0 (RFC 6749), SAML 2.0, OIDC for enterprise SSO per standards.md; JWT (RFC 7519) for API sessions; SCIM 2.0 (RFC 7643) for user provisioning |
| Containerisation | Docker + docker-compose | Self-hosted deployment; multi-service orchestration (API, worker, PostgreSQL, Redis, MinIO) |
| Testing | Vitest + Playwright | Vitest for unit/integration (fast, native ESM); Playwright for E2E browser tests |
| Code quality | ESLint 9 + Prettier + TypeScript strict | Flat config ESLint; Prettier for formatting; TypeScript strict mode with no `any` |
| Package manager | pnpm 9 | Fast, disk-efficient, workspace support for monorepo |
| Monorepo | pnpm workspaces + Turborepo | Shared types package between frontend and backend; parallel builds |
| CI/CD | GitHub Actions | Lint, typecheck, test, build Docker image on every PR; publish images on release |
| Accessibility | WCAG 2.2 AA | Required by standards.md; enforced via axe-core in Playwright tests |

### Data Model Selection

Data Model Suggestion 3 (Hybrid Relational + JSONB) is selected as the foundation. Rationale:
- 17 tables (fewest) accelerates MVP delivery
- JSONB columns for governance, properties, and settings avoid premature schema rigidity
- Unified `permission` table reduces complexity
- GIN-indexed JSONB supports the governance and tagging queries required by the answer bot and stale-content detection
- Graph tables from Suggestion 4 are added in Phase 8 for the knowledge graph feature (2 additional tables: `graph_node`, `graph_edge`)

### Project Structure

```
wiki-knowledge-base/
├── package.json                        # Root workspace config
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── release.yml
├── packages/
│   └── shared/                         # Shared TypeScript types & utilities
│       ├── package.json
│       └── src/
│           ├── types/
│           │   ├── workspace.ts
│           │   ├── user.ts
│           │   ├── document.ts
│           │   ├── collection.ts
│           │   ├── permission.ts
│           │   ├── comment.ts
│           │   ├── revision.ts
│           │   ├── graph.ts
│           │   └── api.ts
│           ├── constants.ts
│           └── index.ts
├── apps/
│   ├── api/                            # Fastify backend API
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts                # Server entrypoint
│   │       ├── config.ts               # Environment config with defaults
│   │       ├── db/
│   │       │   ├── schema.ts           # Drizzle schema definitions
│   │       │   ├── migrations/         # Generated migration SQL files
│   │       │   └── connection.ts
│   │       ├── routes/
│   │       │   ├── auth.ts
│   │       │   ├── workspaces.ts
│   │       │   ├── collections.ts
│   │       │   ├── documents.ts
│   │       │   ├── comments.ts
│   │       │   ├── search.ts
│   │       │   ├── ai.ts
│   │       │   ├── graph.ts
│   │       │   ├── integrations.ts
│   │       │   └── webhooks.ts
│   │       ├── services/
│   │       │   ├── auth.service.ts
│   │       │   ├── document.service.ts
│   │       │   ├── collection.service.ts
│   │       │   ├── permission.service.ts
│   │       │   ├── search.service.ts
│   │       │   ├── embedding.service.ts
│   │       │   ├── answer-bot.service.ts
│   │       │   ├── graph.service.ts
│   │       │   ├── governance.service.ts
│   │       │   ├── integration.service.ts
│   │       │   └── webhook.service.ts
│   │       ├── workers/
│   │       │   ├── embedding.worker.ts
│   │       │   ├── stale-detection.worker.ts
│   │       │   ├── concept-extraction.worker.ts
│   │       │   ├── webhook-delivery.worker.ts
│   │       │   └── link-extraction.worker.ts
│   │       ├── middleware/
│   │       │   ├── auth.ts
│   │       │   ├── permission.ts
│   │       │   ├── rate-limit.ts
│   │       │   └── workspace.ts
│   │       └── lib/
│   │           ├── markdown.ts
│   │           ├── slugify.ts
│   │           ├── chunker.ts
│   │           └── errors.ts
│   ├── web/                            # Next.js frontend
│   │   ├── package.json
│   │   ├── next.config.ts
│   │   ├── tailwind.config.ts
│   │   └── src/
│   │       ├── app/
│   │       │   ├── layout.tsx
│   │       │   ├── page.tsx
│   │       │   ├── (auth)/
│   │       │   │   ├── login/page.tsx
│   │       │   │   └── signup/page.tsx
│   │       │   └── (workspace)/
│   │       │       ├── [workspace]/
│   │       │       │   ├── layout.tsx
│   │       │       │   ├── page.tsx          # Collection listing
│   │       │       │   ├── search/page.tsx
│   │       │       │   ├── graph/page.tsx
│   │       │       │   ├── ask/page.tsx      # Answer bot UI
│   │       │       │   ├── settings/
│   │       │       │   └── [collection]/
│   │       │       │       ├── page.tsx
│   │       │       │       └── [document]/
│   │       │       │           └── page.tsx
│   │       │       └── ...
│   │       ├── components/
│   │       │   ├── editor/
│   │       │   │   ├── Editor.tsx
│   │       │   │   ├── Toolbar.tsx
│   │       │   │   └── extensions/
│   │       │   ├── layout/
│   │       │   ├── search/
│   │       │   ├── graph/
│   │       │   └── ui/
│   │       ├── hooks/
│   │       └── lib/
│   │           ├── api-client.ts
│   │           └── auth.ts
│   └── worker/                         # BullMQ worker process
│       ├── package.json
│       └── src/
│           └── index.ts
└── tests/
    ├── fixtures/
    │   ├── documents/
    │   ├── workspaces/
    │   └── seed.ts
    ├── unit/
    ├── integration/
    └── e2e/
```

---

## Phase 1: Foundation — Project Scaffold, Database, and Configuration

### Purpose

Establish the monorepo structure, database schema, configuration system, and development tooling. After this phase, a developer can clone the repo, run `pnpm install && docker-compose up`, and have a running (empty) API server connected to PostgreSQL and Redis.

### Tasks

#### 1.1 — Monorepo Scaffold and Tooling

**What**: Create the pnpm workspace with three apps (api, web, worker) and one shared package, plus all configuration files.

**Design**:

Root `package.json`:
```json
{
  "name": "wiki-knowledge-base",
  "private": true,
  "packageManager": "pnpm@9.15.0",
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "lint": "turbo lint",
    "typecheck": "turbo typecheck",
    "test": "turbo test",
    "test:e2e": "turbo test:e2e",
    "db:migrate": "pnpm --filter api db:migrate",
    "db:generate": "pnpm --filter api db:generate"
  }
}
```

`pnpm-workspace.yaml`:
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

`turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": { "dependsOn": ["^build"] },
    "typecheck": { "dependsOn": ["^build"] },
    "test": { "dependsOn": ["^build"] }
  }
}
```

Root `tsconfig.json` (base):
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

**Testing**:
- `Unit: pnpm install completes without errors`
- `Unit: turbo build compiles all packages and apps`
- `Unit: turbo lint passes with zero warnings`
- `Unit: turbo typecheck passes with zero errors`

---

#### 1.2 — Configuration System

**What**: Define environment-variable-driven configuration with typed defaults, validation, and `.env.example`.

**Design**:

`apps/api/src/config.ts`:
```typescript
import { z } from 'zod';

const configSchema = z.object({
  // Server
  PORT: z.coerce.number().default(3001),
  HOST: z.string().default('0.0.0.0'),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace']).default('info'),

  // Database
  DATABASE_URL: z.string().url(),

  // Redis
  REDIS_URL: z.string().url().default('redis://localhost:6379'),

  // Auth
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRES_IN: z.string().default('7d'),
  SESSION_SECRET: z.string().min(32),

  // S3 / Object Storage
  S3_ENDPOINT: z.string().url().optional(),
  S3_BUCKET: z.string().default('wiki-attachments'),
  S3_REGION: z.string().default('us-east-1'),
  S3_ACCESS_KEY: z.string().optional(),
  S3_SECRET_KEY: z.string().optional(),

  // LLM / Embeddings
  OPENAI_API_KEY: z.string().optional(),
  ANTHROPIC_API_KEY: z.string().optional(),
  EMBEDDING_MODEL: z.string().default('text-embedding-3-small'),
  EMBEDDING_DIMENSIONS: z.coerce.number().default(1536),

  // Feature Flags
  ENABLE_AI_SEARCH: z.coerce.boolean().default(false),
  ENABLE_ANSWER_BOT: z.coerce.boolean().default(false),
  ENABLE_KNOWLEDGE_GRAPH: z.coerce.boolean().default(false),
  STALE_DETECTION_DAYS: z.coerce.number().default(90),

  // Rate Limiting
  RATE_LIMIT_MAX: z.coerce.number().default(100),
  RATE_LIMIT_WINDOW_MS: z.coerce.number().default(60_000),
});

export type Config = z.infer<typeof configSchema>;

export function loadConfig(): Config {
  const result = configSchema.safeParse(process.env);
  if (!result.success) {
    const formatted = result.error.format();
    console.error('Invalid environment configuration:', formatted);
    process.exit(1);
  }
  return result.data;
}
```

**Testing**:
- `Unit: loadConfig with all required vars → returns typed Config object`
- `Unit: loadConfig with missing DATABASE_URL → exits with error listing missing field`
- `Unit: loadConfig with invalid PORT (non-numeric) → exits with validation error`
- `Unit: default values applied when optional vars omitted → ENABLE_AI_SEARCH=false, PORT=3001`
- `Unit: LOG_LEVEL outside enum → validation error`

---

#### 1.3 — Database Schema and Migrations

**What**: Implement the database schema (based on Data Model Suggestion 3: Hybrid Relational + JSONB) using Drizzle ORM, generate and apply initial migration.

**Design**:

`apps/api/src/db/schema.ts` (core tables — excerpt of key entities):
```typescript
import {
  pgTable, uuid, varchar, text, boolean, integer, bigint,
  timestamp, jsonb, index, uniqueIndex, pgEnum
} from 'drizzle-orm/pg-core';
import { vector } from 'drizzle-orm/pg-core'; // pgvector support

// Enums
export const userRoleEnum = pgEnum('user_role', ['admin', 'member', 'viewer', 'guest']);
export const docStatusEnum = pgEnum('doc_status', ['draft', 'published', 'archived']);
export const permLevelEnum = pgEnum('perm_level', ['read', 'read_write', 'admin']);
export const entityTypeEnum = pgEnum('entity_type', ['collection', 'document']);
export const granteeTypeEnum = pgEnum('grantee_type', ['user', 'team']);

// Workspace
export const workspace = pgTable('workspace', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  subdomain: varchar('subdomain', { length: 100 }).unique(),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// User
export const user = pgTable('user', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  email: varchar('email', { length: 320 }).notNull(),
  name: varchar('name', { length: 255 }).notNull(),
  role: userRoleEnum('role').notNull().default('member'),
  profile: jsonb('profile').notNull().default({}),
  preferences: jsonb('preferences').notNull().default({}),
  passwordHash: text('password_hash'),
  isSuspended: boolean('is_suspended').notNull().default(false),
  lastActiveAt: timestamp('last_active_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  workspaceEmail: uniqueIndex('idx_user_workspace_email').on(table.workspaceId, table.email),
  workspaceIdx: index('idx_user_workspace').on(table.workspaceId),
  emailIdx: index('idx_user_email').on(table.email),
}));

// Team
export const team = pgTable('team', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  metadata: jsonb('metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const teamMember = pgTable('team_member', {
  teamId: uuid('team_id').notNull().references(() => team.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => user.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 20 }).notNull().default('member'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  pk: uniqueIndex('pk_team_member').on(table.teamId, table.userId),
}));

// Collection
export const collection = pgTable('collection', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 255 }).notNull(),
  description: text('description'),
  settings: jsonb('settings').notNull().default({}),
  createdById: uuid('created_by_id').references(() => user.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  workspaceSlug: uniqueIndex('idx_collection_ws_slug').on(table.workspaceId, table.slug),
  workspaceIdx: index('idx_collection_workspace').on(table.workspaceId),
}));

// Document
export const document = pgTable('document', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  collectionId: uuid('collection_id').references(() => collection.id, { onDelete: 'set null' }),
  parentId: uuid('parent_id').references((): any => document.id, { onDelete: 'set null' }),
  title: varchar('title', { length: 1000 }).notNull().default(''),
  slug: varchar('slug', { length: 255 }).notNull(),
  body: text('body').notNull().default(''),
  status: docStatusEnum('status').notNull().default('draft'),
  template: boolean('template').notNull().default(false),
  properties: jsonb('properties').notNull().default({}),
  governance: jsonb('governance').notNull().default({}),
  revisionCount: integer('revision_count').notNull().default(1),
  publishedAt: timestamp('published_at', { withTimezone: true }),
  createdById: uuid('created_by_id').notNull().references(() => user.id),
  lastEditedById: uuid('last_edited_by_id').references(() => user.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
}, (table) => ({
  workspaceIdx: index('idx_document_workspace').on(table.workspaceId),
  collectionIdx: index('idx_document_collection').on(table.collectionId),
  parentIdx: index('idx_document_parent').on(table.parentId),
  statusIdx: index('idx_document_status').on(table.workspaceId, table.status),
  deletedIdx: index('idx_document_deleted').on(table.deletedAt),
}));

// Revision
export const revision = pgTable('revision', {
  id: uuid('id').primaryKey().defaultRandom(),
  documentId: uuid('document_id').notNull().references(() => document.id, { onDelete: 'cascade' }),
  version: integer('version').notNull(),
  title: varchar('title', { length: 1000 }).notNull(),
  body: text('body').notNull(),
  editorId: uuid('editor_id').notNull().references(() => user.id),
  changeSummary: text('change_summary'),
  metadata: jsonb('metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  docVersion: uniqueIndex('idx_revision_doc_version').on(table.documentId, table.version),
  documentIdx: index('idx_revision_document').on(table.documentId),
}));

// Permission (unified)
export const permission = pgTable('permission', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  entityType: entityTypeEnum('entity_type').notNull(),
  entityId: uuid('entity_id').notNull(),
  granteeType: granteeTypeEnum('grantee_type').notNull(),
  granteeId: uuid('grantee_id').notNull(),
  level: permLevelEnum('level').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  entityGrantee: uniqueIndex('idx_perm_entity_grantee').on(
    table.entityType, table.entityId, table.granteeType, table.granteeId
  ),
  entityIdx: index('idx_permission_entity').on(table.entityType, table.entityId),
  granteeIdx: index('idx_permission_grantee').on(table.granteeType, table.granteeId),
}));

// Comment
export const comment = pgTable('comment', {
  id: uuid('id').primaryKey().defaultRandom(),
  documentId: uuid('document_id').notNull().references(() => document.id, { onDelete: 'cascade' }),
  parentId: uuid('parent_id').references((): any => comment.id, { onDelete: 'cascade' }),
  authorId: uuid('author_id').notNull().references(() => user.id),
  body: text('body').notNull(),
  metadata: jsonb('metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  documentIdx: index('idx_comment_document').on(table.documentId),
}));

// Document Embedding (pgvector)
export const documentEmbedding = pgTable('document_embedding', {
  id: uuid('id').primaryKey().defaultRandom(),
  documentId: uuid('document_id').notNull().references(() => document.id, { onDelete: 'cascade' }),
  chunkIndex: integer('chunk_index').notNull().default(0),
  chunkText: text('chunk_text').notNull(),
  embedding: vector('embedding', { dimensions: 1536 }).notNull(),
  metadata: jsonb('metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  docChunk: uniqueIndex('idx_embedding_doc_chunk').on(table.documentId, table.chunkIndex),
  documentIdx: index('idx_embedding_document').on(table.documentId),
}));

// AI Interaction
export const aiInteraction = pgTable('ai_interaction', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => user.id),
  interaction: jsonb('interaction').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  workspaceIdx: index('idx_ai_workspace').on(table.workspaceId),
  createdIdx: index('idx_ai_created').on(table.createdAt),
}));

// Document Link
export const documentLink = pgTable('document_link', {
  sourceId: uuid('source_id').notNull().references(() => document.id, { onDelete: 'cascade' }),
  targetId: uuid('target_id').notNull().references(() => document.id, { onDelete: 'cascade' }),
}, (table) => ({
  pk: uniqueIndex('pk_document_link').on(table.sourceId, table.targetId),
  targetIdx: index('idx_doclink_target').on(table.targetId),
}));

// User-Document State (stars, views, reading progress)
export const userDocumentState = pgTable('user_document_state', {
  userId: uuid('user_id').notNull().references(() => user.id, { onDelete: 'cascade' }),
  documentId: uuid('document_id').notNull().references(() => document.id, { onDelete: 'cascade' }),
  state: jsonb('state').notNull().default({}),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  pk: uniqueIndex('pk_user_doc_state').on(table.userId, table.documentId),
}));

// Attachment
export const attachment = pgTable('attachment', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  documentId: uuid('document_id').references(() => document.id, { onDelete: 'set null' }),
  uploadedById: uuid('uploaded_by_id').notNull().references(() => user.id),
  filename: varchar('filename', { length: 500 }).notNull(),
  contentType: varchar('content_type', { length: 200 }).notNull(),
  byteSize: bigint('byte_size', { mode: 'number' }).notNull(),
  storageKey: text('storage_key').notNull(),
  metadata: jsonb('metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  documentIdx: index('idx_attachment_document').on(table.documentId),
}));

// Integration
export const integration = pgTable('integration', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  type: varchar('type', { length: 50 }).notNull(),
  name: varchar('name', { length: 255 }).notNull(),
  config: jsonb('config').notNull(),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// API Key
export const apiKey = pgTable('api_key', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => user.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  keyHash: text('key_hash').notNull(),
  scopes: text('scopes').array().notNull().default([]),
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  lastUsedAt: timestamp('last_used_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// Activity
export const activity = pgTable('activity', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  actorId: uuid('actor_id').references(() => user.id),
  action: varchar('action', { length: 50 }).notNull(),
  entityType: varchar('entity_type', { length: 50 }).notNull(),
  entityId: uuid('entity_id').notNull(),
  details: jsonb('details').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  workspaceIdx: index('idx_activity_workspace').on(table.workspaceId, table.createdAt),
  entityIdx: index('idx_activity_entity').on(table.entityType, table.entityId),
  actorIdx: index('idx_activity_actor').on(table.actorId),
}));
```

`apps/api/src/db/connection.ts`:
```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';
import type { Config } from '../config';

export function createDb(config: Config) {
  const pool = new Pool({ connectionString: config.DATABASE_URL });
  return drizzle(pool, { schema });
}

export type Db = ReturnType<typeof createDb>;
```

**Testing**:
- `Unit: Drizzle schema compiles without type errors`
- `Integration (real DB): drizzle-kit generate produces valid SQL migration`
- `Integration (real DB): drizzle-kit migrate applies migration to empty PostgreSQL → all 17 tables created`
- `Integration (real DB): insert workspace → select returns same data with defaults applied`
- `Integration (real DB): insert user with duplicate (workspace_id, email) → unique constraint violation`
- `Integration (real DB): delete workspace → cascades to users, collections, documents`

---

#### 1.4 — Docker Compose Development Environment

**What**: Create docker-compose.yml for local development with PostgreSQL 16 (pgvector), Redis 7, and MinIO.

**Design**:

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: wiki_kb
      POSTGRES_USER: wiki
      POSTGRES_PASSWORD: wiki_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wiki -d wiki_kb"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - miniodata:/data

volumes:
  pgdata:
  miniodata:
```

**Testing**:
- `Integration: docker-compose up -d → all 3 services healthy within 30 seconds`
- `Integration: psql connect to PostgreSQL → SELECT 1 returns 1`
- `Integration: redis-cli PING → PONG`
- `Integration: CREATE EXTENSION vector succeeds in PostgreSQL`

---

#### 1.5 — API Server Entrypoint

**What**: Create the Fastify server with health check, OpenAPI documentation, CORS, and graceful shutdown.

**Design**:

`apps/api/src/index.ts`:
```typescript
import Fastify from 'fastify';
import fastifySwagger from '@fastify/swagger';
import fastifySwaggerUI from '@fastify/swagger-ui';
import fastifyCors from '@fastify/cors';
import { loadConfig } from './config';
import { createDb } from './db/connection';

export async function buildApp() {
  const config = loadConfig();
  const db = createDb(config);

  const app = Fastify({
    logger: {
      level: config.LOG_LEVEL,
      transport: config.NODE_ENV === 'development'
        ? { target: 'pino-pretty' }
        : undefined,
    },
  });

  // Plugins
  await app.register(fastifyCors, {
    origin: config.NODE_ENV === 'development' ? true : undefined,
  });

  await app.register(fastifySwagger, {
    openapi: {
      info: {
        title: 'Wiki & Knowledge Base API',
        version: '0.1.0',
        description: 'REST API for the Wiki & Knowledge Base platform',
      },
      servers: [{ url: `http://${config.HOST}:${config.PORT}` }],
    },
  });

  await app.register(fastifySwaggerUI, { routePrefix: '/docs' });

  // Decorate with dependencies
  app.decorate('db', db);
  app.decorate('config', config);

  // Health check
  app.get('/health', {
    schema: {
      response: {
        200: {
          type: 'object',
          properties: {
            status: { type: 'string' },
            timestamp: { type: 'string' },
          },
        },
      },
    },
  }, async () => ({
    status: 'ok',
    timestamp: new Date().toISOString(),
  }));

  return app;
}

// Start
if (import.meta.url === `file://${process.argv[1]}`) {
  const app = await buildApp();
  const config = loadConfig();

  await app.listen({ port: config.PORT, host: config.HOST });

  const shutdown = async () => {
    app.log.info('Shutting down gracefully...');
    await app.close();
    process.exit(0);
  };

  process.on('SIGTERM', shutdown);
  process.on('SIGINT', shutdown);
}
```

**Testing**:
- `Integration: buildApp() → GET /health returns { status: 'ok' } with 200`
- `Integration: GET /docs → Swagger UI HTML page loads`
- `Integration: GET /docs/json → valid OpenAPI 3.1 spec returned`
- `Unit: SIGTERM signal → graceful shutdown initiated`

---

## Phase 2: Authentication and Workspace Management

### Purpose

Implement user authentication (email/password and OAuth 2.0 per RFC 6749), workspace CRUD, team management, and JWT-based API sessions (RFC 7519). After this phase, users can sign up, create workspaces, invite team members, and authenticate API requests.

### Tasks

#### 2.1 — User Registration and Email/Password Authentication

**What**: Implement user signup, login, and JWT token issuance.

**Design**:

```typescript
// packages/shared/src/types/auth.ts
export interface SignupRequest {
  email: string;
  password: string;
  name: string;
  workspaceName: string;
  workspaceSlug: string;
}

export interface LoginRequest {
  email: string;
  password: string;
  workspaceSlug: string;
}

export interface AuthResponse {
  accessToken: string;         // JWT (RFC 7519)
  refreshToken: string;
  expiresIn: number;           // seconds
  user: UserSummary;
  workspace: WorkspaceSummary;
}

export interface UserSummary {
  id: string;
  email: string;
  name: string;
  role: 'admin' | 'member' | 'viewer' | 'guest';
  avatarUrl?: string;
}

export interface WorkspaceSummary {
  id: string;
  name: string;
  slug: string;
}

// JWT payload structure
export interface JWTPayload {
  sub: string;            // user ID
  wid: string;            // workspace ID
  role: string;           // user role in workspace
  iat: number;
  exp: number;
}
```

API endpoints:
- `POST /api/v1/auth/signup` — Create workspace + admin user; hash password with argon2; return AuthResponse
- `POST /api/v1/auth/login` — Verify credentials; return AuthResponse
- `POST /api/v1/auth/refresh` — Exchange refresh token for new access token
- `POST /api/v1/auth/logout` — Invalidate refresh token

Password hashing with argon2id (recommended by OWASP):
```typescript
import { hash, verify } from '@node-rs/argon2';

const ARGON2_OPTIONS = {
  memoryCost: 65536,   // 64MB
  timeCost: 3,
  parallelism: 4,
};

async function hashPassword(password: string): Promise<string> {
  return hash(password, ARGON2_OPTIONS);
}

async function verifyPassword(hash: string, password: string): Promise<boolean> {
  return verify(hash, password, ARGON2_OPTIONS);
}
```

**Testing**:
- `Unit: hashPassword produces argon2id hash`
- `Unit: verifyPassword with correct password → true`
- `Unit: verifyPassword with incorrect password → false`
- `Integration: POST /auth/signup with valid data → 201, workspace created, user with admin role`
- `Integration: POST /auth/signup with existing email in same workspace → 409 Conflict`
- `Integration: POST /auth/login with correct credentials → 200, valid JWT returned`
- `Integration: POST /auth/login with wrong password → 401 Unauthorized`
- `Integration: POST /auth/refresh with valid refresh token → new access token`
- `Integration: POST /auth/refresh with expired refresh token → 401`
- `Integration: authenticated request with valid JWT → 200`
- `Integration: authenticated request with expired JWT → 401`

---

#### 2.2 — Auth Middleware and Permission Checks

**What**: Fastify middleware that extracts and validates JWT from Authorization header, loads user and workspace context, and enforces role-based access.

**Design**:

```typescript
// apps/api/src/middleware/auth.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { JWTPayload } from '@wiki-kb/shared';

declare module 'fastify' {
  interface FastifyRequest {
    currentUser: {
      id: string;
      workspaceId: string;
      role: 'admin' | 'member' | 'viewer' | 'guest';
    };
  }
}

export async function authMiddleware(
  request: FastifyRequest,
  reply: FastifyReply
): Promise<void> {
  const authHeader = request.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return reply.status(401).send({ error: 'Missing or invalid Authorization header' });
  }
  const token = authHeader.slice(7);
  try {
    const payload = verifyJWT<JWTPayload>(token, request.server.config.JWT_SECRET);
    request.currentUser = {
      id: payload.sub,
      workspaceId: payload.wid,
      role: payload.role as any,
    };
  } catch {
    return reply.status(401).send({ error: 'Invalid or expired token' });
  }
}

// Role-based guard factory
export function requireRole(...roles: string[]) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    if (!roles.includes(request.currentUser.role)) {
      return reply.status(403).send({ error: 'Insufficient permissions' });
    }
  };
}
```

**Testing**:
- `Unit: authMiddleware with valid Bearer token → populates request.currentUser`
- `Unit: authMiddleware without Authorization header → 401`
- `Unit: authMiddleware with malformed token → 401`
- `Unit: requireRole('admin') with admin user → passes`
- `Unit: requireRole('admin') with member user → 403`

---

#### 2.3 — Workspace CRUD and Settings

**What**: API endpoints for workspace management including settings update.

**Design**:

```typescript
// packages/shared/src/types/workspace.ts
export interface Workspace {
  id: string;
  name: string;
  slug: string;
  subdomain?: string;
  settings: WorkspaceSettings;
  createdAt: string;
  updatedAt: string;
}

export interface WorkspaceSettings {
  logoUrl?: string;
  defaultLocale: string;
  allowedDomains: string[];
  features: {
    aiAnswerBot: boolean;
    staleDetectionDays: number;
    knowledgeGraph: boolean;
  };
}

export interface UpdateWorkspaceRequest {
  name?: string;
  settings?: Partial<WorkspaceSettings>;
}
```

Endpoints:
- `GET /api/v1/workspaces/:slug` — Get workspace by slug
- `PATCH /api/v1/workspaces/:id` — Update workspace (admin only)
- `DELETE /api/v1/workspaces/:id` — Delete workspace (admin only, requires confirmation)

**Testing**:
- `Integration: GET /workspaces/:slug with valid slug → 200, workspace object`
- `Integration: GET /workspaces/:slug with unknown slug → 404`
- `Integration: PATCH /workspaces/:id as admin → 200, settings updated`
- `Integration: PATCH /workspaces/:id as member → 403`
- `Integration: DELETE /workspaces/:id as admin → 200, workspace soft-deleted`

---

#### 2.4 — Team Management

**What**: CRUD for teams and team membership within a workspace.

**Design**:

```typescript
// packages/shared/src/types/user.ts
export interface Team {
  id: string;
  workspaceId: string;
  name: string;
  memberCount: number;
  createdAt: string;
}

export interface TeamMember {
  userId: string;
  userName: string;
  userEmail: string;
  role: 'admin' | 'member';
  joinedAt: string;
}
```

Endpoints:
- `POST /api/v1/teams` — Create team
- `GET /api/v1/teams` — List teams in workspace
- `POST /api/v1/teams/:id/members` — Add user to team
- `DELETE /api/v1/teams/:id/members/:userId` — Remove user from team
- `DELETE /api/v1/teams/:id` — Delete team (admin only)

**Testing**:
- `Integration: POST /teams → 201, team created`
- `Integration: POST /teams/:id/members with valid user → 200, member added`
- `Integration: POST /teams/:id/members with user from different workspace → 400`
- `Integration: DELETE /teams/:id/members/:userId → 200, member removed`
- `Integration: GET /teams → returns teams with memberCount`

---

## Phase 3: Document Management — CRUD, Hierarchy, and Versioning

### Purpose

Implement the core content management system: document CRUD, collection organisation, document hierarchy (parent/child), revision tracking, and Markdown processing. After this phase, users can create, edit, organise, and version documents — the fundamental wiki functionality.

### Tasks

#### 3.1 — Collection CRUD

**What**: Create, list, update, and delete collections (top-level document groupings).

**Design**:

```typescript
// packages/shared/src/types/collection.ts
export interface Collection {
  id: string;
  workspaceId: string;
  name: string;
  slug: string;
  description?: string;
  settings: CollectionSettings;
  documentCount: number;
  createdBy: UserSummary;
  createdAt: string;
  updatedAt: string;
}

export interface CollectionSettings {
  icon?: string;
  color?: string;
  defaultPermission: 'read_write' | 'read' | 'private';
  sortOrder: number;
}

export interface CreateCollectionRequest {
  name: string;
  description?: string;
  settings?: Partial<CollectionSettings>;
}

export interface ListCollectionsResponse {
  collections: Collection[];
  total: number;
}
```

Endpoints:
- `POST /api/v1/collections` — Create collection; auto-generates slug from name
- `GET /api/v1/collections` — List collections with document counts
- `GET /api/v1/collections/:idOrSlug` — Get single collection
- `PATCH /api/v1/collections/:id` — Update collection
- `DELETE /api/v1/collections/:id` — Delete collection (moves documents to uncategorised)

Slug generation:
```typescript
// apps/api/src/lib/slugify.ts
import slugify from 'slugify';
import { nanoid } from 'nanoid';

export function generateSlug(title: string): string {
  const base = slugify(title, { lower: true, strict: true, locale: 'en' });
  return `${base}-${nanoid(6)}`;
}
```

**Testing**:
- `Unit: generateSlug("Getting Started") → "getting-started-<6chars>"`
- `Unit: generateSlug with special characters → stripped, lowercase, hyphenated`
- `Integration: POST /collections → 201, slug auto-generated`
- `Integration: GET /collections → 200, includes documentCount for each`
- `Integration: PATCH /collections/:id → 200, settings merged`
- `Integration: DELETE /collections/:id → 200, documents moved to null collection`
- `Integration: create two collections with same name → different slugs`

---

#### 3.2 — Document CRUD with Hierarchy

**What**: Create, read, update, and delete documents with parent/child hierarchy and soft delete.

**Design**:

```typescript
// packages/shared/src/types/document.ts
export interface Document {
  id: string;
  workspaceId: string;
  collectionId?: string;
  parentId?: string;
  title: string;
  slug: string;
  body: string;                    // CommonMark Markdown
  status: 'draft' | 'published' | 'archived';
  template: boolean;
  properties: DocumentProperties;
  governance: DocumentGovernance;
  revisionCount: number;
  publishedAt?: string;
  createdBy: UserSummary;
  lastEditedBy?: UserSummary;
  createdAt: string;
  updatedAt: string;
}

export interface DocumentProperties {
  emoji?: string;
  fullWidth?: boolean;
  tags?: string[];
  customFields?: Record<string, unknown>;
}

export interface DocumentGovernance {
  owners?: string[];               // user IDs
  lastVerifiedAt?: string;
  lastVerifiedBy?: string;
  nextReviewAt?: string;
  verificationStatus?: 'verified' | 'needs_update' | 'stale';
  qualityScore?: number;
}

export interface CreateDocumentRequest {
  title: string;
  body?: string;
  collectionId?: string;
  parentId?: string;
  status?: 'draft' | 'published';
  template?: boolean;
  properties?: Partial<DocumentProperties>;
}

export interface UpdateDocumentRequest {
  title?: string;
  body?: string;
  collectionId?: string;
  parentId?: string;
  status?: 'draft' | 'published' | 'archived';
  properties?: Partial<DocumentProperties>;
  changeSummary?: string;
}

export interface DocumentTreeNode {
  id: string;
  title: string;
  slug: string;
  emoji?: string;
  status: string;
  children: DocumentTreeNode[];
}

export interface ListDocumentsQuery {
  collectionId?: string;
  parentId?: string;
  status?: string;
  tag?: string;
  template?: boolean;
  sort?: 'title' | 'updated_at' | 'created_at';
  order?: 'asc' | 'desc';
  limit?: number;
  offset?: number;
}
```

Endpoints:
- `POST /api/v1/documents` — Create document; auto-slug from title; creates initial revision
- `GET /api/v1/documents` — List documents with filtering and pagination
- `GET /api/v1/documents/:id` — Get single document with full body
- `PATCH /api/v1/documents/:id` — Update document; creates new revision if body changed
- `DELETE /api/v1/documents/:id` — Soft delete (set `deleted_at`)
- `POST /api/v1/documents/:id/restore` — Restore soft-deleted document
- `GET /api/v1/documents/:id/children` — List child documents
- `GET /api/v1/collections/:id/tree` — Get full document tree for a collection

Document tree query using PostgreSQL recursive CTE:
```typescript
// apps/api/src/services/document.service.ts
async function getDocumentTree(collectionId: string): Promise<DocumentTreeNode[]> {
  const rows = await db.execute(sql`
    WITH RECURSIVE doc_tree AS (
      SELECT id, title, slug, parent_id,
             properties->>'emoji' AS emoji,
             status, 0 AS depth
      FROM document
      WHERE collection_id = ${collectionId}
        AND parent_id IS NULL
        AND deleted_at IS NULL

      UNION ALL

      SELECT d.id, d.title, d.slug, d.parent_id,
             d.properties->>'emoji' AS emoji,
             d.status, dt.depth + 1
      FROM document d
      JOIN doc_tree dt ON d.parent_id = dt.id
      WHERE d.deleted_at IS NULL
    )
    SELECT * FROM doc_tree ORDER BY depth, title
  `);
  return buildTreeFromFlatRows(rows);
}
```

**Testing**:
- `Integration: POST /documents → 201, document created with revision 1`
- `Integration: POST /documents with parentId → 201, parent-child relationship established`
- `Integration: GET /documents?collectionId=X → 200, filtered list`
- `Integration: GET /documents?tag=onboarding → 200, documents with matching tag`
- `Integration: PATCH /documents/:id with body change → 200, revisionCount incremented`
- `Integration: PATCH /documents/:id with only title change → 200, no new revision`
- `Integration: DELETE /documents/:id → 200, deletedAt set, document excluded from listings`
- `Integration: POST /documents/:id/restore → 200, deletedAt cleared`
- `Integration: GET /collections/:id/tree → 200, nested tree structure`
- `Integration: create document with parentId pointing to different workspace → 400`
- `Fixture: create 3-level document tree → tree query returns correct nesting`

---

#### 3.3 — Revision History

**What**: Store and retrieve document revision history; support version diffing and rollback.

**Design**:

```typescript
// packages/shared/src/types/revision.ts
export interface Revision {
  id: string;
  documentId: string;
  version: number;
  title: string;
  body: string;
  editor: UserSummary;
  changeSummary?: string;
  metadata: RevisionMetadata;
  createdAt: string;
}

export interface RevisionMetadata {
  wordCount: number;
  diffStats?: { added: number; removed: number };
}

export interface RevisionListItem {
  id: string;
  version: number;
  editor: UserSummary;
  changeSummary?: string;
  createdAt: string;
}

export interface RevisionDiff {
  fromVersion: number;
  toVersion: number;
  changes: DiffHunk[];
}

export interface DiffHunk {
  type: 'add' | 'remove' | 'equal';
  value: string;
}
```

Endpoints:
- `GET /api/v1/documents/:id/revisions` — List revision summaries (paginated)
- `GET /api/v1/documents/:id/revisions/:version` — Get specific revision with full body
- `GET /api/v1/documents/:id/revisions/:v1/diff/:v2` — Get diff between two versions
- `POST /api/v1/documents/:id/revisions/:version/restore` — Restore a previous version (creates new revision)

Diff computation using `diff-match-patch`:
```typescript
import { diff_match_patch } from 'diff-match-patch';

function computeDiff(oldText: string, newText: string): DiffHunk[] {
  const dmp = new diff_match_patch();
  const diffs = dmp.diff_main(oldText, newText);
  dmp.diff_cleanupSemantic(diffs);
  return diffs.map(([type, value]) => ({
    type: type === 1 ? 'add' : type === -1 ? 'remove' : 'equal',
    value,
  }));
}
```

**Testing**:
- `Integration: edit document 3 times → GET /revisions returns 3 entries`
- `Integration: GET /revisions/:version → 200, full body of that version`
- `Integration: GET /revisions/1/diff/3 → diff hunks showing changes`
- `Integration: POST /revisions/1/restore → 200, new revision created with v1 body, revisionCount = 4`
- `Unit: computeDiff("hello world", "hello brave world") → [equal "hello ", add "brave ", equal "world"]`
- `Unit: computeDiff with empty old text → all hunks are 'add'`

---

#### 3.4 — Templates

**What**: Support creating documents from templates; templates are documents with `template=true`.

**Design**:

Templates are regular documents with `template: true`. No separate table needed (following Data Model Suggestion 3's philosophy of minimal tables).

Endpoints:
- `GET /api/v1/templates` — List templates in workspace (filters documents where template=true)
- `POST /api/v1/documents` — When `templateId` is provided, copies the template body as initial content

```typescript
export interface CreateDocumentRequest {
  // ... existing fields
  templateId?: string;             // Copy body from this template
}
```

**Testing**:
- `Integration: POST /documents with template=true → 201, template flag set`
- `Integration: GET /templates → 200, only template documents returned`
- `Integration: POST /documents with templateId → 201, body copied from template`
- `Integration: POST /documents with templateId from different workspace → 400`

---

## Phase 4: Permissions, Comments, and Activity Log

### Purpose

Implement the permission system (collection-level and document-level), threaded comments with resolution, and the activity audit trail. After this phase, workspaces have proper access control, collaborative discussion, and a full audit log for compliance (supporting ISO/IEC 27001:2022 A.8.15 logging requirements).

### Tasks

#### 4.1 — Permission System

**What**: Unified permission model for collections and documents, with team and user grantees, and permission inheritance.

**Design**:

Permission resolution logic:
1. Check document-level permissions first (most specific)
2. If no document-level permission, inherit from collection
3. If no collection permission, fall back to workspace role (`admin` → full access, `member` → read_write on unprotected, `viewer` → read only, `guest` → no access unless explicitly granted)

```typescript
// apps/api/src/services/permission.service.ts
export interface ResolvedPermission {
  level: 'none' | 'read' | 'read_write' | 'admin';
  source: 'document' | 'collection' | 'workspace_role';
}

async function resolvePermission(
  userId: string,
  entityType: 'collection' | 'document',
  entityId: string,
  workspaceRole: string,
  db: Db
): Promise<ResolvedPermission> {
  // 1. Get user's team IDs
  const userTeams = await getUserTeamIds(userId, db);

  // 2. Check direct entity permission
  const directPerm = await db.select()
    .from(permission)
    .where(and(
      eq(permission.entityType, entityType),
      eq(permission.entityId, entityId),
      or(
        and(eq(permission.granteeType, 'user'), eq(permission.granteeId, userId)),
        and(eq(permission.granteeType, 'team'), inArray(permission.granteeId, userTeams))
      )
    ))
    .orderBy(desc(permission.level))
    .limit(1);

  if (directPerm.length > 0) {
    return { level: directPerm[0].level, source: entityType };
  }

  // 3. If document, check collection permission
  if (entityType === 'document') {
    const doc = await db.select({ collectionId: document.collectionId })
      .from(document)
      .where(eq(document.id, entityId))
      .limit(1);

    if (doc[0]?.collectionId) {
      return resolvePermission(userId, 'collection', doc[0].collectionId, workspaceRole, db);
    }
  }

  // 4. Fall back to workspace role
  const rolePermMap: Record<string, ResolvedPermission['level']> = {
    admin: 'admin',
    member: 'read_write',
    viewer: 'read',
    guest: 'none',
  };
  return { level: rolePermMap[workspaceRole] ?? 'none', source: 'workspace_role' };
}
```

Endpoints:
- `GET /api/v1/collections/:id/permissions` — List permissions for collection
- `PUT /api/v1/collections/:id/permissions` — Set permission (upsert)
- `DELETE /api/v1/collections/:id/permissions/:permId` — Remove permission
- `GET /api/v1/documents/:id/permissions` — List permissions for document
- `PUT /api/v1/documents/:id/permissions` — Set permission (upsert)
- `DELETE /api/v1/documents/:id/permissions/:permId` — Remove permission

```typescript
export interface SetPermissionRequest {
  granteeType: 'user' | 'team';
  granteeId: string;
  level: 'read' | 'read_write' | 'admin';
}
```

**Testing**:
- `Unit: resolvePermission for admin user with no explicit perms → admin (from workspace_role)`
- `Unit: resolvePermission for member with document-level read → read (from document)`
- `Unit: resolvePermission for member with collection-level read_write, no doc perm → read_write (inherited)`
- `Unit: resolvePermission for guest with no perms → none`
- `Unit: resolvePermission for user in team with team-level perm → team's permission level`
- `Integration: PUT /collections/:id/permissions → 200, permission created`
- `Integration: GET /documents/:id as user with read perm → 200`
- `Integration: PATCH /documents/:id as user with read perm → 403`
- `Integration: GET /documents/:id as user with no perm → 403`

---

#### 4.2 — Threaded Comments

**What**: Document comments with threading (parent/child), resolution, and @mentions.

**Design**:

```typescript
// packages/shared/src/types/comment.ts
export interface Comment {
  id: string;
  documentId: string;
  parentId?: string;
  author: UserSummary;
  body: string;                    // Markdown
  metadata: CommentMetadata;
  createdAt: string;
  updatedAt: string;
  replies?: Comment[];
}

export interface CommentMetadata {
  resolvedAt?: string;
  resolvedBy?: string;
  blockRef?: string;               // Reference to specific block in document
  mentions?: string[];             // User IDs mentioned with @
}

export interface CreateCommentRequest {
  body: string;
  parentId?: string;
  blockRef?: string;
}
```

Endpoints:
- `POST /api/v1/documents/:id/comments` — Create comment or reply
- `GET /api/v1/documents/:id/comments` — List comments (threaded)
- `PATCH /api/v1/comments/:id` — Update comment body (author only)
- `DELETE /api/v1/comments/:id` — Delete comment (author or admin)
- `POST /api/v1/comments/:id/resolve` — Mark comment thread as resolved
- `POST /api/v1/comments/:id/unresolve` — Reopen resolved comment

Mention extraction:
```typescript
function extractMentions(body: string): string[] {
  const mentionRegex = /@\[([^\]]+)\]\(([a-f0-9-]+)\)/g;
  const mentions: string[] = [];
  let match;
  while ((match = mentionRegex.exec(body)) !== null) {
    mentions.push(match[2]); // user ID
  }
  return mentions;
}
```

**Testing**:
- `Integration: POST /documents/:id/comments → 201, comment created`
- `Integration: POST /documents/:id/comments with parentId → 201, reply created`
- `Integration: GET /documents/:id/comments → 200, threaded structure`
- `Integration: POST /comments/:id/resolve → 200, resolvedAt set`
- `Integration: PATCH /comments/:id as different user → 403`
- `Unit: extractMentions("Hey @[Alice](uuid-1) check this") → ["uuid-1"]`
- `Unit: extractMentions with no mentions → []`

---

#### 4.3 — Activity Log

**What**: Record all significant actions as activity entries for audit trail and workspace feed.

**Design**:

```typescript
// packages/shared/src/types/api.ts
export interface ActivityEntry {
  id: string;
  actor: UserSummary;
  action: string;                 // e.g. 'document.created', 'permission.granted'
  entityType: string;             // e.g. 'document', 'collection'
  entityId: string;
  details: Record<string, unknown>;
  createdAt: string;
}

// Activity actions enum
export const ACTIVITY_ACTIONS = {
  DOCUMENT_CREATED: 'document.created',
  DOCUMENT_UPDATED: 'document.updated',
  DOCUMENT_DELETED: 'document.deleted',
  DOCUMENT_RESTORED: 'document.restored',
  DOCUMENT_PUBLISHED: 'document.published',
  COLLECTION_CREATED: 'collection.created',
  COLLECTION_DELETED: 'collection.deleted',
  PERMISSION_GRANTED: 'permission.granted',
  PERMISSION_REVOKED: 'permission.revoked',
  COMMENT_CREATED: 'comment.created',
  COMMENT_RESOLVED: 'comment.resolved',
  MEMBER_INVITED: 'member.invited',
  MEMBER_REMOVED: 'member.removed',
} as const;
```

Activity recording is handled by a service called from route handlers:
```typescript
// apps/api/src/services/activity.service.ts
export async function recordActivity(
  db: Db,
  params: {
    workspaceId: string;
    actorId: string;
    action: string;
    entityType: string;
    entityId: string;
    details?: Record<string, unknown>;
  }
): Promise<void> {
  await db.insert(activity).values(params);
}
```

Endpoints:
- `GET /api/v1/activity` — List activity feed for workspace (paginated, filterable by entity_type)
- `GET /api/v1/documents/:id/activity` — Activity for a specific document

**Testing**:
- `Integration: create document → activity entry with action 'document.created' recorded`
- `Integration: update document → activity with old_title/new_title in details`
- `Integration: GET /activity → paginated list, most recent first`
- `Integration: GET /activity?entityType=document → filtered results`
- `Integration: GET /documents/:id/activity → only activities for that document`

---

## Phase 5: Full-Text Search and File Attachments

### Purpose

Implement PostgreSQL-based full-text search across documents, file upload/download with S3-compatible storage, and document starring/view tracking. After this phase, users can find content quickly, attach files to documents, and track engagement.

### Tasks

#### 5.1 — Full-Text Search (tsvector)

**What**: Index document titles and bodies using PostgreSQL tsvector; provide search API with ranking and filtering.

**Design**:

Search vector update trigger (applied in migration):
```sql
CREATE OR REPLACE FUNCTION update_document_search_vector()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.body, '')), 'B');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_document_search_vector
BEFORE INSERT OR UPDATE OF title, body ON document
FOR EACH ROW EXECUTE FUNCTION update_document_search_vector();
```

```typescript
// packages/shared/src/types/api.ts
export interface SearchRequest {
  query: string;
  collectionId?: string;
  status?: string;
  tag?: string;
  limit?: number;
  offset?: number;
}

export interface SearchResult {
  documentId: string;
  title: string;
  slug: string;
  collectionName?: string;
  snippet: string;                 // ts_headline excerpt
  rank: number;
  updatedAt: string;
}

export interface SearchResponse {
  results: SearchResult[];
  total: number;
  query: string;
}
```

Endpoint:
- `GET /api/v1/search?q=...&collectionId=...&limit=...&offset=...`

Search query:
```typescript
async function searchDocuments(params: SearchRequest, workspaceId: string): Promise<SearchResponse> {
  const tsquery = plainto_tsquery('english', params.query);
  const results = await db.execute(sql`
    SELECT
      d.id AS document_id,
      d.title,
      d.slug,
      c.name AS collection_name,
      ts_headline('english', d.body, ${tsquery},
        'StartSel=<mark>, StopSel=</mark>, MaxWords=50, MinWords=20') AS snippet,
      ts_rank(d.search_vector, ${tsquery}) AS rank,
      d.updated_at
    FROM document d
    LEFT JOIN collection c ON c.id = d.collection_id
    WHERE d.workspace_id = ${workspaceId}
      AND d.deleted_at IS NULL
      AND d.search_vector @@ ${tsquery}
      ${params.collectionId ? sql`AND d.collection_id = ${params.collectionId}` : sql``}
      ${params.status ? sql`AND d.status = ${params.status}` : sql``}
    ORDER BY rank DESC
    LIMIT ${params.limit ?? 20}
    OFFSET ${params.offset ?? 0}
  `);
  return { results, total: results.length, query: params.query };
}
```

**Testing**:
- `Integration: create doc with title "Kubernetes Deployment Guide" → search "kubernetes" returns it`
- `Integration: search "deployment" → snippet contains <mark>deployment</mark>`
- `Integration: search with collectionId filter → only results from that collection`
- `Integration: search with no matches → empty results array`
- `Integration: deleted documents excluded from search results`
- `Integration: search ranking → doc with query term in title ranks higher than body-only`

---

#### 5.2 — File Attachments

**What**: Upload, download, and manage file attachments linked to documents, stored in S3-compatible object storage.

**Design**:

```typescript
// packages/shared/src/types/document.ts
export interface Attachment {
  id: string;
  documentId?: string;
  filename: string;
  contentType: string;
  byteSize: number;
  url: string;                     // pre-signed download URL
  metadata: AttachmentMetadata;
  uploadedBy: UserSummary;
  createdAt: string;
}

export interface AttachmentMetadata {
  width?: number;
  height?: number;
  altText?: string;
}
```

Endpoints:
- `POST /api/v1/documents/:id/attachments` — Upload file (multipart/form-data); max 50MB
- `GET /api/v1/documents/:id/attachments` — List attachments for document
- `GET /api/v1/attachments/:id` — Get attachment metadata + pre-signed download URL
- `DELETE /api/v1/attachments/:id` — Delete attachment (removes from S3)

Storage key pattern: `{workspace_id}/{document_id}/{uuid}-{filename}`

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

function getStorageKey(workspaceId: string, documentId: string, filename: string): string {
  return `${workspaceId}/${documentId}/${crypto.randomUUID()}-${filename}`;
}

async function generateDownloadUrl(storageKey: string): Promise<string> {
  const command = new GetObjectCommand({ Bucket: config.S3_BUCKET, Key: storageKey });
  return getSignedUrl(s3Client, command, { expiresIn: 3600 });
}
```

**Testing**:
- `Integration: POST /documents/:id/attachments with PNG → 201, file stored in S3, metadata saved`
- `Integration: GET /documents/:id/attachments → list includes uploaded file`
- `Integration: GET /attachments/:id → 200, includes pre-signed download URL`
- `Integration: DELETE /attachments/:id → 200, S3 object deleted, DB record removed`
- `Integration: POST attachment exceeding 50MB → 413 Payload Too Large`
- `Integration (mocked S3): upload with S3 failure → 500, no orphan DB record`

---

#### 5.3 — Document Starring and View Tracking

**What**: Allow users to star/unstar documents and track view counts via the `user_document_state` table.

**Design**:

Endpoints:
- `POST /api/v1/documents/:id/star` — Star document
- `DELETE /api/v1/documents/:id/star` — Unstar document
- `GET /api/v1/starred` — List starred documents for current user
- `POST /api/v1/documents/:id/view` — Record a view (idempotent; increments count)

```typescript
// Uses the user_document_state table with JSONB state column
// ON CONFLICT upsert pattern:
async function recordView(userId: string, documentId: string): Promise<void> {
  await db.execute(sql`
    INSERT INTO user_document_state (user_id, document_id, state, updated_at)
    VALUES (${userId}, ${documentId},
      jsonb_build_object('view_count', 1, 'last_viewed_at', now()::text),
      now())
    ON CONFLICT (user_id, document_id) DO UPDATE SET
      state = jsonb_set(
        jsonb_set(
          user_document_state.state,
          '{view_count}',
          to_jsonb(COALESCE((user_document_state.state->>'view_count')::int, 0) + 1)
        ),
        '{last_viewed_at}',
        to_jsonb(now()::text)
      ),
      updated_at = now()
  `);
}
```

**Testing**:
- `Integration: POST /documents/:id/star → 200, state.starred=true`
- `Integration: DELETE /documents/:id/star → 200, state.starred=false`
- `Integration: GET /starred → list of starred documents`
- `Integration: POST /documents/:id/view twice → view_count=2`
- `Integration: star then unstar → starred no longer in GET /starred`

---

## Phase 6: Rich-Text Editor and Real-Time Collaboration

### Purpose

Build the Next.js frontend with the Tiptap rich-text editor, Yjs-based real-time collaborative editing, and the core workspace UI (sidebar, collection browser, document view). After this phase, multiple users can simultaneously edit documents in a browser with live cursor tracking.

### Tasks

#### 6.1 — Next.js Application Shell

**What**: Create the Next.js app with authentication flow, workspace layout, sidebar navigation, and API client.

**Design**:

```typescript
// apps/web/src/lib/api-client.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3001';

export class ApiClient {
  private accessToken: string | null = null;

  setToken(token: string) { this.accessToken = token; }

  async fetch<T>(path: string, options?: RequestInit): Promise<T> {
    const res = await fetch(`${API_BASE}${path}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(this.accessToken ? { Authorization: `Bearer ${this.accessToken}` } : {}),
        ...options?.headers,
      },
    });
    if (!res.ok) {
      const error = await res.json().catch(() => ({}));
      throw new ApiError(res.status, error.message ?? 'Unknown error');
    }
    return res.json();
  }

  // Typed methods
  async getCollections() { return this.fetch<ListCollectionsResponse>('/api/v1/collections'); }
  async getDocument(id: string) { return this.fetch<Document>(`/api/v1/documents/${id}`); }
  async updateDocument(id: string, data: UpdateDocumentRequest) {
    return this.fetch<Document>(`/api/v1/documents/${id}`, {
      method: 'PATCH',
      body: JSON.stringify(data),
    });
  }
  async search(query: string) {
    return this.fetch<SearchResponse>(`/api/v1/search?q=${encodeURIComponent(query)}`);
  }
}
```

Layout structure:
- `(auth)/login` and `(auth)/signup` — public pages
- `(workspace)/[workspace]/layout.tsx` — sidebar with collection tree, search bar, user menu
- `(workspace)/[workspace]/page.tsx` — dashboard with recent documents, starred items
- `(workspace)/[workspace]/[collection]/page.tsx` — collection view
- `(workspace)/[workspace]/[collection]/[document]/page.tsx` — document editor

**Testing**:
- `E2E (Playwright): visit /login → login form visible`
- `E2E: login with valid credentials → redirected to workspace dashboard`
- `E2E: sidebar shows collections → click collection → documents listed`
- `E2E: axe accessibility scan on dashboard → zero WCAG 2.2 AA violations`

---

#### 6.2 — Tiptap Rich-Text Editor

**What**: Integrate Tiptap 3 editor with CommonMark Markdown import/export, toolbar, and slash commands.

**Design**:

```typescript
// apps/web/src/components/editor/Editor.tsx
import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import Placeholder from '@tiptap/extension-placeholder';
import TaskList from '@tiptap/extension-task-list';
import TaskItem from '@tiptap/extension-task-item';
import Table from '@tiptap/extension-table';
import TableRow from '@tiptap/extension-table-row';
import TableCell from '@tiptap/extension-table-cell';
import TableHeader from '@tiptap/extension-table-header';
import CodeBlockLowlight from '@tiptap/extension-code-block-lowlight';
import Image from '@tiptap/extension-image';
import Link from '@tiptap/extension-link';
import Mention from '@tiptap/extension-mention';

interface EditorProps {
  initialContent: string;          // CommonMark Markdown
  onUpdate: (markdown: string) => void;
  editable: boolean;
}

export function DocumentEditor({ initialContent, onUpdate, editable }: EditorProps) {
  const editor = useEditor({
    extensions: [
      StarterKit.configure({ codeBlock: false }),
      CodeBlockLowlight.configure({ lowlight }),
      Placeholder.configure({ placeholder: 'Start writing...' }),
      TaskList,
      TaskItem.configure({ nested: true }),
      Table.configure({ resizable: true }),
      TableRow, TableCell, TableHeader,
      Image,
      Link.configure({ openOnClick: false }),
      Mention.configure({
        suggestion: mentionSuggestion,  // User mention popup
      }),
    ],
    content: markdownToHTML(initialContent),
    editable,
    onUpdate: ({ editor }) => {
      onUpdate(htmlToMarkdown(editor.getHTML()));
    },
  });

  return (
    <div className="editor-container">
      <Toolbar editor={editor} />
      <EditorContent editor={editor} />
    </div>
  );
}
```

Markdown conversion using `turndown` (HTML→MD) and `marked` (MD→HTML):
```typescript
import TurndownService from 'turndown';
import { marked } from 'marked';

const turndown = new TurndownService({
  headingStyle: 'atx',
  codeBlockStyle: 'fenced',
});

export function htmlToMarkdown(html: string): string {
  return turndown.turndown(html);
}

export function markdownToHTML(markdown: string): string {
  return marked.parse(markdown, { gfm: true });
}
```

**Testing**:
- `E2E: type text in editor → Markdown exported via onUpdate`
- `E2E: paste Markdown → rendered as rich text`
- `E2E: create heading via toolbar → # prefix in Markdown output`
- `E2E: create task list → - [ ] items in Markdown output`
- `E2E: insert code block → fenced code block in Markdown output`
- `E2E: insert table → GFM table in Markdown output`
- `Unit: markdownToHTML("# Hello") → <h1>Hello</h1>`
- `Unit: htmlToMarkdown("<h1>Hello</h1>") → "# Hello"`
- `E2E: axe scan on editor → zero WCAG 2.2 AA violations (keyboard nav, ARIA labels)`

---

#### 6.3 — Real-Time Collaborative Editing (Yjs)

**What**: Enable multiple users to edit the same document simultaneously with live cursor positions and awareness.

**Design**:

Backend WebSocket server:
```typescript
// apps/api/src/routes/collaboration.ts
import { WebSocketServer } from 'ws';
import { setupWSConnection } from 'y-websocket/bin/utils';

export function setupCollaborationServer(server: FastifyInstance) {
  const wss = new WebSocketServer({ noServer: true });

  server.server.on('upgrade', (request, socket, head) => {
    const url = new URL(request.url!, `http://${request.headers.host}`);
    if (url.pathname.startsWith('/collaboration/')) {
      // Authenticate via query param token
      const token = url.searchParams.get('token');
      const user = verifyJWT(token);
      if (!user) { socket.destroy(); return; }

      const documentId = url.pathname.split('/')[2];

      // Check permission before allowing connection
      wss.handleUpgrade(request, socket, head, (ws) => {
        setupWSConnection(ws, request, {
          docName: documentId,
          gc: true, // garbage collect
        });
      });
    }
  });
}
```

Frontend Yjs integration with Tiptap:
```typescript
// apps/web/src/components/editor/CollaborativeEditor.tsx
import { HocuspocusProvider } from '@hocuspocus/provider';
import Collaboration from '@tiptap/extension-collaboration';
import CollaborationCursor from '@tiptap/extension-collaboration-cursor';
import * as Y from 'yjs';

export function CollaborativeEditor({ documentId, user }: Props) {
  const ydoc = useMemo(() => new Y.Doc(), []);
  const provider = useMemo(
    () => new HocuspocusProvider({
      url: `${WS_URL}/collaboration/${documentId}`,
      name: documentId,
      document: ydoc,
      token: user.accessToken,
    }),
    [documentId]
  );

  const editor = useEditor({
    extensions: [
      StarterKit.configure({ history: false }),  // Yjs handles undo/redo
      Collaboration.configure({ document: ydoc }),
      CollaborationCursor.configure({
        provider,
        user: { name: user.name, color: userColor },
      }),
      // ... other extensions
    ],
  });

  return <EditorContent editor={editor} />;
}
```

**Testing**:
- `E2E: two browser tabs open same document → changes in tab 1 appear in tab 2 within 1 second`
- `E2E: cursor of user A visible to user B with name label`
- `E2E: disconnect and reconnect → changes sync after reconnection`
- `Integration: WebSocket connection without valid token → connection rejected`
- `Integration: WebSocket connection for document user cannot access → connection rejected`

---

## Phase 7: AI Semantic Search and Answer Bot

### Purpose

Add vector-embedding-based semantic search and the AI answer bot that synthesises answers from workspace documents with citations and confidence scores. This phase delivers the primary AI-native differentiator identified in the research. Studies show semantic search achieves 73%+ accuracy versus 52-58% for keyword-only (features.md).

### Tasks

#### 7.1 — Document Chunking and Embedding Pipeline

**What**: Background worker that chunks documents into passages and generates vector embeddings, stored in the `document_embedding` table with pgvector.

**Design**:

```typescript
// apps/api/src/lib/chunker.ts
export interface Chunk {
  index: number;
  text: string;
  startOffset: number;
  endOffset: number;
}

// Split document into overlapping chunks of ~500 tokens
export function chunkDocument(body: string, options?: {
  maxTokens?: number;   // default 500
  overlap?: number;     // default 50 tokens
}): Chunk[] {
  const maxTokens = options?.maxTokens ?? 500;
  const overlap = options?.overlap ?? 50;
  const paragraphs = body.split(/\n{2,}/);
  const chunks: Chunk[] = [];
  let currentChunk = '';
  let currentIndex = 0;
  let startOffset = 0;

  for (const para of paragraphs) {
    const paraTokens = estimateTokens(para);
    if (estimateTokens(currentChunk) + paraTokens > maxTokens && currentChunk) {
      chunks.push({
        index: currentIndex,
        text: currentChunk.trim(),
        startOffset,
        endOffset: startOffset + currentChunk.length,
      });
      // Overlap: keep last N tokens
      const words = currentChunk.split(/\s+/);
      const overlapWords = words.slice(-overlap);
      currentChunk = overlapWords.join(' ') + '\n\n' + para;
      startOffset += currentChunk.length - overlapWords.join(' ').length;
      currentIndex++;
    } else {
      currentChunk += (currentChunk ? '\n\n' : '') + para;
    }
  }
  if (currentChunk.trim()) {
    chunks.push({
      index: currentIndex,
      text: currentChunk.trim(),
      startOffset,
      endOffset: startOffset + currentChunk.length,
    });
  }
  return chunks;
}

function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4); // rough estimate: ~4 chars per token
}
```

Embedding worker:
```typescript
// apps/api/src/workers/embedding.worker.ts
import { Worker, Job } from 'bullmq';
import OpenAI from 'openai';

interface EmbeddingJob {
  documentId: string;
  workspaceId: string;
  body: string;
  title: string;
}

const embeddingWorker = new Worker<EmbeddingJob>('embedding', async (job) => {
  const { documentId, body, title } = job.data;
  const fullText = `# ${title}\n\n${body}`;

  // 1. Chunk the document
  const chunks = chunkDocument(fullText);

  // 2. Generate embeddings in batch
  const openai = new OpenAI({ apiKey: config.OPENAI_API_KEY });
  const response = await openai.embeddings.create({
    model: config.EMBEDDING_MODEL,
    input: chunks.map(c => c.text),
    dimensions: config.EMBEDDING_DIMENSIONS,
  });

  // 3. Delete old embeddings and insert new ones
  await db.transaction(async (tx) => {
    await tx.delete(documentEmbedding)
      .where(eq(documentEmbedding.documentId, documentId));

    await tx.insert(documentEmbedding).values(
      chunks.map((chunk, i) => ({
        documentId,
        chunkIndex: chunk.index,
        chunkText: chunk.text,
        embedding: response.data[i].embedding,
        metadata: { model: config.EMBEDDING_MODEL, tokenCount: estimateTokens(chunk.text) },
      }))
    );
  });
}, { connection: redis });
```

Embedding jobs are enqueued whenever a document is created or updated (in the document service).

**Testing**:
- `Unit: chunkDocument with short text (< 500 tokens) → single chunk`
- `Unit: chunkDocument with 2000-token text → multiple chunks with overlap`
- `Unit: chunkDocument preserves paragraph boundaries`
- `Unit: chunkDocument with empty text → empty array`
- `Integration (mocked OpenAI): embedding worker processes job → embeddings stored in DB`
- `Integration (mocked OpenAI): re-embed document → old embeddings deleted, new ones inserted`
- `Integration: document update enqueues embedding job → job appears in BullMQ queue`

---

#### 7.2 — Semantic Search API

**What**: Search endpoint that combines full-text search (tsvector) with vector similarity search (pgvector) for hybrid results.

**Design**:

```typescript
// apps/api/src/services/search.service.ts
export interface SemanticSearchResult {
  documentId: string;
  title: string;
  slug: string;
  collectionName?: string;
  snippet: string;
  relevanceScore: number;         // 0-1, combined score
  matchType: 'semantic' | 'keyword' | 'hybrid';
  updatedAt: string;
}

async function semanticSearch(
  query: string,
  workspaceId: string,
  options: { limit?: number; collectionId?: string }
): Promise<SemanticSearchResult[]> {
  // 1. Generate query embedding
  const queryEmbedding = await generateEmbedding(query);

  // 2. Vector similarity search
  const vectorResults = await db.execute(sql`
    SELECT
      de.document_id,
      d.title,
      d.slug,
      c.name AS collection_name,
      de.chunk_text AS snippet,
      1 - (de.embedding <=> ${queryEmbedding}::vector) AS similarity,
      d.updated_at
    FROM document_embedding de
    JOIN document d ON d.id = de.document_id
    LEFT JOIN collection c ON c.id = d.collection_id
    WHERE d.workspace_id = ${workspaceId}
      AND d.deleted_at IS NULL
      AND d.status = 'published'
      ${options.collectionId ? sql`AND d.collection_id = ${options.collectionId}` : sql``}
    ORDER BY de.embedding <=> ${queryEmbedding}::vector
    LIMIT ${(options.limit ?? 10) * 2}
  `);

  // 3. Full-text search (for keyword matching)
  const keywordResults = await fullTextSearch(query, workspaceId, options);

  // 4. Merge and re-rank using Reciprocal Rank Fusion (RRF)
  return reciprocalRankFusion(vectorResults, keywordResults, options.limit ?? 10);
}

function reciprocalRankFusion(
  semanticResults: any[],
  keywordResults: any[],
  limit: number,
  k: number = 60
): SemanticSearchResult[] {
  const scores = new Map<string, number>();

  semanticResults.forEach((r, i) => {
    const id = r.document_id;
    scores.set(id, (scores.get(id) ?? 0) + 1 / (k + i + 1));
  });

  keywordResults.forEach((r, i) => {
    const id = r.document_id;
    scores.set(id, (scores.get(id) ?? 0) + 1 / (k + i + 1));
  });

  // Sort by combined score, return top N
  return Array.from(scores.entries())
    .sort(([, a], [, b]) => b - a)
    .slice(0, limit)
    .map(([docId, score]) => {
      const semantic = semanticResults.find(r => r.document_id === docId);
      const keyword = keywordResults.find(r => r.document_id === docId);
      const source = semantic ?? keyword;
      return {
        documentId: docId,
        title: source.title,
        slug: source.slug,
        collectionName: source.collection_name,
        snippet: source.snippet,
        relevanceScore: score,
        matchType: semantic && keyword ? 'hybrid' : semantic ? 'semantic' : 'keyword',
        updatedAt: source.updated_at,
      };
    });
}
```

Endpoint:
- `GET /api/v1/search?q=...&mode=semantic|keyword|hybrid` — Defaults to hybrid when AI search is enabled

**Testing**:
- `Integration (mocked OpenAI): search "how to deploy to production" → returns doc titled "Deployment Guide" via semantic match`
- `Integration: search "deploy" → returns same doc via keyword match`
- `Integration: hybrid search → RRF re-ranks results from both sources`
- `Integration: search with ENABLE_AI_SEARCH=false → falls back to keyword-only`
- `Unit: reciprocalRankFusion with overlapping results → overlapping docs score higher`
- `Unit: reciprocalRankFusion with disjoint results → all included, sorted by individual RRF score`

---

#### 7.3 — AI Answer Bot

**What**: Natural-language Q&A endpoint that retrieves relevant document chunks, synthesises an answer using Claude API, and returns the answer with citations and confidence score.

**Design**:

```typescript
// apps/api/src/services/answer-bot.service.ts
import Anthropic from '@anthropic-ai/sdk';

export interface AnswerBotRequest {
  question: string;
}

export interface AnswerBotResponse {
  answer: string;
  confidence: number;              // 0.0-1.0
  sources: AnswerSource[];
  followUpQuestions: string[];
}

export interface AnswerSource {
  documentId: string;
  documentTitle: string;
  snippet: string;
  relevance: number;
}

async function askAnswerBot(
  question: string,
  workspaceId: string,
  userId: string
): Promise<AnswerBotResponse> {
  // 1. Retrieve relevant chunks (top 10 by semantic similarity)
  const chunks = await getRelevantChunks(question, workspaceId, 10);

  // 2. Filter by user's permissions
  const accessibleChunks = await filterByPermission(chunks, userId);

  // 3. Build context for LLM
  const context = accessibleChunks.map((chunk, i) => (
    `[Source ${i + 1}: "${chunk.documentTitle}"]\n${chunk.chunkText}`
  )).join('\n\n---\n\n');

  // 4. Call Claude API
  const anthropic = new Anthropic({ apiKey: config.ANTHROPIC_API_KEY });
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: `You are a knowledge base assistant. Answer the user's question based ONLY on the provided source documents. 
Rules:
- Cite sources using [Source N] notation
- If the sources don't contain enough information, say so clearly
- Never fabricate information not present in the sources
- Rate your confidence from 0.0 to 1.0 based on source coverage
- Suggest 2-3 follow-up questions the user might ask

Respond in this JSON format:
{
  "answer": "...",
  "confidence": 0.85,
  "cited_sources": [1, 3],
  "follow_up_questions": ["...", "..."]
}`,
    messages: [
      {
        role: 'user',
        content: `Sources:\n\n${context}\n\n---\n\nQuestion: ${question}`,
      },
    ],
  });

  // 5. Parse response and map cited sources
  const parsed = JSON.parse(response.content[0].text);

  // 6. Log interaction
  await recordAIInteraction(workspaceId, userId, question, parsed);

  return {
    answer: parsed.answer,
    confidence: parsed.confidence,
    sources: parsed.cited_sources.map((i: number) => ({
      documentId: accessibleChunks[i - 1].documentId,
      documentTitle: accessibleChunks[i - 1].documentTitle,
      snippet: accessibleChunks[i - 1].chunkText.slice(0, 200),
      relevance: accessibleChunks[i - 1].similarity,
    })),
    followUpQuestions: parsed.follow_up_questions,
  };
}
```

Endpoint:
- `POST /api/v1/ask` — Submit question, get answer with citations
- `POST /api/v1/ask/:interactionId/feedback` — Submit feedback (helpful, not_helpful, incorrect)

**Testing**:
- `Integration (mocked Anthropic): ask "How do I set up staging?" with relevant docs → answer with citations`
- `Integration (mocked Anthropic): ask about topic with no docs → response indicates insufficient information`
- `Integration: permission-filtered chunks → user only sees answers from docs they can access`
- `Integration: POST /ask/:id/feedback → feedback stored in ai_interaction record`
- `Unit: context builder produces correctly formatted source blocks`
- `Unit: response parser extracts answer, confidence, and source indices`

---

## Phase 8: Knowledge Graph and Graph Visualisation

### Purpose

Add the auto-constructed knowledge graph that infers relationships between documents, concepts, people, and teams. This addresses the "organisational memory" gap identified in the research: no competitor automatically builds relationship graphs from semantic analysis. Uses graph tables from Data Model Suggestion 4.

### Tasks

#### 8.1 — Graph Schema Migration

**What**: Add `graph_node` and `graph_edge` tables to the database, along with the `concept_extraction_job` table.

**Design**:

Migration adds three tables from Data Model Suggestion 4:

```typescript
// apps/api/src/db/schema.ts (additions)
export const graphNode = pgTable('graph_node', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  nodeType: varchar('node_type', { length: 50 }).notNull(),
  sourceId: uuid('source_id'),
  label: varchar('label', { length: 500 }).notNull(),
  description: text('description'),
  properties: jsonb('properties').notNull().default({}),
  embedding: vector('embedding', { dimensions: 1536 }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  workspaceIdx: index('idx_gnode_workspace').on(table.workspaceId),
  typeIdx: index('idx_gnode_type').on(table.nodeType),
  sourceIdx: index('idx_gnode_source').on(table.sourceId),
  labelIdx: index('idx_gnode_label').on(table.workspaceId, table.label),
}));

export const graphEdge = pgTable('graph_edge', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspace.id, { onDelete: 'cascade' }),
  sourceNodeId: uuid('source_node_id').notNull().references(() => graphNode.id, { onDelete: 'cascade' }),
  targetNodeId: uuid('target_node_id').notNull().references(() => graphNode.id, { onDelete: 'cascade' }),
  edgeType: varchar('edge_type', { length: 50 }).notNull(),
  weight: real('weight').notNull().default(1.0),
  properties: jsonb('properties').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  sourceIdx: index('idx_gedge_source').on(table.sourceNodeId),
  targetIdx: index('idx_gedge_target').on(table.targetNodeId),
  typeIdx: index('idx_gedge_type').on(table.edgeType),
  pairIdx: index('idx_gedge_pair').on(table.sourceNodeId, table.targetNodeId, table.edgeType),
}));

export const conceptExtractionJob = pgTable('concept_extraction_job', {
  id: uuid('id').primaryKey().defaultRandom(),
  documentId: uuid('document_id').notNull().references(() => document.id, { onDelete: 'cascade' }),
  revisionVersion: integer('revision_version').notNull(),
  status: varchar('status', { length: 20 }).notNull().default('pending'),
  conceptsFound: integer('concepts_found').default(0),
  edgesCreated: integer('edges_created').default(0),
  error: text('error'),
  startedAt: timestamp('started_at', { withTimezone: true }),
  completedAt: timestamp('completed_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Edge types aligned with SKOS vocabulary (W3C) per Data Model Suggestion 4:
- `links_to` — document contains a link to another document
- `mentions` — document mentions a concept
- `broader` — concept A is a broader term for concept B (SKOS)
- `narrower` — concept A is a narrower term for concept B (SKOS)
- `related` — general semantic relatedness (SKOS)
- `authored_by` — document authored by person
- `owned_by` — document owned by person or team
- `similar_to` — AI-detected semantic similarity

**Testing**:
- `Integration: migration applies successfully → graph_node, graph_edge, concept_extraction_job created`
- `Integration: insert graph_node with embedding → retrieves correctly`
- `Integration: delete graph_node → cascades to graph_edge`

---

#### 8.2 — Automatic Concept Extraction Worker

**What**: Background worker that analyses document content using an LLM to extract concepts, relationships, and topic tags, then populates the knowledge graph.

**Design**:

```typescript
// apps/api/src/workers/concept-extraction.worker.ts
interface ConceptExtractionResult {
  concepts: ExtractedConcept[];
  relationships: ExtractedRelationship[];
}

interface ExtractedConcept {
  label: string;
  aliases: string[];
  definition: string;
  confidence: number;
}

interface ExtractedRelationship {
  source: string;       // concept label
  target: string;       // concept label
  type: 'broader' | 'narrower' | 'related';
  confidence: number;
}

async function extractConcepts(documentTitle: string, documentBody: string): Promise<ConceptExtractionResult> {
  const anthropic = new Anthropic({ apiKey: config.ANTHROPIC_API_KEY });
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 2048,
    system: `Extract key concepts and relationships from the following document.
For each concept, provide:
- label: canonical name
- aliases: alternative names or abbreviations
- definition: one-sentence definition in context
- confidence: 0.0-1.0

For relationships between concepts, use SKOS vocabulary:
- broader: concept A is a broader category containing concept B
- narrower: concept A is a more specific instance of concept B  
- related: concepts are semantically related but not hierarchical

Return JSON: { "concepts": [...], "relationships": [...] }`,
    messages: [{ role: 'user', content: `# ${documentTitle}\n\n${documentBody}` }],
  });

  return JSON.parse(response.content[0].text);
}
```

Worker flow:
1. Dequeue concept extraction job from BullMQ
2. Load document content
3. Call LLM to extract concepts and relationships
4. For each concept: find or create `graph_node` (dedup by label within workspace)
5. Create `graph_edge` entries for document→concept (`mentions`) and concept→concept relationships
6. Create/update shadow node for the document itself
7. Generate embeddings for new concept nodes
8. Update job status

**Testing**:
- `Integration (mocked Anthropic): process document about Kubernetes → extracts "Kubernetes", "Container", "Pod" concepts`
- `Integration: duplicate concept label → reuses existing graph_node`
- `Integration: concept extraction job marked completed with correct counts`
- `Integration: failed LLM call → job marked failed with error message`
- `Unit: concept deduplication matches case-insensitively`

---

#### 8.3 — Graph Query API

**What**: API endpoints for querying the knowledge graph: related documents, concept hierarchy, and graph data for visualisation.

**Design**:

```typescript
// packages/shared/src/types/graph.ts
export interface GraphNode {
  id: string;
  nodeType: 'document' | 'concept' | 'person' | 'team' | 'topic' | 'external';
  label: string;
  description?: string;
  properties: Record<string, unknown>;
  sourceId?: string;
}

export interface GraphEdge {
  id: string;
  sourceNodeId: string;
  targetNodeId: string;
  edgeType: string;
  weight: number;
  properties: Record<string, unknown>;
}

export interface GraphView {
  nodes: GraphNode[];
  edges: GraphEdge[];
}

export interface RelatedDocument {
  documentId: string;
  title: string;
  sharedConcepts: string[];
  relevanceScore: number;
}
```

Endpoints:
- `GET /api/v1/documents/:id/related` — Documents related via shared concepts (2-hop traversal)
- `GET /api/v1/documents/:id/concepts` — Concepts mentioned in a document
- `GET /api/v1/graph?documentId=...` — Graph data centred on a document (for visualisation)
- `GET /api/v1/graph?conceptId=...` — Graph data centred on a concept
- `GET /api/v1/concepts` — List all concepts in workspace
- `GET /api/v1/concepts/:id/hierarchy` — Concept hierarchy (broader/narrower tree)

Related documents query (2-hop: document → concept → document):
```typescript
async function getRelatedDocuments(documentId: string, limit: number = 10): Promise<RelatedDocument[]> {
  return db.execute(sql`
    WITH doc_concepts AS (
      SELECT ge.target_node_id AS concept_id, gn_concept.label
      FROM graph_edge ge
      JOIN graph_node gn_doc ON gn_doc.id = ge.source_node_id
      JOIN graph_node gn_concept ON gn_concept.id = ge.target_node_id
      WHERE gn_doc.source_id = ${documentId}
        AND gn_doc.node_type = 'document'
        AND ge.edge_type = 'mentions'
    ),
    related AS (
      SELECT gn2.source_id AS document_id,
             array_agg(DISTINCT dc.label) AS shared_concepts,
             COUNT(DISTINCT dc.concept_id) AS concept_count,
             AVG(ge2.weight) AS avg_weight
      FROM doc_concepts dc
      JOIN graph_edge ge2 ON ge2.target_node_id = dc.concept_id AND ge2.edge_type = 'mentions'
      JOIN graph_node gn2 ON gn2.id = ge2.source_node_id AND gn2.node_type = 'document'
      WHERE gn2.source_id != ${documentId}
      GROUP BY gn2.source_id
    )
    SELECT r.document_id, d.title, r.shared_concepts, 
           (r.concept_count * r.avg_weight) AS relevance_score
    FROM related r
    JOIN document d ON d.id = r.document_id AND d.deleted_at IS NULL
    ORDER BY relevance_score DESC
    LIMIT ${limit}
  `);
}
```

**Testing**:
- `Integration: two docs mentioning "Kubernetes" → GET /documents/:id/related returns the other`
- `Integration: GET /documents/:id/concepts → list of concepts with confidence`
- `Integration: GET /graph?documentId=X → nodes and edges for visualisation`
- `Integration: GET /concepts/:id/hierarchy → tree with broader/narrower relationships`
- `Integration: document with no concepts → empty related list`

---

#### 8.4 — Knowledge Graph Visualisation (Frontend)

**What**: Interactive force-directed graph view using D3.js, showing document and concept nodes with typed edges.

**Design**:

```typescript
// apps/web/src/components/graph/GraphView.tsx
import { useEffect, useRef } from 'react';
import * as d3 from 'd3';
import type { GraphView, GraphNode, GraphEdge } from '@wiki-kb/shared';

interface GraphViewProps {
  data: GraphView;
  centreNodeId: string;
  onNodeClick: (node: GraphNode) => void;
}

export function KnowledgeGraphView({ data, centreNodeId, onNodeClick }: GraphViewProps) {
  const svgRef = useRef<SVGSVGElement>(null);

  useEffect(() => {
    if (!svgRef.current || !data.nodes.length) return;

    const svg = d3.select(svgRef.current);
    const width = svgRef.current.clientWidth;
    const height = svgRef.current.clientHeight;

    // Color by node type
    const colorScale = d3.scaleOrdinal<string>()
      .domain(['document', 'concept', 'person', 'team', 'topic'])
      .range(['#4A90D9', '#7B68EE', '#2ECC71', '#E67E22', '#9B59B6']);

    const simulation = d3.forceSimulation(data.nodes as any)
      .force('link', d3.forceLink(data.edges as any)
        .id((d: any) => d.id)
        .distance(100))
      .force('charge', d3.forceManyBody().strength(-200))
      .force('center', d3.forceCenter(width / 2, height / 2))
      .force('collision', d3.forceCollide().radius(30));

    // Render links, nodes, labels
    // ... (standard D3 force-directed graph rendering)

    return () => simulation.stop();
  }, [data]);

  return (
    <svg
      ref={svgRef}
      className="w-full h-full"
      role="img"
      aria-label="Knowledge graph visualisation"
    />
  );
}
```

**Testing**:
- `E2E: navigate to /graph → SVG rendered with nodes and edges`
- `E2E: click document node → navigates to document page`
- `E2E: hover concept node → tooltip shows definition`
- `E2E: graph with 50+ nodes renders within 2 seconds`
- `E2E: axe scan → graph SVG has appropriate ARIA labels`

---

## Phase 9: Content Governance — Verification, Stale Detection, and Quality Scoring

### Purpose

Implement the content health features that differentiate this platform from generic wikis: document verification workflows, automated stale-content detection, and quality scoring. Only Guru and Slite offer verification workflows among competitors (features.md); automated stale detection with AI analysis is an underserved area.

### Tasks

#### 9.1 — Document Ownership and Verification Workflow

**What**: Assign owners to documents; owners can verify accuracy on a configurable schedule; verification status tracked in the `governance` JSONB field.

**Design**:

```typescript
// packages/shared/src/types/document.ts (additions)
export interface VerificationRequest {
  status: 'verified' | 'needs_update';
  notes?: string;
  nextReviewDays?: number;         // days until next review; default from workspace settings
}

export interface DocumentOwnership {
  documentId: string;
  owners: UserSummary[];
  verificationStatus: 'verified' | 'needs_update' | 'stale' | 'unverified';
  lastVerifiedAt?: string;
  lastVerifiedBy?: UserSummary;
  nextReviewAt?: string;
}
```

Endpoints:
- `PUT /api/v1/documents/:id/owners` — Set document owners (array of user IDs)
- `GET /api/v1/documents/:id/owners` — Get ownership info
- `POST /api/v1/documents/:id/verify` — Submit verification
- `GET /api/v1/governance/pending-reviews` — List documents due for review (for current user as owner)
- `GET /api/v1/governance/stale` — List stale documents in workspace

Verification updates the `governance` JSONB:
```typescript
async function verifyDocument(documentId: string, userId: string, request: VerificationRequest) {
  const nextReviewDays = request.nextReviewDays ?? config.STALE_DETECTION_DAYS;
  const nextReviewAt = new Date(Date.now() + nextReviewDays * 24 * 60 * 60 * 1000);

  await db.update(document)
    .set({
      governance: sql`
        jsonb_set(
          jsonb_set(
            jsonb_set(
              jsonb_set(governance, '{verification_status}', ${JSON.stringify(request.status)}::jsonb),
              '{last_verified_at}', ${JSON.stringify(new Date().toISOString())}::jsonb
            ),
            '{last_verified_by}', ${JSON.stringify(userId)}::jsonb
          ),
          '{next_review_at}', ${JSON.stringify(nextReviewAt.toISOString())}::jsonb
        )
      `,
      updatedAt: new Date(),
    })
    .where(eq(document.id, documentId));

  await recordActivity(db, {
    workspaceId, actorId: userId,
    action: 'document.verified',
    entityType: 'document', entityId: documentId,
    details: { status: request.status, notes: request.notes },
  });
}
```

**Testing**:
- `Integration: PUT /documents/:id/owners → 200, governance.owners updated`
- `Integration: POST /documents/:id/verify with verified → governance fields updated`
- `Integration: GET /governance/pending-reviews → only documents owned by current user past next_review_at`
- `Integration: verify sets next_review_at to now + configured days`
- `Integration: activity log records verification event`

---

#### 9.2 — Automated Stale Content Detection Worker

**What**: Scheduled worker that scans for documents past their review date or unchanged for a configurable period, marks them as stale, and notifies owners.

**Design**:

```typescript
// apps/api/src/workers/stale-detection.worker.ts
// Runs on a cron schedule (daily at 2am)

async function detectStaleDocuments(workspaceId: string): Promise<void> {
  const staleDays = await getWorkspaceStaleDetectionDays(workspaceId);
  const cutoff = new Date(Date.now() - staleDays * 24 * 60 * 60 * 1000);

  // 1. Documents past their next_review_at
  const pastReview = await db.select()
    .from(document)
    .where(and(
      eq(document.workspaceId, workspaceId),
      isNull(document.deletedAt),
      eq(document.status, 'published'),
      sql`(governance->>'next_review_at')::timestamptz < now()`,
      sql`governance->>'verification_status' != 'stale'`,
    ));

  // 2. Documents not updated in staleDays
  const unchanged = await db.select()
    .from(document)
    .where(and(
      eq(document.workspaceId, workspaceId),
      isNull(document.deletedAt),
      eq(document.status, 'published'),
      lt(document.updatedAt, cutoff),
      sql`governance->>'verification_status' IS NULL OR governance->>'verification_status' != 'stale'`,
    ));

  // 3. Mark as stale
  const staleIds = [...new Set([...pastReview, ...unchanged].map(d => d.id))];
  for (const docId of staleIds) {
    await db.update(document)
      .set({
        governance: sql`jsonb_set(governance, '{verification_status}', '"stale"'::jsonb)`,
      })
      .where(eq(document.id, docId));
  }

  // 4. Notify owners (enqueue notification jobs)
  for (const docId of staleIds) {
    await notificationQueue.add('stale-notification', { documentId: docId, workspaceId });
  }
}
```

**Testing**:
- `Integration: document unchanged for 91 days (config=90) → marked stale`
- `Integration: document verified yesterday → not marked stale`
- `Integration: document past next_review_at → marked stale`
- `Integration: already-stale document → not processed again`
- `Integration: notification job enqueued for each stale document`

---

#### 9.3 — Document Quality Scoring

**What**: Compute a quality score (0-1) for each document based on freshness, completeness, link health, and engagement.

**Design**:

```typescript
// apps/api/src/services/governance.service.ts
export interface QualityScore {
  overall: number;                 // 0.0 - 1.0
  factors: {
    freshness: number;             // based on last update / verification
    completeness: number;          // word count, has title, body length
    linkHealth: number;            // percentage of internal links that resolve
    engagement: number;            // view count relative to workspace average
    ownerAssigned: number;         // 1.0 if owner assigned, 0.0 if not
  };
}

function computeQualityScore(doc: Document, stats: DocumentStats): QualityScore {
  // Freshness: 1.0 if updated today, decays to 0.0 over 180 days
  const daysSinceUpdate = (Date.now() - new Date(doc.updatedAt).getTime()) / (1000 * 60 * 60 * 24);
  const freshness = Math.max(0, 1 - daysSinceUpdate / 180);

  // Completeness: based on word count thresholds
  const wordCount = doc.body.split(/\s+/).length;
  const completeness = Math.min(1, wordCount / 200); // 200+ words = 1.0

  // Link health
  const linkHealth = stats.totalLinks > 0
    ? stats.validLinks / stats.totalLinks
    : 1.0;

  // Engagement: normalised against workspace average
  const engagement = Math.min(1, stats.viewCount / Math.max(stats.avgWorkspaceViews, 1));

  // Owner assigned
  const ownerAssigned = doc.governance.owners?.length ? 1.0 : 0.0;

  const overall = (
    freshness * 0.25 +
    completeness * 0.20 +
    linkHealth * 0.20 +
    engagement * 0.15 +
    ownerAssigned * 0.20
  );

  return { overall, factors: { freshness, completeness, linkHealth, engagement, ownerAssigned } };
}
```

Endpoint:
- `GET /api/v1/documents/:id/quality` — Get quality score breakdown
- `GET /api/v1/governance/quality-report` — Workspace-wide quality distribution

**Testing**:
- `Unit: document updated today → freshness = 1.0`
- `Unit: document not updated in 180 days → freshness = 0.0`
- `Unit: document with 50 words → completeness = 0.25`
- `Unit: document with 300 words → completeness = 1.0`
- `Unit: all links valid → linkHealth = 1.0`
- `Unit: 2 of 4 links broken → linkHealth = 0.5`
- `Unit: document with owner → ownerAssigned = 1.0`
- `Integration: GET /documents/:id/quality → quality score object returned`

---

## Phase 10: Integrations — Slack, Webhooks, and API Keys

### Purpose

Connect the wiki to external tools: Slack integration for answer delivery and notifications, outbound webhooks for automation, and API key management for programmatic access. Slack/Teams integration is table-stakes per features.md.

### Tasks

#### 10.1 — Slack Integration

**What**: Slack bot that responds to questions in channels, sends notifications for document changes, and links back to wiki pages.

**Design**:

```typescript
// packages/shared/src/types/api.ts (additions)
export interface SlackIntegrationConfig {
  botToken: string;
  signingSecret: string;
  defaultChannel: string;
  events: ('document.published' | 'document.updated' | 'comment.created' | 'stale.detected')[];
}

// Slack slash command handler
// POST /api/v1/integrations/slack/command
// Handles: /wiki ask <question>
//          /wiki search <query>
//          /wiki link <document-slug>

interface SlackCommandPayload {
  command: string;
  text: string;
  user_id: string;
  channel_id: string;
  response_url: string;
}

async function handleSlackCommand(payload: SlackCommandPayload): Promise<SlackResponse> {
  const [subcommand, ...args] = payload.text.split(' ');
  const query = args.join(' ');

  switch (subcommand) {
    case 'ask':
      // Defer response, then post answer asynchronously
      const answer = await askAnswerBot(query, workspaceId, userId);
      return {
        response_type: 'in_channel',
        blocks: formatAnswerAsSlackBlocks(answer),
      };
    case 'search':
      const results = await semanticSearch(query, workspaceId, { limit: 5 });
      return {
        response_type: 'ephemeral',
        blocks: formatSearchResultsAsSlackBlocks(results),
      };
    default:
      return { response_type: 'ephemeral', text: 'Usage: /wiki ask|search <query>' };
  }
}
```

Endpoints:
- `POST /api/v1/integrations/slack/events` — Slack Events API endpoint
- `POST /api/v1/integrations/slack/command` — Slack slash command handler
- `POST /api/v1/integrations/slack/interactive` — Slack interactive message handler

**Testing**:
- `Integration (mocked Slack): /wiki ask "how to deploy" → answer posted to channel`
- `Integration (mocked Slack): /wiki search "kubernetes" → ephemeral results posted`
- `Integration: Slack event with invalid signature → 401`
- `Integration: document published → Slack notification sent to configured channel`
- `Unit: formatAnswerAsSlackBlocks produces valid Slack Block Kit JSON`

---

#### 10.2 — Outbound Webhooks

**What**: Allow workspaces to register webhook URLs that receive event notifications for document lifecycle events.

**Design**:

```typescript
// packages/shared/src/types/api.ts
export interface WebhookConfig {
  url: string;
  secret?: string;                 // HMAC secret for signature verification
  events: string[];                // e.g. ['document.created', 'document.published']
}

export interface WebhookPayload {
  event: string;
  timestamp: string;
  workspaceId: string;
  data: Record<string, unknown>;   // event-specific data
}
```

Webhook delivery worker:
```typescript
// apps/api/src/workers/webhook-delivery.worker.ts
async function deliverWebhook(job: Job<WebhookDeliveryJob>) {
  const { webhookId, payload } = job.data;
  const webhook = await getWebhook(webhookId);

  const body = JSON.stringify(payload);
  const signature = webhook.secret
    ? createHmac('sha256', webhook.secret).update(body).digest('hex')
    : undefined;

  const response = await fetch(webhook.url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      ...(signature ? { 'X-Webhook-Signature': `sha256=${signature}` } : {}),
    },
    body,
    signal: AbortSignal.timeout(10_000),
  });

  if (!response.ok) {
    throw new Error(`Webhook delivery failed: ${response.status}`);
  }
}
```

Endpoints:
- `POST /api/v1/webhooks` — Register webhook
- `GET /api/v1/webhooks` — List webhooks
- `PATCH /api/v1/webhooks/:id` — Update webhook
- `DELETE /api/v1/webhooks/:id` — Delete webhook
- `POST /api/v1/webhooks/:id/test` — Send test event

Retry policy: exponential backoff with 3 retries (BullMQ built-in).

**Testing**:
- `Integration: register webhook → test event delivered to URL`
- `Integration: document.created event → webhook payload delivered with correct structure`
- `Integration: webhook with secret → X-Webhook-Signature header present and valid`
- `Integration: webhook URL returns 500 → retried 3 times with backoff`
- `Integration: webhook URL returns 200 → no retry`

---

#### 10.3 — API Key Management

**What**: Create, list, and revoke API keys for programmatic access to the REST API.

**Design**:

```typescript
export interface CreateApiKeyRequest {
  name: string;
  scopes: string[];                // e.g. ['documents:read', 'search:read']
  expiresInDays?: number;
}

export interface ApiKeyResponse {
  id: string;
  name: string;
  key: string;                     // only returned once at creation time
  keyPrefix: string;               // first 8 chars for identification
  scopes: string[];
  expiresAt?: string;
  createdAt: string;
}
```

API key format: `wkb_<32 random chars>` (e.g., `wkb_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`)

Key storage: only the bcrypt hash is stored; the raw key is returned once at creation.

Endpoints:
- `POST /api/v1/api-keys` — Create API key (returns raw key once)
- `GET /api/v1/api-keys` — List API keys (prefix only, no raw keys)
- `DELETE /api/v1/api-keys/:id` — Revoke API key

Authentication: API keys are accepted in the `Authorization: Bearer wkb_...` header alongside JWT tokens. The auth middleware detects the `wkb_` prefix and routes to API key validation.

**Testing**:
- `Integration: POST /api-keys → 201, raw key returned`
- `Integration: GET /api-keys → list with key prefixes, no raw keys`
- `Integration: use API key in Bearer header → authenticated request succeeds`
- `Integration: revoked API key → 401`
- `Integration: expired API key → 401`
- `Integration: API key with documents:read scope → can GET documents, cannot POST`

---

## Phase 11: Import/Export and Document Links

### Purpose

Enable bulk import from Markdown files and Confluence export, bulk export to Markdown/HTML, and automatic extraction of inter-document links for backlink display and graph construction.

### Tasks

#### 11.1 — Markdown Import

**What**: Import a directory of Markdown files into a collection, preserving file hierarchy as document parent/child relationships.

**Design**:

```typescript
export interface ImportRequest {
  collectionId: string;
  source: 'markdown_zip' | 'confluence_export';
}

export interface ImportResult {
  documentsCreated: number;
  documentsSkipped: number;
  errors: ImportError[];
}

export interface ImportError {
  filename: string;
  error: string;
}
```

Endpoints:
- `POST /api/v1/import/markdown` — Upload ZIP of Markdown files (multipart/form-data)
- `POST /api/v1/import/confluence` — Upload Confluence HTML export ZIP
- `GET /api/v1/import/:jobId/status` — Check import job status

Import is processed asynchronously via BullMQ:
1. Extract ZIP to temp directory
2. Walk directory tree, creating documents for each `.md` file
3. Preserve directory structure as parent/child relationships
4. Parse frontmatter (YAML) for title, tags, and metadata
5. Enqueue embedding jobs for each imported document

**Testing**:
- `Integration: upload ZIP with 3 .md files → 3 documents created`
- `Integration: nested directories → parent/child relationships match directory structure`
- `Integration: .md file with YAML frontmatter → title and tags extracted`
- `Integration: invalid ZIP → error response`
- `Integration: file with encoding issues → skipped with error in ImportResult`

---

#### 11.2 — Export

**What**: Export documents from a collection as Markdown files in a ZIP archive or as a single HTML file.

**Design**:

Endpoints:
- `GET /api/v1/export/markdown?collectionId=...` — Download ZIP of Markdown files
- `GET /api/v1/export/html?documentId=...` — Download single document as HTML
- `GET /api/v1/export/markdown?documentId=...` — Download single document as .md

**Testing**:
- `Integration: GET /export/markdown?collectionId=X → ZIP file with .md files`
- `Integration: exported Markdown round-trips (import → export → diff = zero)`
- `Integration: GET /export/html?documentId=X → valid HTML with styles`
- `Integration: export respects permissions (user can only export docs they can read)`

---

#### 11.3 — Link Extraction and Backlinks

**What**: Background worker that parses document Markdown for internal links, populates `document_link` table, and creates `links_to` edges in the knowledge graph.

**Design**:

```typescript
// apps/api/src/workers/link-extraction.worker.ts
function extractInternalLinks(body: string, workspaceDocSlugs: Map<string, string>): string[] {
  const linkRegex = /\[([^\]]*)\]\(\/([^)]+)\)/g;
  const documentIds: string[] = [];
  let match;
  while ((match = linkRegex.exec(body)) !== null) {
    const slug = match[2];
    const docId = workspaceDocSlugs.get(slug);
    if (docId) documentIds.push(docId);
  }
  return documentIds;
}
```

Backlinks endpoint:
- `GET /api/v1/documents/:id/backlinks` — Documents that link to this document

**Testing**:
- `Integration: document A links to document B → document_link row created`
- `Integration: GET /documents/B/backlinks → includes document A`
- `Integration: update document to remove link → document_link row deleted`
- `Unit: extractInternalLinks("[Guide](/setup-guide)") → returns doc ID for setup-guide slug`
- `Unit: extractInternalLinks with external URL → not extracted`

---

## Phase 12: Production Hardening — CI/CD, Docker, Monitoring, and SCIM

### Purpose

Prepare the platform for production deployment: production Dockerfile, CI/CD pipeline, structured logging, health monitoring, rate limiting, and SCIM 2.0 user provisioning for enterprise SSO (per RFC 7643/7644 from standards.md).

### Tasks

#### 12.1 — Production Dockerfile and Docker Compose

**What**: Multi-stage Dockerfile for the API and worker; production docker-compose with health checks, resource limits, and secrets management.

**Design**:

```dockerfile
# Dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@9.15.0 --activate

FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/api/package.json apps/api/
COPY packages/shared/package.json packages/shared/
RUN pnpm install --frozen-lockfile --prod

FROM base AS build
WORKDIR /app
COPY . .
RUN pnpm install --frozen-lockfile
RUN pnpm --filter api build
RUN pnpm --filter shared build

FROM base AS runtime
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/apps/api/dist ./apps/api/dist
COPY --from=build /app/packages/shared/dist ./packages/shared/dist
COPY --from=build /app/apps/api/package.json ./apps/api/
EXPOSE 3001
USER node
CMD ["node", "apps/api/dist/index.js"]
```

**Testing**:
- `Integration: docker build → image builds under 2 minutes`
- `Integration: docker run → container starts, /health returns 200`
- `Integration: docker-compose up (production profile) → all services healthy`
- `Integration: container runs as non-root user (node)`

---

#### 12.2 — CI/CD Pipeline (GitHub Actions)

**What**: GitHub Actions workflow for linting, type checking, testing, and Docker image publishing on merge to main.

**Design**:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env: { POSTGRES_DB: wiki_test, POSTGRES_USER: test, POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/wiki_test
          REDIS_URL: redis://localhost:6379

  docker:
    if: github.ref == 'refs/heads/main'
    needs: [lint-and-typecheck, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

**Testing**:
- `Integration: push to PR → CI runs lint, typecheck, test`
- `Integration: merge to main → Docker image built and pushed`
- `Integration: test job uses real PostgreSQL and Redis service containers`

---

#### 12.3 — Rate Limiting and Security Headers

**What**: Per-user and per-IP rate limiting; security headers (OWASP recommendations from standards.md).

**Design**:

```typescript
// apps/api/src/middleware/rate-limit.ts
import rateLimit from '@fastify/rate-limit';

export async function registerRateLimit(app: FastifyInstance) {
  await app.register(rateLimit, {
    max: app.config.RATE_LIMIT_MAX,
    timeWindow: app.config.RATE_LIMIT_WINDOW_MS,
    keyGenerator: (request) => {
      return request.currentUser?.id ?? request.ip;
    },
  });
}

// Security headers
app.addHook('onSend', async (request, reply) => {
  reply.header('X-Content-Type-Options', 'nosniff');
  reply.header('X-Frame-Options', 'DENY');
  reply.header('X-XSS-Protection', '0');
  reply.header('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  reply.header('Content-Security-Policy', "default-src 'self'");
  reply.header('Referrer-Policy', 'strict-origin-when-cross-origin');
});
```

**Testing**:
- `Integration: 101 requests in 60 seconds → 429 Too Many Requests on 101st`
- `Integration: rate limit resets after window expires`
- `Integration: response includes X-Content-Type-Options: nosniff`
- `Integration: response includes Strict-Transport-Security header`

---

#### 12.4 — SCIM 2.0 User Provisioning

**What**: SCIM 2.0 endpoints (RFC 7643/7644) for automated user provisioning and deprovisioning from identity providers (Okta, Azure AD).

**Design**:

```typescript
// SCIM 2.0 User Schema (RFC 7643)
export interface SCIMUser {
  schemas: ['urn:ietf:params:scim:schemas:core:2.0:User'];
  id: string;
  userName: string;                // mapped to email
  name: {
    givenName: string;
    familyName: string;
  };
  emails: Array<{ value: string; primary: boolean }>;
  active: boolean;
  displayName: string;
}

export interface SCIMListResponse {
  schemas: ['urn:ietf:params:scim:api:messages:2.0:ListResponse'];
  totalResults: number;
  startIndex: number;
  itemsPerPage: number;
  Resources: SCIMUser[];
}
```

Endpoints:
- `GET /scim/v2/Users` — List users (with filtering: `filter=userName eq "alice@company.com"`)
- `GET /scim/v2/Users/:id` — Get user
- `POST /scim/v2/Users` — Create user (provisions into workspace)
- `PUT /scim/v2/Users/:id` — Replace user
- `PATCH /scim/v2/Users/:id` — Update user (partial)
- `DELETE /scim/v2/Users/:id` — Deactivate user (set `is_suspended = true`)

SCIM authentication: Bearer token scoped to the workspace.

**Testing**:
- `Integration: POST /scim/v2/Users → user created in workspace`
- `Integration: GET /scim/v2/Users?filter=userName eq "alice@co.com" → matching user`
- `Integration: DELETE /scim/v2/Users/:id → user suspended, not deleted`
- `Integration: PATCH /scim/v2/Users/:id with name change → user name updated`
- `Integration: SCIM request without Bearer token → 401`
- `Unit: SCIM filter parser handles eq, and, or operators`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Authentication               ─── requires Phase 1
    │
Phase 3: Document Management           ─── requires Phase 2
    │
    ├── Phase 4: Permissions & Comments ─── requires Phase 3
    │       │
    │       └── Phase 5: Search & Attachments ─── requires Phase 3
    │               │
    │               ├── Phase 6: Editor & Real-Time ─── requires Phase 3, Phase 5
    │               │
    │               └── Phase 7: AI Search & Answer Bot ─── requires Phase 5
    │                       │
    │                       └── Phase 8: Knowledge Graph ─── requires Phase 7
    │
    ├── Phase 9: Content Governance     ─── requires Phase 4
    │
    ├── Phase 10: Integrations          ─── requires Phase 4, Phase 7 (for Slack answer bot)
    │
    └── Phase 11: Import/Export         ─── requires Phase 3
         │
         └── Phase 12: Production Hardening ─── requires Phases 1-11
```

**Parallelism opportunities:**
- Phases 4 and 5 can be developed concurrently after Phase 3
- Phases 6 and 7 can be developed concurrently after Phase 5
- Phase 9 can be developed in parallel with Phases 7 and 8 (after Phase 4)
- Phase 11 can be developed in parallel with Phases 6-10 (after Phase 3)

---

## Definition of Done (per phase)

1. All tasks implemented and compiling without errors.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass against real PostgreSQL and Redis (via docker-compose).
4. ESLint passes with zero errors and zero warnings (`pnpm lint`).
5. TypeScript strict-mode type checking passes (`pnpm typecheck`).
6. Docker build succeeds for all affected services.
7. Feature works end-to-end (manual verification or E2E test).
8. New API endpoints appear in auto-generated OpenAPI spec at `/docs/json`.
9. Database migrations created and applied cleanly to fresh database.
10. New configuration options documented in `.env.example`.
11. WCAG 2.2 AA accessibility compliance for any new UI components (verified via axe-core).
12. No secrets or credentials committed to the repository.
