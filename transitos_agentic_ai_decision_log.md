# TRANSITOS Agentic AI — Final Architecture Decision Log (`TRANSITOS_AGENTIC_AI_DECISION_LOG.md`)

**Evaluation Date:** 2026-10-29 (`Revision v3.0` — Final Implementation Blueprint)  
**Scope:** Authoritative record of all 12 Architecture Decision Records (`ADR-AI-001` through `ADR-AI-012`), updated to incorporate the v3.0 critical audit resolutions (`CDR-015` through `CDR-023`).

## 1. Decision Summary Matrix (`v3.0`)

| ADR ID | Decision Title | Resolves Contradictions | Authoritative Blueprint Section |
| :--- | :--- | :--- | :--- |
| **`ADR-AI-001`** | **BFF-Owned Persistence & Zero Database Credentials in `transitos_agentic_ai`** | `CDR-001`, `CDR-013` | `MASTERPLAN.md` §1–3 |
| **`ADR-AI-002`** | **Split-Key Service Auth (`BFF_TO_PLANNER_HMAC_SECRET` vs. BFF-Only `BFF_CAPABILITY_SECRET`), Fail-Closed Replay Cache & Turn-Aligned Deadlines** | `CDR-003`, `CDR-015`, `CDR-016`, `CDR-017` | `TOOL_CONTRACTS.md` §1 |
| **`ADR-AI-003`** | **Server-Authoritative Action Policy, `PendingMutationProposal` Types & JCS-Bound Confirmation Vault (`POST /api/v1/actions/confirm`)** | `CDR-002`, `CDR-011`, `CDR-019` | `TOOL_CONTRACTS.md` §2 |
| **`ADR-AI-004`** | **Complete 12-Table PostgreSQL `FORCE ROW LEVEL SECURITY` Matrix, `NOBYPASSRLS` Runtime Role & Composite Foreign Keys** | `CDR-004`, `CDR-020` | `MEMORY_DESIGN.md` §1, §3 |
| **`ADR-AI-005`** | **Pinned Vercel AI SDK (`ai@^6.0.306`), Two-Channel Streaming, Pre-First-Token Fallback & Heartbeat Deadlines** | `CDR-005`, `CDR-010`, `CDR-023` | `MASTERPLAN.md` §2 S4, `TOOL_CONTRACTS.md` §3 |
| **`ADR-AI-006`** | **Pre-Hydrated Turn Context (`userSettings`, `savedPlacesSummary`, `relevantMemories`) & Deferred MCP** | `CDR-006` | `TOOL_CONTRACTS.md` §1.2, §3 |
| **`ADR-AI-007`** | **Preemptible `worker_threads` Isolation for `unpdf` (`pdf.js` v6.4.299) & `@mozilla/readability` + Dual Stream Byte Caps** | `CDR-007`, `CDR-021`, `CDR-022` | `BROWSING_DESIGN.md` §2–3 |
| **`ADR-AI-008`** | **Explicit Consent Timestamp (`ai_memory_consented_at`), Proposal-Based Memory Writes & Provider Privacy Attestation (`store: false` / ZDR)** | `CDR-008`, `CDR-019`, `CDR-020`, `CDR-022` | `MEMORY_DESIGN.md` §2, `EVALUATION_PLAN.md` §2 |
| **`ADR-AI-009`** | **Phase-Accurate Quota Ledgers (Phase 1 In-Memory Journal vs. Phase 2 Postgres `ai_quota_reservations`) & Midnight-Safe Settlement** | `CDR-009`, `CDR-018` | `EVALUATION_PLAN.md` §1 |
| **`ADR-AI-010`** | **Zero-JSON Binary Command Frame (`0x01`) with Opaque `canonicalParamsUtf8` Preservation for `ridel_shell`** | `CDR-012`, `CDR-019` | `TOOL_CONTRACTS.md` §3.2 |
| **`ADR-AI-011`** | **BFF-Authoritative Taint Latch (`inFlightTurns[reqId].tainted`) via Mandatory Research Lease + OS `nftables` Egress** | `CDR-014`, `CDR-021` | `BROWSING_DESIGN.md` §1–2 |
| **`ADR-AI-012`** | **Bounded Agentic Scope (`stopWhen: stepCountIs(3)` Interactive; Phase 5 `DBOS` Inside BFF / Core Workers)** | `CDR-001`, Lead J | `MASTERPLAN.md` §3.7, `ROADMAP.md` Phase 5 |

## 2. Key v3.0 Refinements Recorded

1. **`ADR-AI-002 v3.0` (Split-Key Service & Capability Signing):**
   - *Why changed from v2.0:* In v2.0, `transitos_agentic_ai` held `INTERNAL_SERVICE_TOKEN` to verify `X-Bff-Signature`, which meant a compromised planner could use that same secret to forge `X-Turn-Capability-Token` for arbitrary users (`CDR-015`).
   - *v3.0 Rule:* `BFF_TO_PLANNER_HMAC_SECRET` is shared for BFF -> Planner calls; `BFF_CAPABILITY_SECRET` is held **exclusively** by `transitos_bff_gateway` to sign and verify `X-Turn-Capability-Token`. Capability token TTL is set to `turnCeilingMs` (`18,000 ms` text / `25,000 ms` multimodal) and checked against `inFlightTurns.get(reqId)` on the BFF (`CDR-016`). Replay cache fails closed (`HTTP 503`) if saturated at `10,000` unexpired entries and is explicitly scoped to single-instance in Phase 1 and PostgreSQL `internal_request_nonces` in Phase 2 (`CDR-017`).
2. **`ADR-AI-003 v3.0` & `ADR-AI-010 v3.0` (Unified Confirmation Contract & Native Parameter Preservation):**
   - *Why changed from v2.0:* Resolved header-vs-body mismatch for `confirmationToken`, added `proposalType: 'UI_ACTION' | 'MEMORY_WRITE' | 'SAVED_PLACE_UPSERT'` so memory and saved-place writes require explicit confirmation without inventing non-existent FlatBuffer `ActionCode` IDs, canonicalized parameters via RFC 8785 JCS (`params_sha256`), and appended `canonicalParamsUtf8` (`u16 LE` length + bytes) to the Native `0x01` binary frame (`CDR-019`).
3. **`ADR-AI-004 v3.0` & `ADR-AI-008 v3.0` (Complete 12-Table RLS Matrix, Legacy Consent Reset & Provider Privacy Tiers):**
   - *Why changed from v2.0:* Extended `ENABLE` + `FORCE ROW LEVEL SECURITY` to all 5 Migration `001` tenant tables (`user_profiles`, `user_settings`, `saved_places`, `attachments`, `chat_messages`) in addition to the 5 Migration `002` tenant tables, added `ai_memory_consented_at TIMESTAMPTZ` to reset legacy unconsented rows to `0` (`CDR-020`), and added startup enforcement of provider privacy tiers (`store: false` on OpenAI; paid logging-disabled or Vertex ZDR on Gemini per the Oct 26, 2026 policy update — `CDR-022`).
4. **`ADR-AI-007 v3.0` & `ADR-AI-011 v3.0` (BFF Taint Authority, Dual Stream Caps, `nftables` & `worker_threads` Isolation):**
   - *Why changed from v2.0:* Moved taint authority to `transitos_bff_gateway` (`acquire_research_lease` sets `inFlightTurns[reqId].tainted = true` in BFF memory so the planner cannot clear taint), added a `5 MiB` decompressed stream cap alongside the `2 MiB` compressed socket cap, defined Linux `nftables` egress rules (`CDR-021`), and isolated CPU-bound `unpdf` and `@mozilla/readability` inside Node.js `worker_threads` (`128 MiB` heap cap, `worker.terminate()` at `3,000 ms` — `CDR-022`).
5. **`ADR-AI-009 v3.0` (Phase-Accurate Quota Ledgers & Midnight-Safe Settlement):**
   - *Why changed from v2.0:* Defined `InMemoryJournaledQuotaLedger` for Phase 1 (single-instance) and `PostgresQuotaLedger` (`ai_quota_ledger` + `ai_quota_reservations`) for Phase 2 (multi-replica), binding every reservation to `quota_date_utc` captured at reservation time and settling idempotently via `WHERE status = 'RESERVED'` (`CDR-018`).
