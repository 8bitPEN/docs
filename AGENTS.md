# Exponent Documentation — Agent Instructions

You are editing the developer documentation for the Exponent protocol on Solana. Every word in these docs can cause a developer to lose funds if wrong. Treat precision as a security requirement.

## Source of Truth

The monorepo at `../exponent-monorepo` is always authoritative. If the docs and the code disagree, the code is correct. Key paths:

- SDK source: `../exponent-monorepo/packages/exponent-sdk/src/`
- SDK exports: `../exponent-monorepo/packages/exponent-sdk/src/index.ts`
- Types: `../exponent-monorepo/packages/exponent-types/src/`
- IDLs: `../exponent-monorepo/packages/exponent-idl/`, `exponent-clmm-idl/`, `exponent-orderbook-idl/`
- Environment: `../exponent-monorepo/packages/exponent-sdk/src/environment.ts`

Before writing or editing any instruction page, read the corresponding method in the SDK source and verify:
1. Parameter names and types match exactly
2. Return type matches exactly
3. The code example compiles

## Terminology (exact usage required)

| Term | Meaning | Never say |
|------|---------|-----------|
| **SY** | Standardized Yield token. Wraps a yield-bearing position. | "wrapped token", "receipt token" |
| **PT** | Principal Token. Redeemable 1:1 for SY at maturity. | "bond", "zero-coupon" (unless explaining the analogy) |
| **YT** | Yield Token. Receives all accrued interest from the underlying. | "coupon token" |
| **Strip** | Convert SY into PT + YT. | "split", "decompose" (in code context) |
| **Merge** | Convert PT + YT back into SY. | "combine", "recombine" |
| **Vault** | On-chain account that manages a single yield source. Class: `Vault`. | "pool" (vaults are not pools) |
| **Market** / **MarketThree** | CLMM pool for PT/SY trading. Class: `MarketThree`. | "AMM" without qualifier |
| **Orderbook** | Limit order book for PT/YT trading. Class: `Orderbook`. | "DEX", "exchange" |
| **Flavor** | The underlying yield protocol (marginfi, kamino, jitoRestaking, perena, generic). | "adapter", "plugin" |
| **Strategy Vault** | Multi-asset pool managed by keyholders via policy-gated execution. Class: `ExponentVault`. | "fund", "pool" (vaults are not generic pools) |
| **Token Entry** | Configuration for an accepted deposit token in a vault. Specifies mint, escrow, interface type. | "deposit config" |
| **LP Token** | Vault LP token representing proportional ownership of vault AUM. | "share token" |
| **Policy** | Squads-level constraint governing which instructions the vault smart account can execute. | "permission", "rule" |
| **InterfaceType** | The yield protocol a token entry is deployed into (Kamino, Marginfi, JitoRestaking, Perena, Generic). Maps to SY flavors. | "adapter type" |
| **Prepared Instruction** | An `ixWrapper*` method that handles account creation, SY minting/redeeming automatically. Returns `{ ix, setupIxs }`. | "wrapped instruction" in docs |
| **Raw Instruction** | A low-level on-chain instruction. Documented in the "Raw Instructions" section. | "prepared instruction" |
| **Virtual Offer** | An orderbook offer backed by SY that auto-strips into PT+YT on match. Enables PT trading without handling YT. | "synthetic offer" |
| **LOCAL_ENV** | The environment config object with all program IDs. Always use this in examples. | Hardcoded program IDs in examples |

## Documentation Structure

The docs have 5 tabs:

| Tab | Purpose |
|-----|---------|
| **Home** | Protocol overview, Learn section (shared concepts), guides, program addresses |
| **Yield Stripping** | Vault SDK — strip/merge, yield positions, emissions |
| **Orderbook** | Limit order SDK — post/market offers, virtual offers, escrows |
| **CLMM** | Concentrated liquidity AMM SDK — buy/sell PT/YT, provide liquidity |
| **Vault SDK** | Strategy vault SDK — deposit/withdraw, Kamino instructions, policy builders, account references |

**The Learn section** (`home/learn/`) is the single source of truth for shared protocol concepts: SY, exchange rate, PT, YT, token lifecycle, pricing formulas, maturity, and architecture. Protocol tabs must NOT re-explain these concepts — they link to Learn pages instead.

### Cross-Reference Pattern

When a protocol tab page needs to mention a shared concept (e.g., PT pricing, fee decay), use a brief description with a link:

```mdx
PT trades at a discount determined by the [implied APY pricing formula](/home/learn/pricing-and-math).
```

Or as a standalone note for deeper topics:

```mdx
<Note>
For a full explanation of how YT yield accrual works, see [YT: Yield Token](/home/learn/yield-token).
</Note>
```

Never duplicate the formulas or extended explanations from Learn pages into protocol tabs.

## Import Rules

SDK classes and environment come from the main SDK package:

```typescript
import { Vault, MarketThree, Orderbook, Router, YtPosition, QuoteDirection, LOCAL_ENV } from "@exponent-labs/exponent-sdk";
```

Strategy Vault classes and helpers:

```typescript
import { ExponentVault, kaminoInstructionBuilder, createVaultInteractionInstructions, KaminoAction, KaminoMarket, LOCAL_ENV } from "@exponent-labs/exponent-sdk";
import { ExponentVaultsPDA } from "@exponent-labs/exponent-vaults-pda";
import { ExponentVaultsFetcher } from "@exponent-labs/exponent-vaults-fetcher";
```

Common types and helpers are also re-exported from the main SDK package:

```typescript
import { OfferType, offerOptions, amount } from "@exponent-labs/exponent-sdk";
```

Raw instructions (codama-generated) come from the SDK's client subpath exports:

```typescript
import { createWrapperPostOfferInstruction } from "@exponent-labs/exponent-sdk/client/orderbook";
import { createStripInstruction } from "@exponent-labs/exponent-sdk/client/core";
import { createBuyPtInstruction } from "@exponent-labs/exponent-sdk/client/clmm";
```

Never reference `@exponent-labs/exponent-clmm-sdk` or `@exponent-labs/exponent-orderbook-sdk`. These do not exist. The old standalone client packages (`exponent-core-client`, `exponent-clmm-client`, `exponent-orderbook-client`) have been merged into the SDK as subpath exports.

## Prepared Instructions vs Raw Instructions

The docs have two sections per protocol:

1. **TypeScript Prepared Instructions** — These are `ixWrapper*` SDK methods. They:
   - Auto-create token accounts (returned in `setupIxs`)
   - Auto-mint SY from base assets when needed
   - Auto-redeem SY to base assets when needed
   - Return `{ ix: TransactionInstruction, setupIxs: TransactionInstruction[] }`
   - Are what 90% of developers should use

2. **Raw Instructions** — These are on-chain program instructions. They:
   - Require the caller to set up all accounts manually
   - Are for developers building custom transaction pipelines
   - Map directly to Solana program instruction discriminators

The Prepared Instructions section only documents `ixWrapper*` methods. Non-wrapper `ix*` methods (e.g., `ixPostOffer`) are available in the SDK but not documented as prepared instructions — they require manual account setup and are for advanced use cases.

## Page Templates

### Learn Page (Shared Concepts)

```mdx
---
title: "PT: Principal Token"
description: "How Principal Tokens provide fixed-rate exposure and predictable redemption value"
---

[Opening paragraph: what the concept is, why it matters, how it fits in the protocol. Link to related Learn pages.]

## Key Properties

- **Property 1** — explanation
- **Property 2** — explanation

## [Core Mechanics Section]

[Detailed explanation with formulas (LaTeX), code examples (TypeScript from SDK), and tables. Every formula must be verified against the monorepo source code.]

## [Additional Sections as Needed]

[Worked numerical examples, warnings, tips. Use `<Warning>`, `<Note>`, `<Tip>` components.]
```

Learn pages are the authoritative source for shared concepts. They should be thorough and self-contained — protocol tab pages link to them for detail.

### Instruction Page (Prepared)

```mdx
---
title: Post Order
description: Post a limit order to the orderbook with automatic account setup
sdk_class: Orderbook
sdk_method: ixWrapperPostOffer
returns: "{ ix: TransactionInstruction, setupIxs: TransactionInstruction[] }"
category: prepared-instruction
protocol: orderbook
---

[1-2 sentence description of what this does and when to use it.]

## Usage

\`\`\`typescript
import { Orderbook, LOCAL_ENV } from "@exponent-labs/exponent-sdk";
import { Connection, Transaction, sendAndConfirmTransaction } from "@solana/web3.js";

const connection = new Connection("https://api.mainnet-beta.solana.com");
const orderbook = await Orderbook.load(LOCAL_ENV, connection, orderbookAddress);

const { ix, setupIxs } = await orderbook.ixWrapperPostOffer({
  // params with real values
});

const tx = new Transaction().add(...setupIxs, ix);
const signature = await sendAndConfirmTransaction(connection, tx, [wallet]);
\`\`\`

## Required Parameters

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| ... | ... | ... |

## Optional Parameters

(if any)

## Returns

| Field | Type | Description |
| ----- | ---- | ----------- |
| `ix` | `TransactionInstruction` | The main instruction |
| `setupIxs` | `TransactionInstruction[]` | Setup instructions (run before `ix`) |
```

### Instruction Page (Low-Level)

Same structure but:
- Title uses `ix` prefix (not `ixWrapper`)
- Note at top: "This is the low-level version. For most use cases, prefer [ixWrapperX](/link)."
- Returns: `TransactionInstruction` (single, not `{ ix, setupIxs }`)

### Read Function Page

```mdx
---
title: getQuote
description: Calculate a trade quote with slippage
sdk_class: Orderbook
sdk_method: getQuote
category: read-function
protocol: orderbook
---

[Description]

## Usage

\`\`\`typescript
// Full working example
\`\`\`

## Parameters

| Parameter | Type | Description |
| --------- | ---- | ----------- |

## Returns

[Exact return type with field descriptions]
```

### Account Reference Page

```mdx
---
title: Vault
description: On-chain account storing vault state
category: account-reference
protocol: yield-stripping
---

[Description of what this account represents]

## Fields

| Field | Type | Description |
| ----- | ---- | ----------- |
```

## Style Rules

- Active voice, second person ("you")
- One idea per sentence
- Sentence case for headings
- All amounts are in lamports (raw u64). Say "lamports" not "tokens" when describing amounts.
- All orderbook prices are APY as a decimal (e.g., `0.12` for 12% APY)
- Code formatting for: method names, parameter names, type names, file paths
- Parameter tables use the colored-code style established in existing pages
- Never add `console.log` or debugging code to examples
- Never truncate code examples with `// ...` — show complete, working code or don't show it

## Mintlify Components

Use only these components:
- `<CardGroup cols={N}>` + `<Card>` — for navigation grids
- `<Tabs>` + `<Tab>` — for multi-view content
- `<Tip>` — for helpful context
- `<Note>` — for important caveats
- Markdown tables — for parameter/field references
- Fenced code blocks with language tag — for all code

## Program IDs

Use `LOCAL_ENV` in examples, never hardcode these. `LOCAL_ENV` contains `exponent_core` and the 5 SY program IDs. The CLMM and Orderbook program IDs come from their IDL packages and are resolved automatically by the SDK.

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
| exponent_vaults | `HycecAnELpjL1pMp435nEKWkcr7aNZ2QGQGXpzK1VEdV` |

## Common Mistakes to Prevent

1. **Wrong package import** — Always `@exponent-labs/exponent-sdk`, never protocol-specific packages
2. **Missing setupIxs** — Wrapper methods return `{ ix, setupIxs }`. Both must be added to the transaction.
3. **Wrong offerType semantics** — `{ sellYt: {} }` means the maker deposits SY/YT and receives the counter-asset when matched. Verify direction against SDK source.
4. **Stale parameter types** — Always check the SDK source before writing parameter tables. Types change between versions.
5. **Confusing `Market` and `MarketThree`** — `Market` is legacy (v1 AMM). `MarketThree` is the active CLMM. Docs should reference `MarketThree` for CLMM content.
