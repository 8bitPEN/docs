# Exponent SDK — Agent Skill

You are an AI coding agent helping a developer integrate with the Exponent protocol on Solana. This file contains everything you need to write correct code. Follow it exactly.

## Environment

- **Chain:** Solana (mainnet-beta)
- **Package:** `@exponent-labs/exponent-sdk` (single package, no per-protocol packages)
- **Dependencies:** `@solana/web3.js ^1.98`, `bn.js ^5.2`
- **All amounts:** lamports (raw `bigint` or `BN`). Never use decimals unless converting for display.
- **All orderbook prices:** APY as a decimal number (e.g., `0.12` = 12% APY)
- **Environment config:** Always use `LOCAL_ENV` from the SDK. Never hardcode program IDs.

## Setup (always start here)

```typescript
import { Connection, PublicKey, Transaction, sendAndConfirmTransaction } from "@solana/web3.js";
import { Vault, MarketThree, Orderbook, Router, LOCAL_ENV } from "@exponent-labs/exponent-sdk";

// Orderbook types and helpers are re-exported from the SDK:
import { OfferType, offerOptions, amount } from "@exponent-labs/exponent-sdk";

const connection = new Connection("https://api.mainnet-beta.solana.com", "confirmed");

// Load the SDK objects you need (async — fetches on-chain state)
const vault = await Vault.load(LOCAL_ENV, connection, vaultAddress);
const market = await MarketThree.load(LOCAL_ENV, connection, marketAddress);
const orderbook = await Orderbook.load(LOCAL_ENV, connection, orderbookAddress);
```

## Decision Tree

Use this to pick the right method:

| Goal | Class | Method |
|------|-------|--------|
| Create PT+YT from SY | `Vault` | `ixStrip({ syIn, depositor })` |
| Create PT+YT from base asset (USDC, SOL) | `Vault` | `ixStripFromBase({ owner, amountBase })` |
| Merge PT+YT back to SY | `Vault` | `ixMerge({ pyIn, depositor })` |
| Merge PT+YT back to base asset | `Vault` | `ixMergeToBase({ owner, amountPy })` |
| Buy PT instantly (pay SY) | `MarketThree` | `ixBuyPt({ trader, amountSy, outConstraint })` |
| Sell PT instantly (receive SY) | `MarketThree` | `ixSellPt({ trader, amountPt, outConstraint })` |
| Buy PT with base asset | `MarketThree` | `ixWrapperBuyPt({ owner, baseIn, minPtOut })` |
| Sell PT for base asset | `MarketThree` | `ixWrapperSellPt({ owner, amount, minBaseOut })` |
| Buy YT instantly | `MarketThree` | `ixBuyYt({ trader, ytOut, maxSyIn })` |
| Sell YT instantly | `MarketThree` | `ixSellYt({ trader, ytIn, minSyOut })` |
| Buy YT with base asset | `MarketThree` | `ixWrapperBuyYt({ owner, ytOut, maxBaseIn })` |
| Sell YT for base asset | `MarketThree` | `ixWrapperSellYt({ owner, amount, minBaseOut })` |
| Post limit order at specific APY | `Orderbook` | `ixWrapperPostOffer({ trader, price, amount, offerType, ... })` |
| Execute market order on orderbook | `Orderbook` | `ixWrapperMarketOffer({ trader, maxPriceApy, amount, ... })` |
| Cancel limit order | `Orderbook` | `ixWrapperRemoveOffer({ trader, offerIdx, mintSy })` |
| Provide concentrated liquidity | `MarketThree` | `ixDepositLiquidity({ ptInIntent, syInIntent, depositor, lowerTickKey, upperTickKey })` |
| Provide liquidity with base asset | `MarketThree` | `ixWrapperProvideLiquidity({ depositor, amountBase, minLpOut, lowerTickApy, upperTickApy })` |
| Remove liquidity | `MarketThree` | `ixWithdrawLiquidity({ lpIn, withdrawer, lpPosition, minPtOut, minSyOut })` |
| Initialize yield position | `Vault` | `ixInitializeYieldPosition({ owner })` |
| Collect YT yield | `Vault` | `ixCollectInterest({ owner, amount })` |
| Collect emission rewards | `Vault` | `ixCollectEmission({ owner, emissionIndex, amount })` |
| Get best quote across venues | `Router` | `getQuote({ direction, inAmount, syExchangeRate })` |
| Get orderbook quote | `Orderbook` | `getQuote({ inAmount, direction, unixNow, syExchangeRate })` |
| List active offers | `Orderbook` | `getOffers()` |
| Get user balances in orderbook | `Orderbook` | `getUserBalances(user)` |
| Withdraw funds from orderbook escrow | `Orderbook` | `ixWrapperWithdrawFunds({ trader, mintSy, ptAmount, ytAmount, syAmount })` |
| Withdraw liquidity to base asset | `MarketThree` | `ixWithdrawLiquidityToBase({ owner, amountLp, minBaseOut, lpPosition })` |
| Provide liquidity (classic mode) | `MarketThree` | `ixProvideLiquidityClassic({ depositor, amountBase, amountPt, minLpOut, lowerTickApy, upperTickApy })` |
| Withdraw liquidity (classic mode) | `MarketThree` | `ixWithdrawLiquidityClassic({ owner, amountLp, lpPosition })` |

## Return Types

Methods return one of two patterns:

**Pattern A — Single instruction:**
```typescript
const ix: TransactionInstruction = vault.ixStrip({ syIn, depositor });
const tx = new Transaction().add(ix);
```

**Pattern B — Instruction + setup:**
```typescript
const { ixs, setupIxs } = market.ixBuyPt({ trader, amountSy, outConstraint });
const tx = new Transaction().add(...setupIxs, ...ixs);
```

Or for wrapper methods:
```typescript
const { ix, setupIxs } = await orderbook.ixWrapperPostOffer({ ... });
const tx = new Transaction().add(...setupIxs, ix);
```

**Always include `setupIxs` before the main instruction(s).** They create required token accounts.

## Constraints

- **Orderbook:** Max 120 filled offers per transaction (compute unit limit).
- **CLMM:** Requires Address Lookup Tables (ALTs) for CPI accounts. The SDK handles this internally.
- **YT yield collection:** Must call `ixInitializeYieldPosition` once per vault per user before depositing YT.
- **YT deposit flow:** Initialize position → `ixDepositYt` → (time passes) → `ixStageYield` → `ixCollectInterest`.
- **Orderbook offer types:** `OfferType.SellYt` = maker sells YT for SY (deposit YT, receive SY when matched). `OfferType.BuyYt` = maker buys YT with SY (deposit SY, receive YT when matched). Import `OfferType` from `@exponent-labs/exponent-sdk`.
- **Virtual offers:** Set `virtualOffer: true` to trade PT without handling YT. The orderbook auto-strips/merges.
- **Offer options:** `offerOptions("FillOrKill", [true])` = fill entirely or revert. `offerOptions("FillOrKill", [false])` = allow partial fills. Import `offerOptions` from `@exponent-labs/exponent-sdk`.

## Common Mistakes

1. **Wrong import:** Never use `@exponent-labs/exponent-clmm-sdk` or `@exponent-labs/exponent-orderbook-sdk`. Everything is in `@exponent-labs/exponent-sdk`.
2. **Missing setupIxs:** Wrapper methods return `{ ix, setupIxs }` or `{ ixs, setupIxs }`. Both must go in the transaction.
3. **Forgetting YieldPosition init:** `ixDepositYt` fails if `ixInitializeYieldPosition` wasn't called first for that vault.
4. **Wrong offerType direction:** Verify against the SDK source. The naming is from the perspective of the YT flow, not the user's intent.
5. **Using `Market` instead of `MarketThree`:** `Market` is the legacy v1 AMM. `MarketThree` is the active CLMM. Use `MarketThree`.
6. **Hardcoding program IDs:** Always use `LOCAL_ENV`. Program IDs are in `environment.ts`.

## Program IDs (reference only — use LOCAL_ENV)

| Program | Address |
|---------|---------|
| exponent_core | `ExponentnaRg3CQbW6dqQNZKXp7gtZ9DGMp1cwC4HAS7` |
| exponent_clmm | `GvHPnJqothHhg5hyTDstNcGEjD8nJL9MeRsU1R6PUc1m` |
| exponent_orderbook | `ExpRUPxMSwYYGjt1eoa2NonhKKrsFB1AQ4vZ6ujCHviP` |
| marginfi_sy | `XPMfipyhcbq3DBvgvxkbZY7GekwmGNJLMD3wdiCkBc7` |
| kamino_sy | `XPK1ndTK1xrgRg99ifvdPP1exrx8D1mRXTuxBkkroCx` |
| jito_restaking_sy | `XPJitopeUEhMZVF72CvswnwrS2U2akQvk5s26aEfWv2` |
| perena_sy | `XPerenaJPyvnjseLCn7rgzxFEum6zX1k89C13SPTyGZ` |
| generic_sy | `XP1BRLn8eCYSygrd8er5P4GKdzqKbC3DLoSsS5UYVZy` |

`LOCAL_ENV` contains `exponent_core` and the 5 SY program IDs. The CLMM and Orderbook program IDs come from their IDL packages and are resolved automatically by the SDK.

## Flavors (Yield Sources)

Each vault has one flavor (the underlying yield protocol):

| Flavor | Protocol | What it wraps |
|--------|----------|---------------|
| `marginfi` | MarginFi | Lending positions (USDC, SOL, mSOL) |
| `kamino` | Kamino (Klend) | Lending vault shares |
| `jitoRestaking` | Jito | Liquid staking / restaking |
| `perena` | Perena | Stablecoin pool shares |
| `generic` | Various | Meteora, Adrena, Fragmetric, Sanctum, Jupiter Perps, and more |

Access flavor info via `vault.flavor`.

## Key Formulas

```
PT price:   P_PT = e^(-APY × t)           where t = secondsToMaturity / 31536000
YT price:   P_YT = (1 - P_PT) / rate      where rate = SY exchange rate
YT yield:   earned_sy = (1/old_rate - 1/new_rate) × yt_balance
Fee rate:   fee_rate = e^(ln_fee_rate_root × τ)    decays to zero at maturity
Strip:      1 SY → r × PT + r × YT
Merge:      r × PT + r × YT → 1 SY
```

## Documentation

### Learn (core concepts)
- [Standardized Yield](home/learn/standardized-yield) — What SY is, the exchange rate, 5 standard programs, instruction interface
- [Principal Token](home/learn/principal-token) — PT as zero-coupon bond, pricing, backing, fixed-rate strategy
- [Yield Token](home/learn/yield-token) — YT yield accrual formula, leverage, time decay, collection workflow
- [Token Lifecycle](home/learn/token-lifecycle) — Full strip → trade → yield → merge → redeem flow
- [Pricing & Math](home/learn/pricing-and-math) — All pricing formulas, fee decay, arbitrage boundaries
- [Maturity](home/learn/maturity) — Before/at/after maturity, emergency mode, vault invariant
- [Architecture](home/learn/architecture) — 3 Solana programs, how they relate, SDK class mapping

### Protocol SDKs
- [Yield Stripping](yield-stripping/sdk-quickstart) — Vault and YtPosition instructions, read functions, account references
- [CLMM](clmm/sdk-quickstart) — MarketThree instructions, read functions, account references
- [Orderbook](orderbook/sdk-quickstart) — Orderbook instructions, read functions, account references

### Reference
- [llms-full.txt](llms-full.txt) — Complete API reference with every method signature
- [Integration Guide](home/integration-guide) — Production code examples combining all protocols
