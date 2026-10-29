# TRANSITOS Agentic AI — Final Memory, 12-Table RLS Matrix, Consent & Retention Design (`TRANSITOS_AGENTIC_AI_MEMORY_DESIGN.md`)

**Evaluation Date:** 2026-10-29 (`Revision v3.0` — Final Implementation Blueprint)  
**Database Ownership:** Owned **100% by `transitos_bff_gateway`** (`ADR-AI-001`). `transitos_agentic_ai` never connects to PostgreSQL.

## 1. Complete 12-Table Access-Control & Row-Level Security Matrix (`ADR-AI-004 v3.0`, Resolves `CDR-020`)

Every table across Migration `001` (`001_initial_bff_schema.up.sql`) and Migration `002` (`002_agentic_memory.up.sql`) is audited below. All 10 tenant-scoped tables enforce **both** `ENABLE ROW LEVEL SECURITY` and `FORCE ROW LEVEL SECURITY` under the `transitos_app_runtime` (`NOSUPERUSER NOBYPASSRLS`) role:

| # | Table Name | Migration | Primary Key & Composite Constraints | Tenant Column | `ENABLE` + `FORCE RLS`? | `transitos_app_runtime` Policy (`USING` & `WITH CHECK`) | Cascade / Retention Rule |
| :---: | :--- | :---: | :--- | :--- | :---: | :--- | :--- |
| **1** | `schema_migrations` | `001` | `PRIMARY KEY (version)` | *System-wide* | No (No grant to runtime role) | `REVOKE ALL FROM transitos_app_runtime` (Read/written only by `transitos_migrations_admin`). | Permanent DDL ledger. |
| **2** | `user_profiles` | `001` & `002` | `PRIMARY KEY (user_id)` | `user_id` | **Yes (`ENABLE` + `FORCE`)** | `user_id = NULLIF(current_setting('app.current_user_id', true), '')` | Root parent record; deleting `(user_id)` cascades to all 9 child tenant tables. |
| **3** | `user_settings` | `001` & `002` | `PRIMARY KEY (user_id)`<br>`FK (user_id) -> user_profiles ON DELETE CASCADE` | `user_id` | **Yes (`ENABLE` + `FORCE`)** | `user_id = NULLIF(current_setting('app.current_user_id', true), '')` | `ON DELETE CASCADE` from `user_profiles`. Includes `ai_memory DEFAULT 0` & `ai_memory_consented_at`. |
| **4** | `saved_places` | `001` & `002` | `PRIMARY KEY (place_id)`<br>`FK (owner_id) -> user_profiles ON DELETE CASCADE` | `owner_id` | **Yes (`ENABLE` + `FORCE`)** | `owner_id = NULLIF(current_setting('app.current_user_id', true), '')` | `ON DELETE CASCADE` from `user_profiles`. |
| **5** | `attachments` | `001` & `002` | `PRIMARY KEY (attachment_id)`<br>`UNIQUE (attachment_id, owner_id)`<br>`FK (owner_id) -> user_profiles ON DELETE CASCADE` | `owner_id` | **Yes (`ENABLE` + `FORCE`)** | `owner_id = NULLIF(current_setting('app.current_user_id', true), '')` | `ON DELETE CASCADE` from `user_profiles` + disk/object blob unlink. |
| **6** | `chat_messages` | `001` & `002` | `PRIMARY KEY (message_id)`<br>`FK (owner_id) -> user_profiles ON DELETE CASCADE` | `owner_id` | **Yes (`ENABLE` + `FORCE`)** | `owner_id = NULLIF(current_setting('app.current_user_id', true), '')` | `ON DELETE CASCADE` + `90-day` rolling TTL cleanup job. |
| **7** | `revoked_tokens` | `001` & `002` | `PRIMARY KEY (jti)` | `user_id` | **Yes (`ENABLE` + `FORCE`)** | `SELECT` allowed for auth check (`USING (true)`), `INSERT`/`DELETE` restricted to `user_id = current_setting('app.current_user_id', true)`. | Auto-pruned when `expires_at < EXTRACT(EPOCH FROM NOW())`. |
| **8** | `chat_thread_summaries` | `002` | `PRIMARY KEY (owner_id, thread_id)`<br>`FK (owner_id) -> user_profiles ON DELETE CASCADE` | `owner_id` | **Yes (`ENABLE` + `FORCE`)** | `owner_id = NULLIF(current_setting('app.current_user_id', true), '')` | `ON DELETE CASCADE` + `90-day` rolling TTL. |
| **9** | `user_memories` | `002` | `PRIMARY KEY (memory_id)`<br>`FK (owner_id) -> user_profiles ON DELETE CASCADE` | `owner_id` | **Yes (`ENABLE` + `FORCE`)** | `owner_id = NULLIF(current_setting('app.current_user_id', true), '')` | `ON DELETE CASCADE` + `expires_at` (`<= 180 days` default) + instant user delete. |
| **10** | `user_attachment_chunks` | `002` | `PRIMARY KEY (chunk_id)`<br>**Composite `FK (attachment_id, owner_id) -> attachments(attachment_id, owner_id) ON DELETE CASCADE`** | `owner_id` | **Yes (`ENABLE` + `FORCE`)** | `owner_id = NULLIF(current_setting('app.current_user_id', true), '')` | `ON DELETE CASCADE` when parent attachment or user is deleted. |
| **11** | `pending_action_confirmations` | `002` | `PRIMARY KEY (token_hash)`<br>`FK (owner_id) -> user_profiles ON DELETE CASCADE` | `owner_id` | **Yes (`ENABLE` + `FORCE`)** | `owner_id = NULLIF(current_setting('app.current_user_id', true), '')` | `ON DELETE CASCADE` + `30s` (`UI_ACTION`) / `300s` (`MEMORY_WRITE`) expiry. |
| **12** | `ai_quota_ledger` & `ai_quota_reservations` | `002` | `PRIMARY KEY (owner_id, quota_date_utc)` / `PRIMARY KEY (request_id)`<br>`FK (owner_id) -> user_profiles ON DELETE CASCADE` | `owner_id` | **Yes (`ENABLE` + `FORCE`)** | `owner_id = NULLIF(current_setting('app.current_user_id', true), '')` | `ON DELETE CASCADE` + `30-day` billing ledger retention. |

*(System-level `ai_audit_logs` and `deleted_accounts_tombstone` store only `user_id_hash = HMAC_SHA256(AUDIT_SALT, user_id)` with zero PII or prompts; `transitos_app_runtime` has `INSERT, SELECT` on `ai_audit_logs` and `deleted_accounts_tombstone`).*

## 2. Consent Migration, Memory Approval & Deletion Semantics (`ADR-AI-008 v3.0`)

### 2.1 Legacy User Migration & Explicit Consent (`CDR-020`)
In Migration `001`, `user_settings.ai_memory` had `DEFAULT 1`. Simply running `ALTER COLUMN ai_memory SET DEFAULT 0` would leave existing rows at `1` without recorded consent. Migration `002` resolves this deterministically:
```sql
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS ai_memory_consented_at TIMESTAMPTZ DEFAULT NULL;
ALTER TABLE user_settings ALTER COLUMN ai_memory SET DEFAULT 0;
-- Reset any legacy row that lacks an explicit consent timestamp:
UPDATE user_settings SET ai_memory = 0 WHERE ai_memory_consented_at IS NULL;
```
When a user toggles `Memory` **ON** in Settings (`PUT /api/v1/settings` with `{ aiMemory: true }`), the BFF sets `ai_memory = 1` and `ai_memory_consented_at = NOW()`.

### 2.2 Enabling Memory vs. Approving a Specific Memory Write (`CDR-019`)
1. **Enabling the Feature (`ai_memory == 1 AND ai_memory_consented_at IS NOT NULL`):** Permits `transitos_bff_gateway` to read existing `user_memories` and allows the planner to call `propose_memory_write`.
2. **Approving a Specific Memory Write (`MEMORY_WRITE` Proposal -> Commit):**
   - Calling `propose_memory_write` **never inserts directly into `user_memories`**.
   - Instead, BFF validates that `turnState.tainted === false`, scrubs PII, rejects prohibited sensitive categories (health, religion, caste, political beliefs, sexual orientation, government IDs, financial account numbers, biometrics), and inserts a pending proposal into `pending_action_confirmations` (`proposal_type = 'MEMORY_WRITE'`, `expires_at = NOW() + INTERVAL '300 seconds'`).
   - The client displays a compact *"Save preference to memory? [Save] [Dismiss]"* chip on the assistant message. Only when the user clicks **Save** (`POST /api/v1/actions/confirm`) does the BFF compute the embedding and upsert into `user_memories`.

### 2.3 Opt-Out & Account Deletion Matrix (Honest Cross-System Guarantees)

| User Action | Live PostgreSQL Tables (`user_memories`, `user_attachment_chunks`, etc.) | Object Storage (`data/objects/` / R2 / S3) | Encrypted PostgreSQL Backups & WAL Archives | Privacy-Scrubbed Audit Logs (`ai_audit_logs`) | External Model Provider Logs (`Gemini / OpenAI / Anthropic`) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **User Toggles `ai_memory` OFF (`1 -> 0`)** | Immediately disables all memory reads/writes (`ai_memory = 0`, `ai_memory_consented_at = NULL`). UI prompts user: **Keep existing memories paused** OR **Delete all saved memories now** (`DELETE /api/v1/memories?all=true`). | Unchanged (attachments remain accessible in chat unless deleted). | Unchanged until rows are deleted. | Unchanged (`0` PII stored). | Future prompts no longer include `relevantMemories`. |
| **User Deletes Single Memory (`DELETE /api/v1/memories/:id`)** | **Immediate synchronous `DELETE`** in PostgreSQL. | N/A | Aged out within **`<= 14 days`** as rolling backups expire. | Unchanged (`0` PII stored). | Cannot retroactively alter prior completed inference requests; covered by provider `store: false` / ZDR policy. |
| **User Deletes Account (`DELETE /api/v1/account` with `X-Confirm-Account-Delete: DELETE`)** | **Immediate synchronous ACID `DELETE`** on `user_profiles` cascading (`ON DELETE CASCADE`) to all 9 tenant tables + inserts `user_id_hash` into `deleted_accounts_tombstone`. | **Immediate synchronous file unlink** of all user attachment blobs before HTTP `200` returns. | Rolling backups expire within **`<= 14 days`**. If a backup is restored during disaster recovery, startup script purges any `user_id` matching `deleted_accounts_tombstone` before opening traffic. | Retains only non-reversible `user_id_hash` + token/cost counters (`0` prompts, `0` PII) for `90 days`. | **Not atomic with DB delete:** Protected proactively by sending `store: false` (OpenAI), disabling project logging (Gemini Paid/Vertex), and scrubbing PII before egress (`ADR-AI-008 v3.0`). |

## 3. Complete Migration `002_agentic_memory.up.sql` (Including Migration `001` Table Hardening)

```sql
-- Migration 002 UP: Complete 12-Table RLS Hardening, Quotas, Confirmations & Hybrid Memory Schema
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'transitos_app_runtime') THEN
    CREATE ROLE transitos_app_runtime LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE;
  END IF;
END
$$;

-- 1. Harden Migration 001 Tables (Add FK cascades, consent timestamp, and FORCE RLS)
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS ai_memory_consented_at TIMESTAMPTZ DEFAULT NULL;
ALTER TABLE user_settings ALTER COLUMN ai_memory SET DEFAULT 0;
UPDATE user_settings SET ai_memory = 0 WHERE ai_memory_consented_at IS NULL;

CREATE UNIQUE INDEX IF NOT EXISTS idx_attachments_id_owner ON attachments(attachment_id, owner_id);

ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_profiles FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_user_profiles ON user_profiles FOR ALL TO transitos_app_runtime
  USING (user_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (user_id = NULLIF(current_setting('app.current_user_id', true), ''));

ALTER TABLE user_settings ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_settings FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_user_settings ON user_settings FOR ALL TO transitos_app_runtime
  USING (user_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (user_id = NULLIF(current_setting('app.current_user_id', true), ''));

ALTER TABLE saved_places ENABLE ROW LEVEL SECURITY;
ALTER TABLE saved_places FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_saved_places ON saved_places FOR ALL TO transitos_app_runtime
  USING (owner_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (owner_id = NULLIF(current_setting('app.current_user_id', true), ''));

ALTER TABLE attachments ENABLE ROW LEVEL SECURITY;
ALTER TABLE attachments FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_attachments ON attachments FOR ALL TO transitos_app_runtime
  USING (owner_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (owner_id = NULLIF(current_setting('app.current_user_id', true), ''));

ALTER TABLE chat_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_messages FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_chat_messages ON chat_messages FOR ALL TO transitos_app_runtime
  USING (owner_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (owner_id = NULLIF(current_setting('app.current_user_id', true), ''));

-- 2. New Phase 2 Tables (Summaries, Memories, Attachment Chunks, Confirmations, Quotas, Nonces)
CREATE UNLOGGED TABLE IF NOT EXISTS internal_request_nonces (
  request_id TEXT PRIMARY KEY,
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE IF NOT EXISTS chat_thread_summaries (
  owner_id TEXT NOT NULL REFERENCES user_profiles(user_id) ON DELETE CASCADE,
  thread_id TEXT NOT NULL,
  summary_text TEXT NOT NULL CHECK (char_length(summary_text) <= 4000),
  last_message_id TEXT NOT NULL,
  turn_count INTEGER NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (owner_id, thread_id)
);

CREATE TABLE IF NOT EXISTS user_memories (
  memory_id TEXT PRIMARY KEY,
  owner_id TEXT NOT NULL REFERENCES user_profiles(user_id) ON DELETE CASCADE,
  category TEXT NOT NULL CHECK (category IN ('commute_pref', 'place_note', 'vehicle_pref', 'schedule_habit', 'general')),
  fact_text TEXT NOT NULL CHECK (char_length(fact_text) BETWEEN 3 AND 1000),
  source_message_id TEXT,
  user_confirmed BOOLEAN NOT NULL DEFAULT TRUE,
  embedding_model TEXT NOT NULL DEFAULT 'text-embedding-3-small-1536',
  embedding_dim INTEGER NOT NULL DEFAULT 1536 CHECK (embedding_dim = 1536),
  embedding halfvec(1536) NOT NULL,
  search_tsv tsvector GENERATED ALWAYS AS (to_tsvector('simple', fact_text)) STORED,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL DEFAULT (NOW() + INTERVAL '180 days')
);

CREATE TABLE IF NOT EXISTS user_attachment_chunks (
  chunk_id TEXT PRIMARY KEY,
  attachment_id TEXT NOT NULL,
  owner_id TEXT NOT NULL,
  page_number INTEGER NOT NULL DEFAULT 1,
  chunk_index INTEGER NOT NULL DEFAULT 0,
  chunk_text TEXT NOT NULL CHECK (char_length(chunk_text) <= 4000),
  embedding_model TEXT NOT NULL DEFAULT 'text-embedding-3-small-1536',
  embedding halfvec(1536) NOT NULL,
  search_tsv tsvector GENERATED ALWAYS AS (to_tsvector('simple', chunk_text)) STORED,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT fk_chunk_attachment_owner
    FOREIGN KEY (attachment_id, owner_id)
    REFERENCES attachments(attachment_id, owner_id)
    ON DELETE CASCADE
);

CREATE TABLE IF NOT EXISTS pending_action_confirmations (
  token_hash TEXT PRIMARY KEY,
  owner_id TEXT NOT NULL REFERENCES user_profiles(user_id) ON DELETE CASCADE,
  session_jti TEXT NOT NULL,
  proposal_type TEXT NOT NULL CHECK (proposal_type IN ('UI_ACTION', 'MEMORY_WRITE', 'SAVED_PLACE_UPSERT')),
  action_code TEXT NOT NULL DEFAULT 'CMD_NONE',
  params_json TEXT NOT NULL,
  params_sha256 TEXT NOT NULL,
  security_level TEXT NOT NULL CHECK (security_level IN ('CONFIRM_LEVEL_2', 'BIOMETRIC_LEVEL_3')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL,
  consumed_at TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS ai_quota_ledger (
  owner_id TEXT NOT NULL REFERENCES user_profiles(user_id) ON DELETE CASCADE,
  quota_date_utc DATE NOT NULL,
  used_tokens INTEGER NOT NULL DEFAULT 0 CHECK (used_tokens >= 0),
  reserved_tokens INTEGER NOT NULL DEFAULT 0 CHECK (reserved_tokens >= 0),
  daily_token_limit INTEGER NOT NULL DEFAULT 150000,
  used_cost_micros BIGINT NOT NULL DEFAULT 0 CHECK (used_cost_micros >= 0),
  reserved_cost_micros BIGINT NOT NULL DEFAULT 0 CHECK (reserved_cost_micros >= 0),
  daily_cost_limit_micros BIGINT NOT NULL DEFAULT 500000,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (owner_id, quota_date_utc)
);

CREATE TABLE IF NOT EXISTS ai_quota_reservations (
  request_id TEXT PRIMARY KEY,
  owner_id TEXT NOT NULL REFERENCES user_profiles(user_id) ON DELETE CASCADE,
  quota_date_utc DATE NOT NULL,
  reserved_tokens INTEGER NOT NULL CHECK (reserved_tokens > 0),
  reserved_cost_micros BIGINT NOT NULL CHECK (reserved_cost_micros >= 0),
  actual_tokens INTEGER,
  actual_cost_micros BIGINT,
  status TEXT NOT NULL CHECK (status IN ('RESERVED', 'SETTLED', 'REFUNDED')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  settled_at TIMESTAMPTZ,
  FOREIGN KEY (owner_id, quota_date_utc) REFERENCES ai_quota_ledger(owner_id, quota_date_utc) ON DELETE CASCADE
);

-- Enable + Force RLS on all new tenant tables
ALTER TABLE chat_thread_summaries ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_thread_summaries FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_chat_thread_summaries ON chat_thread_summaries FOR ALL TO transitos_app_runtime
  USING (owner_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (owner_id = NULLIF(current_setting('app.current_user_id', true), ''));

ALTER TABLE user_memories ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_memories FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_user_memories ON user_memories FOR ALL TO transitos_app_runtime
  USING (owner_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (owner_id = NULLIF(current_setting('app.current_user_id', true), ''));

ALTER TABLE user_attachment_chunks ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_attachment_chunks FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_user_attachment_chunks ON user_attachment_chunks FOR ALL TO transitos_app_runtime
  USING (owner_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (owner_id = NULLIF(current_setting('app.current_user_id', true), ''));

ALTER TABLE pending_action_confirmations ENABLE ROW LEVEL SECURITY;
ALTER TABLE pending_action_confirmations FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_pending_confirmations ON pending_action_confirmations FOR ALL TO transitos_app_runtime
  USING (owner_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (owner_id = NULLIF(current_setting('app.current_user_id', true), ''));

ALTER TABLE ai_quota_ledger ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_quota_ledger FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_ai_quota_ledger ON ai_quota_ledger FOR ALL TO transitos_app_runtime
  USING (owner_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (owner_id = NULLIF(current_setting('app.current_user_id', true), ''));

ALTER TABLE ai_quota_reservations ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_quota_reservations FORCE ROW LEVEL SECURITY;
CREATE POLICY rls_ai_quota_reservations ON ai_quota_reservations FOR ALL TO transitos_app_runtime
  USING (owner_id = NULLIF(current_setting('app.current_user_id', true), ''))
  WITH CHECK (owner_id = NULLIF(current_setting('app.current_user_id', true), ''));
```
