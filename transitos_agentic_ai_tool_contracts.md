# TRANSITOS Agentic AI — Canonical Service, Tool, Action & Streaming Contracts (`TRANSITOS_AGENTIC_AI_TOOL_CONTRACTS.md`)

**Evaluation Date:** 2026-10-29 (`Revision v3.0` — Final Implementation Blueprint)  
**Pinned SDK Versions (`ADR-AI-005`):** [Vercel AI SDK `ai@^6.0.306`](https://www.npmjs.com/package/ai), `@ai-sdk/google@^2.x`, `@ai-sdk/openai@^2.x`, `@ai-sdk/anthropic@^2.x`, and **[`zod@^3.25.76` (or `>=4.0.4`)](https://github.com/vercel/ai/issues/7189)** (avoiding `zod@4.0.0–4.0.3` build issues).

## 1. Split-Key Internal Service Authentication & Replay Protection (`ADR-AI-002 v3.0`, Resolves `CDR-015`..`CDR-017`)

To guarantee that `transitos_agentic_ai` **cannot mint or forge its own authorized turn capability tokens**, `transitos_bff_gateway` uses **two cryptographically distinct secrets**:

| Secret Name | Held By `transitos_bff_gateway`? | Held By `transitos_agentic_ai`? | Purpose & Direction |
| :--- | :---: | :---: | :--- |
| **`BFF_TO_PLANNER_HMAC_SECRET`** (`>= 32` bytes) | **Yes (Signer)** | **Yes (Verifier)** | Authenticates `POST /internal/v1/plan` (`X-Bff-Signature`). Supports `BFF_TO_PLANNER_HMAC_SECRET_PREVIOUS` for zero-downtime key rotation. |
| **`BFF_CAPABILITY_SECRET`** (`>= 32` bytes, or Ed25519 private key) | **Yes (Sole Signer & Verifier)** | **NO (`0` access)** | Mints and verifies `X-Turn-Capability-Token`. Passed to the planner as an **opaque bearer string** that the planner echoes back on `/internal/v1/tools/*` callbacks. |

### 1.1 BFF-to-Planner Request Signing (`POST /internal/v1/plan`)
- **Canonical String to Sign:**
  `"v1\nPOST\n/internal/v1/plan\n${timestampSec}\n${requestId}\n${userId}\n${sha256Hex(rawRequestBodyBytes)}"`
- **Headers:** `X-Request-Id`, `X-Timestamp`, `X-User-Id`, `X-Bff-Key-Id: current|previous`, `X-Bff-Signature: <hex>`.
- **Verification & Replay Protection:**
  1. Reject if `Math.abs(nowSec - Number(timestampSec)) > 60` (`HTTP 401`).
  2. Recompute expected HMAC using `BFF_TO_PLANNER_HMAC_SECRET` (or `_PREVIOUS` if `X-Bff-Key-Id === 'previous'`) over the raw body buffer and verify via `crypto.timingSafeEqual` (`HTTP 401` on mismatch).
  3. Assert `parsedBody.requestId === headerRequestId` and `parsedBody.userContext.userId === headerUserId` (`HTTP 400` on mismatch).
  4. **Replay Cache Semantics (`CDR-017`):**
     - **Phase 1 (Single-Instance Deployment):** In-memory `Map<requestId, expiresAtMs>` (`TTL = 120s`, `MAX_ENTRIES = 10,000`). If `requestId` is already present, reject with `HTTP 409 REPLAY_DETECTED`. If the cache reaches `10,000` unexpired entries (saturation attack), **fail closed (`HTTP 503 REPLAY_CACHE_SATURATED`)** rather than evicting unexpired nonces.
     - **Phase 2+ (Multi-Replica Deployment):** Backed by PostgreSQL `UNLOGGED TABLE internal_request_nonces (request_id TEXT PRIMARY KEY, expires_at TIMESTAMPTZ NOT NULL)` via `INSERT INTO internal_request_nonces VALUES ($1, NOW() + INTERVAL '120 seconds') ON CONFLICT DO NOTHING RETURNING request_id` (`0` rows -> `HTTP 409 REPLAY_DETECTED`).

### 1.2 Turn-Scoped Capability Token (`X-Turn-Capability-Token` — Resolves `CDR-015`, `CDR-016`)
- **Payload Minted by BFF:**
  `{ v: 1, reqId: requestId, sub: userId, jti: sessionJti, scopes: string[], aiMemory: boolean, deadlineMs: nowMs + turnCeilingMs }`
  - `turnCeilingMs` is **`18,000 ms`** for standard turns (`15,000 ms` turn deadline + `3,000 ms` grace) and **`25,000 ms`** for multimodal turns (`20,000 ms` turn deadline + `5,000 ms` grace), ensuring the token never expires mid-operation (`CDR-016`).
- **Verification on Planner-to-BFF Tool Callbacks (`POST /internal/v1/tools/:toolName`):**
  1. Verify HMAC signature using BFF-only `BFF_CAPABILITY_SECRET` via `crypto.timingSafeEqual`.
  2. Check `Date.now() <= token.deadlineMs`.
  3. Look up `turnState = inFlightTurns.get(token.reqId)` on the BFF; reject (`HTTP 403 TURN_CLOSED_OR_ABORTED`) if the turn has already completed or the client disconnected (`req.on('close')`).
  4. Enforce `turnState.toolCallsCount < 3` and increment `turnState.toolCallsCount`.
  5. Bind the tool execution strictly to `userId = token.sub` (ignoring any `userId` field in the request body).

## 2. Canonical Action Policy Table, Mutation Proposals & Confirmation Contract (`ADR-AI-003 v3.0`, Resolves `CDR-019`)

### 2.1 Planner Raw Output vs. BFF Authoritative Policy
The LLM / Planner outputs only `rawProposal` and optional `pendingProposals[]` (never `actionId`, `securityLevel`, `requiresConfirmation`, or `confirmationToken`):

```typescript
import { z } from 'zod';

export const PrimitiveParamSchema = z.union([
  z.string().max(512),
  z.number().finite(),
  z.boolean(),
]);

export const PlannerRawActionProposalSchema = z.object({
  actionCode: CanonicalActionCodeSchema.default('CMD_NONE'),
  destination: z.string().max(200).default(''),
  parameters: z.record(z.string().max(64), PrimitiveParamSchema).default({}),
}).strict();

export const PlannerPendingMutationProposalSchema = z.discriminatedUnion('proposalType', [
  z.object({
    proposalType: z.literal('MEMORY_WRITE'),
    category: z.enum(['commute_pref', 'place_note', 'vehicle_pref', 'schedule_habit', 'general']),
    factText: z.string().min(3).max(500),
    ttlDays: z.number().int().min(1).max(365).default(180),
  }).strict(),
  z.object({
    proposalType: z.literal('SAVED_PLACE_UPSERT'),
    label: z.string().min(1).max(64),
    name: z.string().min(1).max(128),
    subtitle: z.string().min(1).max(256),
  }).strict(),
]);
```

### 2.2 Complete 26-Entry `CANONICAL_ACTION_POLICY_TABLE` (`0..25`)
Aligned 1-to-1 across `ridel_ui/4_sdui_schema/action_registry_v1.json`, `ridel_shell/.../permission_guard.dart`, `ridel_web_shell/src/actions/action_dispatcher.ts`, and `transitos_bff_gateway`:

| `actionId` | `actionCode` | Security Level | `requiresConfirmation` | Allowed for AI Proposal? | Allowed Screen Contexts | Tainted-Turn Behavior (`turnState.tainted === true`) |
| :---: | :--- | :---: | :---: | :---: | :--- | :--- |
| `0` | `CMD_NONE` | `SAFE_LEVEL_1` | `false` | **Yes (Phase 1+)** | All screens | Permitted |
| `1` | `CMD_ROUTE_TO_DEST` | `SAFE_LEVEL_1` | `false` | **Yes (Phase 1+)** | `main_canvas_v1`, `main_chat_v1`, `sidebar_v1` | Permitted (renders route preview card only) |
| `9` | `CMD_MAP_INTERACTION` | `SAFE_LEVEL_1` | `false` | **Yes (Phase 1+)** | `main_canvas_v1`, `main_chat_v1`, `sidebar_v1` | Permitted |
| `17` | `CMD_GET_DIRECTIONS` | `SAFE_LEVEL_1` | `false` | **Yes (Phase 1+)** | `main_canvas_v1`, `main_chat_v1` | Permitted |
| `18` | `CMD_EXPLORE_AI` | `SAFE_LEVEL_1` | `false` | **Yes (Phase 1+)** | `main_canvas_v1`, `main_chat_v1`, `sidebar_v1` | Permitted |
| `24` | `CMD_EDIT_PROFILE` | `CONFIRM_LEVEL_2` | **`true`** | **Phase 2+ (With Token)** | `profile_v1`, `main_chat_v1` | **Hard-downgraded to `CMD_NONE`** |
| `25` | `CMD_LOGOUT` | `CONFIRM_LEVEL_2` | **`true`** | **Phase 2+ (With Token)** | `sidebar_v1`, `profile_v1`, `main_chat_v1` | **Hard-downgraded to `CMD_NONE`** |
| `4` | `CMD_BOOK_MOVIE` | `BIOMETRIC_LEVEL_3` | **`true`** | **Phase 4+ (With Token + Step-Up)** | `main_chat_v1` | **Hard-downgraded to `CMD_NONE`** |
| `6` | `CMD_BOOK_NOW` | `BIOMETRIC_LEVEL_3` | **`true`** | **Phase 4+ (With Token + Step-Up)** | `main_chat_v1` | **Hard-downgraded to `CMD_NONE`** |
| `2,3,5,7,8,10..16,19..23` | Client-only UI Chrome Actions (`CMD_TOGGLE_SIDEBAR`, `CMD_SELECT_MODEL`, etc.) | `SAFE_LEVEL_1` | `false` | **No (Client UI Only)** | Per `SCREEN_CONTEXT_ACTION_ALLOWLIST` | **Hard-downgraded to `CMD_NONE`** |

### 2.3 Canonical Confirmation Endpoint (`POST /api/v1/actions/confirm` — Resolves `CDR-019`)
- **Parameter Canonicalization (RFC 8785 JCS):** Before minting or verifying a `confirmationToken`, `transitos_bff_gateway` serializes `parameters` with lexicographically sorted keys and no whitespace (`canonicalizeParams(parameters)`) and computes `params_sha256 = sha256Hex(canonicalJson)`.
- **Request Contract (`POST /api/v1/actions/confirm`):**
  - **Headers:** `Authorization: Bearer <jwt>`, `Idempotency-Key: <uuid>`, `Content-Type: application/json` (optional `X-Confirmation-Token` header is accepted only if identical to `body.confirmationToken`).
  - **JSON Body:**
    ```json
    {
      "confirmationToken": "ctok_9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4c",
      "proposalType": "UI_ACTION",
      "actionCode": "CMD_LOGOUT",
      "parameters": {},
      "stepUpVerified": false
    }
    ```
    *(Where `proposalType` is `'UI_ACTION' | 'MEMORY_WRITE' | 'SAVED_PLACE_UPSERT'`; for `BIOMETRIC_LEVEL_3`, `stepUpVerified: true` or platform biometric assertion is required).*
- **Atomic Redemption Rule in BFF:**
  - Executes `UPDATE pending_action_confirmations SET consumed_at = NOW() WHERE token_hash = $1 AND owner_id = $2 AND session_jti = $3 AND proposal_type = $4 AND action_code = $5 AND params_sha256 = $6 AND expires_at > NOW() AND consumed_at IS NULL RETURNING *`.
  - If `0` rows match: returns `HTTP 409 CONFIRMATION_ALREADY_CONSUMED` (if `consumed_at IS NOT NULL`) or `HTTP 403 INVALID_OR_EXPIRED_CONFIRMATION`.

## 3. Streaming & Native `0x00`/`0x01` Binary Frame Contracts (`ADR-AI-005` & `ADR-AI-010 v3.0`)

### 3.1 Internal Planner-to-BFF Stream (`application/x-ndjson`)
- `{"type":"progress","phase":"planning"|"tool_call","toolName":"get_route_eta"}\n`
- `{"type":"token","token":"Leave by 9:10 "}\n`
- `{"type":"done","replyText":"...","durationText":"46 min","distanceViaText":"19.1 km • No tolls","rawProposal":{"actionCode":"CMD_ROUTE_TO_DEST","destination":"Office via Ring Road","parameters":{"avoidTolls":true}},"pendingProposals":[],"citations":[],"usage":{"modelId":"gemini-2.5-pro","inputTokens":410,"outputTokens":48,"latencyMs":1120,"toolCallsExecuted":1}}\n`
- `{"type":"error","code":"STREAM_INTERRUPTED"|"LLM_TIMEOUT","message":"...","retryable":true}\n`

### 3.2 Native `ridel_shell` Binary WebSocket Frame (`0x01` — Resolves `CDR-012`, `CDR-019`)
To preserve `parameters` for confirmation round-trips without calling `jsonDecode()` on the Flutter UI thread:
- **Frame `0x00` (Text Token):** `[0x00, ...utf8Bytes]` (always split on complete UTF-8 code-point boundaries via `TextDecoder` stream mode).
- **Frame `0x01` (Validated Command Proposal):**
  - Byte `0`: `0x01`
  - Byte `1`: `actionId` (`1..25`)
  - Byte `2`: `securityLevel` (`1` = Safe, `2` = Confirm, `3` = Biometric)
  - Bytes `3..4` (`u16 LE`): `destination` UTF-8 byte length (`N`), followed by `N` bytes
  - Byte `5+N` (`u8`): `durationText` UTF-8 byte length (`M`), followed by `M` bytes
  - Byte `6+N+M` (`u8`): `distanceViaText` UTF-8 byte length (`K`), followed by `K` bytes
  - Byte `7+N+M+K` (`u8`): `confirmationToken` ASCII byte length (`C`), followed by `C` bytes
  - Bytes `8+N+M+K+C .. 9+N+M+K+C` (`u16 LE`): `canonicalParamsUtf8` byte length (`P`), followed by `P` opaque bytes (stored in `ConfirmationMemory` and echoed back verbatim on confirmation without UI-thread JSON parsing).
