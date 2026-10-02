# Aivexa AI

An AI workspace platform for research, knowledge, and building. This repository currently contains the complete planning and engineering specification — no application code yet.

## Documentation

| Doc | Contents | Release |
| --- | --- | --- |
| `docs/01-architecture.md` | Full build-ready architecture: stack, layers, agents, RAG, data layer, workers, security, phases | All |
| `docs/02-v1-build-spec.md` | Ready-to-paste V1 builder prompt (chat + files + research + memory/RAG) | V1 |
| `docs/03-model-gateway.md` | Model Registry, provider abstraction, routing + failover, cost choke point, DDL | V1/V2 |
| `docs/04-agent-runtime.md` | Agent Registry, executor state machine, task DAG, retries, timeouts, runaway protection | V2 |
| `docs/05-tool-registry.md` | Tool catalog, metadata schema, execution path, confirmation handshake, tool author SDK | V2 |
| `docs/06-security-and-permissions.md` | Permission Engine, RBAC, AI safety/moderation pipeline, expanded audit logging | V1/V2 |
| `docs/07-sandbox-and-code-execution.md` | Sandbox Manager, resource limits, network policy, secret isolation, lifecycle | V2/V3 |
| `docs/08-data-and-database-schema.md` | Org → Workspace → Project hierarchy, full DB schema, RLS, data lifecycle/privacy, backup/DR | V1+ |
| `docs/09-evaluation-and-observability.md` | Observability stack, SLOs, AI evaluation platform, regression gating | V2+ |
| `docs/10-admin-billing-platform.md` | Admin console, billing tiers, cost controller, notifications, feature flags | V4 |
| `docs/11-training-and-model-roadmap.md` | Training/data-governance roadmap, API developer platform, V1-V5 release roadmap | V5 |

## Release roadmap

V1 intentionally stays small — a real foundation, not a collection of disconnected features:

```text
V1  Chat, Files, RAG, Memory, Research, Model Gateway
V2  Agent Runtime, Tool Registry, Coding Agent, Sandbox, Permission Engine
V3  Aivexa Builder (app generation, live preview, deployment)
V4  Automation (workflows, browser agent, integrations) + Billing + Admin Console
V5  Aivexa proprietary models + API developer platform + enterprise workspaces
```

## Status

- No application code has been written yet.
- Next step: create the "Aivexa AI" app at https://app.base44.com and paste the V1 build spec (`docs/02-v1-build-spec.md`) as the first prompt.
- The Model Gateway, Agent Runtime, Tool Registry, permissions, sandbox, and evaluation subsystems should be implemented against their specs (docs 03-09) — do not let a builder improvise them.
