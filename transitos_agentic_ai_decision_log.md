# TRANSITOS Agentic AI — Final Architecture Decision Log (`TRANSITOS_AGENTIC_AI_DECISION_LOG.md`)

**Evaluation Date:** 2026-10-30 (`Revision v3.1` — Final Documentation & Architecture Cross-Verification)  
**Scope:** Authoritative record of all 13 Architecture Decision Records (`ADR-AI-001` through `ADR-AI-013`), governing `transitos_agentic_ai` and its integration with `transitos_bff_gateway`, `ridel_ui`, `ridel_web_shell`, and `ridel_shell`.

## 1. Decision Summary Matrix (`v3.1`)

| ADR ID | Decision Title | Resolves Contradictions | Authoritative Blueprint Section |
| :--- | :--- | :--- | :--- |
| **`ADR-AI-001`** | **BFF-Owned Persistence & Zero Database Credentials in `transitos_agentic_ai`** | `CDR-001`, `CDR-013` | `MASTERPLAN.md` §1–3, `STAGE_01` |
| **`ADR-AI-002`** | **Split-Key Service Auth (`BFF_TO_PLANNER_HMAC_SECRET` vs. BFF-Only `BFF_CAPABILITY_SECRET`), Fail-Closed Replay Cache & Turn-Aligned Deadlines** | `CDR-003`, `CDR-015`, `CDR-016`, `CDR-017` | `TOOL_CONTRACTS.md` §1, `STAGE_01` |
| **`ADR-AI-003`** | **Server-Authoritative Action Policy, `PendingMutationProposal` Types & JCS-Bound Confirmation Vault (`POST /api/v1/actions/confirm`)** | `CDR-002`, `CDR-011`, `CDR-019` | `TOOL_CONTRACTS.md` §2, `STAGE_01`, `STAGE_02` |
| **`ADR-AI-004`** | **Complete 12-Table PostgreSQL `FORCE ROW LEVEL SECURITY` Matrix, `NOBYPASSRLS` Runtime Role & Composite Foreign Keys** | `CDR-004`, `CDR-020` | `MEMORY_DESIGN.md` §1, §3, `STAGE_02` |
| **`ADR-AI-005`** | **Pinned Vercel AI SDK (`ai@^6.0.306`, `stopWhen: stepCountIs(3)`), Two-Channel Streaming, Pre-First-Token Fallback & Heartbeat Deadlines** | `CDR-005`, `CDR-010`, `CDR-023`, `CDR-025` | `MASTERPLAN.md` §2 S4, `TOOL_CONTRACTS.md` §3 |
| **`ADR-AI-006`** | **Pre-Hydrated Turn Context (`userSettings`, `savedPlacesSummary`, `relevantMemories`) & Deferred MCP** | `CDR-006` | `TOOL_CONTRACTS.md` §1.2, §3, `STAGE_01` |
| **`ADR-AI-007`** | **Preemptible `worker_threads` Isolation for `unpdf` (`pdf.js` v6.4.299) & `@mozilla/readability` + Dual Stream Byte Caps** | `CDR-007`, `CDR-021`, `CDR-022` | `BROWSING_DESIGN.md` §2–3, `STAGE_03`, `STAGE_04` |
| **`ADR-AI-008`** | **Explicit Consent Timestamp (`ai_memory_consented_at`), Transactional Blob-Cleanup Outbox (`deleted_blob_cleanup_queue`) & Provider Privacy Attestation (`store: false` / ZDR)** | `CDR-008`, `CDR-019`, `CDR-020`, `CDR-022`, `CDR-026` | `MEMORY_DESIGN.md` §2–3, `EVALUATION_PLAN.md` §2 |
| **`ADR-AI-009`** | **Phase-Accurate Quota Ledgers (Phase 1 In-Memory Journal vs. Phase 2 Postgres `ai_quota_reservations`) & Midnight-Safe Settlement** | `CDR-009`, `CDR-018` | `EVALUATION_PLAN.md` §1, `STAGE_01`, `STAGE_02` |
| **`ADR-AI-010`** | **Zero-JSON Binary Command Frame (`0x01`) with Opaque `canonicalParamsUtf8` & Web SSE `citations`/`pendingProposals` Pass-Through** | `CDR-012`, `CDR-019`, `CDR-027` | `TOOL_CONTRACTS.md` §3.2, `STAGE_02`, `STAGE_03` |
| **`ADR-AI-011`** | **BFF-Authoritative Taint Latch (`inFlightTurns[reqId].tainted`) via Mandatory Research Lease + OS `nftables` Egress** | `CDR-014`, `CDR-021` | `BROWSING_DESIGN.md` §1–2, `STAGE_03` |
| **`ADR-AI-012`** | **Bounded Agentic Scope (`stopWhen: stepCountIs(3)` Interactive; Phase 5 `DBOS` Inside BFF / Core Workers)** | `CDR-001`, Lead J | `MASTERPLAN.md` §3.7, `STAGE_05` |
| **`ADR-AI-013`** | **Dual-Gate Node.js Runtime Policy (`>=18.20.0` Dev Container + `Iterator` Polyfill vs. `STRICT_NODE_24_GATE=1` Release CI)** | `CDR-024` | `DOCUMENTATION_AUDIT.md` §1, `STAGE_01`, `STAGE_03` |

## 2. Key `v3.1` Additions (`ADR-AI-008 v3.1`, `ADR-AI-010 v3.1`, `ADR-AI-013`)

1. **`ADR-AI-008 v3.1` — Transactional Blob-Cleanup Outbox (`deleted_blob_cleanup_queue`, Resolves `CDR-026`):**
   - To guarantee that a transient filesystem/S3 unlink error during `DELETE /api/v1/account` or `DELETE /api/v1/attachments/:id` never leaves orphaned user files after the SQL transaction commits, Migration `002` adds `deleted_blob_cleanup_queue (object_path TEXT PRIMARY KEY, user_id_hash TEXT NOT NULL, queued_at TIMESTAMPTZ NOT NULL DEFAULT NOW(), retry_count INTEGER NOT NULL DEFAULT 0, last_error TEXT)`.
   - Inside `withTenantTransaction`, before deleting `user_profiles` or `attachments`, the BFF inserts the target `object_path` rows into `deleted_blob_cleanup_queue`. Post-commit, the BFF unlinks each file and removes its queue entry (`ENOENT` = success); any remaining entry is retried by a background sweeper.
2. **`ADR-AI-010 v3.1` — Web Shell `parseSseChatResponse` Field Pass-Through (Resolves `CDR-027`):**
   - `ridel_web_shell/src/features/chat_session_controller.ts` (`parseSseChatResponse`) is updated in Phase 2/3 to preserve `citations: donePayload.citations ?? []` and `pendingProposals: donePayload.pendingProposals ?? []` alongside inline `[Source Title](https://...)` markdown links in `replyText`.
3. **`ADR-AI-013` — Dual-Gate Node.js Runtime Policy (Resolves `CDR-024`):**
   - Because the current development container runs Node `v18.20.4` while production targets Node 24 LTS (`>=24.0.0 <25.0.0`), `transitos_agentic_ai/package.json` specifies `"engines": { "node": ">=18.20.0" }` for local test execution, enforces `STRICT_NODE_24_GATE=1` in `npm run ci:release` (matching `ridel_web_shell/package.json`), and loads `core-js/actual/iterator` inside `extractor_worker.ts` when `typeof globalThis.Iterator === 'undefined'` so [`unpdf`](https://github.com/unjs/unpdf) (`pdf.js` v6.4.299) runs cleanly on both Node 18 and Node 24.
