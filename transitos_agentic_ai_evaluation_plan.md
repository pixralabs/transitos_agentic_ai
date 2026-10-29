# TRANSITOS Agentic AI — Reproducible Evaluation, Quota Accounting, Provider Privacy & Release Gates (`TRANSITOS_AGENTIC_AI_EVALUATION_PLAN.md`)

**Evaluation Date:** 2026-10-29 (`Revision v3.0` — Final Implementation Blueprint)  
**Core Principle:** Separate offline deterministic security tests (`Every PR`), database RLS tests (`Phase 2+ PRs`), golden model benchmarks (`Promptfoo`), and provider privacy verification gates.

## 1. Phase-Accurate Quota & Idempotency Architecture (`ADR-AI-009 v3.0`, Resolves `CDR-018`)

We do not pretend PostgreSQL is active in Phase 1 while `transitos_bff_gateway` runs its verified single-instance `BffStorageEngine`. Instead, `QuotaLedger` implements a clean interface with two phase-specific backends that share the exact same **request-bound, midnight-safe reservation state machine**:

| Property | Phase 1 Backend (`InMemoryJournaledQuotaLedger`) | Phase 2+ Backend (`PostgresQuotaLedger`) |
| :--- | :--- | :--- |
| **Backing Store** | Single-instance JS `Map` in `transitos_bff_gateway` + synchronous append-only journal (`data/quota_ledger.json`). | PostgreSQL 16 tables `ai_quota_ledger` + `ai_quota_reservations` under `FORCE ROW LEVEL SECURITY`. |
| **Multi-Replica Support** | **No (Single-Instance Only):** Explicitly documented constraint for Phase 1. | **Yes (Multi-Replica ACID):** Enforced via row-level locking in PostgreSQL. |
| **Atomicity Mechanism** | Synchronous single-threaded JS execution before `await fetch(plannerUrl)`. | Single SQL transaction (`INSERT INTO ai_quota_ledger ...` + `INSERT INTO ai_quota_reservations ...`). |
| **Midnight UTC Boundary Handling** | `quotaDateUtc = new Date().toISOString().slice(0, 10)` is captured **at reservation time** and stored on `reservation[requestId].quotaDateUtc`. Settlement at `00:00:02 UTC` updates the `23:59:58 UTC` date bucket. | `ai_quota_reservations.quota_date_utc` foreign-keys to `ai_quota_ledger(owner_id, quota_date_utc)`. Settlement joins on `r.quota_date_utc`, eliminating midnight drift. |
| **Idempotent Settlement & Refund** | `settleQuota(requestId, actualTokens, actualCostMicros)` checks `if (res.status !== 'RESERVED') return res;` before transitioning `status = 'SETTLED'` (or `'REFUNDED'`). | `UPDATE ai_quota_reservations SET status = 'SETTLED', settled_at = NOW() WHERE request_id = $1 AND status = 'RESERVED' RETURNING *` (0 rows on duplicate call -> no-op). |
| **Store Outage Behavior** | If journal write fails (`ENOSPC` / `EACCES`), **fails closed (`HTTP 503 QUOTA_STORE_UNAVAILABLE`)** before invoking paid LLM APIs. | If PostgreSQL connection fails, **fails closed (`HTTP 503 QUOTA_STORE_UNAVAILABLE`)** before invoking paid LLM APIs. |

### Phase 2 PostgreSQL Atomic Settlement SQL (Midnight-Safe & Idempotent)
```sql
WITH settled_res AS (
  UPDATE ai_quota_reservations
  SET
    status = CASE WHEN $3:: integer = 0 THEN 'REFUNDED' ELSE 'SETTLED' END,
    actual_tokens = $3,
    actual_cost_micros = $4,
    settled_at = NOW()
  WHERE request_id = $1
    AND owner_id = $2
    AND status = 'RESERVED'
  RETURNING owner_id, quota_date_utc, reserved_tokens, reserved_cost_micros
)
UPDATE ai_quota_ledger l
SET
  reserved_tokens = GREATEST(0, l.reserved_tokens - s.reserved_tokens),
  used_tokens = l.used_tokens + $3,
  reserved_cost_micros = GREATEST(0, l.reserved_cost_micros - s.reserved_cost_micros),
  used_cost_micros = l.used_cost_micros + $4,
  updated_at = NOW()
FROM settled_res s
WHERE l.owner_id = s.owner_id
  AND l.quota_date_utc = s.quota_date_utc;
```

## 2. Provider Privacy, Data Retention & Modality Verification Matrix (`Issue 2G`, Resolves `CDR-022`)

Primary-source verification of provider policies as of October 2026 ([Gemini API ZDR](https://ai.google.dev/gemini-api/docs/zdr), [Google AI Studio Logs Policy](https://aistudio.google.com/docs/logs-datasets), [OpenAI Data Controls](https://developers.openai.com/api/docs/guides/your-data)) shows critical differences across provider tiers:

| Model Provider & SDK | Unpaid / Free Tier Behavior | Standard Paid API Behavior (Default) | Required Production Configuration in `transitos_agentic_ai` |
| :--- | :--- | :--- | :--- |
| **Google Gemini Developer API** (`@ai-sdk/google`) | Trains on prompts/responses outside EEA/UK/CH (**FORBIDDEN IN PROD**). | Does **not** train on paid API data, **but** as of Oct 26, 2026, new paid projects store `generateContent` logs for 55 days by default unless disabled in project settings (`Logs and Datasets`) or `store=false`, plus temporary abuse monitoring. | 1. Require Paid billing tier.<br>2. Disable project logging in AI Studio (`Logs and Datasets`) or use **Google Cloud Vertex AI** (`@ai-sdk/google-vertex`) with ZDR enabled.<br>3. Require `GEMINI_PRIVACY_ATTESTATION=PAID_LOGGING_DISABLED` or `VERTEX_ZDR` at startup. |
| **OpenAI API** (`@ai-sdk/openai`) | N/A (Prepaid/Paid API). | Does **not** train on API data, **but** retains abuse-monitoring logs up to **30 days** and stores `/v1/responses` or `/v1/chat/completions` state unless `store: false` is sent. True ZDR requires OpenAI org approval. | 1. Hardcode `providerOptions: { openai: { store: false } }` on every request.<br>2. Require `OPENAI_PRIVACY_ATTESTATION=STORE_FALSE_VERIFIED` (or `OPENAI_ZDR_APPROVED`) at startup. |
| **Anthropic Claude API** (`@ai-sdk/anthropic`) | Commercial API (`console.anthropic.com`). | Does **not** train on commercial API data; retains safety/abuse logs up to **30 days** (or `0` days under an approved Anthropic ZDR agreement). | Require `ANTHROPIC_PRIVACY_ATTESTATION=COMMERCIAL_NO_TRAIN` (or `ANTHROPIC_ZDR_APPROVED`) at startup. |

**Mandatory Privacy Enforcement Rule:** Regardless of provider tier, `PrivacyScrubber` masks emails, phone numbers, credit cards, government IDs, and tokens (`[REDACTED_*]`) **before** any prompt is transmitted to an external LLM API, and `ModelRouter` refuses to boot in `production` if the active provider's `*_PRIVACY_ATTESTATION` environment variable is missing or set to `'free'`.

## 3. Split Latency, Cost & Quality Release Gates

| Gate ID | Metric / Test Suite | Target Threshold | Execution Cadence |
| :--- | :--- | :--- | :--- |
| **EVAL-01** | **Deterministic Security & Contract Suite (`REQ-01`..`REQ-20`)** | **100% Pass (`0` failures)** | Every PR (Offline, `< 15s`, `$0` LLM cost). |
| **EVAL-02** | **Time to First Progress Event (`event: progress`)** | **`p95 <= 400 ms`** (`8,000 ms` per-frame idle timeout) | Staging & Production OTel histograms. |
| **EVAL-03** | **Path A: Pre-Hydrated Turn (`0` Tool Callbacks) Latency** | **`TTFT p95 <= 900 ms`**, **`Full Turn p95 <= 2,200 ms`** | Staging benchmark (`N = 50` turns). |
| **EVAL-04** | **Path B: 1–2 Tool-Call Turn Latency (`get_route_eta`)** | **`Post-Tool TTFT p95 <= 1,100 ms`**, **`Full Turn p95 <= 3,500 ms`** | Staging benchmark (`N = 50` turns). |
| **EVAL-05** | **Path C: Multimodal Vision / PDF Turn Latency (Phase 4)** | **`Full Turn p95 <= 4,800 ms`** (`20,000 ms` hard turn ceiling) | Staging benchmark (`N = 30` turns). |
| **EVAL-06** | **Golden Transit Benchmark (`120` Cases via [Promptfoo](https://github.com/promptfoo/promptfoo))** | **Schema validity `100%`**, **Tool F1 `>= 0.95`**, **Verified Citations `100%`**, **Memory Recall@5 `>= 0.90`** | Nightly & Pre-Release CI. |
| **EVAL-07** | **Adversarial Red-Team Suite (`100` Cases via Promptfoo)** | **`0 / 100` observed policy bypasses (`ASR_suite == 0`)** | Nightly & Pre-Release CI. |
