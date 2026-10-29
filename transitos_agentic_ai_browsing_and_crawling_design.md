# TRANSITOS Agentic AI — Safe Web Retrieval, BFF Taint Authority, Worker-Thread Extraction & SSRF Isolation (`TRANSITOS_AGENTIC_AI_BROWSING_AND_CRAWLING_DESIGN.md`)

**Evaluation Date:** 2026-10-29 (`Revision v3.0` — Final Implementation Blueprint)  
**Core Security Invariant:** `transitos_bff_gateway` never relies on a planner-provided boolean to track external taint (`CDR-021`). Before `transitos_agentic_ai` executes a web search or webpage fetch, it must acquire a research lease from `transitos_bff_gateway` (`POST /internal/v1/tools/acquire_research_lease`), which permanently latches `inFlightTurns.get(requestId).tainted = true` inside BFF memory.

## 1. BFF-Authoritative Taint Latch & Research Lease Protocol (`ADR-AI-011 v3.0`)

1. **Initial Turn Taint Assessment in BFF:**
   - At the start of `POST /api/v1/chat/messages`, `transitos_bff_gateway` initializes `inFlightTurns.set(requestId, { tainted: verifiedAttachments.length > 0 || recentConversation.some(m => m.taintedSource) })`.
2. **Mandatory Research Lease Acquisition (`POST /internal/v1/tools/acquire_research_lease`):**
   - Whenever `transitos_agentic_ai` invokes `web_search_transit` or `fetch_and_extract_webpage`, its tool wrapper first calls `POST /internal/v1/tools/acquire_research_lease` on `transitos_bff_gateway` with `X-Turn-Capability-Token` and `{ toolName, queryOrUrl }`.
   - `transitos_bff_gateway` atomically sets `inFlightTurns.get(requestId).tainted = true` (a monotonic one-way latch that can never be reset to `false`), logs the tool name in the turn's audit record, and returns `{ leaseId, allowed: true }`.
3. **BFF Enforcement When `turnState.tainted === true`:**
   - `propose_memory_write` and `propose_saved_place_upsert` callbacks are hard-rejected by the BFF (`HTTP 403 TAINTED_TURN_WRITE_BLOCKED`).
   - At turn completion (`done`), any Level 2 or Level 3 `rawProposal` (`CMD_EDIT_PROFILE`, `CMD_LOGOUT`, `CMD_BOOK_MOVIE`, `CMD_BOOK_NOW`) is deterministically downgraded by the BFF to `CMD_NONE`, regardless of what `taintedByExternalContent` flag the planner reports.

## 2. Defense-in-Depth SSRF Pipeline (`SafeWebFetcher` + Kernel Egress Firewall)

### 2.1 Application-Layer SSRF & Decompression-Bomb Pipeline (`src/research/safe_fetcher.ts`)
1. **Strict URL & Port Validation:**
   - Parse with `new URL(rawUrl)`. Permit only `https:` on default port `443` (or `http:` on port `80` only for allowlisted legacy municipal transit domains). Reject any URL with `username` or `password` credentials.
2. **Connect-Time DNS Resolution & IP Pinning (`undici.Agent`):**
   - Resolve `hostname` inside the custom `undici.Agent({ connect: { lookup } })` hook ([`undici` Issue #2019](https://github.com/nodejs/undici/issues/2019)) so the exact IP address checked against the blocklist is the IP address passed to `net.connect` / `tls.connect` (eliminating TOCTOU DNS rebinding).
   - **Blocked Non-Public CIDRs (IPv4, IPv6, IPv4-Mapped IPv6 `::ffff:0:0/96`, & NAT64 `64:ff9b::/96`):**
     - `0.0.0.0/8`, `10.0.0.0/8`, `100.64.0.0/10`, `127.0.0.0/8`, `169.254.0.0/16` (Cloud Metadata `169.254.169.254`), `172.16.0.0/12`, `192.0.0.0/24`, `192.0.2.0/24`, `192.168.0.0/16`, `198.18.0.0/15`, `198.51.100.0/24`, `203.0.113.0/24`, `224.0.0.0/4`, `240.0.0.0/4`.
     - `::/128`, `::1/128`, `100::/64`, `2001:db8::/32`, `fc00::/7`, `fe80::/10`, `ff00::/8`.
3. **Manual Redirect Validation (`maxRedirects = 3`):**
   - `redirect: 'manual'`. Every `301/302/303/307/308` `Location` target is resolved and re-validated through Steps 1 and 2.
4. **Dual Stream Byte Ceilings (Compressed `<= 2 MiB` AND Decompressed `<= 5 MiB` — Resolves `CDR-021`):**
   - Disable automatic unbounded decompression in `undici`.
   - Pipe the raw socket stream through a `CompressedByteLimitTransform(2 * 1024 * 1024)` (`2 MiB`), then through `zlib.createGunzip()` / `createBrotliDecompress()` (when `Content-Encoding` is present), and immediately through a `DecompressedByteLimitTransform(5 * 1024 * 1024)` (`5 MiB`) with a hard `5,000 ms` `AbortSignal`. A 10 KB gzip bomb is aborted the instant decompressed output crosses `5 MiB`.

### 2.2 Network/OS-Level Egress Firewall (`nftables` / Container Network Policy)
Application URL checks are never treated as a substitute for network-level isolation. The `transitos_agentic_ai` container/host enforces an outbound `nftables` policy dropping all non-public egress from the planner UID (`uid = transitos_ai`), except to the internal BFF loopback/subnet port:
```nft
table inet transitos_ai_egress {
  chain output {
    type filter hook output priority 0; policy accept;
    meta skuid != "transitos_ai" accept
    # Allow loopback / internal subnet ONLY to transitos_bff_gateway port 4000
    ip daddr 127.0.0.1 tcp dport 4000 accept
    # Block all private, loopback, link-local, metadata, and CGNAT IPv4 & IPv6 ranges
    ip daddr { 0.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10, 127.0.0.0/8, 169.254.0.0/16, 172.16.0.0/12, 192.168.0.0/16, 224.0.0.0/4, 240.0.0.0/4 } drop
    ip6 daddr { ::1/128, fc00::/7, fe80::/10, ff00::/8, ::ffff:0.0.0.0/96 } drop
    # Allow public outbound HTTPS (443) and HTTP (80) only
    tcp dport { 80, 443 } accept
    drop
  }
}
```

## 3. Preemptible `worker_threads` Content Extraction (`ADR-AI-007 v3.0`, Resolves `CDR-022`)

Because both `linkedom` + [@mozilla/readability](https://github.com/mozilla/readability) and [`unpdf` (`unjs/unpdf`)](https://github.com/unjs/unpdf) execute CPU-intensive parsing synchronously on the JavaScript thread ([`unpdf` Documentation](https://github.com/unjs/unpdf)), running them on the main planner thread could stall active SSE token streams.

1. **Bounded Worker Thread Pool (`src/workers/extractor_worker.ts`):**
   - `transitos_agentic_ai` maintains a pool of `maxWorkers = 2` Node.js `worker_threads` configured with `resourceLimits: { maxOldGenerationSizeMb: 128, maxYoungGenerationSizeMb: 32 }`.
   - The verified buffer is transferred zero-copy (`postMessage( { mimeType, buffer: arrayBuffer }, [arrayBuffer] )`) to the worker.
2. **Strict MIME + Magic-Byte Dispatch Inside Worker:**
   - `text/html` / `application/xhtml+xml` (not starting with `%PDF-`, `MZ`, `\x7FELF`) -> parsed via `linkedom` + `@mozilla/readability`, stripping `<script>`, `<style>`, `<iframe>`, and CSS-hidden elements.
   - `application/pdf` (verified `%PDF-` header) -> parsed via `unpdf` (`getDocumentProxy(bytes, { maxImageSize: 16_777_216 })`), enforcing `pdf.numPages <= 10` (`20` for user uploads).
   - `text/plain` (valid UTF-8, zero NUL bytes) -> normalized plain text.
3. **Hard Preemption on Timeout (`3,000 ms`):**
   - If the worker does not return within `3,000 ms`, the main thread calls `await worker.terminate()`, spawns a replacement worker, and returns `{ status: "error", code: "EXTRACTION_TIMEOUT" }` without stalling the main event loop for even `1 ms`.
