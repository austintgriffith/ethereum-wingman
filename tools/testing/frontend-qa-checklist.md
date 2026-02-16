# Frontend QA Checklist — Three-Phase Browser Testing

A build is NOT done when the code compiles. A build is done when you've tested it like a real user in the browser. If you have browser access, a wallet, and funds — **use them.**

---

## Phase 1: Localhost + Local Chain + Burner Wallet

**Environment:** `yarn fork` + `yarn start` + SE2 burner wallet
**Cost of bugs:** Free

### Browser Test Protocol

```
1. Open http://localhost:3000 — take a snapshot, verify it loaded
2. Check page title in browser tab — is it correct, not "Scaffold-ETH 2"?
3. Click the faucet — fund the burner wallet
4. Navigate to every page/route — do they all render?
5. Click every button — does each one do something?
6. Do the full user flow start to finish:
   - Fill inputs → click action → verify state change
   - Check that data displays correctly after transactions
7. Test edge cases:
   - Zero amounts → proper validation?
   - Very large numbers → handled?
   - Empty inputs → error message?
8. Open browser console — any errors?
9. Verify events fire and UI updates in response
```

### Superpowers Available

```bash
# Impersonate any address
cast rpc anvil_impersonateAccount 0x...

# Fast-forward time (86400 = 1 day)
cast rpc evm_increaseTime 86400 && cast rpc evm_mine

# Infinite ETH via faucet (in header)

# Transfer whale tokens
cast send <TOKEN> "transfer(address,uint256)" <BURNER> <AMOUNT> --from <WHALE> --unlocked

# Snapshot and revert chain state
cast rpc evm_snapshot  # returns snapshot ID
cast rpc evm_revert <snapshot_id>
```

### Exit Gate
All items must pass before moving to Phase 2:
- [ ] App loads without errors
- [ ] Every page renders
- [ ] Every button works (no dead buttons)
- [ ] Full user flow end-to-end
- [ ] `forge test` passes
- [ ] Edge cases handled
- [ ] No console errors

---

## Phase 2: Localhost + Live L2 + MetaMask

**Environment:** `yarn start` + contract deployed on Base/L2 + MetaMask wallet
**Cost of bugs:** Real gas, real tokens

### What Changes from Phase 1

- MetaMask is in the loop (wallet connect, tx signing, popups)
- Transactions take 2-3 seconds (loaders are CRITICAL)
- Real RPC (rate limits, polling matters)
- Users can be on wrong network
- Tx failures cost real money

### Browser Test Protocol

```
1. Open http://localhost:3000
2. Connect MetaMask — verify wallet address shows correctly
3. WRONG NETWORK TEST:
   → Switch MetaMask to a wrong network (e.g., Ethereum mainnet)
   → Does "Switch to Base" (or target chain) button appear?
   → Click it — does it switch correctly?
4. APPROVE FLOW TEST (if app uses token approvals):
   → With insufficient allowance, does "Approve" button show?
   → Click Approve:
     - Does button disable immediately?
     - Does it show "Approving..." text?
     - Does MetaMask popup appear?
   → Sign the tx
   → Wait for confirmation:
     - Does the approve button stay disabled while pending?
     - When confirmed, does the action button ("Stake", "Deposit") appear?
   → Click action button — same checks (disable, loader, confirm)
5. DOUBLE-CLICK PREVENTION:
   → Click any onchain button — can you click it again immediately?
   → It MUST be disabled after first click
6. REJECT TEST:
   → Start a transaction, reject it in MetaMask
   → Does the button re-enable?
   → Does an error message show?
7. TX REVERT TEST:
   → Trigger a transaction that will revert (e.g., insufficient balance)
   → Does the UI show a meaningful error?
8. RPC HEALTH CHECK:
   → Open browser DevTools → Network tab
   → Watch for 3-5 seconds
   → Polling should be ~1 request every 3 seconds
   → If 15+ requests/second → something is wrong
   → Any 429 (rate limit) errors? → RPC config is bad
9. ADDRESS DISPLAY:
   → Find every place an address is shown
   → All should show ENS/blockies, not raw hex (0x1234...)
10. FULL USER FLOW:
    → Complete the entire app flow with real tokens/ETH
    → Verify final state is correct on-chain
```

### Mobile Wallet Deep Linking

On mobile, when a user connects via Rainbow, Phantom, Coinbase Wallet, or any non-MetaMask wallet, clicking a transaction button must open **their connected wallet** — not MetaMask. SE2's default `wagmiConnectors.tsx` includes `phantomWallet` but the app has no mobile deep linking out of the box.

**If your app calls `openWallet()` or does any manual deep linking on mobile, it MUST detect the connected wallet:**

```typescript
import { useAccount } from "wagmi";

const { connector } = useAccount();

const openWallet = useCallback(() => {
  if (typeof window === "undefined") return;
  const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);
  if (!isMobile) return;
  // Already inside a wallet's in-app browser
  if (window.innerWidth < 500 && window.ethereum) return;

  const currentUrl = window.location.href;
  const strippedUrl = currentUrl.replace(/^https?:\/\//, "");
  const id = (connector?.id || "").toLowerCase();
  const name = (connector?.name || "").toLowerCase();

  if (name.includes("rainbow") || id.includes("rainbow")) {
    window.location.href = `rainbow://dapp/${strippedUrl}`;
  } else if (name.includes("phantom") || id.includes("phantom")) {
    window.location.href = `https://phantom.app/ul/browse/${encodeURIComponent(currentUrl)}?ref=${encodeURIComponent(currentUrl)}`;
  } else if (name.includes("coinbase") || id.includes("coinbase")) {
    window.location.href = `cbwallet://dapp/${strippedUrl}`;
  } else if (name.includes("trust") || id.includes("trust")) {
    window.location.href = `trust://open_url?coin_id=60&url=${encodeURIComponent(currentUrl)}`;
  } else if (name.includes("metamask") || id.includes("metamask")) {
    window.location.href = `metamask://dapp/${strippedUrl}`;
  }
  // WalletConnect and others — SDK handles the redirect
}, [connector]);
```

**Common mistake:** Hardcoding `metamask://dapp/...` for all wallets. This hijacks users into MetaMask even when they connected with Rainbow or Phantom.

**Verify `wagmiConnectors.tsx` includes Phantom:**
```typescript
import { phantomWallet } from "@rainbow-me/rainbowkit/wallets";
// ... include phantomWallet in the wallets array
```

### Mobile Wallet Test Protocol
```
1. Open production URL on phone
2. Connect via Rainbow (or Phantom, Coinbase Wallet)
3. Click any transaction button (approve, action, etc.)
4. Verify it opens THE SAME WALLET you connected with
5. ❌ If it opens MetaMask instead → deep linking is broken
```

### Exit Gate
All items must pass before moving to Phase 3:
- [ ] Wallet connects cleanly
- [ ] Wrong network detection + switch works
- [ ] Every onchain button has its OWN loading state
- [ ] Every onchain button disables on click (no double-click)
- [ ] Approve flow: network → approve → action (three buttons, one at a time)
- [ ] Allowance reads via hook (auto-updates)
- [ ] Reject in wallet → UI recovers
- [ ] Tx revert → error message shown
- [ ] RPC polling is sensible (~1 req/3sec)
- [ ] All addresses use `<Address/>`
- [ ] Real transaction end-to-end works
- [ ] No console errors
- [ ] Mobile: transaction buttons open the connected wallet (not hardcoded MetaMask)
- [ ] `wagmiConnectors.tsx` includes `phantomWallet`

---

## Phase 3: Live Frontend + Live Chain + Browser Wallets

**Environment:** Vercel/IPFS + Base mainnet + any wallet
**Cost of bugs:** Highest — broken deploys, confused users, wasted builds, rate limits

### What Changes from Phase 2

- Frontend is public (anyone can see it)
- URL sharing matters (OG tags, Twitter cards)
- No more localhost (env vars, hardcoded values will bite)
- CDN/hosting rate limits (too many redeploys = trouble)

### Browser Test Protocol

```
1. Open the PRODUCTION URL (not localhost)
2. Verify page loads, title correct, favicon correct
3. Run all Phase 2 checks on the live URL
4. URL UNFURL TEST:
   → Copy the URL
   → Paste into Twitter/Telegram/Discord/Slack
   → Does an image preview show?
   → Is the title correct?
   → Is the description correct?
   → Is the image the right one (not SE2 default)?
5. INCOGNITO TEST:
   → Open production URL in incognito/private window
   → Does it work without cached state?
6. CONSOLE CHECK:
   → Open browser console on production URL
   → Any errors? Any warnings about missing env vars?
7. NO LOCALHOST ARTIFACTS:
   → Search page source for "localhost", "127.0.0.1", "31337"
   → None should be present
```

### Exit Gate
All items must pass before sharing publicly:
- [ ] All Phase 2 criteria pass on live URL
- [ ] OG image loads (correct image, title, description)
- [ ] Twitter card type is `summary_large_image`
- [ ] No localhost/testnet artifacts
- [ ] Works in incognito
- [ ] No console errors
- [ ] Favicon and page title are correct
- [ ] Footer links to actual repo
- [ ] README describes the project

---

## Automated QA Script

Run `tools/qa-check.sh` against any SE2 project to catch code-level issues:

```bash
./tools/qa-check.sh /path/to/project
```

Checks for: missing RPC overrides, bad OG meta, raw addresses, missing button loading states, default SE2 text, hardcoded localhost values.

**Use before Phase 3.** Catches syntax-level issues. Browser testing catches UX issues.

---

## The Golden Rule

| Phase | Cost of a bug | Fix time |
|-------|--------------|----------|
| 1 — Local/Local | Free | Seconds |
| 2 — Local/Live chain | Gas money | Minutes |
| 3 — Production | Broken deploys, angry users | 10+ minutes |

**Every bug found in Phase 3 means Phase 1 or 2 testing failed.**
