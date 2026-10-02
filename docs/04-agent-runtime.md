---
title: "Aivexa AI — Subsystem Spec 04: Agent Runtime"
summary: "Forward specification for the V2 Agent Runtime: state machine, Redis/BullMQ worker architecture, DAG planner, DDL schemas, resilience, runaway controls, and end-to-end trace."
---

# 04. Agent Runtime Subsystem Spec (V2 Forward Spec)

## 1. Overview

While Aivexa V1 operates as a synchronous, single-turn RAG and chat assistant over SSE streams, V2 introduces the **Agent Runtime** — an asynchronous execution engine capable of executing complex, multi-step goals such as *"Research Acme Corp, analyze their financial dataset, synthesize a report, and email it to stakeholders."*

Long-running agent workflows cannot be tied to held-open HTTP/SSE connections. The Agent Runtime converts incoming user requests into durable jobs processed asynchronously by worker nodes using Redis + BullMQ, updating state deterministically in PostgreSQL and broadcasting status updates over WebSockets/SSE events.

---

## 2. Architecture & Components

```text
User Request ──► API Gateway ──► Enqueue Run ──► Redis Queue (BullMQ)
                                                     │
 ┌───────────────────────────────────────────────────┴───────────────────────────────────────────────────┐
 │ BullMQ Worker Node                                                                                    │
 │                                                                                                       │
 │ ┌──────────────┐   ┌───────────────┐   ┌────────────────┐   ┌─────────────────┐   ┌─────────────────┐ │
 │ │ Agent        │──►│ Task Planner  │──►│ Context        │──►│ Permission      │──►│ Tool Executor   │ │
 │ │ Registry     │   │ (DAG Builder) │   │ Manager        │   │ Manager         │   │ (Idempotent)    │ │
 │ └──────────────┘   └───────────────┘   └────────────────┘   └─────────────────┘   └────────┬────────┘ │
 │         ▲                  │                   │                     │                     │      │
 │         │                  ▼                   ▼                     ▼                     ▼      │
 │ ┌───────┴──────┐   ┌───────────────┐   ┌────────────────┐   ┌─────────────────┐   ┌─────────────────┐ │
 │ │ Result       │◄──│ Cost & Timeout│◄──│ Loop & Runaway │◄──│ Retry Manager   │◄──│ Postgres State  │ │
 │ │ Validator    │   │ Managers      │   │ Detector       │   │ & Backoff       │   │ (Runs & Steps)  │ │
 │ └──────────────┘   └───────────────┘   └────────────────┘   └─────────────────┘   └─────────────────┘ │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Agent Registry
Stores definitions, capabilities, budgets, and constraints for registered agents.

```sql
CREATE TABLE agents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  name VARCHAR(128) NOT NULL,
  slug VARCHAR(128) NOT NULL UNIQUE,
  description TEXT,
  system_instructions_id UUID NOT NULL,
  model_config JSONB NOT NULL DEFAULT '{"provider": "anthropic", "model": "claude-3-5-sonnet", "temperature": 0.2}',
  allowed_tools TEXT[] NOT NULL DEFAULT '{}',
  permissions TEXT[] NOT NULL DEFAULT '{"READ_FILE", "WEB_ACCESS"}',
  default_token_budget INT NOT NULL DEFAULT 100000,
  default_cost_limit_usd NUMERIC(10,4) NOT NULL DEFAULT 2.0000,
  output_schema JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 2.2 Agent Executor
A deterministic state machine executing inside BullMQ background worker threads. Each run transitions through five phases:
`queued` ──► `planning` ──► `executing` ──► `verifying` ──► `done` / `failed`

- **queued**: Job ingested into BullMQ `agent-execution-queue`; worker claims job lock.
- **planning**: LLM calls decompose user intent into a Directed Acyclic Graph (DAG) of steps.
- **executing**: Steps executed sequentially or in parallel based on DAG dependencies.
- **verifying**: Final output validated against `agents.output_schema`.
- **done / failed**: Final state written to PostgreSQL; job released; user notified via SSE/WebSocket.

### 2.3 Task Planner & DAG Persistence
Decomposes goals into discrete, dependency-aware steps. Step graphs are saved to the `tasks` table before execution begins.

```sql
CREATE TABLE tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id UUID NOT NULL REFERENCES agent_runs(id) ON DELETE CASCADE,
  name VARCHAR(256) NOT NULL,
  execution_order INT NOT NULL DEFAULT 1,
  dependencies UUID[] DEFAULT '{}',
  tool_name VARCHAR(128),
  input_params JSONB NOT NULL DEFAULT '{}',
  output_result JSONB,
  status VARCHAR(32) NOT NULL DEFAULT 'pending', -- pending, ready, running, completed, failed, skipped
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_tasks_run_id ON tasks(run_id);
```

### 2.4 Task State & Resumability
Every state change, step input, and step result is persisted synchronously to Postgres. If a BullMQ worker crashes mid-execution, the replacement worker inspects `agent_runs` and `tasks`, identifies uncompleted node dependencies, and resumes execution seamlessly without re-running completed steps.

### 2.5 Tool Selection
Before executing a step, the worker resolves requested tools against `agents.allowed_tools` and system-wide Tool Registry definitions, rejecting any unauthorized tool invocation before model routing.

### 2.6 Context Manager
Constructs a minimal, targeted context window per step consisting of:
1. Agent system instructions & goal prompt.
2. Short-term conversation history & relevant user memory items.
3. RAG vector chunks retrieved for the specific step task.
4. Outputs from parent prerequisite DAG steps (pruned to schema outputs).

### 2.7 Permission Manager
Enforces strict server-side authorization checks on every tool call independently of the model.
- Checks user RBAC (`organization_id`, `workspace_id`).
- Verifies tool scope (e.g., `EMAIL_SEND` or `WRITE_FILE`).
- Escalates high-risk operations (e.g., financial transactions, external webhooks) to a `human_approval_required` state.

### 2.8 Retry Manager
Handles transient execution failures using exponential backoff with full jitter (base 2s, max 30s, max 3 attempts per step).
- **Retryable**: Network timeouts, HTTP 429/502/503, rate limits, transient tool errors.
- **Non-retryable**: Permission denied, tool schema validation errors, cost cap exceeded.

### 2.9 Timeout Manager
- **Per-step timeout**: Default 60 seconds per tool/step execution.
- **Total-run timeout**: Default 15 minutes per agent run.
- **Expiry handling**: On expiry, the timeout manager cancels pending BullMQ jobs, marks the step `timed_out`, surfaces partial results, and updates `agent_runs.status = 'failed'`.

### 2.10 Cost Manager
Monitors aggregate token usage and dollar costs in real time across steps. When cumulative cost exceeds `agent_runs.budget_limit_usd`, execution halts immediately, logs `error_detail = {"reason": "BUDGET_EXCEEDED"}`, and saves partial outputs.

### 2.11 Result Validator
Before transitioning a run to `done`, the Result Validator validates the step output payload against `agents.output_schema` using Zod/JSON-Schema validation. If validation fails, it triggers a single re-planning/repair step.

---

## 3. Core Schemas (agent_runs & agent_steps DDL)

```sql
CREATE TYPE agent_run_status AS ENUM (
  'queued', 'planning', 'executing', 'verifying', 'done', 'failed', 'cancelled', 'halted_user_input'
);

CREATE TYPE agent_step_status AS ENUM (
  'pending', 'running', 'completed', 'failed', 'retrying', 'skipped'
);

CREATE TABLE agent_runs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID NOT NULL REFERENCES agents(id),
  user_id UUID NOT NULL,
  conversation_id UUID,
  status agent_run_status NOT NULL DEFAULT 'queued',
  input JSONB NOT NULL,
  output JSONB,
  total_tokens_used INT NOT NULL DEFAULT 0,
  total_cost_usd NUMERIC(10, 6) NOT NULL DEFAULT 0.000000,
  budget_limit_usd NUMERIC(10, 6) NOT NULL DEFAULT 2.000000,
  max_steps INT NOT NULL DEFAULT 25,
  current_step_count INT NOT NULL DEFAULT 0,
  error_detail JSONB,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE agent_steps (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id UUID NOT NULL REFERENCES agent_runs(id) ON DELETE CASCADE,
  step_number INT NOT NULL,
  status agent_step_status NOT NULL DEFAULT 'pending',
  tool_name VARCHAR(128),
  input_payload JSONB NOT NULL DEFAULT '{}',
  output_payload JSONB,
  tokens_used INT NOT NULL DEFAULT 0,
  cost_usd NUMERIC(10, 6) NOT NULL DEFAULT 0.000000,
  idempotency_key VARCHAR(256) NOT NULL UNIQUE,
  retry_count INT NOT NULL DEFAULT 0,
  error_detail JSONB,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agent_runs_user ON agent_runs(user_id, status);
CREATE INDEX idx_agent_steps_run_step ON agent_steps(run_id, step_number);
```

---

## 4. Failure Semantics & Resilience

- **Idempotency Keys**: Generated as `hash(run_id + ":" + step_number + ":" + retry_count)`. Tools (e.g., email sending, S3 writes) pass this key to prevent double execution on worker re-claims.
- **Poison-Message Handling**: Jobs failing 3 consecutive attempts due to unhandled worker crashes are flagged as poison messages by BullMQ and auto-removed from active processing.
- **Dead-Letter Queue (DLQ)**: Failed runs/steps are pushed to `agent-execution-dlq` for telemetry inspection, manual re-triggering, or offline debugging.
- **Partial Results Recovery**: Step outputs are committed atomically upon step completion. If step $N$ fails, results from steps $1 \dots N-1$ remain stored and accessible to the user in the UI.

---

## 5. Runaway Protection & Safety Controls

1. **Max Steps per Run**: Capped at `agent_runs.max_steps` (default: 25). Run terminates automatically if threshold is crossed.
2. **Max Tool Calls per Step**: Individual steps are limited to 5 sub-tool invocations.
3. **Loop Detection**: The executor maintains a rolling hash window of the last 5 tool calls `(tool_name, SHA256(canonical_args))`. If 3 identical calls occur consecutively, the run transitions to `halted_user_input`, alerting the user: *"Agent detected a potential execution loop during search. Proceed or cancel?"*

---

## 6. End-to-End Trace: Company Research Task

**Goal**: *"Research Acme Corp, analyze the data, create a report, and send it to me via email."*

```text
1. API Gateway ──► Validates request, creates `agent_runs` record (status: queued), enqueues BullMQ job.
2. Worker Claim ──► BullMQ worker acquires lock; updates `agent_runs.status = 'planning'`.
3. Agent Registry ──► Loads 'Research & Analysis Agent' definition, tools, permissions, and cost limits.
4. Task Planner ──► Calls LLM planner; builds DAG persisted in `tasks`:
   - Task 1: `web_search`("Acme Corp company overview and financial data") [deps: []]
   - Task 2: `data_analyzer`("Analyze web_search financial metrics") [deps: [Task 1]]
   - Task 3: `report_generator`("Compile markdown report from analysis") [deps: [Task 2]]
   - Task 4: `email_sender`("Send compiled report to user email") [deps: [Task 3]]
5. Step 1 Executing ──► Context Manager retrieves query; Permission Manager verifies `WEB_ACCESS`;
   Tool Executor calls search tool (idempotency: `run_123:step_1:0`); saves output to `tasks` and `agent_steps`.
6. Step 2 Executing ──► Context Manager injects Task 1 output; code sandbox executes analysis script;
   Cost & Timeout managers update accumulated tokens and run time.
7. Step 3 Executing ──► Report generator formats final markdown document artifact.
8. Step 4 Executing ──► Permission Manager identifies `EMAIL_SEND` capability; executes email tool;
   persists sent message ID to step output.
9. Verification ──► Result Validator checks output against `agents.output_schema`; marks `agent_runs.status = 'done'`.
10. Notification ──► Realtime SSE event sent to UI; user views report & delivery status.
```

---

## 7. Open Questions

1. **Long-Running Workflow Engine**: Should multi-day human-in-the-loop approvals stay on BullMQ with delayed jobs or transition to Temporal in V3?
2. **Dynamic Tool Sandbox**: What CPU/RAM limits and kernel isolation mechanisms (e.g., Firecracker microVMs vs gVisor containers) will code execution steps enforce?
3. **Agent-to-Agent Communication**: When an agent delegates a task to another specialized agent, should child tasks run inside a nested BullMQ child job or as an independent top-level `agent_runs` instance?
