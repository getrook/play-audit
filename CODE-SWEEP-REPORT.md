# PLAY Project — Full Code Sweep Report

*Date: 2025-03-15*
*Auditor: Rook (via sub-agent)*

---

## Summary

**Total issues: 32**

| Priority | Count |
|----------|-------|
| 🔴 CRITICAL | 3 |
| 🟠 HIGH | 8 |
| 🟡 MEDIUM | 10 |
| 🔵 LOW | 6 |
| ⚪ INFO | 5 |

---

## 🔴 CRITICAL Issues

### C-1: OwnershipAssigned handler sets owner to PREVIOUS owner instead of new owner

**File:** `packages/indexer/src/sync/event-handler.ts:515`
**Lines:** 508–528

```ts
await this.nftRepo.updateOwner(nftAddress, toRawAddress(event.prevOwner));
```

**What:** The `handleOwnershipAssigned` handler is called when an NFT is transferred. Per TEP-62, the `OwnershipAssigned` message is sent to the **new owner** and contains `prev_owner` (the old owner). The handler calls `updateOwner(nftAddress, event.prevOwner)` — this sets the NFT's owner to the *previous* owner, not the new one.

**Impact:** Every NFT transfer recorded by the indexer will show the WRONG owner. The original minter will appear to own NFTs forever, regardless of transfers. Users who buy/receive NFTs won't see them in their profile. This is a **data integrity bug on mainnet** that affects all post-mint ownership tracking.

**Fix:** The new owner isn't directly in the `OwnershipAssigned` event payload — it's the *recipient* of the message. The handler needs to either:
1. Track the `Transfer` message's `new_owner` field (requires handling a different event), or
2. Look up the NFT's on-chain owner via TonAPI after seeing this event, or
3. Parse the transaction's destination account as the new owner from the SSE/polling context.

The current `TransactionContext.accountAddress` is the NFT item address (which emits the event), not the new owner. This needs architectural thought.

---

### C-2: `revealNft()` SDK method calls non-existent backend endpoint

**File:** `packages/sdk/src/client/playproject-client.ts:304–317`
**Also:** `packages/webapp/src/lib/playproject.tsx:245–270` (dead `useRevealUserNft` hook)

**What:** The SDK's `revealNft(address)` method sends `POST /api/nfts/${address}/reveal`. **This endpoint does not exist** in the backend routes (`packages/indexer/src/api/routes/nfts.ts`). The backend only has `GET /nfts/:address` and `GET /nfts/:address/history`.

**Impact:** If `useRevealUserNft` is ever called (it's currently dead code — the frontend uses `useUserRevealTransaction` for on-chain reveal instead), it would return a 404 or be caught by API key auth. However, the SDK is a public package — any third-party consumer calling `client.revealNft()` will get a silent failure.

The previous finding about "reveal endpoint may lack auth/ownership check" is **partially moot** — the endpoint doesn't exist at all. But this means:
1. The SDK's public API is broken for this method
2. The `useRevealUserNft` hook in the webapp is dead code that will fail if ever wired up

**Fix:** Either implement the endpoint with proper ownership verification, or remove `revealNft()` from the SDK and `useRevealUserNft` from the webapp.

---

### C-3: No rate limiting or anti-replay on mint transaction construction

**File:** `packages/webapp/src/hooks/useMintWithOptimisticUpdate.ts:48–65`
**Also:** `packages/webapp/src/components/mint/MintForm.tsx:185–196`

**What:** The mint button has no debounce, throttle, or `isPending` guard. The `MintForm` component passes `isMintDisabled` to the button, but this only checks balance/supply — it does NOT check if a mint transaction is already in-flight. A user can rapidly tap the mint button multiple times before the first TonConnect popup appears, potentially queuing multiple transactions.

**Impact:** Double-mint race condition. User could end up minting 2× the intended quantity, spending real TON on mainnet. The `useMutation` from react-query does prevent concurrent `mutateAsync` calls while one is pending, BUT the MintForm doesn't pass `isPending` to the button's disabled state.

**Fix:** Thread `isPending` from the `useMintWithOptimisticUpdate` mutation through to `MintForm` and add it to `isMintDisabled`. Also consider a local `isMinting` state flag or debounce.

---

## 🟠 HIGH Issues

### H-1: API key auth is optional — all endpoints publicly accessible when `API_KEY` unset

**File:** `packages/indexer/src/api/server.ts:124–137`

**What:** If `API_KEY` is not set in environment variables, the `onRequest` hook is never registered — all endpoints are completely open. The code only logs a warning: `'API_KEY not set — all endpoints are publicly accessible'`.

**Impact:** In development/staging this is fine. In production on mainnet, if the env var is accidentally omitted, any user data (wallet addresses, NFT ownership, mint history) is queryable without authentication. The user-facing routes (`/api/users/:address/nfts`, `/api/users/:address/activity`) expose wallet-linked activity.

**Fix:** In production mode (`NODE_ENV=production`), either require `API_KEY` or at minimum protect write endpoints and sensitive user data queries.

---

### H-2: Rate limiter uses spoofable `x-wallet-address` header as bucket key

**File:** `packages/indexer/src/api/server.ts:95–103`

**What:** The rate limiter's `keyGenerator` prefers `x-wallet-address` header over IP:
```ts
keyGenerator: (request) => {
  const wallet = request.headers['x-wallet-address'];
  if (typeof wallet === 'string' && wallet.length > 0) {
    return `wallet:${wallet}`;
  }
  return request.ip;
}
```

This header is client-supplied and trivially spoofable. An attacker can rotate wallet addresses to bypass rate limiting entirely, or set another user's wallet address to consume their rate limit quota.

**Impact:** Rate limiting is effectively bypassable. Under load (launch day), a single actor could overwhelm the API.

**Fix:** Use IP-based rate limiting as primary, and optionally use wallet address as a secondary bucket (with validation that the address is well-formed).

---

### H-3: Reveal polling timeout silently drops to idle with no user feedback

**File:** `packages/webapp/src/pages/NFTDetailPage.tsx:157–165`

**What:** After a user initiates reveal and sends the on-chain transaction, the frontend polls for 120 seconds. If the indexer hasn't picked up the revealed metadata by then:
```ts
revealPollTimeoutRef.current = window.setTimeout(() => {
  console.warn('[Reveal] Reveal polling timed out...');
  clearRevealPolling();
  setRevealStatus('idle');
  setRevealStage('idle');
  resolve();
}, REVEAL_POLL_TIMEOUT_MS);
```

The UI silently returns to idle state. The user paid gas for the reveal transaction but sees no confirmation and no error — they might think it failed and try again (wasting more gas).

**Impact:** Bad UX on mainnet. If the indexer is slow (which is likely under launch load), many users will hit this timeout. They'll either think the reveal failed or re-try, wasting gas.

**Fix:** Show a user-facing message like "Reveal is processing — please check back shortly" instead of silently returning to idle. Also consider extending the timeout or switching to a "check back later" pattern after the on-chain tx is confirmed.

---

### H-4: No balance pre-check before constructing mint transaction

**File:** `packages/webapp/src/pages/MintPage.tsx:78–115`
**File:** `packages/webapp/src/components/mint/MintForm.tsx:185–196`

**What:** The `MintForm` component shows a balance warning and disables the button when balance is insufficient. However, the `handleMint` function in `MintPage` doesn't re-check the balance at execution time — it trusts the stale balance from the last query. If the balance dropped between render and click (e.g., user sent TON from another tab), the transaction will be submitted and fail on-chain, costing gas.

**Impact:** Users lose gas fees on failed transactions. The price sanity check exists (line 91-95), but there's no balance sanity check at the mutation level.

**Fix:** Add a balance re-fetch or re-check inside `handleMint` before calling `mint()`.

---

### H-5: Hardcoded wallet limit check uses magic number 10 instead of `drop.maxPerWallet`

**File:** `packages/webapp/src/components/mint/MintForm.tsx:239–248`

**What:** The "Items per Wallet limit exceeded" check in the Balance Card uses a hardcoded `10`:
```tsx
{userMinted + selectedQuantity > 10 ? (
```

But the actual limit comes from `drop.maxPerWallet` (used correctly elsewhere in the component, e.g., `exceedsWalletLimit` on line 62). If the collection's `maxPerWallet` is changed to anything other than 10, this UI check will be wrong.

**Impact:** Incorrect validation message shown to users if `maxPerWallet` differs from 10. Could either block valid mints (if limit > 10) or allow invalid ones to proceed to on-chain failure (if limit < 10).

**Fix:** Replace `> 10` with `> drop.maxPerWallet`.

---

### H-6: Bounced mint handler decrements `next_item_index` but doesn't refund buyer

**File:** `packages/contracts/contracts/nft_collection.tact:548–558`

**What:** When an NFT deploy bounces, the contract decrements `next_item_index`:
```tact
bounced(msg: bounced<InitializeItem>) {
    if (self.next_item_index > 0) {
        self.next_item_index = self.next_item_index - 1;
    }
}
```

However, the buyer's TON is not refunded. The comment says "buyer's funds remain in the collection contract and the owner can withdraw and manually refund if needed." Additionally, `minted_per_wallet` is NOT decremented, so the buyer's per-wallet quota is permanently consumed.

**Impact:** On a batch mint of multiple NFTs where one deployment bounces, the buyer loses funds AND their per-wallet mint count is wrong. For a batch of 5 where #4 bounces, `next_item_index` is decremented by 1 but `minted_per_wallet` still shows 5. Manual owner intervention required for refunds.

**Fix:** At minimum, decrement `minted_per_wallet` in the bounce handler. Consider storing buyer address for automatic refund (though bounced messages have limited data).

---

### H-7: SSE client and collection poller have no backpressure or circuit breaker

**File:** `packages/indexer/src/sync/sse-client.ts` (entire file)
**File:** `packages/indexer/src/sync/collection-poller.ts` (entire file)

**What:** The SSE client reconnects with a simple delay on error but has no exponential backoff, jitter, or circuit breaker. If TonAPI is down, the client will hammer it with reconnection attempts. Similarly, the collection poller runs on a fixed interval with no backoff on repeated failures.

**Impact:** Under TonAPI outage or rate limiting, the indexer becomes a denial-of-service client. TonAPI rate limit violations could get the API key banned.

**Fix:** Implement exponential backoff with jitter on SSE reconnection. Add circuit breaker logic that pauses after N consecutive failures.

---

### H-8: `minted_per_wallet` map grows unboundedly in contract storage

**File:** `packages/contracts/contracts/nft_collection.tact:175`

**What:** `minted_per_wallet: map<Address, Int as uint32>` grows with every unique wallet that mints. On TON, contract storage costs real TON (storage fees). There's no mechanism to prune this map.

**Impact:** For popular collections with thousands of minters, storage costs will grow linearly. The contract's minimum balance requirement increases over time, potentially trapping funds. This is a well-known TON smart contract concern.

**Fix:** Consider using a separate "minting ticket" contract pattern per wallet (like Jetton wallets) instead of a monolithic map, or document the storage cost implications for collection creators.

---

## 🟡 MEDIUM Issues

### M-1: `text-1xl` CSS class doesn't exist in Tailwind

**File:** `packages/webapp/src/pages/NFTDetailPage.tsx:310`

```tsx
<h1 className="text-1xl bg-gray-100 px-3 py-0 pt-2 text-gray-500">
```

**What:** Tailwind CSS doesn't have a `text-1xl` utility class. Valid sizes go `text-xs`, `text-sm`, `text-base`, `text-lg`, `text-xl`, `text-2xl`, etc. `text-1xl` will be ignored, and the heading will use the browser's default `<h1>` size (typically very large and bold).

**Impact:** NFT name heading on the detail page likely renders with incorrect/oversized font size.

**Fix:** Replace with `text-xl` or `text-lg`.

---

### M-2: Telegram back button doesn't work outside Telegram webview

**File:** `packages/webapp/src/hooks/useTelegramBackButton.ts`

**What:** The `useTelegramBackButton` hook relies on `window.Telegram?.WebApp?.BackButton`. Outside the Telegram Mini App context (e.g., testing in a regular browser), this API doesn't exist. The hook does handle this gracefully (no-op), but the NFTDetailPage's manual back button (`handleBack` → `navigate(-1)`) is the only fallback.

**Impact:** Standard browser back navigation works, but the UI shows both a manual back button AND a Telegram back button in-app, or no native back in browser. This is a known issue from the task brief.

---

### M-3: `useWalletGuard` redirects aren't protected against race conditions

**File:** `packages/webapp/src/hooks/useWalletGuard.ts`

**What:** If the TonConnect wallet is slow to reconnect on page load, `useWalletGuard` may trigger a redirect to the home page before the wallet state is restored. This could bounce authenticated users back to the welcome page on every MintPage navigation.

**Impact:** Transient UX issue — users may see a flash redirect before wallet reconnects.

**Fix:** Add a "wallet connecting" loading state before triggering redirects.

---

### M-4: MoonPay buy URL has no parameters for chain/amount

**File:** `packages/webapp/src/components/mint/MintForm.tsx:277`

```tsx
const url = `https://buy.moonpay.com/v2/buy`;
window.open(url, '_blank');
```

**What:** The MoonPay URL opens with no query parameters — no currency code, no amount, no wallet address. Users land on a generic buy page and have to manually select TON, enter their address, etc.

**Impact:** High friction for users who need to top up. Many will abandon the flow.

**Fix:** Add `?currencyCode=ton_ton&walletAddress=${walletAddress}&baseCurrencyAmount=${tonNeeded}` to the URL.

---

### M-5: `useRevealUserNft` hook is dead code

**File:** `packages/webapp/src/lib/playproject.tsx:245–270`

**What:** `useRevealUserNft` calls `client.revealNft()` which hits a non-existent endpoint (see C-2). The actual reveal flow uses `useUserRevealTransaction`. This hook is dead code that will confuse future developers.

**Impact:** Code maintenance burden. If someone wires it up thinking it's the correct reveal hook, reveals will break silently.

**Fix:** Remove `useRevealUserNft` and `revealNft()` from the SDK (or implement the backend endpoint).

---

### M-6: Metadata fetch during `ContentConfigured` has no concurrency limit

**File:** `packages/indexer/src/sync/event-handler.ts:352–397`

**What:** When `ContentConfigured` fires and the collection is revealed, the handler fetches metadata for ALL existing NFTs that don't have it. It processes in pages of 100, but within each page, it fetches metadata sequentially. For large collections, this could take a very long time. There's no concurrency limit, timeout per fetch, or overall timeout.

**Impact:** Processing a single `ContentConfigured` event could block the event handler for minutes on a collection with thousands of NFTs. Other events queue behind it.

**Fix:** Add `Promise.allSettled` with concurrency limit (e.g., 10) per batch, and add an overall timeout.

---

### M-7: No input sanitization on collection/NFT address route parameters

**File:** `packages/indexer/src/api/routes/collections.ts`, `nfts.ts`, `users.ts`

**What:** Route handlers use `tryParseAddress()` for validation on some endpoints (e.g., `GET /nfts/:address`), but others like the collections routes may pass raw user input to database queries. The `tryParseAddress` utility normalizes to raw form, which is good, but not all routes consistently apply it.

**Impact:** Potential for malformed addresses to cause unexpected database behavior (though Drizzle's parameterized queries prevent SQL injection).

**Fix:** Ensure all `:address` route parameters consistently pass through `tryParseAddress()` before any database operation.

---

### M-8: Optimistic NFT store never cleans up stale entries

**File:** `packages/webapp/src/stores/optimisticNfts.ts`

**What:** `useOptimisticNfts` adds optimistic NFTs after mint but `removeByCollection` and `removeByAddress` require explicit calls. If the polling/indexing watcher fails or times out (see H-3), optimistic NFTs remain in the Zustand store indefinitely, showing phantom NFTs in the UI.

**Impact:** Users may see "ghost" NFTs that don't actually exist on-chain. On page refresh the store is cleared (it's not persisted), but within a session, stale entries accumulate.

**Fix:** Add a TTL-based auto-cleanup (e.g., remove optimistic entries older than 5 minutes) or tie cleanup to the mint polling timeout.

---

### M-9: `SECURITY.md` references responsible disclosure but provides no contact

**File:** `SECURITY.md`

**What:** The security policy file exists but if it doesn't provide a clear security contact email or PGP key, white-hat researchers have no way to report vulnerabilities privately.

**Impact:** Potential responsible disclosure findings go unreported or get posted publicly.

**Fix:** Add a security contact email and ideally a PGP key.

---

### M-10: No health check for Redis connection in health endpoint

**File:** `packages/indexer/src/api/routes/health.ts`

**What:** The health/readiness check verifies database connectivity but does not check Redis connectivity. If Redis is down, the cache layer fails silently (cache misses are treated as no-ops), but this degrades performance significantly under load.

**Impact:** Orchestration tools (Railway, Docker health checks) may report the service as healthy when Redis is down, leading to degraded performance without alerting.

**Fix:** Add Redis ping check to the readiness endpoint.

---

## 🔵 LOW Issues

### L-1: `SdkTestPage` exists in production build

**File:** `packages/webapp/src/pages/SdkTestPage.tsx`
**File:** `packages/webapp/src/App.tsx` (route registration)

**What:** The SDK test page is registered as a route in the app. It exposes internal SDK methods, collection addresses, and debugging information.

**Impact:** Low security risk (it only reads data), but it exposes internal tooling to end users who discover the route.

**Fix:** Gate behind a `NODE_ENV !== 'production'` check or remove from production routing.

---

### L-2: `from_nano` precision in `MintForm` gas fee display

**File:** `packages/webapp/src/components/mint/MintForm.tsx:10`

```ts
const MINT_GAS_FEE = Number(fromNano(PlayProjectClient.mintGas));
```

**What:** Converting nanoTON to a JavaScript `Number` via `fromNano` then to `Number` can lose precision for very small or very large values due to floating-point representation. For typical gas fees (~0.15 TON) this is fine, but the pattern is fragile.

**Impact:** Negligible for current values. Could cause display rounding errors for unusual fee amounts.

---

### L-3: Collection cover image has fixed 3:4 aspect ratio

**File:** `packages/webapp/src/components/mint/MintForm.tsx:101`

```tsx
className="w-full aspect-[3/4] object-cover mb-3 mt-6 rounded-3xl"
```

**What:** All collection cover images are forced into 3:4 aspect ratio with `object-cover`. If a collection has a square or landscape cover, it will be cropped.

**Impact:** Visual cropping for non-standard cover images.

---

### L-4: Multiple TON logo references without fallback

**Files:** `packages/webapp/src/components/mint/MintForm.tsx:192`

```tsx
<img src="/ton-logo.png" alt="TON" className="w-6 h-6 object-contain" />
```

**What:** References a static `/ton-logo.png` asset. If this file is missing or fails to load, only the alt text "TON" shows.

**Impact:** Minor visual glitch if asset is missing.

---

### L-5: Console.error/warn statements in production code

**Files:** Multiple files throughout `packages/webapp/src/`

**What:** `console.error('Mint failed:', error)`, `console.warn('[Reveal] polling timed out...')` and similar statements are scattered through production code. These are visible in the browser console to any user.

**Impact:** Information leakage in browser console. Not a security issue but unprofessional.

**Fix:** Use a proper logger that can be silenced in production, or strip console statements in the build.

---

### L-6: `HowToPlayPage` content quality

**File:** `packages/webapp/src/pages/HowToPlayPage.tsx`

**What:** The "How To Play" page contains hardcoded instructional content. If the game mechanics change (e.g., reveal flow, pricing), this page will become stale.

**Impact:** User confusion if game mechanics evolve.

**Fix:** Consider making this content CMS-driven or at least document where to update it.

---

## ⚪ INFO / Architecture Notes

### I-1: Smart contract architecture is solid

The Tact contracts follow TEP-62 (NFT Collection/Item) and TEP-66 (Royalties) standards correctly. Notable security positives:
- Two-step ownership transfer (`ChangeOwner` → `AcceptOwnership`) prevents accidental loss
- Bounce handler on `InitializeItem` recovers `next_item_index` on failed deploys
- Bounce handler on `OwnershipAssigned` in NftItem reverts ownership on failed transfer notification
- Per-wallet mint limits enforced on-chain
- `requireOwner()` guard on all admin functions
- Minimum balance check on withdrawals (`0.1 TON` reserve)
- Reveal requires base_content to be configured first
- NFT item validates that only the collection can initialize/update content
- RequestRevealContent validates sender against expected NFT address

---

### I-2: Event-driven indexer architecture is well-designed

The indexer uses an event-sourcing pattern with idempotent event handling (duplicate detection via `eventRepo.exists()`), paginated metadata fetching (100/page), and proper cache invalidation. The `TransactionContext` pattern cleanly separates blockchain metadata from event data.

---

### I-3: Specs directory indicates mature engineering process

13 specification documents covering production dependencies, service modes, CORS, API auth, DB indexes, Redis caching, rate limiting, SSE recovery, query optimization, webapp tuning, multi-frontend support, Railway deployment, and monitoring. This is above-average for a project at this stage.

---

### I-4: Frontend price sanity check is a good defense-in-depth pattern

Both `MintPage.tsx` (line 91-95) and `useMintWithOptimisticUpdate.ts` (line 52-55) independently verify that mint price doesn't exceed 100 TON. This protects against a compromised indexer feeding inflated prices.

---

### I-5: Open GitHub issues alignment

The 4 open GitHub issues (#55 design tokens, #57 BTC icon, #58 logo position, #59 wallet logo oversized) are all cosmetic/UI issues. The critical and high-priority issues found in this sweep are NOT covered by existing issues and should be filed.

---

## Files Reviewed

### Smart Contracts
- [x] `packages/contracts/contracts/nft_collection.tact` (654 lines)
- [x] `packages/contracts/contracts/nft_item.tact` (~280 lines)
- [x] `packages/contracts/scripts/deploy.ts`
- [x] `packages/contracts/scripts/enable-mint.ts`
- [x] `packages/contracts/scripts/enable-reveal.ts`
- [x] `packages/contracts/scripts/mint.ts`
- [x] `packages/contracts/scripts/config.ts`

### Backend/Indexer
- [x] `packages/indexer/src/api/server.ts`
- [x] `packages/indexer/src/api/routes/collections.ts`
- [x] `packages/indexer/src/api/routes/nfts.ts`
- [x] `packages/indexer/src/api/routes/users.ts`
- [x] `packages/indexer/src/api/routes/events.ts`
- [x] `packages/indexer/src/api/routes/health.ts`
- [x] `packages/indexer/src/db/schema.ts`
- [x] `packages/indexer/src/db/client.ts`
- [x] `packages/indexer/src/db/collections.ts`
- [x] `packages/indexer/src/db/nfts.ts`
- [x] `packages/indexer/src/db/events.ts`
- [x] `packages/indexer/src/config.ts`
- [x] `packages/indexer/src/env-validator.ts`
- [x] `packages/indexer/src/cache/redis.ts`
- [x] `packages/indexer/src/utils/cache.ts`
- [x] `packages/indexer/src/sync/collection-loader.ts`
- [x] `packages/indexer/src/sync/collection-poller.ts`
- [x] `packages/indexer/src/sync/event-handler.ts`
- [x] `packages/indexer/src/sync/metadata-fetcher.ts`
- [x] `packages/indexer/src/sync/sse-client.ts`
- [x] `packages/indexer/src/utils/address.ts`
- [x] `packages/indexer/src/utils/ipfs.ts`
- [x] `packages/indexer/src/utils/nft-address.ts`
- [x] `packages/indexer/src/utils/retry.ts`

### SDK
- [x] `packages/sdk/src/client/playproject-client.ts`
- [x] `packages/sdk/src/client/messages.ts`
- [x] `packages/sdk/src/client/types.ts`
- [x] `packages/sdk/src/events.ts`
- [x] `packages/sdk/src/tonapi-client.ts`

### Frontend/Webapp
- [x] `packages/webapp/src/App.tsx`
- [x] `packages/webapp/src/pages/WelcomePage.tsx`
- [x] `packages/webapp/src/pages/HomePage.tsx`
- [x] `packages/webapp/src/pages/MintPage.tsx`
- [x] `packages/webapp/src/pages/CollectionPage.tsx`
- [x] `packages/webapp/src/pages/NFTDetailPage.tsx`
- [x] `packages/webapp/src/pages/ProfilePage.tsx`
- [x] `packages/webapp/src/pages/SdkTestPage.tsx`
- [x] `packages/webapp/src/pages/HowToPlayPage.tsx`
- [x] `packages/webapp/src/hooks/useMintWithOptimisticUpdate.ts`
- [x] `packages/webapp/src/hooks/useMintPolling.ts`
- [x] `packages/webapp/src/hooks/useMergeOptimisticNfts.ts`
- [x] `packages/webapp/src/hooks/useWalletBalance.ts`
- [x] `packages/webapp/src/hooks/useWalletGuard.ts`
- [x] `packages/webapp/src/hooks/useTonBalance.ts`
- [x] `packages/webapp/src/hooks/useUserNftIndexingWatcher.ts`
- [x] `packages/webapp/src/hooks/usePullToRefresh.ts`
- [x] `packages/webapp/src/hooks/useTelegramTheme.ts`
- [x] `packages/webapp/src/hooks/useTelegramBackButton.ts`
- [x] `packages/webapp/src/hooks/useTelegramHaptic.ts`
- [x] `packages/webapp/src/components/mint/MintForm.tsx`
- [x] `packages/webapp/src/components/mint/MintSuccessScreen.tsx`
- [x] `packages/webapp/src/components/mint/MintErrorScreen.tsx`
- [x] `packages/webapp/src/components/layout/Layout.tsx`
- [x] `packages/webapp/src/components/layout/BottomNav.tsx`
- [x] `packages/webapp/src/components/layout/WalletMenu.tsx`
- [x] `packages/webapp/src/components/home/LiveDropCard.tsx`
- [x] `packages/webapp/src/components/ui/RevealingLoadingScreen.tsx`
- [x] `packages/webapp/src/components/ui/MintingLoadingScreen.tsx`
- [x] `packages/webapp/src/components/ErrorBoundary.tsx`
- [x] `packages/webapp/src/stores/optimisticNfts.ts`
- [x] `packages/webapp/src/lib/playproject.tsx`

### Config & Infra
- [x] `docker-compose.yml`
- [x] `packages/indexer/drizzle/0000_worried_scorpion.sql`
- [x] `packages/indexer/drizzle/0001_remove_registry_columns.sql`
- [x] `packages/indexer/drizzle/0002_add_is_revealed_by_user.sql`
- [x] `packages/indexer/drizzle/0003_add_starts_at_to_collections.sql`
- [x] `SECURITY.md`
- [x] `specs/01-dockerfile-prod-deps.md` through `specs/13-monitoring-observability.md`
- [x] `specs/README.md`

---

*End of report. 32 issues found across 70+ files. Priority action: Fix C-1 (wrong owner on transfer) before any transfers occur on mainnet.*
