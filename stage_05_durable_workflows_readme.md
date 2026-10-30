# Stage 05 — Durable Workflows, HITL Governance & Production Readiness (`05-durable-workflows`)

> **Stage ID:** `05-durable-workflows`  
> **Revision:** `v3.1 (Final Complete Documentation & Cross-Verification Audit)`  
> **Implementation Status:** `PLANNED` (Target Build Window: Weeks 9–10)  
> **Estimated Effort:** 2 Weeks (Solo Developer — `TASK-5.1` through `TASK-5.4`)  
> **Governing ADRs:** \[suspicious link removed\], \[suspicious link removed\], \[suspicious link removed\], \[suspicious link removed\]

## 1. Stage Overview

Stage 05 completes the `transitos_agentic_ai` platform by adding durable background workflow orchestration (`pg-boss` `^12.14.0` backed by the existing PostgreSQL 16 instance), cryptographic Human-in-the-Loop (HITL) proposal approval (`POST /api/v1/proposals/:proposalId/confirm`), automated Promptfoo red-team security gates (`>= 100` adversarial test cases across 6 attack categories), and the strict Node 24 LTS release CI pipeline (`STRICT_NODE_24_GATE=1`) (`PLANNED`). By reusing PostgreSQL via `pg-boss` (`ADR-AI-005`), a solo developer gets transactional job queueing (`singletonKey` idempotency, exponential backoff, dead-letter queues, and cron schedules) without operating a separate Temporal cluster or Kafka broker.

## 2. Why This Stage Exists

1. **In-Process Timers Do Not Survive Container Restarts:** Commuters rely on recurring disruption checks (e.g., *"Check Metro Yellow Line delays 30 minutes before my 08:30 AM commute"*). In-memory `setTimeout` jobs vanish on container deploy or crash; `pg-boss` persists job state inside PostgreSQL (`pgboss.job`).
2. **State-Mutating Agent Proposals Need Cryptographic User Approval (`T-MEM-01` / `T-MM-01`):** Stages 02–04 allow the agent to *propose* schedule reminders or memory updates (`pendingProposals`), but the user must explicitly confirm or reject those proposals via an authenticated, CSRF-protected, idempotent endpoint (`POST /api/v1/proposals/:proposalId/confirm`).
3. **Background Workers Must Never Run with Elevated Agent Privileges (`T-WF-01` / `ADR-AI-012`):** When a scheduled background job runs while the user is offline, it cannot use an interactive user JWT. Instead, it must run under a restricted `system_worker` capability (`allowTools: ['get_transit_routes', 'get_weather_context']`) that is structurally prohibited from calling web search, URL browse, or destructive tools (`ADR-AI-012`).
4. **Release Gate Verification (`CDR-024` / `ADR-AI-013`):** Before production sign-off, all unit, integration, RLS, worker isolation, and Promptfoo red-team suites must pass deterministically under the pinned Node 24 LTS release gate (`STRICT_NODE_24_GATE=1`).

## 3. Scope

### In Scope (`PLANNED`)
- Durable workflow queue and worker (`src/workflows/pg_boss_engine.ts`) using `pg-boss` (`^12.14.0`) with `singletonKey` deduplication, max 3 retries with exponential backoff, dead-letter queue (`transit-monitor-dlq`), and per-user active workflow cap (`<= 5` active recurring monitors per `user_id`) (`TASK-5.1`).
- Restricted `system_worker` capability tokenminting and verification (`src/security/capability_token.ts` + `src/workflows/monitor_worker.ts`) enforcing `allowTools: ['get_transit_routes', 'get_weather_context']` and a daily background budget cap (`<= $0.05/day` per user) (`TASK-5.2`).
- HITL Proposal Store and Confirmation API (`src/workflows/proposal_service.ts`, `POST /api/v1/proposals/:proposalId/confirm`, and BFF proxy route) with 15-minute proposal TTL and single-use atomic state transition (`pending -> confirmed | rejected | expired`) (`TASK-5.3`).
- Automated Promptfoo red-team evaluation harness (`evals/promptfoo_redteam.yaml`, `>= 100` adversarial prompts across 6 categories) and release CI script (`scripts/ci_release_gate.sh` enforcing `STRICT_NODE_24_GATE=1` in release CI) (`TASK-5.4`).

### Out of Scope (`DEFERRED` / Non-Goals)
- Multi-node Temporal Server + Cassandra/Elasticsearch clusters (`ADR-AI-005` — rejected due to operational overhead for a solo developer).
- Unbounded autonomous background web browsing or recursive agent loops while the user is offline (`ADR-AI-012`).

## 4. Prerequisites & Dependencies from Previous Stages

| Prerequisite | Source Stage | Verification Check Before Starting Stage 05 |
| :--- | :--- | :--- |
| Fastify Core, HMAC Capability Token & Tool Registry | Stage 01 (`01-safe-agent-core`) | `npm run test:stage1` passes (`100%`) |
| PostgreSQL 16 + RLS Policies + Outbox Sweeper | Stage 02 (`02-persistent-memory`) | `npm run test:stage2` passes (`100%`) |
| Spotlighting, SSRF Guards & Grounded Citations | Stage 03 (`03-web-research`) | `npm run test:stage3` passes (`100%`) |
| Worker Thread Multimodal Extraction & Proposal Tool | Stage 04 (`04-multimodal`) | `npm run test:stage4` passes (`100%`) |

## 5. Architecture & Component Design

```
[User Creates Schedule Proposal via Chat or Attachment]
       │
       ▼
[pending_proposals Table (PostgreSQL RLS)] (PLANNED - TASK-5.3)
  ├─ Status: 'pending' (expires_at = NOW() + 15 minutes)
  ├─ Surfaced to Client via SSE `donePayload.pendingProposals` (CDR-027)
       │
       ▼
[User Clicks "Confirm" in UI -> POST /api/v1/proposals/:id/confirm]
  ├─ Verifies session JWT + X-CSRF-Token + user_id ownership
  ├─ Atomically transitions status: 'pending' -> 'confirmed'
  └─ Enqueues durable job in `pg-boss` with `singletonKey: "${userId}:${proposalId}"`
       │
       ▼
[pg-boss Durable Worker: src/workflows/monitor_worker.ts] (PLANNED - TASK-5.1 & TASK-5.2)
  ├─ Mints restricted `system_worker` capability token:
  │    allowTools: ['get_transit_routes', 'get_weather_context']
  │    denyTools:  ['search_web_grounded', 'fetch_public_page', 'delete_memory', 'save_memory_preference']
  ├─ Checks per-user background daily spend (<= $0.05/day)
  ├─ Executes deterministic route + weather check inside RLS transaction (`app.current_user_id`)
  └─ Persists notification in `workflow_notifications` (or routes dead-letter on 3 failures to `transit-monitor-dlq`)
```

## 6. Exact Directory & File Structure Created/Modified in This Stage

```text
transitos_agentic_ai/
├── src/
│   ├── workflows/
│   │   ├── pg_boss_engine.ts        # PLANNED (TASK-5.1: pg-boss lifecycle, queues, DLQ, singletonKey)
│   │   ├── monitor_worker.ts        # PLANNED (TASK-5.2: restricted system_worker execution & budget check)
│   │   └── proposal_service.ts      # PLANNED (TASK-5.3: pending_proposals CRUD, 15m TTL, atomic confirm/reject)
│   └── routes/
│       └── proposals.ts             # PLANNED (TASK-5.3: POST /api/v1/proposals/:id/confirm & /reject)
├── migrations/
│   └── 0002_stage5_workflows_and_proposals.sql # PLANNED (TASK-5.1 & TASK-5.3: pending_proposals & workflow_runs tables)
├── evals/
│   ├── promptfoo_redteam.yaml       # PLANNED (TASK-5.4: 100+ adversarial cases across 6 categories)
│   └── fixtures/
│       └── adversarial_prompts.json # PLANNED (TASK-5.4: deterministic offline & live red-team dataset)
├── scripts/
│   └── ci_release_gate.sh           # PLANNED (TASK-5.4: full 5-stage verification + STRICT_NODE_24_GATE check)
└── tests/
    └── stage5/
        ├── pg_boss_durability.test.ts    # PLANNED (TASK-5.1: singletonKey idempotency, retry & DLQ)
        ├── system_worker_scope.test.ts   # PLANNED (TASK-5.2: blocks web/memory tools in background worker)
        ├── hitl_proposals.test.ts        # PLANNED (TASK-5.3: 15m expiry, replay prevention, cross-user 403)
        └── redteam_regression.test.ts    # PLANNED (TASK-5.4: 100-case adversarial prompt injection suite)
```

Cross-repository BFF files updated in `TASK-5.3` (`PLANNED`):
- `transitos_bff_gateway/src/routes/chat.js` (or `proposals.js`) — Proxies authenticated `POST /api/v1/proposals/:proposalId/confirm` with `X-CSRF-Token` validation and signed `X-Transit-Capability`.

## 7. Dependencies, Packages & External Services

| Package / Dependency | Pinned Version | License | Purpose |
| :--- | :--- | :--- | :--- |
| `pg-boss` | `^12.14.0` | MIT | PostgreSQL-backed durable job queue (`SKIP LOCKED`, cron scheduling, retries, dead-letter queues) |
| `promptfoo` | `^0.109.0` (dev) | MIT | Automated red-team and grounding evaluation runner (`evals/promptfoo_redteam.yaml`) |

## 8. Data Models, Schemas & Database Changes

### Migration `migrations/0002_stage5_workflows_and_proposals.sql` (`PLANNED`)

```sql
BEGIN;

CREATE TABLE IF NOT EXISTS pending_proposals (
  proposal_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id TEXT NOT NULL REFERENCES user_profiles(user_id) ON DELETE CASCADE,
  tool_name TEXT NOT NULL,
  proposed_action TEXT NOT NULL CHECK (proposed_action IN ('create_reminder', 'update_commute_alert', 'save_preference')),
  payload JSONB NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'confirmed', 'rejected', 'expired')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL DEFAULT (NOW() + INTERVAL '15 minutes'),
  confirmed_at TIMESTAMPTZ
);

CREATE INDEX IF NOT EXISTS idx_pending_proposals_user_status
  ON pending_proposals (user_id, status, expires_at);

CREATE TABLE IF NOT EXISTS user_workflows (
  workflow_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id TEXT NOT NULL REFERENCES user_profiles(user_id) ON DELETE CASCADE,
  proposal_id UUID NOT NULL UNIQUE REFERENCES pending_proposals(proposal_id) ON DELETE CASCADE,
  route_id TEXT NOT NULL,
  origin_stop TEXT NOT NULL,
  cron_expression TEXT,
  next_run_at TIMESTAMPTZ NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  daily_cost_usd NUMERIC(8, 5) NOT NULL DEFAULT 0.00000,
  cost_reset_date DATE NOT NULL DEFAULT CURRENT_DATE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE pending_proposals ENABLE ROW LEVEL SECURITY;
ALTER TABLE pending_proposals FORCE ROW LEVEL SECURITY;
CREATE POLICY pending_proposals_tenant_isolation ON pending_proposals
  USING (user_id = current_setting('app.current_user_id', true))
  WITH CHECK (user_id = current_setting('app.current_user_id', true));

ALTER TABLE user_workflows ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_workflows FORCE ROW LEVEL SECURITY;
CREATE POLICY user_workflows_tenant_isolation ON user_workflows
  USING (user_id = current_setting('app.current_user_id', true))
  WITH CHECK (user_id = current_setting('app.current_user_id', true));

GRANT SELECT, INSERT, UPDATE, DELETE ON pending_proposals, user_workflows TO transitos_app;

COMMIT;
```

## 9. Tool, API & Inter-Service Contracts

### 9.1 HITL Proposal Confirmation API (`POST /api/v1/proposals/:proposalId/confirm` — `PLANNED`)
- **Request Headers:**
  - `Authorization: Bearer <access_token>` (at BFF)
  - `X-CSRF-Token: <csrf_token>` (at BFF)
  - `X-Internal-Service-Token` + `X-Transit-Capability` (BFF -> `transitos_agentic_ai`)
- **Request Body:**
  ```json
  {
    "decision": "confirm"
  }
  ```
- **Response (`200 OK`):**
  ```json
  {
    "status": "confirmed",
    "proposalId": "c390512c-84f7-41b3-9d1a-09e442f019a2",
    "workflowId": "91f8b2d4-3120-4f9e-8b1a-77c52a109e11",
    "nextRunAt": "2026-04-02T00:45:00.000Z"
  }
  ```
- **Error Responses:**
  - `404 Not Found` if `proposalId` does not exist or belongs to another `user_id` (enforced by RLS).
  - `409 Conflict` (`PROPOSAL_ALREADY_PROCESSED`) if `status !== 'pending'`.
  - `410 Gone` (`PROPOSAL_EXPIRED`) if `NOW() > expires_at`.

### 9.2 Restricted `system_worker` Capability Contract (`ADR-AI-012` — `PLANNED`)

```typescript
export interface SystemWorkerCapabilityClaims {
  sub: string; // user_id on whose behalf the monitor runs
  role: 'system_worker';
  workflowId: string;
  allowTools: readonly ['get_transit_routes', 'get_weather_context'];
  iat: number;
  exp: number; // Strictly <= iat + 60 seconds
}
```

## 10. Security Model, Threat Controls & Trust Boundaries

| Threat ID | Threat Description | Stage 05 Mitigation (`PLANNED`) | Verification Test |
| :--- | :--- | :--- | :--- |
| **`T-WF-01`** | **Unattended Background Privilege Escalation:** A scheduled job triggers web browsing, external data exfiltration, or memory deletion while the user is offline. | `monitor_worker.ts` mints a short-lived (`60s`) `system_worker` capability whose `allowTools` is hardcoded to `['get_transit_routes', 'get_weather_context']` (`ADR-AI-012`). | `tests/stage5/system_worker_scope.test.ts` |
| **`T-WF-02`** | **Job Queue Flood / Denial of Wallet:** A user or prompt injection creates thousands of recurring cron jobs in `pg-boss`. | 1. Every workflow requires an explicit user click on `POST /api/v1/proposals/:id/confirm`.<br>2. Hard limit of `5` active `user_workflows` per `user_id`.<br>3. Background spend capped at `$0.05/day` per user. | `tests/stage5/pg_boss_durability.test.ts` |
| **`T-WF-03`** | **Proposal Replay / Cross-Tenant Approval:** Attacker replays a confirmation request or guesses another user's `proposalId`. | `UPDATE pending_proposals SET status = 'confirmed' WHERE proposal_id = $1 AND status = 'pending' AND expires_at > NOW()` inside an RLS-scoped transaction (`app.current_user_id`). | `tests/stage5/hitl_proposals.test.ts` |

## 11. Step-by-Step Implementation Guide

### Step 1 (`TASK-5.1`): Configure `pg-boss` Durable Queue (`src/workflows/pg_boss_engine.ts`)
Initialize `PgBoss` over `DATABASE_URL` (`pgboss` schema), create the primary queue `'transit-commute-monitor'` with dead-letter queue `'transit-commute-monitor-dlq'`, `retryLimit: 3`, `retryDelay: 30`, `retryBackoff: true`, and `expireInSeconds: 60`. Enforce `singletonKey: `${userId}:${proposalId}`` when scheduling jobs so duplicate confirmations never enqueue duplicate runs.

### Step 2 (`TASK-5.2`): Implement Restricted `monitor_worker.ts` (`ADR-AI-012`)
In the `pg-boss` job handler:
1. Verify that the user's `user_workflows.is_active === true` and `daily_cost_usd < 0.05`.
2. Mint a `60s` `system_worker` capability token restricted to `['get_transit_routes', 'get_weather_context']`.
3. Query deterministic route + weather status and record any delay alert inside the user's RLS-scoped transaction.

### Step 3 (`TASK-5.3`): Implement HITL Proposal Lifecycle (`src/workflows/proposal_service.ts` & `src/routes/proposals.ts`)
1. Persist proposals generated during chat or multimodal extraction into `pending_proposals` (`expires_at = NOW() + 15 minutes`).
2. Expose `POST /api/v1/proposals/:proposalId/confirm` to atomically transition `status` from `'pending'` to `'confirmed'`, verify the user has `< 5` active workflows, insert into `user_workflows`, and enqueue the `pg-boss` job.

### Step 4 (`TASK-5.4`): Wire Red-Team Evaluation Suite & Release CI Gate (`scripts/ci_release_gate.sh`)
Create `tests/stage5/redteam_regression.test.ts` (100 adversarial cases covering direct override, indirect web/OCR injection, SSRF, cross-tenant RLS/IDOR, tool schema fuzzing, and cost exhaustion) and `scripts/ci_release_gate.sh`. When `STRICT_NODE_24_GATE=1` is set in CI, `ci_release_gate.sh` verifies `node -v` matches `v24.*` before executing all Stage 01–05 test suites.

## 12. Configuration & Environment Variables

| Variable Name | Default / Example | Required | Description |
| :--- | :--- | :--- | :--- |
| `PG_BOSS_SCHEMA` | `pgboss` | Yes | Dedicated PostgreSQL schema managed by `pg-boss` |
| `WORKFLOW_MAX_ACTIVE_PER_USER` | `5` | Yes | Maximum concurrent recurring workflows per `user_id` |
| `WORKFLOW_MAX_DAILY_COST_USD` | `0.05` | Yes | Maximum background LLM/API spend per `user_id` per UTC day |
| `PROPOSAL_TTL_SECONDS` | `900` | Yes | Expiration window (15 minutes) for unconfirmed `pending_proposals` |
| `STRICT_NODE_24_GATE` | `0` (local) / `1` (CI) | Yes | Enforces Node 24 LTS runtime check during release CI (`ADR-AI-013`) |

## 13. Build, Run & Local Development Commands

```bash
# 1. Install Stage 05 durable workflow dependencies
npm install pg-boss@^12.14.0
npm install -D promptfoo@^0.109.0

# 2. Apply Stage 05 migration (pending_proposals & user_workflows + RLS)
npm run db:migrate

# 3. Run Stage 05 durability, HITL proposal, and red-team test suites
npm run test:stage5

# 4. Run full cumulative Stage 01-05 verification
npm test
```

## 14. Verification, Testing & Acceptance Criteria

### Stage 05 Test Commands
```bash
npx vitest run tests/stage5/pg_boss_durability.test.ts
npx vitest run tests/stage5/system_worker_scope.test.ts
npx vitest run tests/stage5/hitl_proposals.test.ts
npx vitest run tests/stage5/redteam_regression.test.ts
```

### Stage 05 Acceptance Criteria (`AC-S5-01` through `AC-S5-06`)
- **`AC-S5-01` (`pg-boss` Crash Recovery & `singletonKey` Idempotency):** Submitting two identical job schedule calls with the same `singletonKey` results in exactly **1** queued job in `pgboss.job`, and jobs in `'active'` state recover automatically after simulated worker restart.
- **`AC-S5-02` (Dead-Letter Queue Routing):** A job that throws 3 consecutive transient errors is moved deterministically to `'transit-commute-monitor-dlq'` with full error metadata.
- **`AC-S5-03` (`system_worker` Least Privilege):** Attempting to invoke `search_web_grounded`, `fetch_public_page`, `delete_memory`, or `save_memory_preference` under a `role: 'system_worker'` token throws `CAPABILITY_TOOL_FORBIDDEN` (`100%` blocked).
- **`AC-S5-04` (HITL Proposal Atomic Confirmation & 15m TTL):** Confirming a valid `'pending'` proposal transitions it to `'confirmed'` (`200 OK`); a second confirmation call returns `409 Conflict` (`PROPOSAL_ALREADY_PROCESSED`); confirming a proposal past `expires_at` returns `410 Gone` (`PROPOSAL_EXPIRED`).
- **`AC-S5-05` (Per-User Workflow Cap & GDPR Cascade):** Attempting to confirm a 6th active workflow for the same `user_id` returns `429 Too Many Requests` (`WORKFLOW_LIMIT_REACHED`). Deleting the user profile cascades deletion across `pending_proposals`, `user_workflows`, and cancels pending `pg-boss` jobs for that user.
- **`AC-S5-06` (100-Case Adversarial Red-Team Gate):** `tests/stage5/redteam_regression.test.ts` executes all `>= 100` adversarial fixtures across 6 attack categories with **0** unauthorized tool executions, **0** cross-user leaks, and **0** SSRF bypasses.

## 15. Failure Modes, Rollback & Recovery Procedures

| Failure Scenario | Automatic System Behavior (`PLANNED`) | Operator Rollback / Recovery Action |
| :--- | :--- | :--- |
| Downstream route/weather API fails during background monitor run | `pg-boss` retries up to 3 times with exponential backoff (`30s`, `60s`, `120s`), then moves the job to `transit-commute-monitor-dlq`. | Inspect `pgboss.job` where `name = 'transit-commute-monitor-dlq'` and replay once downstream service recovers. |
| Runaway background job spend | `monitor_worker.ts` checks `daily_cost_usd >= 0.05` before any LLM call and skips LLM summarization (falling back to deterministic template notification). | Set `ENABLE_BACKGROUND_WORKFLOWS=false` to pause `pg-boss` polling without affecting interactive `/api/v1/chat`. |
| Stage 05 schema rollback needed | Disable worker polling (`boss.stop({ graceful: true })`). | Run `npm run db:migrate:down -- 0002_stage5_workflows_and_proposals.sql`. |

## 16. Performance, Cost & Latency Budgets

| Metric | Target Budget | Hard Cap / Enforcement Mechanism |
| :--- | :--- | :--- |
| `POST /api/v1/proposals/:id/confirm` latency | `p95 <= 80ms` | Single PostgreSQL transaction (`UPDATE pending_proposals` + `boss.send`) |
| Background monitor execution duration | `p95 <= 2,500ms` | `expireInSeconds: 60` per `pg-boss` job |
| Background LLM/API cost per user | `<= $0.01/day` typical | Hard ceiling `$0.05/day` per `user_id` (`WORKFLOW_MAX_DAILY_COST_USD`) |
| Active recurring monitors per user | `1–2` typical | Hard ceiling `5` active workflows per `user_id` (`WORKFLOW_MAX_ACTIVE_PER_USER`) |

## 17. Observability, Logging & Metrics

- **`transitos_ai_workflows_active_gauge`:** Gauge of currently active `user_workflows`.
- **`transitos_ai_pgboss_jobs_total{queue, state}`:** Counter of `completed`, `retry`, and `dlq` job transitions.
- **`transitos_ai_proposals_total{action, decision}`:** Counter tracking `created`, `confirmed`, `rejected`, and `expired` proposals.
- **`transitos_ai_redteam_pass_rate`:** CI metric asserting `1.0` (`100%`) pass rate across the 100-case adversarial suite.

## 18. Known Limitations, Risks & Technical Debt

1. **PostgreSQL Polling Interval:** `pg-boss` polls PostgreSQL every `2,000ms` (`newJobCheckIntervalSeconds: 2`). Sub-second background job dispatch is not required for transit schedule reminders (which operate on minute-level commute windows).
2. **Push Notification Delivery Channel:** Stage 05 persists triggered alerts in PostgreSQL and surfaces them via BFF polling/SSE; native APNs/FCM mobile push delivery is handled separately by `ridel_shell` and `transitos_messaging_engine`.

## 19. How This Stage Completes the System Lifecycle

Stage 05 closes the loop across all five stages of `transitos_agentic_ai`:
- **Stage 01** established the authenticated, schema-validated tool-calling core (`stopWhen: stepCountIs(3)`).
- **Stage 02** added RLS-isolated PostgreSQL + `pgvector` hybrid memory and transactional GDPR deletion (`deleted_blob_cleanup_queue`).
- **Stage 03** added SSRF-hardened, spotlighted web search and crawl grounding with verifiable citations (`CDR-027`).
- **Stage 04** added `worker_threads`-isolated PDF/image extraction (`unpdf` + `sharp` + Gemini Vision) with `confirm_attachment_schedule_action`.
- **Stage 05** turns user-confirmed proposals from Stages 02–04 into durable, least-privilege `pg-boss` workflows verified by a 100-case adversarial red-team suite.

## 20. Reference Links to Deep-Dive Architecture Docs

- \[suspicious link removed\] — Section 7 (Durable Workflows & HITL Lifecycle)
- \[suspicious link removed\] — Section 6 (`T-WF-01` through `T-WF-03`)
- \[suspicious link removed\] — Section 4 (100-Case Adversarial Red-Team Matrix)
- \[suspicious link removed\] — `ADR-AI-005` (`pg-boss` v12), `ADR-AI-007` (HITL Proposal Gate), `ADR-AI-012` (`system_worker` Least Privilege), `ADR-AI-013` (Node 24 Release Gate)
- \[suspicious link removed\] — `TASK-5.1` through `TASK-5.4`

## 21. Exact Parameter & Payload Examples

### 21.1 Example `pg-boss` Job Payload (`transit-commute-monitor`)
```json
{
  "workflowId": "91f8b2d4-3120-4f9e-8b1a-77c52a109e11",
  "proposalId": "c390512c-84f7-41b3-9d1a-09e442f019a2",
  "userId": "usr_99281a",
  "routeId": "Airport Express",
  "originStop": "New Delhi Metro Station",
  "scheduledDepartureIso": "2026-04-02T00:45:00.000Z"
}
```

### 21.2 Example `POST /api/v1/proposals/:proposalId/confirm` cURL Verification
```bash
curl -i -X POST http://127.0.0.1:4000/api/v1/proposals/c390512c-84f7-41b3-9d1a-09e442f019a2/confirm \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-CSRF-Token: ${CSRF_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"decision":"confirm"}'
```

## 22. Exact Security Enforcement Points

| Enforcement Point | Source File (`PLANNED`) | Control Applied |
| :--- | :--- | :--- |
| Proposal Ownership & Single-Use Atomic Update | `src/workflows/proposal_service.ts` | `UPDATE pending_proposals SET status = 'confirmed' WHERE proposal_id = $1 AND status = 'pending' AND expires_at > NOW()` inside RLS transaction |
| Background Worker Capability Scope | `src/workflows/monitor_worker.ts` | Mints `role: 'system_worker'` token with `allowTools: ['get_transit_routes', 'get_weather_context']` (`60s` expiry) |
| Per-User Workflow & Daily Spend Cap | `src/workflows/proposal_service.ts` & `monitor_worker.ts` | Hard limit of `<= 5` active workflows and `<= $0.05/day` background cost per `user_id` |
| Release Runtime Gate | `scripts/ci_release_gate.sh` | Enforces Node 24 LTS when `STRICT_NODE_24_GATE=1` |

## 23. Exact Test Commands & Expected Outputs

| Test Suite Command | Expected Output Summary |
| :--- | :--- |
| `npx vitest run tests/stage5/pg_boss_durability.test.ts` | `PASS tests/stage5/pg_boss_durability.test.ts (5 tests) - singletonKey dedup, retry backoff & DLQ verified` |
| `npx vitest run tests/stage5/system_worker_scope.test.ts` | `PASS tests/stage5/system_worker_scope.test.ts (4 tests) - web & destructive tools 100% blocked for system_worker` |
| `npx vitest run tests/stage5/hitl_proposals.test.ts` | `PASS tests/stage5/hitl_proposals.test.ts (6 tests) - atomic confirm, 409 replay block, 410 expiry, 403 cross-user` |
| `npx vitest run tests/stage5/redteam_regression.test.ts` | `PASS tests/stage5/redteam_regression.test.ts (100 tests) - 100/100 adversarial cases contained` |

## 24. Stage Completion Checklist (Definition of Done)

- [ ] `TASK-5.1` completed: `pg-boss` v12 initialized with `'transit-commute-monitor'`, `'transit-commute-monitor-dlq'`, and `singletonKey` idempotency.
- [ ] `TASK-5.2` completed: `monitor_worker.ts` enforces `role: 'system_worker'` least-privilege tool allowlist and `$0.05/day` per-user budget cap.
- [ ] `TASK-5.3` completed: `pending_proposals` and `user_workflows` tables migrated with `FORCE ROW LEVEL SECURITY`, and `POST /api/v1/proposals/:id/confirm` enforces 15m TTL and single-use confirmation.
- [ ] `TASK-5.4` completed: 100-case adversarial red-team suite (`tests/stage5/redteam_regression.test.ts`) and `scripts/ci_release_gate.sh` (`STRICT_NODE_24_GATE`) pass (`100%`).
- [ ] All cumulative test suites (`test:stage1` through `test:stage5`) pass with zero regressions.
