# PLAY Project — Comprehensive Testing Plan
*Date: 2026-03-15*
*Author: Rook*

## Executive Summary

PLAY is a mainnet TON blockchain app. Real money. Users mint NFT stickerpacks inside Telegram, then reveal them. Testing must cover smart contracts → backend API → frontend UX → integration. This plan separates what we can automate (code analysis, API testing) from what needs Thomas on his phone (wallet connect, mint flow, Telegram-specific UX).

**Biggest blockers:** Production indexer URL (Railway returning 404), reveal endpoint auth verification, contract address confirmation.

---

## Phase 1: Code Analysis (Automated)

### Smart Contract Audit
- **CRITICAL:** Review mint/reveal logic for access control, reentrancy, price enforcement
- **CRITICAL:** Verify reveal can ONLY be called by NFT owner (not anyone)
- **HIGH:** Check admin functions have proper access control
- **HIGH:** Verify no hardcoded keys/secrets in deploy scripts

### Backend/API Analysis  
- **CRITICAL:** Reveal endpoint (`POST /api/nfts/:address/reveal`) — must verify ownership before revealing
- **HIGH:** All endpoints must validate input (Zod schemas) and reject invalid data
- **HIGH:** No SQL injection vectors — all queries must be parameterized
- **HIGH:** Rate limiting must be enforced (prevent hammering during launch)
- **HIGH:** Error responses must NOT leak stack traces in production
- **MEDIUM:** Cache invalidation after mint/reveal (no stale data)
- **MEDIUM:** IPFS gateway fallback (if primary fails)
- **MEDIUM:** SSE reconnection logic for real-time updates

### Frontend Analysis
- **HIGH:** Mint button must have debounce (prevent double-mint race condition)
- **HIGH:** Optimistic updates must rollback on failure
- **HIGH:** Wallet disconnect mid-flow must be handled gracefully
- **HIGH:** SdkTestPage should NOT be accessible in production
- **MEDIUM:** All error states shown to user (no silent failures)
- **MEDIUM:** Dark mode rendering (all text readable, no invisible elements)
- **MEDIUM:** Invalid Tailwind classes (e.g., `text-1xl` doesn't exist)
- **LOW:** All image assets load correctly (logo, TON icon, etc.)

### SDK Analysis
- **CRITICAL:** Transaction builders must create valid TON messages
- **CRITICAL:** MAX_PRICE guard implemented correctly
- **HIGH:** No hardcoded endpoints/keys in SDK source

---

## Phase 2: API Testing (Once Indexer URL Available)

Assuming David provides the production indexer URL:

### Basic Connectivity
- `GET /api/health` → should return 200
- `GET /api/collections` → should return collection list
- `GET /api/collections/:address/nfts` → should return NFTs

### Input Validation
- Send malformed params (invalid addresses, negative page numbers, huge limits)
- Expect graceful 400 responses with error messages
- Verify pagination limits are enforced (max 100)

### Security Tests
- Send SQL injection payloads in address/params fields
- Expect rejection or sanitization (no DB errors)
- Hit reveal endpoint WITHOUT auth header
- Expect 401 or 403 (not 500)

### Rate Limiting
- Hit endpoints rapidly (100 reqs/sec for 10 seconds)
- Expect 429 response after threshold
- Verify backoff works (waits then retries successfully)

### Cache Behavior
- Mint an NFT → poll collection endpoint
- Verify NFT appears within 5-10 seconds (cache updated)
- Reveal an NFT → verify reveal status updated in cache

---

## Phase 3: E2E on Phone (Thomas)

### Happy Path
1. **Wallet Connect:** Open bot link → tap Connect Wallet → scan QR in TonKeeper → approve → see connected state
2. **Browse Home:** Home page shows collections/live drops, stats look reasonable
3. **Mint Flow:** Navigate to mint page → select quantity → see total price → approve in wallet → see "minting..." → NFT appears in collection
4. **Reveal Flow:** Tap unrevealed NFT → tap reveal → approve in wallet → see animation → see revealed image + rarity + attributes
5. **Profile:** Check profile page shows correct wallet address, mint count, activity
6. **Share/Download:** Tap share button (generates link/image), tap download (saves image)

### Error Cases
- **Insufficient Balance:** Try to mint more than balance allows → see graceful error
- **Reject Approval:** Reject wallet approval mid-flow → app recovers to previous state
- **Disconnect Mid-Flow:** Disconnect wallet while minting → app handles it (doesn't crash)
- **Rapid Clicks:** Click mint button 3x rapidly → only one transaction fires
- **Network Loss:** Briefly enable airplane mode → see loading states, recover when network returns

### Telegram-Specific
- **Dark Mode:** Toggle TG dark mode → verify all text readable, no invisible elements
- **Back Button:** Tap back → navigates correctly (or returns to Telegram if on home)
- **Haptic Feedback:** Tap mint/reveal → should feel haptic response if available
- **Safe Area:** Safe area insets respected (notch-aware on iPhone, etc.)
- **Viewport:** Test on multiple screen sizes (small Android phone, large iPhone, iPad)

---

## Phase 4: Load Testing (Before Launch)

Once backend is stable:

- **Concurrent Mints:** Simulate 100 users minting simultaneously → verify no double-mints, all transactions atomic
- **Concurrent Reveals:** Simulate 100 users revealing simultaneously → verify proper state transitions
- **Database Connections:** Monitor connection pool during concurrent load → should not exhaust pool
- **Redis Cache:** Monitor cache hit rate during load → should stay high (>80%)
- **API Response Times:** Mint endpoint should respond <2s under 100 RPS. Reveal should respond <1s.

---

## Pre-Requisites (Blockers)

| Item | Status | Owner | Action |
|------|--------|-------|--------|
| Production indexer URL | 🔴 BLOCKING | David | Provide Railway public URL or alternate endpoint |
| Reveal endpoint auth | 🔴 CRITICAL | David | Confirm: does it verify on-chain ownership? |
| Contract addresses | 🟡 NEEDED | David | Mainnet collection + item contract addresses |
| API key | 🟡 NEEDED | David | Is `x-api-key` header enforced? What's the key? |
| Test TON budget | 🟡 NEEDED | Thomas | 10 TON available — how many mints possible? |
| Mini app link | 🟡 NEEDED | Thomas | The @bot link to open the Telegram mini app |

**ACTION: Thomas reaches out to David for all 6 items. These unblock everything.**

---

## Execution Order

1. **Day 1:** Rook completes code sweep report (all files analyzed, issues flagged)
2. **Day 1:** Thomas gets indexer URL from David
3. **Day 2:** Rook runs API tests against indexer (test connectivity, validation, rate limiting)
4. **Day 2-3:** Thomas tests E2E on phone (wallet connect → mint → reveal flow)
5. **Day 3:** Consolidate findings, fix critical issues
6. **Day 4:** Load testing (if needed before launch)
7. **Day 5:** Final checklist before go-live

---

## Test Matrix (Full)

| ID | Category | Test Case | Method | Runner | Priority | Status |
|----|----------|-----------|--------|--------|----------|--------|
| C-01 | Contract | Mint logic access control | Code review | Rook | 🔴 | TODO |
| C-02 | Contract | Reveal only by owner | Code review | Rook | 🔴 | TODO |
| A-01 | API | All endpoints map + params | Code review | Rook | 🟠 | TODO |
| A-02 | API | Input validation on each endpoint | curl tests | Rook | 🟠 | TODO |
| A-03 | API | SQL injection resistance | curl + payloads | Rook | 🔴 | TODO |
| A-04 | API | Reveal endpoint auth check | curl | Rook | 🔴 | BLOCKED (need URL) |
| A-05 | API | Rate limiting enforced | curl hammer | Rook | 🟡 | BLOCKED |
| F-01 | Frontend | Mint button debounce | Code review | Rook | 🟠 | TODO |
| F-02 | Frontend | Optimistic update rollback | Code review | Rook | 🟠 | TODO |
| F-03 | Frontend | Error states visible | Code review | Rook | 🟡 | TODO |
| E-01 | E2E | Wallet connect works | Manual | Thomas | 🔴 | BLOCKED (need link) |
| E-02 | E2E | Mint flow successful | Manual | Thomas | 🔴 | BLOCKED |
| E-03 | E2E | Reveal flow successful | Manual | Thomas | 🔴 | BLOCKED |
| E-04 | E2E | Insufficient balance error | Manual | Thomas | 🟠 | BLOCKED |
| E-05 | E2E | Reject wallet approval | Manual | Thomas | 🟠 | BLOCKED |
| L-01 | Load | 100 concurrent mints | Load test script | Rook | 🟡 | BLOCKED (need stability first) |
| L-02 | Load | API response times under load | Load test | Rook | 🟡 | BLOCKED |

---

## Risk Mitigation

**Risk:** Reveal endpoint lacks auth → anyone can reveal anyone's NFT
- **Mitigation:** David confirms on-chain ownership check in contract/API
- **Fallback:** Manual code review of both contract + API before launch

**Risk:** Double-mint race condition if button not debounced
- **Mitigation:** Code review confirms debounce, manual rapid-click test on phone
- **Fallback:** Add debounce during code review if missing

**Risk:** High load during launch crashes backend
- **Mitigation:** Load test before launch, identify bottlenecks (DB, Redis, API)
- **Fallback:** Rate limit aggressively at launch, scale up gradually

**Risk:** Telegram-specific UX issues (dark mode, safe area, viewport)
- **Mitigation:** Thomas tests on real phone with multiple screen sizes
- **Fallback:** Deploy hotfixes quickly if UX issues found post-launch

---

## Success Criteria

✅ Code sweep report completed — all critical/high issues flagged
✅ All 🔴 CRITICAL issues resolved before launch
✅ All 🟠 HIGH issues resolved or documented as acceptable risk
✅ E2E flow verified on Thomas's phone (connect → mint → reveal)
✅ API responds <2s under normal load
✅ Zero SQL injection vulnerabilities
✅ Reveal endpoint confirmed to check ownership
✅ No silent failures — all error states visible to user