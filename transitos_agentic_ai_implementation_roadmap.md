# TRANSITOS Agentic AI — Final Step-by-Step Implementation Roadmap (`TRANSITOS_AGENTIC_AI_IMPLEMENTATION_ROADMAP.md`)

**Evaluation Date:** 2026-10-29 (`Revision v3.0` — Final Implementation Blueprint)  
**Execution Policy:** Follow this exact dependency-ordered sequence. Do not advance from Phase $N$ to Phase $N+1$ until every deterministic test and release gate for Phase $N$ passes with reproducible evidence.

## Phase 1 — Safe End-to-End Core (`BFF <-> Stateless Planner <-> Web & Native Clients`)

- **Deployment Model:** Single-instance `transitos_bff_gateway` + single-instance stateless `transitos_agentic_ai` (`0` databases added; uses hardened `BffStorageEngine` + `InMemoryJournaledQuotaLedger`).
- **Prerequisites:**
  - Existing verified `transitos_bff_gateway` (`4/4` tests pass) and `ridel_web_shell` (`30/30` tests pass).
  - Provision two distinct 32-byte secrets: `BFF_TO_PLANNER_HMAC_SECRET` (shared between BFF and Planner) and `BFF_CAPABILITY_SECRET` (held **exclusively** by BFF).
  - Provision one paid-tier LLM API key with privacy attestation (`GEMINI_API_KEY` + `GEMINI_PRIVACY_ATTESTATION=PAID_LOGGING_DISABLED` or `OPENAI_API_KEY` + `OPENAI_PRIVACY_ATTESTATION=STORE_FALSE_VERIFIED`).
- **Step-by-Step Implementation Sequence:**
  1. **BFF Security & Policy Hardening (`transitos_bff_gateway`):**
     - Implement `src/security/turn_capability.js` (`BFF_CAPABILITY_SECRET` signer/verifier + `inFlightTurns` registry).
     - Implement `src/security/action_policy.js` (`CANONICAL_ACTION_POLICY_TABLE` mapping `0..25`, overriding any planner security fields).
     - Implement `src/services/quota_ledger.js` (`InMemoryJournaledQuotaLedger` binding `(requestId, userId, quotaDateUtc)` with midnight-safe idempotent settlement).
     - Upgrade `src/services/ai_planner_client.js` and `src/server.js` to sign `/internal/v1/plan` with `BFF_TO_PLANNER_HMAC_SECRET`, pre-hydrate `userSettings` + `savedPlacesSummary` (`<= 10`), expose `/internal/v1/tools/get_route_eta`, `/internal/v1/tools/get_weather_forecast`, and `/internal/v1/tools/list_user_saved_places`, pipe incremental SSE `event: progress` / `event: token` / `event: done`, and emit `0x00`/`0x01` binary WebSocket frames.
  2. **Stateless Planner Service (`transitos_agentic_ai`):**
     - Scaffold `package.json` pinning Node.js `>=24.0.0`, `ai@^6.0.306`, `@ai-sdk/google` / `@ai-sdk/openai`, and `zod@^3.25.76` (with zero `pg` or DB packages).
     - Implement `src/config/env_guard.ts` (fails closed if `DATABASE_URL` or `BFF_CAPABILITY_SECRET` is present in planner env, or if `*_PRIVACY_ATTESTATION` is missing in `production`).
     - Implement `src/security/service_auth.ts` (constant-time `X-Bff-Signature` verification + `120s` replay cache failing closed at `10,000` entries).
     - Implement `src/security/pii_redactor.ts` and `src/orchestrator/stream_orchestrator.ts` (`streamText` with `stopWhen: stepCountIs(3)`, `store: false`, and `AbortSignal` propagation).
- **Mandatory Phase 1 Release Gates:**
  - `REQ-01` (Zero DB creds in planner), `REQ-02` (Canonical HMAC + replay rejection `409`), `REQ-03` (Planner cannot forge `X-Turn-Capability-Token` -> `401/403`), `REQ-04` (`CANONICAL_ACTION_POLICY_TABLE` overrides spoofed fields), `REQ-14` (`event: token` streaming + `event: done` validation), `REQ-15` (`< 50 ms` disconnect abort + quota refund), `REQ-16` (Web SSE + Native `0x00`/`0x01` parity), `REQ-18` (Concurrent & midnight UTC quota tests), `REQ-19` (PII redaction + `store: false` verification), plus [OSV-Scanner](https://github.com/google/osv-scanner) / [Trivy](https://github.com/aquasecurity/trivy) in CI.
- **Explicit Out-of-Scope for Phase 1:** Multi-replica BFF/Planner deployments, PostgreSQL `pgvector`, Level 2/3 AI confirmations, web search/fetching, and PDF/image parsing.

---

## Phase 2 — Persistent, User-Controlled Memory & Multi-Replica PostgreSQL (`FORCE RLS`)

- **Prerequisites:** Phase 1 gates `PASS`; PostgreSQL 16+ instance with `pgvector` and `pg_trgm` extensions provisioned.
- **Step-by-Step Implementation Sequence (`transitos_bff_gateway` & `transitos_agentic_ai`):**
  1. Apply reviewed `migrations/002_agentic_memory.up.sql` under `transitos_migrations_admin`, enabling `ENABLE` + `FORCE ROW LEVEL SECURITY` across **all 10 tenant tables** (Migrations `001` and `002`), creating `transitos_app_runtime` (`NOSUPERUSER NOBYPASSRLS`), adding `ai_memory_consented_at`, and resetting unconsented legacy rows to `ai_memory = 0`.
  2. Implement `withTenantTransaction` (`SELECT set_config('app.current_user_id', $1, true)`), `PostgresQuotaLedger` (`ai_quota_ledger` + `ai_quota_reservations`), `internal_request_nonces` (enabling multi-replica replay protection), and `ConfirmationVault` (`POST /api/v1/actions/confirm` with RFC 8785 JCS `params_sha256` binding).
  3. Implement `PgMemoryStore` (`halfvec(1536)` + `simple` FTS + `pg_trgm` RRF filtered by `embedding_model`), `/api/v1/memories` CRUD/export/wipe endpoints, and `/internal/v1/tools/search_user_memory`, `propose_memory_write`, and `propose_saved_place_upsert`.
- **Mandatory Phase 2 Release Gates:**
  - `REQ-05` (Single-use `confirmationToken` for `UI_ACTION`, `MEMORY_WRITE`, and `SAVED_PLACE_UPSERT`), `REQ-06` (12-table RLS audit + 100-concurrency pool isolation test), `REQ-07` (Composite FK `(attachment_id, owner_id)` test), `REQ-08` (Legacy `ai_memory = 0` reset + opt-out transition), `REQ-09` (Multilingual RRF recall@5 `>= 0.90`), `REQ-20` (Cascading account deletion + tombstone verification).

---

## Phase 3 — Safe Web Research, BFF Taint Authority & Worker-Thread Extraction

- **Prerequisites:** Phase 2 gates `PASS`; Brave Search or Tavily API key configured; Linux `nftables` / container egress policy applied to planner.
- **Step-by-Step Implementation Sequence:**
  1. Implement `POST /internal/v1/tools/acquire_research_lease` in `transitos_bff_gateway` so the BFF authoritatively latches `inFlightTurns.get(requestId).tainted = true` before any external search or fetch executes (`ADR-AI-011 v3.0`).
  2. Implement `SafeWebFetcher` (`undici.Agent` connect-time DNS IP pinning + `<= 2 MiB` compressed / `<= 5 MiB` decompressed stream caps + manual redirect validation).
  3. Implement `src/workers/extractor_worker.ts` using Node.js `worker_threads` (`maxWorkers = 2`, `128 MiB` heap limit, `3,000 ms` `worker.terminate()` preemption) dispatching HTML to `linkedom` + [@mozilla/readability](https://github.com/mozilla/readability) and PDFs (`%PDF-`) to [`unpdf`](https://github.com/unjs/unpdf) (`maxPages = 10`, `maxImageSize = 16 MP`).
  4. Implement `CitationValidator` stripping any URL not returned by a tool in that turn.
- **Mandatory Phase 3 Release Gates:**
  - `REQ-10` (BFF enforces taint downgrade even if planner reports `tainted: false`), `REQ-11` (SSRF, DNS-rebinding, and 10x gzip-bomb tests), `REQ-12` (`worker_threads` termination at `3,000 ms` with `< 5 ms` main event-loop lag), `REQ-13` (`0` unretrieved citation URLs).

---

## Phase 4 — Multimodal Processing, Image Normalization & Pre-Token Provider Fallback

- **Prerequisites:** Phase 3 gates `PASS`; secondary LLM provider configured and privacy-attested.
- **Step-by-Step Implementation Sequence:**
  1. Integrate [Sharp (`lovell/sharp`)](https://github.com/lovell/sharp) (`limitInputPixels: 16_777_216`, EXIF strip, `<= 1568px` resize, WebP/PNG re-encode) and `read_verified_attachment_excerpt` (`unpdf` in `worker_threads`, `maxPages = 20`).
  2. Enable `ModelRouter` pre-first-token failover (`tokensEmitted === 0`).
  3. Optional Pilot: Connect an isolated [IBM Docling](https://github.com/docling-project/docling) container via `@modelcontextprotocol/sdk` only if scanned image-only PDFs require OCR beyond Vision LLMs.
- **Mandatory Phase 4 Release Gates:**
  - `REQ-14` (Pre-token failover passes; mid-stream failure never splices providers), `REQ-17` (16 MP pixel-bomb, EXIF-strip, and 20-page PDF limits verified).

---

## Phase 5 — Durable Background Workflows (`DBOS` in BFF) & Advanced Operations

- **Prerequisites:** Phases 1–4 verified in staging.
- **Step-by-Step Implementation Sequence:**
  1. Integrate [DBOS Transact TS (`@dbos-inc/dbos-sdk`)](https://github.com/dbos-inc/dbos-transact-ts) inside `transitos_bff_gateway` (or `transitos_core_workers`), checkpointing asynchronous background jobs in PostgreSQL while invoking `transitos_agentic_ai` as a stateless step.
  2. Enable [Promptfoo](https://github.com/promptfoo/promptfoo) (`120` golden + `100` red-team cases) as a blocking release gate and optional isolated [Crawlee](https://github.com/apify/crawlee) + [Playwright](https://github.com/microsoft/playwright-mcp) worker for scheduled transit bulletins.