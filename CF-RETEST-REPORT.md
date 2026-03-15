# PLAY CF Pages Re-Test — 2026-03-16 01:20 GMT+2

## Test URL
- **Old:** https://stickerpad-ton-nft-webapp.vercel.app (Vercel)
- **New:** https://play-project-8mk.pages.dev (Cloudflare Pages)

## Test Execution

### Step 1: Welcome Page ✅
- **Status:** Clean, minimal design
- **Elements visible:** PLAY logo, "Collect unique Sticker packs" subtitle, Connect Wallet button
- **Design notes:** White background, centered layout, blue CTA

### Step 2: Wallet Auto-Connection ✅
- **Status:** Wallet `0:a0518e7b...NWZ0` auto-connected (carried over from Vercel session via localStorage)
- **Balance shown:** 0.00 TON (note: This is a display issue — actual balance is 5.9438 TON per on-chain audit)
- **Issue:** Balance sync broken between old and new deploy

### Step 3: Home Page Collections ✅
- **Status:** Collections loaded successfully
- **Collections visible:**
  - Bear Bros V2 (0.01 TON, 5/18 minted, status: LIVE)
  - Reveal Test V2 (0.10 TON, 3/20 minted, status: LIVE)
- **Navigation:** Home, Collection, Profile tabs functional
- **Design:** Cards display with image, name, price, minting progress, Get Now button

### Step 4: Mint Modal ✅
- **Status:** Modal opened correctly on "Get Now" click
- **Modal shows:**
  - Quantity selector (0-5)
  - Price breakdown: Item price, transaction fee (0.15 TON), total
  - Balance check: "Your Balance: 0.00 TON" (incorrect)
  - "Insufficient balance" warning
  - Quantity validation: "Items per Wallet below limit ✓"
- **Fee accuracy:** Quoted 0.15 TON transaction fee (matches on-chain `USER_MINT_GAS`)

### Step 5: Collection Page ⚠️
- **Status:** Page loads but shows "0 Owned Stickers"
- **Issue:** Wallet state not syncing between Vercel → CF Pages transition
- **Root cause:** TonConnect session is ephemeral; switching URLs breaks wallet connection context

## Key Findings

| Finding | Severity | Status | Notes |
|---------|----------|--------|-------|
| Welcome page loads | ✅ | PASS | Clean design |
| Collections render | ✅ | PASS | Bear Bros + Reveal Test V2 visible |
| Mint modal functional | ✅ | PASS | Modal opens, shows correct fees |
| Balance display broken | 🔴 | FAIL | Shows 0.00 TON instead of 5.9438 TON |
| Collection page empty | 🟡 | FAIL | "0 Owned Stickers" despite 21 NFTs on-chain |
| TonConnect persistence | 🟡 | ISSUE | Wallet disconnects on URL change |

## What's Different From Vercel

**Vercel (`stickerpad-ton-nft-webapp.vercel.app`):**
- Directly linked to NFT in URL (e.g., `/stickers/0:ba010fc0...`)
- Wallet session persisted better

**CF Pages (`play-project-8mk.pages.dev`):**
- Clean URLs, new routing structure
- Wallet state not carrying over from page transitions
- Balance sync issue suggests API endpoint mismatch

## Hypothesis: API Endpoint Configuration

The CF Pages deploy may be pointing to a different API origin or endpoint structure. Possible causes:

1. **API Base URL:** Vercel vs CF Pages may have different API_BASE_URL env var
   - Old: `https://stickerpad-api.vercel.app` (hypothetical)
   - New: `https://play-api.pages.dev` or `https://api.play-project.com`
2. **Wallet Data:** Collections load (so API is reachable) but user NFT query fails
3. **LocalStorage:** Wallet connection info carries over but NFT ownership query doesn't

## Recommendation

1. **Verify API endpoint in CF Pages deploy:** 
   - Check `vite.config.js` or `.env` for API_BASE_URL
   - Confirm it matches the current API backend URL

2. **Check Network tab for failed requests:**
   - GET `/api/nfts?owner=UQCg...` should return list of owned NFTs
   - If 404 or 500, API endpoint is misconfigured

3. **Test fresh wallet connection on CF Pages:**
   - Clear localStorage
   - Reconnect wallet
   - Verify balance loads correctly

4. **Verify indexer API sync:**
   - David's PR #92 fixed indexer sync
   - Confirm new stickers from our mint are indexing properly

## Next Steps

1. **For David:** Confirm CF Pages API_BASE_URL configuration
2. **For Rook:** Test with fresh wallet connection after API verification
3. **Monitor indexer:** Verify the newly minted stickers appear in next sync cycle

---

*Test completed: 2026-03-16 01:25 GMT+2*
