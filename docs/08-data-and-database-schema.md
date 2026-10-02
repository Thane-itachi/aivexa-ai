---
title: Data & Database Schema Architecture
summary: Consolidated PostgreSQL schema, pgvector hybrid search, tenancy hierarchy, RLS security policies, data privacy, and disaster recovery strategy for Aivexa AI.
---

# 08 — Data & Database Schema Architecture

## 1. Tenancy & Resource Hierarchy

Aivexa AI uses a multi-tenant resource hierarchy to isolate customer data, streamline access control, and provide a clear upgrade path from self-serve users to enterprise deployments.

### Hierarchy Model

```text
┌────────────────────────────────────────────────────────────────────────┐
│                          ORGANIZATION                                  │
│             (Enterprise Billing, SAML SSO, Governance)                 │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ 1:N
┌──────────────────────────────────▼─────────────────────────────────────┐
│                           WORKSPACE                                    │
│       (Team Boundary, Member Roles, Usage Quotas, RLS Scope)           │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ 1:N
┌──────────────────────────────────▼─────────────────────────────────────┐
│                            PROJECT                                     │
│      (Context Container for Scoped Assets & Vector Search Boundaries)   │
└───────┬──────────────┬──────────────┬──────────────────┬───────────────┘
        │ 1:N          │ 1:N          │ 1:N              │ 1:N
┌───────▼──────┐┌──────▼──────┐┌──────▼──────┐    ┌──────▼──────┐
│ Conversations││    Files    ││ Knowledge   │    │ Agent Runs  │
│  & Messages  ││ & Documents ││   Bases     │    │   & Tasks   │
└──────────────┘└─────────────┘└─────────────┘    └─────────────┘
```

### Path to Team & Enterprise Tiers

1. **Organization**: Corporate billing and compliance root. Handles enterprise SSO/SAML, audit log streaming, and cross-workspace governance.
2. **Workspace**: Multi-tenant team container. Memberships assign explicit roles (`owner`, `admin`, `member`, `viewer`). Shared API quotas and subscriptions attach here.
3. **Project**: Logical grouping for Conversations, Files, Knowledge Bases, Agent Runs, and Tasks. Isolates vector context searches so retrievals remain strictly project-scoped.
4. **Tenancy Enforcement**: Every user-facing table carries `workspace_id UUID NOT NULL` and a nullable or required `project_id UUID` for Row-Level Security (RLS) enforcement.

---

## 2. Consolidated PostgreSQL Schema

Aivexa AI utilizes PostgreSQL with `pgvector` for both transactional state and vector embeddings, eliminating dual-write sync issues and simplifying disaster recovery.

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS vector;
```

### Full DDL for Core 10 Tables

```sql
-- 1. Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    status VARCHAR(32) NOT NULL DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

-- 2. Workspaces
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

-- 3. Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

-- 4. Conversations
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    title VARCHAR(255) NOT NULL DEFAULT 'New Conversation',
    is_archived BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

-- 5. Messages
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    sender_role VARCHAR(32) NOT NULL CHECK (sender_role IN ('user', 'assistant', 'system', 'tool')),
    content TEXT NOT NULL,
    token_count INT DEFAULT 0,
    metadata JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

-- 6. Documents
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    knowledge_base_id UUID REFERENCES knowledge_bases(id) ON DELETE SET NULL,
    file_name VARCHAR(255) NOT NULL,
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(128) NOT NULL,
    storage_path TEXT NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'pending',
    created_by UUID NOT NULL REFERENCES users(id),
    deleted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

-- 7. Document Chunks
CREATE TABLE document_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index INT NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536),
    tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', content)) STORED,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

-- 8. Memories
CREATE TABLE memories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category VARCHAR(64) NOT NULL,
    content TEXT NOT NULL,
    confidence NUMERIC(3,2) NOT NULL DEFAULT 1.00,
    embedding vector(1536),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

-- 9. Usage Records
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    model_id VARCHAR(128) NOT NULL,
    prompt_tokens INT NOT NULL DEFAULT 0,
    completion_tokens INT NOT NULL DEFAULT 0,
    total_cost NUMERIC(10,6) NOT NULL DEFAULT 0.000000,
    recorded_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

-- 10. Audit Events
CREATE TABLE audit_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    actor_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(128) NOT NULL,
    resource_type VARCHAR(64) NOT NULL,
    resource_id UUID,
    ip_address INET,
    payload JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);
```

### Compact Schema Summaries for Remaining Domain Tables

- **Auth, Core & Tenancy**: `organizations` (`id`, `name`, `domain`, `sso_config`, `created_at`, `updated_at`); `memberships` (`id`, `workspace_id`, `user_id`, `role_id`, `created_at`, `updated_at`); `roles` (`id`, `workspace_id`, `name`, `permissions_mask`, `created_at`, `updated_at`); `sessions` (`id`, `user_id`, `token_hash`, `expires_at`, `ip_address`, `user_agent`, `created_at`); `oauth_connections` (`id`, `user_id`, `provider`, `provider_user_id`, `access_token`, `refresh_token`, `expires_at`, `created_at`); `api_keys` (`id`, `workspace_id`, `user_id`, `key_hash`, `name`, `scopes`, `last_used_at`, `created_at`).
- **Knowledge & Memory**: `knowledge_bases` (`id`, `workspace_id`, `project_id`, `name`, `description`, `created_by`, `created_at`, `updated_at`); `memory_items` (`id`, `workspace_id`, `memory_id`, `key`, `value`, `source_message_id`, `created_at`).
- **Agent & Orchestration Engine**: `agents` (`id`, `workspace_id`, `project_id`, `name`, `system_prompt`, `tools_enabled`, `created_by`, `created_at`, `updated_at`); `agent_runs` (`id`, `workspace_id`, `project_id`, `agent_id`, `status`, `input`, `output`, `created_at`, `updated_at`); `agent_steps` (`id`, `workspace_id`, `agent_run_id`, `step_number`, `step_type`, `input`, `output`, `created_at`); `tasks` (`id`, `workspace_id`, `project_id`, `title`, `status`, `assigned_agent_id`, `created_at`, `updated_at`).
- **Models & Execution**: `models` (`id`, `provider`, `model_name`, `context_window`, `input_cost_per_k`, `output_cost_per_k`, `is_active`, `created_at`); `model_runs` (`id`, `workspace_id`, `model_id`, `prompt_tokens`, `completion_tokens`, `latency_ms`, `created_at`); `prompts` (`id`, `workspace_id`, `project_id`, `name`, `description`, `created_at`, `updated_at`); `prompt_versions` (`id`, `prompt_id`, `version`, `template_text`, `variables`, `created_by`, `created_at`); `tools` (`id`, `name`, `description`, `schema_json`, `is_system`, `created_at`); `tool_calls` (`id`, `workspace_id`, `message_id`, `tool_id`, `arguments`, `result`, `latency_ms`, `created_at`); `sandbox_runs` (`id`, `workspace_id`, `project_id`, `runtime_language`, `code_snippet`, `status`, `output`, `created_at`).
- **Research & Sources**: `sources` (`id`, `workspace_id`, `project_id`, `url`, `title`, `content_summary`, `created_at`); `research_runs` (`id`, `workspace_id`, `project_id`, `query`, `status`, `summary_result`, `created_at`, `updated_at`).
- **Platform & Billing**: `notifications` (`id`, `workspace_id`, `user_id`, `type`, `title`, `body`, `read_at`, `created_at`); `feature_flags` (`id`, `key`, `description`, `is_enabled`, `rules_json`, `updated_at`); `subscriptions` (`id`, `workspace_id`, `plan_id`, `stripe_subscription_id`, `status`, `current_period_end`, `created_at`); `plans` (`id`, `name`, `monthly_price_cents`, `token_allowance`, `max_members`, `created_at`).

---

## 3. Indexing & pgvector Hybrid Search Strategy

### Vector Index Configuration

Vector similarity uses `pgvector` HNSW (Hierarchical Navigable Small World) indexing for $O(\log N)$ nearest neighbor retrieval:

```sql
CREATE INDEX idx_document_chunks_embedding ON document_chunks 
USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);

CREATE INDEX idx_document_chunks_tsv ON document_chunks USING gin (tsv);

CREATE INDEX idx_document_chunks_workspace_project ON document_chunks (workspace_id, project_id);
```

### Retrieval Pipeline

```text
Question ──► Embedding ──► Vector Search (HNSW) ──┐
                                                 ├──► Hybrid Fusion (RRF) ──► Reranking ──► LLM Context
                           Keyword Search (TSV)  ──┘
```

#### Hybrid Retrieval Query (Reciprocal Rank Fusion)

```sql
WITH vector_search AS (
    SELECT id, content, RANK() OVER (ORDER BY embedding <=> $1) as v_rank
    FROM document_chunks
    WHERE workspace_id = $2 AND project_id = $3
    ORDER BY embedding <=> $1 LIMIT 20
),
text_search AS (
    SELECT id, content, RANK() OVER (ORDER BY ts_rank_cd(tsv, websearch_to_tsquery('english', $4)) DESC) as t_rank
    FROM document_chunks
    WHERE workspace_id = $2 AND project_id = $3 AND tsv @@ websearch_to_tsquery('english', $4)
    ORDER BY t_rank DESC LIMIT 20
)
SELECT 
    COALESCE(v.id, t.id) as id,
    COALESCE(v.content, t.content) as content,
    (COALESCE(1.0 / (60 + v.v_rank), 0.0) + COALESCE(1.0 / (60 + t.t_rank), 0.0)) AS rrf_score
FROM vector_search v
FULL OUTER JOIN text_search t ON v.id = t.id
ORDER BY rrf_score DESC LIMIT 10;
```

---

## 4. Row-Level Security (RLS) & Security Policies

Multi-tenant isolation is enforced inside PostgreSQL via session-bound RLS policies.

```sql
-- Application sets runtime scope per request:
-- SET LOCAL app.current_workspace_id = '11111111-1111-1111-1111-111111111111';

ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_chunks ENABLE ROW LEVEL SECURITY;

CREATE POLICY workspace_isolation_policy ON conversations
    FOR ALL USING (workspace_id = NULLIF(current_setting('app.current_workspace_id', true), '')::UUID)
    WITH CHECK (workspace_id = NULLIF(current_setting('app.current_workspace_id', true), '')::UUID);

CREATE POLICY workspace_isolation_chunks ON document_chunks
    FOR ALL USING (workspace_id = NULLIF(current_setting('app.current_workspace_id', true), '')::UUID);

-- Service-Role Bypass Policy strictly reserved for platform background workers
CREATE POLICY service_role_bypass_conversations ON conversations
    FOR ALL TO service_role USING (TRUE) WITH CHECK (TRUE);
```

---

## 5. Data Lifecycle, Privacy & Deletion Policies

### Data Retention Matrix

| Data Class | Retention Period | Deletion Mechanism |
| :--- | :--- | :--- |
| **Sessions** | 30 days | Automated TTL purge job |
| **Audit Logs** | 1 year minimum (configurable) | Cold S3 archive + DB partition purge |
| **Conversations / Messages** | User / Workspace policy controlled | Soft-delete (`deleted_at`) + 30d background purge |
| **Document Chunks & Embeddings**| Tied to parent Document lifecycle | Immediate CASCADE on document purge |
| **Usage Records** | 7 years (tax/accounting compliance) | Retained for audit reconciliation |

### Deletion & Data Export Workflows

- **Account / Org Deletion**: Cascades across user resources with a 30-day graceful recovery window before hard purge.
- **Conversation / File / Memory Deletion**: Sets `deleted_at = NOW()`, hides record from UI via RLS, and cleans up storage files and embeddings via async purge job.
- **RLS-Safe Purging**: Purge jobs execute under `service_role` using parameterized checks scoped to marked `deleted_at` records.
- **User Data Export**: Users can export a structured JSON archive containing conversations, files, memories, and audit logs.

### Strict AI Training Governance

> **MANDATORY PRIVACY DIRECTIVE**: User data, files, prompts, and memories **NEVER** automatically become AI model training data. Any future model training use requires explicit user opt-in consent, administrative governance controls, and an isolated training data pipeline (see `docs/12-model-training-roadmap.md`).

---

## 6. Backup & Disaster Recovery (DR)

### Backup Strategy & Resilience

- **Continuous WAL Archiving**: PostgreSQL Write-Ahead Logs pushed to S3 via `WAL-G`/`pgBackRest` enabling Point-In-Time Recovery (PITR).
- **Daily Snapshots**: Daily full encrypted database snapshots (AES-256 KMS).
- **Object Storage Versioning**: S3 versioning and Cross-Region Replication (CRR) with 30-day lifecycle retention.
- **Targets**: **RPO $\le 5$ minutes** (via WAL archiving), **RTO $\le 1$ hour** (instance restoration).
- **Restore Drills**: Monthly automated restore testing into an isolated staging environment.

### Disaster Recovery Procedure

1. Provision target PostgreSQL instance in backup region.
2. Restore base snapshot: `pgbackrest --stanza=aivexa_prod --type=time "--target=YYYY-MM-DD HH:MM:SS" restore`.
3. Apply continuous WAL logs up to the recovery target time.
4. Verify table checksums and RLS permissions.
5. Re-route application database poolers to the restored primary.

---

## 7. Migration Strategy & Zero-Downtime Deployment

### Versioning & Execution

Migrations are version-controlled using Goose/Prisma files stored in `migrations/`.

### Zero-Downtime Rules

1. **Expand-Contract Pattern**: Schema changes must be backward-compatible. Add new columns as nullable, deploy updated code, backfill historical rows, and apply non-null/drop operations in a separate release.
2. **Non-Destructive Guarantee**: Destructive DDL (`DROP TABLE`, `DROP COLUMN`) is prohibited without an explicit rollback plan and migration dry-run.

---

## 8. Open Questions

1. **pgvector Dimension Expansion**: If upgrading from 1536-dim embeddings to 3072-dim models, should we support dual-vector columns per document chunk or migration pipelines?
2. **Audit Log Partitioning**: At what volume scale (e.g., $10\text{M}+$ events) should `audit_events` transition to PostgreSQL monthly range partitioning?
3. **Multi-Region Vector Replicas**: Will read-heavy global deployments require localized read replicas for low-latency vector search in non-primary regions?
