---
title: Sandbox Manager & Code Execution Specification
summary: Technical specification for secure, isolated code execution environments in Aivexa AI using gVisor container isolation, strict resource bounds, network egress proxies, and lifecycle management.
---

# Sandbox Manager & Code Execution Specification

## 1. Overview & Trust Boundaries

The Coding Agent and Aivexa Builder generate, install, and execute arbitrary code on behalf of users. Platform security dictates that **generated code NEVER executes directly on the API server or background job worker nodes**. Untrusted code, untrusted package dependencies (npm, PyPI), and prompt-injected payloads present severe security threats, including remote code execution (RCE), host compromise, credential exfiltration, and lateral movement.

### Trust Boundary Architecture

```text
┌────────────────────────────────────────────────────────────────────────┐
│ TRUST ZONE 0: API Server & Core Services (Next.js Node.js Server)      │
│ - Serves UI, validates requests, enforces RBAC/quotas                  │
│ - Direct access to PostgreSQL, Redis, Platform S3 Buckets, Master Keys │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ Enqueues Execution Task
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ TRUST ZONE 1: Execution Workers (BullMQ Worker Fleet)                  │
│ - Orchestrates agent loops and calls Sandbox Manager API              │
│ - Holds temporary S3 presigned URLs, scoped execution tokens           │
│ - NO execution of user/generated scripts                               │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ Docker API / gVisor Runtime Call
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ TRUST ZONE 2: Isolated Sandbox Container (gVisor runsc Runtime)        │
│ - Runs generated user code, package installs, tests, and builds        │
│ - Ephemeral, zero platform secrets, read-only root FS                 │
│ - Strictly isolated network (Default Deny) via Proxy                   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Technology Choice & Architecture

### Concrete Starting Stack
- **Container Engine**: Docker Engine managed via Node.js Dockerode / Container API.
- **Runtime**: **gVisor (`runsc`)** sandbox runtime. gVisor intercepts application system calls in user space, presenting a virtualized Linux kernel interface to prevent kernel privilege escalation or escape.
- **Environment Ephemerality**: One container per execution request (or active agent iteration). Containers are spawned on-demand and strictly destroyed immediately after task completion or timeout.

### Scalability & Upgrade Path
- **Phase 1 (V1)**: Docker + gVisor (`runsc`) on dedicated Linux worker nodes.
- **Phase 2 (Scale)**: AWS Fargate / Fly.io Machines or Firecracker microVMs (via Firecracker containerd plugin) for hardware-assisted VM-level multi-tenant isolation across distributed worker pools.

---

## 3. Sandbox Manager API

The Sandbox Manager is exposed as a TypeScript module (`@aivexa/sandbox-manager`) used by BullMQ worker processes.

```typescript
export interface SandboxLimits {
  cpuCores: number;        // e.g. 2.0 vCPU
  memoryMb: number;        // e.g. 2048 MiB
  diskQuotaMb: number;     // e.g. 10240 MiB (10 GiB)
  pidsLimit: number;       // e.g. 256
  timeoutSeconds: number;  // e.g. 300 (default 5m, max 1800)
}

export interface EnvironmentConfig {
  agentRunId: string;
  userId: string;
  baseImage: 'node:20-alpine' | 'python:3.11-slim' | 'ubuntu:22.04';
  limits?: Partial<SandboxLimits>;
  envVars?: Record<string, string>;
}

export interface ExecutionOptions {
  command: string;
  args?: string[];
  workingDir?: string;
  timeoutMs?: number;
  env?: Record<string, string>;
}

export interface ExecutionResult {
  exitCode: number;
  stdout: string;
  stderr: string;
  truncated: boolean;
  durationMs: number;
}

export interface ArtifactResult {
  key: string;
  s3Url: string;
  sizeBytes: number;
}

export interface ISandboxManager {
  createEnvironment(config: EnvironmentConfig): Promise<string>; // returns sandboxId
  destroyEnvironment(sandboxId: string): Promise<void>;
  executeCommand(sandboxId: string, options: ExecutionOptions): Promise<ExecutionResult>;
  writeFile(sandboxId: string, filePath: string, content: string | Buffer): Promise<void>;
  readFile(sandboxId: string, filePath: string): Promise<string>;
  runTests(sandboxId: string, testCommand?: string): Promise<ExecutionResult>;
  runBuild(sandboxId: string, buildCommand?: string, artifactGlob?: string): Promise<{ result: ExecutionResult; artifacts: ArtifactResult[] }>;
  getLogs(sandboxId: string, tailLines?: number): Promise<string[]>;
}
```

---

## 4. Resource Limits per Sandbox

Hard limits are enforced at the gVisor / Cgroups v2 layer per sandbox instance:

| Resource | Default Limit | Maximum Allowed | Enforcement Mechanism |
| :--- | :--- | :--- | :--- |
| **CPU** | 2 vCPU (`--cpus=2.0`) | 4 vCPU | Docker Cgroups (`cpu.max`) |
| **Memory** | 2 GiB (`--memory=2g`) | 4 GiB | OOM Killer inside gVisor |
| **Disk Space** | 10 GiB tmpfs/volume | 20 GiB | Overlay FS quota / Docker `storage-opt` |
| **Processes** | 256 PIDs (`--pids-limit=256`) | 512 PIDs | Cgroups `pids.max` (Fork bomb prevention) |
| **Execution Timeout** | 5 minutes (300s) | 30 minutes (1800s) | Worker watchdog timer + Docker timeout |
| **Concurrency** | 3 active sandboxes / user | 10 active (Tier 2/3) | Redis semaphore in Sandbox Orchestrator |

---

## 5. Network Policy & Secret Isolation

### Network Policy
- **Default Action**: `DEFAULT DENY ALL` (Container spawned with `--network aivexa-sandbox-net`).
- **Internal Protection**: IPTables and network namespaces explicitly block access to:
  - Internal subnet (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`)
  - PostgreSQL DB server, Redis cache, BullMQ internal queues
  - Cloud Instance Metadata Service (`169.254.169.254`)
- **Egress Proxy**: Outbound HTTP/HTTPS traffic is routed strictly through an egress caching proxy (e.g., Squid / Envoy).
  - Allowed targets: Approved package registries (`registry.npmjs.org`, `pypi.org`, `github.com` git read-only).

### Secret Isolation
- Platform credentials (database connection strings, master API keys, S3 admin keys) are **never** mounted or injected into sandbox containers.
- If an agent task requires external API credentials (e.g., user's target app key), only scoped, short-lived runtime environment variables provided explicitly for that task are injected.

---

## 6. Filesystem & Dependency Isolation

### Filesystem Isolation
- **Base Image**: Mounted as **read-only** (`--read-only`).
- **Workspace**: `/workspace` mounted as a clean, temporary tmpfs/ephemeral volume.
- **Temp Directories**: `/tmp` and `/run` mounted as isolated `tmpfs` mounts with `noexec` where feasible.
- **Teardown**: Container and scratch volumes are wiped completely upon destruction.

### Dependency Installation
- Package installations (`npm install`, `pip install`) run inside `/workspace`.
- Requests pass through an egress proxy caching mirror (e.g., Verdaccio / DevPi) to ensure speed, integrity, and registry allowlisting.
- **Safety Controls**:
  - Max download payload cap per run: 500 MB.
  - Package install timeout: 3 minutes.
  - Lockfile enforcement (`npm ci` / `pip install --require-hashes`).

---

## 7. Execution, Logs, & Lifecycle

### Test & Build Execution
- **Exit Codes**: Standard POSIX exit codes captured (0 = success, non-zero = failure).
- **Log Buffering**: stdout and stderr streams captured with a 10 MB in-memory ring buffer ceiling. Excess log output is truncated with a visual truncation marker.
- **Artifact Extraction**: Output artifacts (e.g. `dist/`, coverage reports) are extracted directly from `/workspace` by the worker and uploaded to S3 with pre-signed ephemeral keys.

### Logging Architecture
- Execution logs are streamed real-time via Server-Sent Events (SSE) / WebSockets to the frontend console UI.
- Final stdout/stderr logs are archived to S3 and referenced in the audit log database table.

### Lifecycle & Capacity Management
1. **Provision**: Worker requests sandbox from pool manager. If capacity is exhausted, job queues in BullMQ.
2. **Execute**: Container runs command under supervisor timeout watchdog.
3. **Reap**:
   - Immediate destroy upon job completion or execution timeout.
   - **Idle Reaper Daemon**: Scans every 60s for stale sandboxes older than TTL and forcibly removes them (`docker rm -f`).

---

## 8. Observability & Data Model

The `sandbox_runs` PostgreSQL table records all sandbox lifecycle metrics and execution audit logs.

```sql
CREATE TYPE sandbox_status AS ENUM (
    'pending',
    'running',
    'completed',
    'failed',
    'timed_out',
    'reaped'
);

CREATE TABLE sandbox_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_run_id UUID NOT NULL REFERENCES agent_runs(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    container_id VARCHAR(128),
    base_image VARCHAR(255) NOT NULL,
    status sandbox_status NOT NULL DEFAULT 'pending',
    limits JSONB NOT NULL DEFAULT '{"cpuCores": 2, "memoryMb": 2048, "diskQuotaMb": 10240, "timeoutSeconds": 300}',
    command_executed TEXT,
    exit_code INT,
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    execution_duration_ms INT,
    logs_s3_key VARCHAR(512),
    artifacts_s3_keys TEXT[],
    estimated_cost_usd NUMERIC(10, 6) DEFAULT 0.000000,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sandbox_runs_agent_run ON sandbox_runs(agent_run_id);
CREATE INDEX idx_sandbox_runs_user_status ON sandbox_runs(user_id, status);
```

---

## 9. Example Flow: Coding Agent Execution

```text
[ Coding Agent ] ──► 1. Write Source Files to /workspace via writeFile()
                      │
                      ▼
                 2. Execute `npm install` (via Egress Caching Proxy)
                      │
                      ▼
                 3. Execute `npm test` (Capture Exit Code & Stdout/Stderr)
                      │
                      ▼
                 4. Execute `npm run lint` & Static Security Scan
                      │
                      ▼
                 5. Extract Dist Artifacts to S3 & Stream Results to UI
                      │
                      ▼
                 6. Destroy Ephemeral Sandbox & Log Metrics to `sandbox_runs`
```

---

## 10. Open Questions

1. **Pre-warmed Container Pools**: Should we maintain a pool of pre-warmed gVisor containers (e.g. 5 Node.js, 5 Python) to reduce container startup latency from ~800ms to <100ms?
2. **Browser / Headless Testing**: Will sandboxes require Playwright/Chromium capabilities, and what additional memory/GPU/binary dependencies does that introduce?
3. **Long-Running Dev Environments**: Should Aivexa Builder support persistent workspace state across agent steps via attached network volumes (e.g., EFS/Ceph), or remain strictly 100% ephemeral per execution run?
