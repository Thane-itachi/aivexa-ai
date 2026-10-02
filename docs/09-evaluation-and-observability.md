---
title: Evaluation & Production Observability Architecture
summary: Technical architecture for Aivexa AI production observability (OpenTelemetry, Prometheus, Grafana, Loki, Sentry) and the offline/CI AI evaluation engine (prompt registry, LLM-as-judge, tool/coding benchmarks, human feedback, regression gating).
---

# 09 — Evaluation & Production Observability

Aivexa AI operates two distinct telemetry systems: **Production Observability** for real-time operational health and streaming performance, and the **AI Evaluation Platform** for deterministic and model-based quality testing across releases.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                   AIVEXA TELEMETRY                                     │
├───────────────────────────────────────────┬────────────────────────────────────────────┤
│       PART A: PRODUCTION OBSERVABILITY    │        PART B: AI EVALUATION PLATFORM      │
│  • Operational health & live user traffic  │  • Release gating & prompt version diffs   │
│  • Latency, throughput, errors, cost      │  • Accuracy, hallucinations, tool precision│
│  • OTel, Prometheus, Loki, Sentry, Grafana │  • Benchmark datasets, LLM-as-judge, CI/CD │
└───────────────────────────────────────────┴────────────────────────────────────────────┘
```

---

## PART A — Production Observability

Production observability captures real-time runtime diagnostics across the API Gateway, Agent Runtime, Worker Queues, and Sandbox environments.

### A.1 Architecture & Starting Telemetry Stack

Aivexa uses standard CNCF observability components to collect metrics, logs, and traces without vendor lock-in:

* **OpenTelemetry (OTel) Collector**: In-process SDKs emit traces and metrics via OTLP/gRPC to a unified collector daemon.
* **Prometheus & Grafana**: Pulls time-series metrics every 15s from OTel Collector for dashboards, alerts, and SLO monitoring.
* **Grafana Loki**: Tail and ingest structured JSON logs tagged with `trace_id`, `tenant_id`, and `agent_run_id`.
* **Sentry**: Captures unhandled client/server exceptions, attaching active span contexts and LLM context IDs.

```
┌─────────────┐    OTLP/gRPC    ┌─────────────────┐    PromQL    ┌─────────────┐
│ Next.js API ├────────────────►│                 ├─────────────►│ Prometheus  │
├─────────────┤                 │                 │              └──────┬──────┘
│ Agent Core  ├────────────────►│  OTel Collector │                     │ Grafana
├─────────────┤                 │   DaemonSet     │              ┌──────▼──────┐
│ Sandbox/Pool├────────────────►│                 ├─────────────►│    Loki     │
└─────────────┘                 └────────┬────────┘              └─────────────┘
                                         │ OTLP/HTTP
                                ┌────────▼────────┐
                                │     Sentry      │
                                └─────────────────┘
```

### A.2 Key Metrics & Tracing Hierarchy

Every inbound request creates an OpenTelemetry trace context. Spans propagate hierarchically:
`http.request` ➔ `agent.run` ➔ `model.completion` ➔ `tool.execute` ➔ `sandbox.command`.

#### Core Metric Categories
1. **Endpoint Latency Percentiles**: `http_request_duration_seconds{quantile="p50|p95|p99", endpoint="..."}`.
2. **Queue Depth & Worker Lag**: `bullmq_queue_depth{queue="agent_jobs"}` and `bullmq_job_latency_seconds`.
3. **Sandbox Pool Utilization**: `sandbox_pool_active_instances` vs `sandbox_pool_capacity` and warm-start allocation time.
4. **Token Throughput**: `llm_tokens_total{direction="input|output", provider="...", model="..."}` and `llm_tokens_per_second`.
5. **Error Rates**: `http_requests_failed_total{code="5xx"}` and `llm_api_errors_total{reason="rate_limit|timeout|context_length"}`.

### A.3 Correlation IDs & Structured Logging

Logs are formatted in JSON and enriched with correlation identifiers across all asynchronous boundaries (BullMQ, Redis PubSub, WebSockets):

```json
{
  "timestamp": "2026-10-02T06:26:00.120Z",
  "level": "info",
  "service": "agent-runtime",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "org_99x21",
  "agent_run_id": "run_881a02f",
  "prompt_version_id": "prompt_res_v3.1",
  "tool_call_id": "call_exec_py_41",
  "message": "Tool execution initiated in warm sandbox",
  "sandbox_id": "sbx_f8a912"
}
```

### A.4 Infrastructure Health & Token Usage Dashboards

* **Infra Monitoring**: PostgreSQL connection pool saturation (`pg_stat_activity`), pgvector HNSW index search latency, Redis memory usage/evictions, and S3 multipart upload success rate.
* **Token Cost Dashboard**: Aggregates `prompt_tokens` and `completion_tokens` per tenant, model, and prompt version to enforce real-time billing quotas and track cost efficiency over time.

### A.5 Service Level Objectives (SLOs)

| Metric | Target | Measurement Window | Alert Trigger Threshold |
| :--- | :--- | :--- | :--- |
| **Chat p95 Latency** | < 2,500 ms | 5-minute rolling | > 3,000 ms over 3m |
| **Stream Time-to-First-Token (TTFT)** | < 600 ms (p95) | 5-minute rolling | > 900 ms over 3m |
| **Agent Run Success Rate** | > 99.0% | 24-hour window | < 98.0% over 15m |
| **Tool Call Failure Rate** | < 1.5% (tech error) | 1-hour window | > 3.0% over 10m |

---

## PART B — AI Evaluation Platform

The AI Evaluation Platform verifies whether model updates, prompt changes, router logic, or tool additions improve system performance without causing regressions.

### B.1 Components

1. **Test Datasets (`eval_datasets`)**: Collections of domain-specific test cases across reasoning, coding, tool selection, and safety.
2. **Golden Answers**: Ground-truth references, accepted regexes, deterministic AST matches, or expected tool call schemas.
3. **Model Evaluation**: Direct assessment of raw model capability (latency, context window retention, format compliance).
4. **Agent Evaluation**: End-to-end evaluation of multi-turn trajectories, state transitions, and step limits.
5. **Tool-Use Evaluation**: Measures selection precision, argument payload validity, and missing-parameter handling.
6. **Safety Evaluation**: Automated red-teaming for indirect prompt injection, jailbreaks, data exfiltration, and PII leakage.
7. **Regression Tests**: Gatekeeping test suites executed against baseline outputs during code/prompt pull requests.
8. **Human Feedback Loop**: Production user ratings (`thumbs_up`, `thumbs_down`, explicit edits) ingested to grow test sets.
9. **Benchmark Dashboard**: Cross-version side-by-side performance matrix (e.g. Prompt v2 vs v3, Model v1.3 vs v1.4).

### B.2 Prompt Registry & Dynamic Resolution

Prompts are stored as immutable versioned database records rather than hardcoded string literals:

```
┌────────────────────────────────────────────────────────────────────────┐
│                             PROMPT REGISTRY                            │
├───────────────────┬───────────────────┬────────────────────────────────┤
│ Agent Prompts     │ System Prompts    │ Tool Instructions              │
│ Safety Guards     │ Routing Rules     │ Evaluation Prompts             │
└─────────┬─────────┴─────────┬─────────┴───────────────┬────────────────┘
          │                   │                         │
          ▼                   ▼                         ▼
   Tag: `@active`     Tag: `@staging`            Tag: `@eval-v3`
```

Engineers run evaluation suites against specific prompt database IDs without code deployments.

### B.3 Continuous Evaluation Pipeline

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Eval Case   │────►│ Agent / Model│────►│ Evaluator    │────►│ Metrics      │────►│ Regression   │
│  Dataset     │     │  Execution   │     │ Engine       │     │ Calculation  │     │ CI Gate      │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

The pipeline runs automatically on every PR modifying prompts, tool definitions, model routing parameters, or system context generation.

### B.4 Evaluation Metrics & Threshold Matrix

| Metric | Measurement Method | Target Threshold | Description |
| :--- | :--- | :--- | :--- |
| **Accuracy** | LLM-as-Judge + Exact Match | $\ge 92.0\%$ | Semantic correctness against golden answer |
| **Hallucination Rate** | LLM-as-Judge (Factuality) | $\le 2.0\%$ | Unsupported claims absent from context |
| **Citation Correctness** | Deterministic URL/Doc Check | $\ge 98.0\%$ | Verifiable source attribution |
| **Coding Success** | Sandbox Unit Test Execution | $\ge 95.0\%$ | Generated code passes unit test suite |
| **Reasoning Accuracy** | GSM8K / HumanEval Benchmark | $\ge 88.0\%$ | Step-by-step logic verification |
| **Tool Selection** | Deterministic JSON Schema | $\ge 98.0\%$ | Correct tool choice & valid parameters |
| **Instruction Adherence** | AST / Schema Validator | $\ge 99.0\%$ | Strict adherence to JSON/Markdown output rules |
| **Safety Guard** | Injection Resistance Suite | $100.0\%$ | Zero successful prompt injection/jailbreak |
| **Eval Latency** | Measured Runtime Duration | $\le 3,000$ ms | Average execution time per test case |
| **Eval Cost** | Token Pricing Calculation | $\le \$0.04$ | Average monetary cost per test case |

### B.5 Human Feedback Flywheel

1. **Capture**: Chat UI records user feedback (`thumbs_down` with optional text correction) alongside full trace context.
2. **Ingestion**: Event worker writes negative feedback instances to `feedback_store` table.
3. **Curation**: Domain expert reviews and converts high-signal feedback into new `eval_cases` with golden answers.
4. **Integration**: Evaluator incorporates new cases into nightly regression runs.

```
User Action (Thumbs Down) ──► feedback_store ──► Expert Curation ──► eval_cases ──► CI Regression Suite
```

### B.6 CI/CD Regression Gating

Pull requests execute `aivexa-eval-cli run --candidate-prompt-version=PR_104 --baseline=active`.
The CI runner compares candidate run scores against active baseline metrics:
* If `candidate.accuracy < baseline.accuracy - 0.01` ➔ **BLOCKED**
* If `candidate.safety < 1.0` ➔ **BLOCKED**
* If `candidate.tool_selection < baseline.tool_selection` ➔ **BLOCKED**

### B.7 Database Schema (DDL)

```sql
-- Evaluation Datasets
CREATE TABLE eval_datasets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(128) NOT NULL UNIQUE,
    description TEXT,
    category VARCHAR(64) NOT NULL,
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Individual Test Cases within a Dataset
CREATE TABLE eval_cases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES eval_datasets(id) ON DELETE CASCADE,
    input_prompt TEXT NOT NULL,
    system_context JSONB DEFAULT '{}'::jsonb,
    golden_answer TEXT,
    expected_tools JSONB DEFAULT '[]'::jsonb,
    assertion_rules JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Evaluation Suite Runs
CREATE TABLE eval_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES eval_datasets(id),
    prompt_version_id VARCHAR(128) NOT NULL,
    model_identifier VARCHAR(128) NOT NULL,
    environment VARCHAR(32) NOT NULL DEFAULT 'ci',
    status VARCHAR(32) NOT NULL DEFAULT 'pending',
    summary_metrics JSONB DEFAULT '{}'::jsonb,
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

-- Detailed Per-Case Results
CREATE TABLE eval_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES eval_runs(id) ON DELETE CASCADE,
    case_id UUID NOT NULL REFERENCES eval_cases(id) ON DELETE CASCADE,
    actual_output TEXT,
    actual_tool_calls JSONB DEFAULT '[]'::jsonb,
    scores JSONB NOT NULL,
    evaluator_reasoning TEXT,
    passed BOOLEAN NOT NULL DEFAULT FALSE,
    latency_ms INT NOT NULL,
    token_cost NUMERIC(10, 6) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_eval_cases_dataset ON eval_cases(dataset_id);
CREATE INDEX idx_eval_runs_dataset_env ON eval_runs(dataset_id, environment);
CREATE INDEX idx_eval_results_run ON eval_results(run_id);
```

---

## Open Questions

1. **LLM-as-Judge Calibration**: Should evaluator models be pinned to a specific external model version (e.g. GPT-4o-2024-08-06) or an internally fine-tuned evaluation model to ensure evaluator deterministic stability?
2. **Synthetic Dataset Generation**: What ratio of synthetic production-derived test cases versus hand-curated expert cases achieves optimal bug discovery without overfitting?
3. **Eval Run Cost Limits**: Should CI gating limit deep multi-step agent evaluations to nightly scheduled builds while running fast deterministic checks (AST, tool schema) on every individual PR commit?
