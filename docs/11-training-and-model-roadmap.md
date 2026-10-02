---
title: "Aivexa AI — Training, Model & Release Roadmap"
summary: "Engineering architecture for proprietary model training pipelines, API developer platform, and the corrected V1–V5 release roadmap."
---

# Aivexa AI — Training, Model & Developer Platform Roadmap

## Overview

This document defines the engineering architecture for Aivexa AI's long-term training and proprietary model pipeline, the post-V1 API developer ecosystem, and the corrected release roadmap (V1–V5). Aivexa AI is built as a modular monolith in Next.js/React/TypeScript backed by PostgreSQL with `pgvector`, Redis with BullMQ, and Amazon S3. This roadmap resolves earlier specification ambiguities by strictly isolating V1 to a foundational conversational search, document RAG, and memory engine, ensuring that advanced agentic execution, application building, and proprietary models scale sequentially on a stable substrate.

---

## Part A: Training & Proprietary Model Roadmap

The proprietary model engineering strategy is strictly scheduled post-V1 and will be executed only after the platform reaches production scale and validates core traffic patterns.

### 1. Model Engineering Pipeline Architecture
- **Data Ingestion & Filtering**:
  - Ingest licensed third-party datasets and explicit user opt-in telemetry into partitioned S3 staging buckets (`s3://aivexa-ml-staging/`).
  - Pipeline processing: Text extraction, language identification (fastText), exact-match deduplication, and MinHash LSH (Locality-Sensitive Hashing) for near-duplicate removal across text chunks.
- **PII Scrubbing & Provenance Tracking**:
  - Sanitize raw text via Microsoft Presidio and customized regex masking rules for PII (names, phone numbers, SSNs), API keys, passwords, and IP addresses.
  - Every training sample receives a immutable JSON provenance manifest containing `source_dataset_id`, `license_type`, `checksum_sha256`, `opt_in_audit_id`, and `ingestion_timestamp`.
- **Fine-Tuning Strategy**:
  - Base Models: Fine-tune state-of-the-art open-weight models (e.g., Llama 3.3 70B, Qwen 2.5 72B, Mistral Small).
  - Techniques: Parameter-Efficient Fine-Tuning (PEFT via QLoRA 4-bit/8-bit) for rapid iteration, followed by full parameter fine-tuning for production checkpoints.
  - Platform-Specific Target Tasks:
    - *Research Synthesis*: Academic grounding, multi-document conflict resolution, and structured inline citation generation (`[1]`, `[2]`).
    - *Builder Codegen*: Clean Next.js App Router code generation, Tailwind CSS component composition, and Prisma schema generation.
    - *Tool-Call Formatting*: Strict JSON schema compliance, parameter extraction, and zero-shot function-calling accuracy.
- **Evaluation Platform**:
  - Continuous benchmark evaluation against standard suites (HumanEval, SWE-bench, GSM8K) plus custom Aivexa task suites (citation accuracy, tool-call syntax validation, vector grounding precision).
  - Regression thresholding: Any fine-tuned candidate must match or exceed base frontier model performance on task-specific metrics before staging promotion.
- **Model Registry & Gateway Promotion**:
  - Trained weights and hyperparameter artifacts stored in MLflow and backed by S3 model registries.
  - Deployment Pipeline: Staging → Shadow Traffic Proxy (asynchronous execution parallel to live API, evaluating latency and generation quality with 0% user impact) → Canary Rollout (1% → 5% → 25% → 100%) through the Aivexa Model Gateway with automated fallback routing.

### 2. Data Governance & Privacy Rights
- **Explicit Opt-In Consent System**: User telemetry and chat interactions are NEVER automatically ingested into training datasets. Opt-in requires explicit UI confirmation, logged in PostgreSQL with timestamped audit records.
- **Right to Withdraw**: Users can revoke data consent at any time via Account Settings. Revocation triggers an automated BullMQ worker job that purges candidate records from training queues and retraining datasets within 30 days.
- **License Compliance & Auditing**: Strict license verification pipeline blocking unverified open-source or scraped content lacking explicit commercial redistribution rights.

### 3. Immediate Constraints (What NOT to Do Yet)
- **No Self-Hosted GPU Clusters**: Zero bare-metal GPU procurement or long-term cluster reservations today; leverage serverless compute providers (Modal, RunPod, Anyscale) for batch fine-tuning jobs.
- **No Pretraining**: Zero model training from scratch; rely entirely on top-tier open-weight base checkpoints.
- **No Speculative Data Hoarding**: Never collect or retain telemetry without an active, explicitly governed ML pipeline target.

---

## Part B: API Developer Platform (Post-V1 Ecosystem)

The API platform transforms Aivexa AI into an extensible ecosystem, enabling third-party developers and enterprise customers to integrate agentic capabilities programmatically.

### 1. Public APIs & SDK Capabilities
- **Aivexa API (`/v1/chat`, `/v1/research`)**: High-level REST and Server-Sent Events (SSE) streaming endpoints for conversational search, multi-source research synthesis, and document grounding.
- **Agents API (`/v1/agents`)**: Lifecycle management endpoints to instantiate, configure, execute, pause, and inspect persistent AI agents.
- **Tools API (`/v1/tools`)**: Schema registration endpoints allowing developers to register custom OpenAPI 3.1 tool definitions, HTTP webhooks, and sandboxed functions.
- **Models API (`/v1/models`)**: Model discovery, dynamic routing parameters, token usage telemetry, and fine-tuned model endpoint selection.
- **Developer SDKs**: First-party, fully typed client libraries in TypeScript (`@aivexa/sdk`) and Python (`aivexa-python`), featuring native SSE streaming, exponential backoff retries, and automatic schema validation.

### 2. Developer Surface & System Architecture
- **Custom Surface Integration**: Developers can register and expose custom agents, custom tool sets, custom vector knowledge bases (external PostgreSQL/Pinecone/Qdrant endpoints), and complex multi-step workflows.
- **API Keys & Granular Scopes**: Token generation using secure prefix tokens (`ax_live_...`, `ax_sand_...`) with scope enforcement (`agent:read`, `agent:write`, `tools:execute`, `knowledge:query`, `models:route`).
- **Rate Limits & Metering**: Redis sliding-window counter rate limiting per API key and organisation tier (requests/min, tokens/min).
- **Sandbox Keys & Dry-Run Mode**: Isolated sandbox environment keys that route all tool calls and code executions to isolated mock environments without side effects.
- **Webhooks & Event Bus**: Distributed Redis/BullMQ event publishing sending Webhook payloads (`agent.completed`, `workflow.failed`, `usage.threshold_exceeded`) with HMAC-SHA256 signature verification.

---

## Part C: Corrected Release Roadmap (V1–V5)

This roadmap replaces earlier inconsistent specifications. V1 is deliberately bounded to provide a bulletproof core conversational foundation before higher-order execution layers land.

### V1 — Core Foundation
- **Goal**: Establish a production-ready, highly responsive AI workspace for conversational search, document intelligence, and persistent user memory.
- **Components Landed**:
  - Next.js App Router UI with modern dark design.
  - PostgreSQL schema + `pgvector` index for semantic search.
  - Redis + BullMQ background processing queues for document ingestion and text chunking.
  - S3 file storage integration for PDF, TXT, CSV, and code uploads.
  - Aivexa Model Gateway for multi-provider routing (OpenAI, Anthropic) with streaming SSE responses.
  - Research synthesis module providing inline citation grounding.
  - Memory engine for cross-session entity/fact extraction.
- **Definition of Done**: Users can sign up, upload documents, execute multi-step research queries with inline citations, view/edit saved memories, and stream chat responses with under 200ms Time-To-First-Token (TTFT).
- **Dependencies**: None.

### V2 — Agent Runtime & Code Execution
- **Goal**: Expand from single-turn RAG chat into multi-step agentic planning, tool execution, code sandboxing, and safe authorization.
- **Components Landed**:
  - ReAct Agent execution loop with step-by-step reasoning logs.
  - Centralized Tool Registry for system and custom tools.
  - Dedicated Coding Agent capable of reading, modifying, and generating multi-file codebases.
  - Secure isolated code execution sandboxes (E2B / Docker container runtime).
  - Fine-grained Permission Engine prompting users for approval before performing state-mutating tool actions.
- **Definition of Done**: Agents autonomously draft multi-step execution plans, execute generated Python/JS code safely within sandboxes, call external tools, and enforce interactive user permission prompts for write actions.
- **Dependencies**: V1 Model Gateway, Auth, Storage, and Memory infrastructure.

### V3 — Aivexa Builder
- **Goal**: Enable end-to-end full-stack web application generation, live hot-reloading preview, and deployment directly within Aivexa.
- **Components Landed**:
  - App generation engine turning high-level prompts into full Next.js/React codebases.
  - In-browser live preview sandbox using isolated iframes / WebContainers.
  - Code visual editor and side-by-side git diff viewer.
  - One-click deployment agent integration (Vercel, Fly.io, Netlify).
- **Definition of Done**: Users can prompt a full-stack web application, view a live hot-reloading preview instantly, request code modifications via chat, and deploy the live app with a public URL.
- **Dependencies**: V2 Agent Runtime, Coding Agent, and Sandbox runtime.

### V4 — Automation & Enterprise Hardening
- **Goal**: Deliver enterprise-grade scheduled background workflows, headless browser automation, usage metering, and administrative security.
- **Components Landed**:
  - Workflow engine supporting CRON schedules, webhooks, and multi-agent DAG execution.
  - Headless Browser Agent powered by Playwright for automated web data retrieval and interaction.
  - Production OAuth connector integrations (Slack, GitHub, Google Workspace, Jira).
  - Stripe billing, usage metering, and organization token/compute quota enforcement.
  - Admin Console with user management, audit logging, and team workspace isolation.
- **Definition of Done**: Scheduled multi-step workflows execute reliably in background worker queues, browser agents navigate external sites, billing accurately meters token/compute usage, and workspace admins enforce organization policies.
- **Dependencies**: V2 Tool Registry and V3 System Infrastructure.

### V5 — Proprietary Models & Developer Ecosystem
- **Goal**: Launch specialized in-house fine-tuned models, open the public API/SDK developer platform, and provide enterprise custom workspaces.
- **Components Landed**:
  - In-house fine-tuned model fleet integrated into the Aivexa Model Gateway.
  - Public Developer Platform (Aivexa API, Agents API, Tools API, Models API, TS & Python SDKs).
  - Enterprise workspace tier featuring SAML/SSO, audit log streaming, and custom vector store connections (Qdrant/Pinecone).
- **Definition of Done**: Proprietary models serve production traffic with higher citation accuracy and lower latency than base models, external developers build applications using `@aivexa/sdk`, and enterprise tenants manage compliance-ready workspaces.
- **Dependencies**: V1–V4 production data pipelines, evaluation benchmarks, and stable API platform infrastructure.

---

## Open Questions

1. **Synthetic Data Generation Strategies**: Should V5 proprietary fine-tuning rely heavily on synthetic dataset pipelines generated from frontier teacher models (e.g., Claude 3.5 Sonnet / GPT-4o) or strictly on human-curated expert domain datasets?
2. **Local vs. Server-Side Agent Sandboxing**: Should V2 code sandboxing support WebAssembly / WebContainers directly in the user browser for lower backend latency, or strictly rely on server-side isolated microVMs (E2B)?
3. **Enterprise Vector Store Architecture**: For V5 enterprise tenants, should `pgvector` Row-Level Security (RLS) remain the primary vector isolation mechanism, or should high-scale tenants be automatically provisioned on dedicated vector clusters (Qdrant/Pinecone)?
