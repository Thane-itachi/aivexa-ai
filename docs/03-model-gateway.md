---
title: Aivexa Model Gateway Specification
summary: Technical specification for the single-entry Aivexa Model Gateway covering provider abstraction, dynamic routing, PostgreSQL DDL schemas, streaming SSE token tracking, budget enforcement, secrets management, and observability.
---

# Aivexa Model Gateway Specification

## 1. Overview & Responsibilities

The **Aivexa Model Gateway** is the single entry point for all AI calls within the platform (chat, reasoning, coding, vision, embeddings, STT, TTS, image, video). Application services and background workers never invoke external AI APIs directly.

```text
┌──────────────────────────────────────────────────────────────────┐
│             Aivexa Core Services (Task Planner/Agents)            │
└─────────────────────────────────┬────────────────────────────────┘
                                  │ Unified Request
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Aivexa Model Gateway                       │
│  [Budget Choke Pt] ──► [Model Registry] ──► [Router & Fallback]  │
│                                 │                                │
│                     [Provider Abstraction Layer]                 │
└────────┬────────────────────────┼───────────────────────┬────────┘
         ▼                        ▼                       ▼
 ┌───────────────┐        ┌───────────────┐       ┌───────────────┐
 │  Provider A   │        │  Provider B   │       │  Provider C   │
 │   (OpenAI)    │        │  (Anthropic)  │       │(Google/DeepS) │
 └───────────────┘        └───────────────┘       └───────────────┘
```

### Key Responsibilities
- **Provider Abstraction**: Normalizes divergent request/response formats into a single interface.
- **Cost Accounting & Budget Enforcement**: Performs pre-flight token and budget checks against tenant balances.
- **Failover & Resilience**: Handles HTTP 429/5xx and timeouts via exponential backoff, retries, and fallback chains.
- **Usage Metering & Rate Limiting**: Tracks token consumption per user and organization in real time.
- **Observability**: Records every execution into PostgreSQL `model_runs` for audit, billing, and latency tracking.

---

## 2. Provider Abstraction Layer

Decouples Aivexa Core from vendor SDKs. Providers implement the standard `ModelProvider` interface:

```typescript
export interface ModelProvider {
  providerId: string;
  listModels(): Promise<ProviderModelDescriptor[]>;
  complete(request: NormalizedCompletionRequest): Promise<NormalizedCompletionResponse>;
  stream(request: NormalizedStreamRequest): Promise<ReadableStream<NormalizedStreamChunk>>;
  embed(request: NormalizedEmbedRequest): Promise<NormalizedEmbedResponse>;
  transcribe(request: NormalizedAudioRequest): Promise<NormalizedTranscriptionResponse>;
  tts(request: NormalizedTTSRequest): Promise<NormalizedTTSResponse>;
  generateImage(request: NormalizedImageRequest): Promise<NormalizedImageResponse>;
  generateVideo(request: NormalizedVideoRequest): Promise<NormalizedVideoResponse>;
}
```

### Adding a New Provider
1. Create provider adapter in `src/gateway/providers/<provider-name>.ts` implementing `ModelProvider`.
2. Map vendor payload structures and stream events into `NormalizedCompletionResponse` / `NormalizedStreamChunk`.
3. Register the instance in `ProviderRegistry` and inject API keys via the Secrets Manager.

---

## 3. Model Registry & Database Schema

The DB-backed Model Registry maintains live configuration, pricing, capabilities, and health metrics.

```sql
CREATE TABLE models (
    model_id VARCHAR(64) PRIMARY KEY, -- e.g. 'gpt-4o-mini', 'claude-3-5-sonnet-20241022'
    provider VARCHAR(32) NOT NULL,    -- e.g. 'openai', 'anthropic', 'deepseek'
    capabilities TEXT[] NOT NULL,     -- {'chat','reasoning','coding','vision','embedding','stt','tts','image','video'}
    context_window INT NOT NULL,      -- Max token limit
    input_cost_per_1k NUMERIC(10, 6) NOT NULL,  -- USD per 1,000 input tokens
    output_cost_per_1k NUMERIC(10, 6) NOT NULL, -- USD per 1,000 output tokens
    avg_latency_ms INT DEFAULT 0,
    availability NUMERIC(3, 2) DEFAULT 1.00,
    version VARCHAR(32) NOT NULL,
    status VARCHAR(16) NOT NULL CHECK (status IN ('active', 'deprecated', 'testing')),
    routing_priority INT NOT NULL DEFAULT 10,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE model_aliases (
    alias VARCHAR(64) PRIMARY KEY, -- e.g. 'default-chat', 'default-reasoning'
    target_model_id VARCHAR(64) NOT NULL REFERENCES models(model_id) ON DELETE CASCADE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_models_provider_status ON models(provider, status);
CREATE INDEX idx_models_capabilities ON models USING GIN(capabilities);
```

---

## 4. Router Logic & Failover Policy

Selects the model based on task category, performance requirements, and real-time provider health.

### Routing Decision Table & Fallback Chains

| Task Type | Capability | Primary Model | Secondary Fallback | Tertiary Fallback |
| :--- | :--- | :--- | :--- | :--- |
| **Simple Question** | `chat` | `gpt-4o-mini` | `claude-3-5-haiku` | `deepseek-v3` |
| **Complex Reasoning** | `reasoning` | `o3-mini` | `deepseek-r1` | `claude-3-5-sonnet` |
| **Large Codebase** | `coding` | `claude-3-5-sonnet` | `gpt-4o` | `deepseek-v3` |
| **Image Vision Question** | `vision` | `gpt-4o` | `claude-3-5-sonnet` | `gemini-1-5-pro` |
| **Document Search (RAG)** | `embedding` + `chat` | `text-embedding-3-large` + `claude-3-5-sonnet` | `text-embedding-3-small` + `gpt-4o-mini` | `bge-m3` + `deepseek-v3` |

### Provider Failover & Retry Strategy
- **Failover Triggers**: Executed on HTTP `429` (Rate Limit), `5xx` (Server Error), or request `Timeout` (>15s).
- **Retry Policy**: Up to 3 max attempts with exponential backoff ($delay = min(10000, 500 \times 2^{attempt} + jitter)$).
- **Circuit Breaker**: Tracks provider errors over a 60s window. If error rate > 30%, circuit trips to `OPEN` for 30s, routing requests directly to the secondary fallback model.

---

## 5. Streaming Architecture & Token Accounting

Gateway streams responses using standard Server-Sent Events (SSE).

```text
Provider API Stream ──► Gateway Stream Accumulator ──► Client SSE Stream
                                │
                                └── Token Counter ──► Finalize model_runs Log
```

- **Native Token Usage**: Extracts `usage` stats from provider stream completion frames when supported (e.g., OpenAI `include_usage: true`).
- **Tokenizer Fallback**: If provider stream usage is absent, a fast stream token accumulator (`tiktoken` / `cl100k_base`) counts tokens dynamically before closing the SSE connection.

---

## 6. Pre-Flight Budget Enforcement Hook

Acts as the gateway choke point checking estimated cost against organization budget limits before execution.

```typescript
export async function enforceBudget(userId: string, orgId: string, estimatedCostUsd: number): Promise<void> {
  const currentSpend = await budgetService.getSpend(orgId);
  const budgetLimit = await budgetService.getLimit(orgId);

  if (currentSpend + estimatedCostUsd > budgetLimit) {
    throw new GatewayError({
      code: 'BUDGET_EXCEEDED',
      message: `Budget exceeded. Current spend: $${currentSpend.toFixed(2)}, Limit: $${budgetLimit.toFixed(2)}.`,
      details: { orgId, currentSpend, budgetLimit, estimatedCostUsd }
    });
  }
}
```

---

## 7. Observability & `model_runs` Schema

Every model run is logged asynchronously to `model_runs` for audit, analytics, and billing.

```sql
CREATE TABLE model_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id VARCHAR(64) NOT NULL,
    user_id VARCHAR(64) NOT NULL,
    org_id VARCHAR(64) NOT NULL,
    task_type VARCHAR(32) NOT NULL,
    model_id VARCHAR(64) NOT NULL REFERENCES models(model_id),
    provider VARCHAR(32) NOT NULL,
    tokens_in INT NOT NULL DEFAULT 0,
    tokens_out INT NOT NULL DEFAULT 0,
    latency_ms INT NOT NULL,
    cost_usd NUMERIC(10, 6) NOT NULL DEFAULT 0.000000,
    success BOOLEAN NOT NULL,
    error_code VARCHAR(64),
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_model_runs_org_created ON model_runs(org_id, created_at DESC);
CREATE INDEX idx_model_runs_user_created ON model_runs(user_id, created_at DESC);
CREATE INDEX idx_model_runs_model_id ON model_runs(model_id);
```

---

## 8. Secrets Management & Key Rotation

- **Vault Storage**: Provider API keys are stored in AWS Secrets Manager / Vault (never in database or code).
- **In-Memory Caching**: Credential decryption cached with a 5-minute TTL.
- **Key Rotation**: Supports key rings (`PRIMARY`, `SECONDARY`) per provider for zero-downtime key rotation.

---

## 9. Core Gateway TypeScript Interfaces

```typescript
export interface GatewayCompletionRequest {
  requestId: string;
  userId: string;
  orgId: string;
  taskType: 'simple_question' | 'complex_reasoning' | 'coding' | 'vision' | 'rag_chat';
  messages: Array<{ role: 'system' | 'user' | 'assistant'; content: string }>;
  overrideModelAlias?: string;
  maxTokens?: number;
  temperature?: number;
}

export interface GatewayCompletionResponse {
  requestId: string;
  modelId: string;
  provider: string;
  content: string;
  tokensIn: number;
  tokensOut: number;
  costUsd: number;
  latencyMs: number;
}

export interface GatewayStreamChunk {
  requestId: string;
  delta: string;
  finishedReason?: 'stop' | 'length' | 'tool_calls';
  usage?: { tokensIn: number; tokensOut: number; costUsd: number };
}

export interface ModelGatewayService {
  routeAndComplete(request: GatewayCompletionRequest): Promise<GatewayCompletionResponse>;
  routeAndStream(request: GatewayCompletionRequest): Promise<ReadableStream<GatewayStreamChunk>>;
}
```

---

## 10. Model Deprecation & Versioning Strategy

- **Virtual Aliases**: Applications reference aliases (e.g. `default-chat`, `default-reasoning`) mapped to concrete `model_id`s in `model_aliases`.
- **Lifecycle States**: Models transition through `testing` ──► `active` ──► `deprecated`.
- **Zero-Downtime Migration**: Updating an alias target FK allows seamless model upgrades without modifying application code.

---

## 11. Open Questions

1. **Semantic Caching**: Should a Redis vector cache sit ahead of the gateway router to short-circuit repeated prompts?
2. **Local Fallback Nodes**: What error thresholds should trigger failover to self-hosted vLLM/Ollama fallback nodes?
3. **Enterprise BYOK**: How will tenant-provided API keys (Bring Your Own Key) integrate with the secrets manager cache?
