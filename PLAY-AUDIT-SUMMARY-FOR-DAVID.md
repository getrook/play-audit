# PLAY Project — Audit Summary for David
*March 15, 2026 | By Rook (Thomas's agent)*

---

## What We Did

Over the weekend, we ran a **full code sweep** (70+ files) and **live API testing** against the production indexer at `api-production-c67e.up.railway.app`. This document is the executive summary. Three detailed reports are attached as markdown files for your agent to parse.

---

## Code Sweep: 32 Issues Found

| Priority | Count | Key Issues |
|----------|-------|------------|
| 🔴 CRITICAL | 3 | Wrong owner on transfer, broken SDK reveal, no mint debounce |
| 🟠 HIGH | 8 | Optional API auth, spoofable rate limiter, silent reveal timeout, no refund on bounced mint |
| 🟡 MEDIUM | 10 | Invalid CSS class, dead code, no concurrency limit on metadata fetch |
| 🔵 LOW | 6 | Test page in prod, console logs, minor UX |
| ⚪ INFO | 5 | Architecture positives, solid contract design |

### The 3 Criticals

**C-1: OwnershipAssigned handler sets WRONG owner**
`event-handler.ts:515` calls `updateOwner(nftAddress, event.prevOwner)` — this records the *previous* owner as the current owner on every NFT transfer. Every post-mint transfer will show incorrect ownership in the indexer. This is a **data integrity bug** that affects all ownership tracking.

**C-2: `revealNft()` SDK method calls non-existent endpoint**
`POST /api/nfts/:address/reveal` doesn't exist in the backend. The frontend correctly uses `useUserRevealTransaction` for on-chain reveal, so the app works — but the SDK's public API is broken. The `useRevealUserNft` hook is dead code.

**C-3: No debounce on mint button**
`isPending` from the mutation isn't threaded to the button's disabled state. Rapid taps can queue multiple wallet approval popups, burning real TON on accidental double-mints.

---

## API Testing: 3 Critical Findings

We tested all 16 endpoints with auth, validation, security payloads, and rate limit stress.

### ✅ What Passed
- **Authentication** — Enforced on all protected endpoints. Wrong keys rejected. Health endpoint correctly public.
- **Input Validation** — Zod schemas working. Invalid addresses → 400. Negative pages → 400. Huge limits → 400.
- **Security** — SQL injection, XSS, path traversal all blocked. No stack traces leaked.
- **All 16 Read Endpoints** — Collections, NFTs, users, events, metrics all returning correct data.

### 🔴 What Failed

**1. Indexer Sync Stalled (4 Days Behind)**
```
lastSync: "2026-03-11T19:51:54.748Z"
syncLag: 352,835 seconds
```
Both collections haven't synced since Mar 11. Any mints/reveals after that won't appear in the API. The SSE client or collection poller is stuck.

**2. No Rate Limiting Active**
50 rapid-fire requests all returned HTTP 200. Zero 429 responses. The rate limiter in `server.ts` either isn't configured or isn't firing. Vulnerable to hammering on launch day.

**3. Reveal Endpoint Confirmed Missing**
`POST /api/nfts/:address/reveal` returns 404. Confirms C-2 from the code sweep. The SDK method should be removed.

---

## Action Items for David

### Before Launch (Blockers)

| # | Issue | File | Action |
|---|-------|------|--------|
| 1 | **Wrong owner on transfer** | `event-handler.ts:515` | Fix `handleOwnershipAssigned` to track new owner, not `prevOwner` |
| 2 | **Sync stalled** | SSE client / poller | Investigate why sync stopped Mar 11. Restart and verify real-time sync. |
| 3 | **No rate limiting** | `server.ts:95-103` | Verify rate limiter config. Test that 429s fire under load. |
| 4 | **Mint button debounce** | `MintForm.tsx:185-196` | Thread `isPending` from mutation to button disabled state |

### Before Go-Live (Important)

| # | Issue | File | Action |
|---|-------|------|--------|
| 5 | Remove `revealNft()` from SDK | `playproject-client.ts:304-317` | Dead code — endpoint doesn't exist |
| 6 | Remove `useRevealUserNft` hook | `playproject.tsx:245-270` | Dead code — will confuse future devs |
| 7 | Fix hardcoded wallet limit `10` | `MintForm.tsx:239-248` | Replace `> 10` with `> drop.maxPerWallet` |
| 8 | Add balance re-check in `handleMint` | `MintPage.tsx:78-115` | Prevent gas waste on stale balance |
| 9 | Show message on reveal timeout | `NFTDetailPage.tsx:157-165` | Don't silently return to idle — tell user |
| 10 | Add backoff to SSE client | `sse-client.ts` | Exponential backoff with jitter on reconnect |

### Nice-to-Have

| # | Issue | Action |
|---|-------|--------|
| 11 | `minted_per_wallet` unbounded map | Document storage cost implications |
| 12 | Bounced mint doesn't decrement wallet count | Decrement `minted_per_wallet` in bounce handler |
| 13 | MoonPay URL has no params | Add `currencyCode=ton_ton&walletAddress=...` |
| 14 | Remove SdkTestPage from prod routing | Gate behind `NODE_ENV !== 'production'` |

---

## Architecture Positives

The audit wasn't all bad. Notable strengths:
- **Smart contracts are solid** — TEP-62/66 compliant, two-step ownership transfer, per-wallet limits on-chain, bounce handlers, min balance checks
- **Event-driven indexer** — Idempotent handling, duplicate detection, paginated metadata fetch
- **13 spec documents** — Above average engineering discipline for this stage
- **Price sanity check** — Both frontend and SDK independently verify mint price < 100 TON

---

## Attached Reports

1. **CODE-SWEEP-REPORT.md** — Full 32-issue breakdown with file paths, code snippets, and fixes
2. **TESTING-PLAN-FINAL.md** — 4-phase testing plan (code analysis → API → E2E → load)
3. **API-TEST-RESULTS.md** — Complete API test results with request/response data

---

*Report generated by Rook. Questions → Thomas or ping the agent.*
