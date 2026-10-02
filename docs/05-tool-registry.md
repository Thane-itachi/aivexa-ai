---
title: "05 - Tool Registry Specification"
summary: "Precise architecture specification for Aivexa AI's central Tool Registry, schema definitions, execution pipeline, safety controls, and developer SDK."
---

# 05 — Tool Registry Specification

## 1. Overview
The Tool Registry is the single source of truth for all external capabilities actionable by agents within Aivexa AI. Every execution capability—whether reading local files, querying a database, invoking LLMs, or executing code—is encapsulated as a registered tool. Both the Agent Runtime and the server-side Permission Engine query the registry prior to tool invocation: the Agent Runtime retrieves parameter schemas and execution routing metadata, while the Permission Engine verifies that the calling agent's role and user grants satisfy the tool's access requirements.

## 2. Tool Catalog
The platform provides thirteen core tools across various operational domains:

- **web-search**: Performs real-time web indexing and retrieval queries via search engine APIs to gather external context. **Risk Profile: Low.** Read-only internet queries with minimal risk of system mutation or data exfiltration.
- **browser**: Launches a headless Playwright instance in an isolated container to render pages, click elements, extract dynamic DOM content, and take screenshots. **Risk Profile: Medium.** Can access untrusted external websites, risking prompt injection and SSRF; isolated via sandbox network rules.
- **filesystem**: Provides scoped read/write operations within the tenant's dedicated workspace volume in object/container storage. **Risk Profile: Medium.** Manipulates user project files; strictly constrained by path traversal checks and tenant root jail.
- **terminal**: Executes shell commands inside an isolated, resource-capped container sandbox with strict CPU, memory, and timeout bounds. **Risk Profile: High.** Direct code execution capability; restricted to unprivileged sandbox containers with no host access.
- **github**: Interacts with GitHub repositories to create commits, open pull requests, read issues, and manage code reviews using tenant GitHub OAuth credentials. **Risk Profile: Medium.** Mutates remote codebases; actions are scoped by OAuth scope grants.
- **database**: Executes read/write SQL queries and schema migrations against designated user databases using tenant connection strings. **Risk Profile: High.** Can modify or corrupt production datasets; governed by transactional rollbacks and query safety analyzers.
- **image-generation**: Synthesizes images using Diffusion models or external APIs given structured prompts and style parameters. **Risk Profile: Low.** Stateless, content-filtered creative asset generation with quota limits.
- **speech**: Handles text-to-speech rendering and audio-to-text transcription via Whisper and neural voice synthesis pipelines. **Risk Profile: Low.** Pure data transformation and processing tool with minimal security impact.
- **email**: Composes and dispatches outbound email communications via SMTP or connected user Gmail/Outlook OAuth integrations. **Risk Profile: High.** Direct communication channel with external recipients; requires confirmation for external recipients.
- **calendar**: Fetches, creates, and updates event bookings across connected Google Calendar or Outlook Calendar accounts. **Risk Profile: Medium.** Alters user schedules and sends event invitations to third parties.
- **payments**: Triggers payment processing, invoice generation, and subscription modifications via Stripe / payment gateways. **Risk Profile: Critical.** Direct financial implications; strictly governed by dual-actor approval and hard spend caps.
- **deployment**: Triggers production deployments, serverless updates, and cloud infrastructure changes across target environments. **Risk Profile: Critical.** Modifies live application availability and infrastructure state; strictly requires user confirmation.
- **external-apis**: Invokes arbitrary REST/GraphQL HTTP endpoints configured with custom OpenAPI specs and auth tokens. **Risk Profile: Medium.** Varies by target endpoint scope; outbound requests pass through an egress proxy filter.

## 3. Tool Metadata & Schema
Tools are cataloged in PostgreSQL. The `tools` table defines the primary tool entity, while `tool_versions` preserves immutable historical versions and JSON Schemas.

```sql
CREATE TYPE tool_risk_level AS ENUM ('low', 'medium', 'high', 'critical');
CREATE TYPE tool_auth_type AS ENUM ('user_oauth', 'platform_key', 'none');
CREATE TYPE cost_model_type AS ENUM ('per_call', 'per_minute', 'per_byte');

CREATE TABLE tools (
    tool_id VARCHAR(64) PRIMARY KEY,
    name VARCHAR(128) NOT NULL,
    description TEXT NOT NULL,
    risk_level tool_risk_level NOT NULL DEFAULT 'low',
    auth_type tool_auth_type NOT NULL DEFAULT 'none',
    default_timeout_ms INTEGER NOT NULL DEFAULT 30000,
    cost_model cost_model_type NOT NULL DEFAULT 'per_call',
    cost_unit_amount NUMERIC(12, 6) NOT NULL DEFAULT 0.000000,
    enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE tool_versions (
    version_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tool_id VARCHAR(64) NOT NULL REFERENCES tools(tool_id) ON DELETE CASCADE,
    version VARCHAR(32) NOT NULL,
    input_schema JSONB NOT NULL,
    output_schema JSONB NOT NULL,
    required_permissions JSONB NOT NULL DEFAULT '[]'::jsonb,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_tool_version UNIQUE (tool_id, version)
);
```

## 4. Registration and Lifecycle
1. **Manifest Submission**: Developers declare a tool using a manifest file containing its metadata, version, permissions, and OpenAPI/JSON Schemas.
2. **Schema Validation**: The registration service validates `input_schema` and `output_schema` against JSON Schema Draft 2020-12 specs.
3. **Contract Test Suite**: Before enabling a new version, the registry runs an automated contract suite in a staging environment to verify execution conformance, failure modes, and schema compliance.
4. **Activation**: Upon passing contract tests, `is_active` is set to `true`, and the version becomes available for runtime dispatch.
5. **Deprecation**: Older versions are marked inactive (`is_active = false`). The runtime issues deprecation warnings if an agent invokes a deprecated version, forcing fallback or migration.

## 5. Execution Path & Audit Logging
The tool execution pipeline balances speed, isolation, and auditability:

```text
Agent Runtime ──► Permission Engine ──► Input Schema Validation
                                               │
                        ┌──────────────────────┴──────────────────────┐
                        ▼                                             ▼
             [Fast Tools: Inline Execution]                [Long-Running: BullMQ Queue]
             (e.g., web-search, speech)                    (e.g., browser, terminal)
                        │                                             │
                        └──────────────────────┬──────────────────────┘
                                               ▼
                                      Timeout Enforcement
                                               │
                                               ▼
                                    Output Schema Validation
                                               │
                                               ▼
                                      Persist tool_calls Row
```

Every execution generates an audit record in the `tool_calls` table:

```sql
CREATE TABLE tool_calls (
    call_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_run_id UUID NOT NULL,
    tool_id VARCHAR(64) NOT NULL,
    version VARCHAR(32) NOT NULL,
    input JSONB NOT NULL,
    output JSONB,
    status VARCHAR(32) NOT NULL, -- 'pending', 'succeeded', 'failed', 'timed_out', 'rejected'
    duration_ms INTEGER,
    cost NUMERIC(12, 6) DEFAULT 0.000000,
    error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 6. High-Risk Tool Confirmation Handshake
High-risk and critical tools (`deployment`, `payments`, `email`) require explicit human-in-the-loop authorization before execution.

```text
1. Agent Runtime detects tool.risk_level IN ('high', 'critical').
2. System inserts a row into `pending_confirmations` and sets agent_run status to 'paused'.
3. Real-time WebSocket event delivers a confirmation card to the frontend UI.
4. User reviews tool call parameters and selects 'Approve' or 'Deny'.
5. UI submits response:
   - Approved: Agent run unpauses, tool execution dispatches to BullMQ / inline execution.
   - Denied: Agent run resumes with a 'Tool execution rejected by user' error response.
```

## 7. Per-User Tool Connections & Auth
Tools requiring user account access (Gmail, GitHub, Calendar) utilize per-user OAuth tokens:
- **Token Isolation**: OAuth tokens are retrieved from an external Secrets Manager (e.g., AWS Secrets Manager or HashiCorp Vault) using a composite lookup key (`user_id:provider`).
- **Postgres Separation**: Postgres stores only connection metadata (e.g., connection status, provider ID, granted scopes) and secret reference URIs. Plaintext tokens or refresh keys are never stored in Postgres.
- **Runtime Injection**: During tool dispatch, the execution worker fetches the user token from Secrets Manager via short-lived in-memory retrieval and injects it into the tool execution context.

## 8. Tool Author SDK
Every tool within the platform implements the standardized TypeScript `ITool` interface:

```typescript
import { ZodSchema } from 'zod';

export interface ExecuteContext {
  userId: string;
  agentRunId: string;
  authToken?: string;
  signal: AbortSignal;
}

export interface HealthCheckResult {
  status: 'healthy' | 'degraded' | 'unhealthy';
  message?: string;
}

export interface ITool<TInput = unknown, TOutput = unknown> {
  id: string;
  name: string;
  description: string;
  version: string;
  inputSchema: ZodSchema<TInput>;
  outputSchema: ZodSchema<TOutput>;
  execute(input: TInput, ctx: ExecuteContext): Promise<TOutput>;
  healthCheck(): Promise<HealthCheckResult>;
}
```

## 9. Open Questions
1. **Dynamic Schema Hot-Reloading**: How to safely handle hot-swapping tool JSON schemas without restarting worker processes or interrupting active agent runs?
2. **Sub-second Queue Overhead**: For interactive agent sessions, does BullMQ queue dispatch add noticeable latency to fast tools that need fallback isolation?
3. **Granular Financial Approvals**: Should payment tool confirmations support dynamic policy thresholds (e.g., auto-approve payments under $10 while requiring confirmation for >$10)?
4. **Third-Party Plugin Sandboxing**: What runtime isolation model (Wasm vs microVM) should be mandated if third-party developers submit custom tool binaries?
