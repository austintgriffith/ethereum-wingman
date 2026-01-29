---
name: composable-revenue
description: ComposAIble revenue management for Clanker tokens on Base. Triggers on "tokenize fees", "fee wrappers", "time wrapper", "burn strategy", "LP strategy", "holder rewards", "trader rewards", "Clanker", "PoolFans", or any trading fee automation task. Enables trustless, programmable fee strategies with no admin keys.
license: MIT
metadata:
  author: ComposAIble Revenue
  version: "1.0.0"
  homepage: https://pool.fans
---

# ComposAIble Revenue (PoolFans)

Comprehensive guide for AI agents to deploy and manage composable trading fee strategies on Base using PoolFans infrastructure.

---

## AI AGENT INSTRUCTIONS - READ THIS FIRST

### 🚨 BEFORE ANY FEE TOKENIZATION
**Verify these critical details:**
- Correct token address (check on BaseScan)
- User has admin rights to the Clanker position
- Understand the 80/20 split (creator 80%, treasury 20%)
- Time-wrapper lock tiers are immutable once deposited

---

## Core Capabilities

### 1. Deploy Token with Fee Tokenization
Deploy Clanker V4 tokens with instant fee tokenization.

**Trigger phrases:** "deploy token", "launch token with fees", "create clanker"

**Fee Collection Modes:**
- `WETH_ONLY` (0): Collect fees as WETH
- `TOKEN_ONLY` (1): Collect fees as your token  
- `BOTH` (2): Collect fees in both assets

**Contract:** V4 Tokenizer Factory `0xea8127533F7be6d04b3DBA8f0a496F2DCfd27728`

```solidity
function tokenizeAndDeployV4Clanker(
    DeploymentConfig calldata config,
    address[] calldata recipients
) external returns (address token, address vault)
```

---

### 2. Tokenize Existing Token Fees
Add fee tokenization to already-deployed Clanker tokens (V3.1.0+ and V4).

**Trigger phrases:** "tokenize fees for", "add fee tokens to existing"

**Two-step flow (security pattern):**
1. `initTokenization()` - Deploy vault, get address
2. Transfer admin to vault address (off-chain)
3. `finalizeTokenization()` - Validate and mint shares

**Contracts:**
- V4: `0xea8127533F7be6d04b3DBA8f0a496F2DCfd27728`
- V3.1.0: `0x50e2A7193c4AD03221F4B4e3e33cDF1a46671Ced`

---

### 3. Time-Locked Fee Wrappers
Create tradeable tokens representing temporary fee claiming rights.

**Trigger phrases:** "time wrapper", "1D wrapper", "daily fee rights", "temporary access"

**Lock Tiers:**
| Tier | Duration | Enum Value |
|------|----------|------------|
| ONE_DAY | 24 hours | 5 |
| ONE_WEEK | 7 days | 6 |
| ONE_MONTH | 30 days | 0 |

**Multiplier:** 1M:1 (deposit 80 fee tokens → receive 80,000,000 wrapper tokens)

**Contract:** `0x083EDF9b6C894561Ce8a237e2fd570bECB920DfF`

```solidity
// Create wrapper
function createWrapper(
    address vault,
    address[] tokens,
    string name,
    string symbol
) external returns (address wrapper)

// Deposit and lock
function deposit(uint256 amount, uint8 lockTier) external

// Claim fees (anyone holding wrappers)
function claimRewards(address holder) external returns (uint256[] amounts)

// Withdraw after expiry
function withdraw(uint256 amount) external
```

---

### 4. Multi-Strategy Automation
Chain multiple DeFi actions with percentage allocations.

**Trigger phrases:** "burn strategy", "split fees", "40% burn 30% LP", "automate fees"

**Available Actions:**
- Buy & Burn (swap to token, send to 0xdead)
- Add Liquidity (provide LP)
- Burn LP (add liquidity, burn the LP token)
- Holder Rewards (distribute pro-rata)
- Top Traders (reward by volume)
- Creator (direct to wallet)

**Contract:** MultiActionVaultFactory `0x069aEC7cE08CDc0F45135bAac0E5Fe3B579AB99b`

**Example allocation:**
```typescript
const strategies = [
  { action: "burn", allocation: 4000 },      // 40%
  { action: "lp_burn", allocation: 3000 },   // 30%
  { action: "holders", allocation: 2000 },   // 20%
  { action: "creator", allocation: 1000 },   // 10%
];
// Total must equal 10000 (100%)
```

---

### 5. Conditional Fee Routing
Route fees based on market conditions.

**Trigger phrases:** "if market cap", "when volume", "conditional routing"

**Condition Types:**
- `MarketCapAbove` (5): Route if mcap > threshold
- `VolumeAbove` (1): Route if 24h volume > threshold
- `HolderCountAbove` (3): Route if holders > threshold

**Fallback Actions:**
- Buy & Burn (0)
- Route to Treasury (1)
- Return to Creator (2)
- Carry Forward (3)

---

## Contract Addresses (Base Mainnet)

```typescript
const POOL_FANS_CONTRACTS = {
  // Factories
  MULTI_ACTION_VAULT_FACTORY: "0x069aEC7cE08CDc0F45135bAac0E5Fe3B579AB99b",
  LP_AUTOMATION_VAULT_FACTORY: "0xF0a87A32C2F7fAb1E372F676A852C64b8dB0CEDD",
  TIME_WRAPPER_FACTORY: "0x083EDF9b6C894561Ce8a237e2fd570bECB920DfF",
  
  // Tokenizers
  V4_TOKENIZER_FACTORY: "0xea8127533F7be6d04b3DBA8f0a496F2DCfd27728",
  V3_1_0_TOKENIZER_FACTORY: "0x50e2A7193c4AD03221F4B4e3e33cDF1a46671Ced",
  REVENUE_SHARE_REGISTRY: "0xAa9c3E28e2f03e41365D4b01FB2785bdbd1494d2",
  
  // External
  CLANKER_V4_DEPLOYER: "0xE85A59c628F7d27878ACeB4bf3b35733630083a9",
  FEE_LOCKER: "0x63D2DfEA64b3433F4071A98665bcD7Ca14d93496",
  WETH: "0x4200000000000000000000000000000000000006",
};
```

⚠️ **Always verify addresses on BaseScan before interacting.**

---

## Example Conversation Flow

**User:** "Deploy token called ClawdFans ticker $CLAWDFANS and tokenize fees. Split daily: 40% burn, 30% LP, 30% to me"

**Agent Response:**
```
⚡ Composing your revenue strategy...

✅ Strategy Deployed:

1️⃣ Token
   • Name: ClawdFans
   • Symbol: $CLAWDFANS
   • Fee Mode: WETH_ONLY

2️⃣ Fee Tokenization
   • You received: 80 fee tokens (80%)
   • Treasury: 20 fee tokens (20%)

3️⃣ Multi-Strategy Router
   • Route A (40%): Auto-swap → Burn
   • Route B (30%): Add liquidity → Burn LP
   • Route C (30%): Direct to you
   • Automation: Daily via Chainlink

Monitor: https://pool.fans/vault/0x...
```

---

## Common Patterns

### Maximum Deflation
```
User: "100% of fees should burn"
→ Deploy MultiActionVault with single BuyAndBurn action at 100%
```

### Holder-First Strategy
```
User: "Reward my holders, burn the rest"
→ 60% distribute to holders, 40% burn
```

### Time-Wrapper Auction
```
User: "Create daily fee auctions"
→ Deploy 1D wrappers, sell on Uniswap, rotate daily
```

### Conditional Growth
```
User: "If mcap > $1M distribute to holders, else burn"
→ ConditionalFeeRouter with MarketCapAbove condition
```

---

## Critical Gotchas

### Treasury Fee is Immutable
- 80% creator / 20% treasury split is protocol-level
- Cannot be changed after deployment

### Time-Wrapper Locks are Final
- Once deposited with a lock tier, cannot change duration
- Wrapper tokens ARE tradeable during lock
- Original tokens locked until expiry

### Two-Step Tokenization Required
- For existing tokens, MUST complete all 3 steps
- If admin transferred to wrong address, recovery is complex

### Allocation Must Equal 100%
- MultiActionVault allocations must sum to 10000 bps
- Partial allocations will fail deployment

---

## Resources

- **UI:** https://pool.fans
- **Docs:** https://pool.fans/docs
- **Contracts:** https://pool.fans/docs#contracts
- **Architecture Diagrams:** See references/architecture.md

---

*⚡ ComposAIble Revenue — Trust through code, not promises.*
