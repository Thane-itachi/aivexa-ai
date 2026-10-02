---
title: "Aivexa AI — Admin Console, Billing, Cost Control, Notifications & Feature Flags"
summary: "Production architecture specification for platform administration, subscription billing, pre-execution cost control and runaway protection, multi-channel notifications, and server-side feature flags."
---

# Admin Console, Billing, Cost Control, Notifications & Feature Flags

## 1. Overview & System Context

Aivexa AI operates as a modular monolith backed by PostgreSQL, Redis/BullMQ, and S3. Because agents execute long-running, multi-step tasks across external tools and LLMs, the platform requires unified administration, strict usage tracking, pre-execution cost protection, reliable alerting, and safe capability deployment. This document defines the architecture across these five platform foundation services.

---

## 2. Part A — Admin Console (`/admin`)

The Admin Console is a role-gated operational interface restricted to internal administrators with active multi-factor authentication and role assignments (`super_admin`, `support_admin`, `security_admin`, `billing_admin`).

### 2.1 Console Sections & Telemetry Views

1. **Users (`/admin/users`)**:
   - Accounts, active sessions, subscription tier, billing status, total spend, daily/monthly token consumption.
   - User detail view displays per-agent run history, connected OAuth accounts, rate-limit status, and storage utilization.
2. **AI Engine (`/admin/ai`)**:
   - Provider status (OpenAI, Anthropic, Bedrock, local fallbacks), model latencies (p50/p95/p99), error rates, token throughput.
   - Prompt registry version tree, active agent configurations, tool catalog status, and automated evaluation benchmark scores.
3. **Infrastructure (`/admin/infrastructure`)**:
   - BullMQ queue metrics (waiting, active, completed, failed jobs across `agent-execution`, `notifications`, `media-gen`).
   - Node process memory, CPU utilization, Redis cache hit ratios, S3 storage usage (GB per tier), and database connection pool stats.
4. **Security & Compliance (`/admin/security`)**:
   - Live audit stream, security event log (failed log-ins, privilege escalations, suspicious API bursts), tool execution safety logs, flagged prompts, and abuse reports.

### 2.2 Privileged Admin Actions & Immediate Effects

- **Block User**: Instantly revokes session tokens, marks user status as `suspended`, halts active BullMQ agent tasks, and rejects inbound API calls.
- **Disable Tool**: Toggles tool status in the Tool Registry to `disabled`. Model Gateway automatically omits disabled tool definitions from context windows.
- **Roll Back Prompt Version**: Updates active semver pointer in `prompt_versions` table. Next agent iteration uses the restored system prompt instantly.
- **Kill Runaway Agent Run**: Issues a cancellation signal to BullMQ worker, revokes agent sandbox context, and writes a `terminated_by_admin` outcome record.

### 2.3 Audit Logging Specification

Every admin action triggers a synchronous insert into `audit_events` within the same database transaction where applicable:

```sql
CREATE TABLE audit_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id UUID NOT NULL REFERENCES users(id),
    action VARCHAR(64) NOT NULL, -- e.g., 'user.block', 'tool.disable', 'prompt.rollback', 'agent_run.kill'
    target_type VARCHAR(32) NOT NULL, -- 'user', 'tool', 'prompt', 'agent_run'
    target_id VARCHAR(128) NOT NULL,
    reason TEXT NOT NULL,
    payload JSONB DEFAULT '{}'::jsonb,
    ip_address INET NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_events_admin_time ON audit_events(admin_id, created_at DESC);
CREATE INDEX idx_audit_events_target ON audit_events(target_type, target_id);
```

---

## 3. Part B — Billing & Usage Metering

### 3.1 Tier Structure & Limit Definitions

| Tier | Monthly Price | Monthly Messages / Tokens | Knowledge Bases | Automations | Storage | Special Features |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **FREE** | $0 | 50 msg/day (~100k tokens) | 3 KBs | 0 (Manual only) | 100 MB | Shared queue, basic tools |
| **PRO** | $20 / mo | 3,000 msg/mo (~5M tokens) | 25 KBs | 10 active | 10 GB | Priority queue, web search, code sandbox |
| **TEAM** | $80 / mo (5 seats) | 15,000 msg/mo (~25M tokens) | 100 KBs | 50 active | 100 GB | Shared workspaces, custom tools, RBAC |
| **BUSINESS**| $300 / mo (20 seats)| 75,000 msg/mo (~120M tokens)| 500 KBs | 250 active | 500 GB | Dedicated sandbox pool, fine-tuned models |
| **ENTERPRISE**| Custom | Custom token allocation | Unlimited | Unlimited | Custom | SSO/SAML, custom SLAs, single-tenant |

### 3.2 Centrally Tracked Metering Choke Points

Metering is strictly enforced at two choke points: **Model Gateway** (tracking LLM tokens) and **Tool Registry** (tracking browser sandbox minutes, media generation, S3 storage, API calls).

```sql
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    metric VARCHAR(64) NOT NULL, -- 'tokens_input', 'tokens_output', 'image_gen', 'sandbox_sec', 'storage_bytes', 'api_req'
    quantity NUMERIC(14, 4) NOT NULL,
    unit_cost NUMERIC(10, 6) NOT NULL,
    total_cost NUMERIC(12, 6) GENERATED ALWAYS AS (quantity * unit_cost) STORED,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_usage_records_org_time ON usage_records(org_id, occurred_at DESC);
CREATE INDEX idx_usage_records_user_metric ON usage_records(user_id, metric, occurred_at DESC);
```

### 3.3 Subscription Lifecycle & Webhooks

- **Provider**: Stripe Billing handles payment methods, invoices, recurring subscriptions, and tax calculation.
- **Webhook Handlers**:
  - `customer.subscription.created`: Provision tier quotas in Redis and PostgreSQL.
  - `customer.subscription.updated`: Re-evaluate seats and limits immediately. Handle mid-cycle tier upgrades via Stripe proration logic.
  - `customer.subscription.deleted`: Revert organization to FREE tier; archive excess knowledge bases and automations.
  - `invoice.payment_failed`: Trigger 7-day grace period; notify billing admin via notification service. If unpaid after 7 days, set status to `past_due` and downgrade limits.

---

## 4. Part C — Cost Control System

Agent loops (e.g., search → model → search → model) risk exponential cost explosion without strict pre-execution budget gating.

### 4.1 Cost Controller Architecture & Budget Matrix

Budgets are checked against Redis rolling meters synced with `usage_records`:
- **Token Budget**: Max input/output tokens per turn, per run, and per billing cycle.
- **Agent-Run Budget**: Hard upper limit on cumulative dollars per agent invocation (e.g., max $2.50 per run).
- **Tool Budget**: Max executions per tool type (e.g., max 10 web searches or 5 code sandbox executions per run).
- **Request & Quota Limits**: Per-minute rate limits, daily quotas, and org-level monthly dollar caps.

### 4.2 Pre-Execution Gate Flow

Every step in an agent workflow must request execution clearance from the Pre-Execution Gate prior to calling an LLM or executing a tool.

```text
  ┌───────────────────────────┐
  │ Agent Turn / Tool Request │
  └─────────────┬─────────────┘
                │
                ▼
  ┌───────────────────────────┐
  │   Pre-Execution Gate      │
  │ (Model Gateway / Tool Reg)│
  └─────────────┬─────────────┘
                │
     Estimate Step Cost &
     Check Redis Usage Counters
                │
        ┌───────┴───────┐
        ▼               ▼
  [ Budget OK ]   [ Budget Exceeded ]
        │               │
        │               ├──► Write usage breach log
        │               └──► Return structured error:
        │                    {
        │                      "status": "budget_exceeded",
        │                      "metric": "agent_run_dollar_cap",
        │                      "limit": 2.50,
        │                      "current": 2.51,
        │                      "reset_at": "2026-10-02T07:00:00Z"
        │                    }
        ▼
  Execute Step &
  Record Usage
```

### 4.3 Runaway-Loop Protection

1. **Loop Signature Hashing**: Tool Registry maintains a sliding window of the last 10 tool calls per run. If `hash(tool_name + normalized_args)` repeats > 3 times consecutively or > 5 times in 10 turns, the run is terminated with `loop_detected`.
2. **Turn Depth Cap**: Hard limit of 25 iterations per agent run.
3. **Hard Cost Cap**: Agent context immediately terminates if cumulative run spend hits $2.50 (PRO) or $10.00 (ENTERPRISE).
4. **Velocity Anomaly Alerts**: If an org consumes > 300% of its average hourly token usage in under 10 minutes, trigger an automated freeze on high-cost tools and alert admins via Slack/PagerDuty.

---

## 5. Part D — Notification Engine

### 5.1 Multi-Channel Architecture

The notification service routes system and user events across four delivery channels:
- **In-App**: Real-time push over SSE/WebSocket, backed by persistent database storage.
- **Email**: Transactional delivery via Resend/SendGrid for billing and summary digests.
- **Push**: WebPush / FCM for urgent human-in-the-loop approvals.
- **Webhooks**: Outbound HTTP POST payloads for external developer integrations.

### 5.2 Schema & Storage

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    event_type VARCHAR(64) NOT NULL, -- 'task_completed', 'tool_approval_required', 'quota_warning', 'billing_failed'
    channel VARCHAR(32) NOT NULL, -- 'in_app', 'email', 'push', 'webhook'
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    data JSONB DEFAULT '{}'::jsonb,
    status VARCHAR(32) NOT NULL DEFAULT 'pending', -- 'pending', 'sent', 'failed'
    retry_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sent_at TIMESTAMPTZ
);

CREATE INDEX idx_notifications_user_status ON notifications(user_id, status) WHERE status = 'pending';

CREATE TABLE notification_preferences (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    in_app_enabled BOOLEAN NOT NULL DEFAULT true,
    email_enabled BOOLEAN NOT NULL DEFAULT true,
    push_enabled BOOLEAN NOT NULL DEFAULT false,
    webhook_url TEXT,
    event_settings JSONB NOT NULL DEFAULT '{
        "quota_warning": ["in_app", "email"],
        "tool_approval": ["in_app", "push"],
        "task_completed": ["in_app"]
    }'::jsonb
);
```

### 5.3 Event Delivery & Retry Policy

All notifications pass through a dedicated BullMQ queue (`notification-delivery`). Fails trigger exponential backoff:
- Attempt 1: Immediate.
- Attempt 2: +15 seconds.
- Attempt 3: +2 minutes.
- Attempt 4: +15 minutes.
After 4 failed attempts, the job is moved to the Dead-Letter Queue (DLQ) and flagged as `failed`.

---

## 6. Part E — Feature Flags & Experimentation

### 6.1 Server-Side Evaluation Architecture

Feature flags are strictly evaluated on the server side (in Next.js Server Components, API middleware, and BullMQ worker tasks) to eliminate client-side tampering or leak of internal feature names.

### 6.2 Schema Definition

```sql
CREATE TABLE feature_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key VARCHAR(64) UNIQUE NOT NULL, -- e.g., 'builder_v2', 'new_model_router', 'new_research_engine'
    description TEXT,
    percentage_rollout INT NOT NULL DEFAULT 0 CHECK (percentage_rollout BETWEEN 0 AND 100),
    enabled BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE flag_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_key VARCHAR(64) NOT NULL REFERENCES feature_flags(key) ON DELETE CASCADE,
    target_type VARCHAR(32) NOT NULL, -- 'user', 'org', 'cohort'
    target_id VARCHAR(128) NOT NULL, -- user_id, org_id, or cohort name e.g. 'beta'
    enabled BOOLEAN NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(flag_key, target_type, target_id)
);
```

### 6.3 Evaluation Logic & Risky Subsystem Kill Switches

1. **Explicit Overrides**: Check `flag_overrides` for `target_type = 'user'`, `target_type = 'org'`, or `target_type = 'cohort'`.
2. **Deterministic Hash Rollout**: Calculate `fnv1a(user_id + ":" + flag_key) % 100 < percentage_rollout`.
3. **Subsystem Flag Allocations**:
   - `builder_v2`: 10% percentage rollout.
   - `new_model_router`: 5% percentage rollout.
   - `new_research_engine`: Override enabled for target_type='cohort', target_id='beta'.
4. **Global Kill Switch**: Setting `enabled = false` on `feature_flags` immediately disables the feature globally across all nodes via Redis Pub/Sub flag state invalidation.

---

## 7. Open Questions

1. **Real-time Quota Synchronization**: How can high-throughput token counters in Redis best handle sub-second multi-region synchronization without introducing write latency on agent turns?
2. **Human-in-the-Loop Timeout Behavior**: When an agent pauses pending user notification approval for a high-risk tool call, how long should the BullMQ task hold its lock before timing out and rolling back?
3. **Proration Abuse Protection**: Should instant mid-cycle subscription upgrades re-credit full token quotas immediately, or should token top-ups be strictly pro-rated to prevent upgrade/downgrade quota manipulation?
4. **Fine-Grained Audit Retention**: What retention schedule and cold-storage tiering policy (e.g. S3 Glacier after 90 days) should be enforced for high-volume tool execution audit events?
