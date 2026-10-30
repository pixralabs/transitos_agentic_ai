# TRANSITOS Agentic AI — Central Documentation Index (`docs/README.md`)

**Revision:** `v3.1` (`2026-10-30`)  
**Purpose:** Single navigational index for all architecture specifications, phase implementation guides, runbooks, and governance records in `transitos_agentic_ai`. Every link below points to a verified, existing file in the repository tree.

## 1. Project Overview & Execution Runbooks

* **[Root Project Overview (../README.md)](https://www.google.com/search?q=../README.md)** — High-level system architecture, how `transitos_agentic_ai` differs from an ordinary chatbot, ecosystem boundaries (`transitos_bff_gateway`, `ridel_ui`, `ridel_web_shell`, `ridel_shell`), and phase summary.
* **[Master Implementation Runbook (IMPLEMENTATION\_RUNBOOK.md)](https://www.google.com/search?q=IMPLEMENTATION_RUNBOOK.md)** — Practical, task-by-task execution guide (`TASK-1.1` through `TASK-5.4`) with priorities (`P0`–`P3`), target files, commands, expected outputs, and phase entry/exit gates.
* **[Requirements Traceability Matrix (TRACEABILITY\_MATRIX.md)](https://www.google.com/search?q=TRACEABILITY_MATRIX.md)** — Maps every requirement (`REQ-01` through `REQ-24`) to its Architecture Decision (`ADR-AI-*`), component, implementation task (`TASK-*`), verification test, acceptance criterion, and target phase.
* **[Master Cross-Verification \& Documentation Audit (DOCUMENTATION\_AUDIT.md)](https://www.google.com/search?q=DOCUMENTATION_AUDIT.md)** — Cross-audit of all 13 architecture documents against actual workspace code and verification of Critical Architecture Risks `5A` through `5N`.

## 2. System Architecture & Technology Decisions

* **[Authoritative Masterplan \& Target Architecture (architecture/TRANSITOS\_AGENTIC\_AI\_MASTERPLAN.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_MASTERPLAN.md)** — Canonical subsystem blueprint (`S1`–`S8`), service ownership matrix, degraded-mode recovery, and 7 end-to-end Mermaid sequence/architecture diagrams.
* **[Open-Source Landscape Research (architecture/TRANSITOS\_AGENTIC\_AI\_OPEN\_SOURCE\_LANDSCAPE.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_OPEN_SOURCE_LANDSCAPE.md)** — Primary-source evaluation of orchestration, MCP, web retrieval, memory, multimodal, model routing, and security libraries (`Categories A–G`).
* **[Technology Selection \& Build-vs-Buy Matrix (architecture/TRANSITOS\_AGENTIC\_AI\_TECHNOLOGY\_DECISIONS.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_TECHNOLOGY_DECISIONS.md)** — `KEEP` / `ADOPT` / `PILOT` / `DEFER` / `REJECT` decision matrix and explicit justification for every custom TypeScript module.

## 3. Security, Threat Model & Tool/API Contracts

* **[Threat Model \& Defense-in-Depth Controls (architecture/TRANSITOS\_AGENTIC\_AI\_THREAT\_MODEL.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_THREAT_MODEL.md)** — Trust zones, STRIDE / OWASP LLM threat matrix (`THR-01`–`THR-11`), split-key auth, BFF-authoritative taint latch, and `worker_threads` parser isolation.
* **[Canonical Service, Tool, Action \& Streaming Contracts (architecture/TRANSITOS\_AGENTIC\_AI\_TOOL\_CONTRACTS.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_TOOL_CONTRACTS.md)** — `POST /internal/v1/plan` HMAC contract, `X-Turn-Capability-Token` (`BFF_CAPABILITY_SECRET`), 26-entry `CANONICAL_ACTION_POLICY_TABLE`, `POST /api/v1/actions/confirm` JCS contract, and Web SSE / Native `0x00`/`0x01` binary frames.

## 4. Memory, Browsing, Multimodal Processing & Evaluation

* **[Memory, 12-Table RLS Matrix, Consent \& Retention Design (architecture/TRANSITOS\_AGENTIC\_AI\_MEMORY\_DESIGN.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_MEMORY_DESIGN.md)** — Complete 12-table PostgreSQL `FORCE ROW LEVEL SECURITY` matrix (`transitos_app_runtime` `NOBYPASSRLS`), `ai_memory_consented_at` legacy reset, multilingual (`simple` + `pg_trgm` + `pgvector`) RRF search, and `deleted_blob_cleanup_queue` outbox.
* **[Safe Web Retrieval, SSRF Pinning \& Worker Extraction (architecture/TRANSITOS\_AGENTIC\_AI\_BROWSING\_AND\_CRAWLING\_DESIGN.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_BROWSING_AND_CRAWLING_DESIGN.md)** — BFF `acquire_research_lease` taint latch, `undici.Agent` connect-time DNS IP pinning, `2 MiB` raw / `5 MiB` decompressed stream caps, Linux `nftables` egress firewall, and `worker_threads` extraction (`@mozilla/readability` + [unpdf](https://www.google.com/search?q=https://github.com/unjs/unpdf)).
* **[Evaluation, Split SLOs, Quota Ledgers \& Provider Privacy (architecture/TRANSITOS\_AGENTIC\_AI\_EVALUATION\_PLAN.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_EVALUATION_PLAN.md)** — Phase 1 (`InMemoryJournaledQuotaLedger`) vs. Phase 2 (`PostgresQuotaLedger`) midnight-safe quota accounting, Oct 2026 provider privacy attestation (`store: false` / ZDR), split latency SLOs, and [Promptfoo](https://www.google.com/search?q=https://github.com/promptfoo/promptfoo) evaluation gates.

## 5. Step-by-Step Implementation Phases (`Stages 01–05`)

* **[Phase 1: Safe Agent Core (stages/01-safe-agent-core/README.md)](https://www.google.com/search?q=stages/01-safe-agent-core/README.md)** — Split-key service auth, in-memory journaled quotas, `CANONICAL_ACTION_POLICY_TABLE`, stateless planner (`ai@^6.0.306`), 3 read-only tools, and Web SSE / Native `0x00`/`0x01` streaming.
* **[Phase 2: Persistent Memory \& Retrieval (stages/02-persistent-memory/README.md)](https://www.google.com/search?q=stages/02-persistent-memory/README.md)** — PostgreSQL 16 + `pgvector`, 12-table `FORCE RLS`, `ConfirmationVault` (`POST /api/v1/actions/confirm`), explicit memory consent & approval, and cascading account deletion with `deleted_blob_cleanup_queue`.
* **[Phase 3: Web Research \& Grounded Answers (stages/03-web-research/README.md)](https://www.google.com/search?q=stages/03-web-research/README.md)** — Search adapter, BFF research lease taint latch, `SafeWebFetcher` connect-time DNS IP pinning, `worker_threads` HTML/PDF extraction, and `CitationValidator`.
* **[Phase 4: Multimodal Understanding (stages/04-multimodal/README.md)](https://www.google.com/search?q=stages/04-multimodal/README.md)** — [Sharp](https://www.google.com/search?q=https://github.com/lovell/sharp) image sanitization (`16 MP` cap, EXIF strip), `read_verified_attachment_excerpt` (`unpdf` worker, `<= 20` pages), Vision LLM routing, and pre-first-token failover.
* **[Phase 5: Durable Agentic Workflows (stages/05-durable-workflows/README.md)](https://www.google.com/search?q=stages/05-durable-workflows/README.md)** — [DBOS Transact TS](https://www.google.com/search?q=https://github.com/dbos-inc/dbos-transact-ts) checkpointed workflows in `transitos_bff_gateway`, human approval suspension (`DBOS.recv()`), Promptfoo CI red-team gates, and optional isolated transit crawler.
* **[High-Level Implementation Roadmap (architecture/TRANSITOS\_AGENTIC\_AI\_IMPLEMENTATION\_ROADMAP.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_IMPLEMENTATION_ROADMAP.md)** — Executive summary of phase dependencies and release gates.

## 6. Governance & Architecture Audits

* **[Final Architecture Audit \& Approval Report (architecture/TRANSITOS\_AGENTIC\_AI\_ARCHITECTURE\_AUDIT.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_ARCHITECTURE_AUDIT.md)** — Executive audit report, severity counts, verified vs. planned capabilities, residual risks, and `APPROVE FOR PHASE 1` recommendation.
* **[Complete Cross-Document Contradiction Register (architecture/TRANSITOS\_AGENTIC\_AI\_CONTRADICTION\_REGISTER.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_CONTRADICTION_REGISTER.md)** — Full ledger of all 27 contradictions (`CDR-001` through `CDR-027`) discovered and resolved across v1.0, v2.0, and v3.1.
* **[Architecture Decision Log (architecture/TRANSITOS\_AGENTIC\_AI\_DECISION\_LOG.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_DECISION_LOG.md)** — Authoritative record of `ADR-AI-001` through `ADR-AI-013`.
* **[Core Requirements Traceability (architecture/TRANSITOS\_AGENTIC\_AI\_REQUIREMENTS\_TRACEABILITY.md)](https://www.google.com/search?q=architecture/TRANSITOS_AGENTIC_AI_REQUIREMENTS_TRACEABILITY.md)** — Security invariant to test mapping (`REQ-01`..`REQ-20`, extended to `REQ-24` in [TRACEABILITY\_MATRIX.md](https://www.google.com/search?q=TRACEABILITY_MATRIX.md)).
