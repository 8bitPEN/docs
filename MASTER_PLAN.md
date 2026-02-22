# Exponent Docs — Master Plan

## Context

Exponent is a Solana yield-stripping protocol with 3 sub-protocols (Yield Stripping, CLMM, Orderbook) and a single TypeScript SDK (`@exponent-labs/exponent-sdk`). This docs repo is Mintlify-based with 173+ pages. The monorepo at `../exponent-monorepo` is the source of truth for all SDK behavior.

This master plan defines every documentation improvement we will implement. Each section is a discrete deliverable.

---

## Phase A — Foundation (Done)

### LLMS.txt + LLMS-full.txt
Machine-readable index files at the repo root following the [llms.txt standard](https://llmstxt.org/). `llms.txt` is the concise index (~90 lines). `llms-full.txt` is the complete API reference with every method signature, parameter type, and return type (~400 lines). Agents fetch these to avoid hallucinating APIs.

### SKILL.md
Agent skill file with operational knowledge — setup patterns, decision tree (goal → class → method), constraints (120 offer CU limit, ALT requirements), and common mistakes. This is what an agent loads to become a competent SDK user.

### AGENTS.md
Overhauled from Mintlify boilerplate to Exponent-specific instructions. Contains exact terminology glossary, single-package rule, prepared-vs-raw instruction mapping, page templates for every doc type, program IDs, and style rules.

### Integration Guide Fix
Fixed stale imports in `home/integration-guide.mdx`. Removed references to non-existent `@exponent-labs/exponent-clmm-sdk` and `@exponent-labs/exponent-orderbook-sdk`. All imports now use the single `@exponent-labs/exponent-sdk` package with `LOCAL_ENV`.

### AI & LLM Integration Page
New page at `home/ai-integration.mdx` under the Guides group. Documents how to use `llms.txt`, `SKILL.md`, and MCP integration with AI coding tools.

---

## Phase A.5 — Learn Section Restructure (Done)

Centralized duplicated core concepts into a new Learn section under the Home tab. Created 7 Learn pages:
- `home/learn/standardized-yield` — SY wrappers, the exchange rate, 5 standard programs, instruction interface
- `home/learn/principal-token` — PT as zero-coupon bond, pricing, backing, fixed-rate strategy
- `home/learn/yield-token` — YT yield accrual, leverage, time decay, collection workflow
- `home/learn/token-lifecycle` — Full strip → trade → yield → merge → redeem flow
- `home/learn/pricing-and-math` — All PT/YT pricing formulas, fee decay model, arbitrage bounds
- `home/learn/maturity` — Before/at/after maturity, emergency mode, vault invariant
- `home/learn/architecture` — 4 programs, how they relate, SDK class mapping

Protocol tab pages (yield-stripping, clmm, orderbook) slimmed to remove duplicated concept explanations, replaced with brief descriptions + links to Learn pages. Deleted `home/how-it-works.mdx` (content distributed across Learn pages) and old `home/architecture.mdx`.

Updated all cross-references, `llms.txt`, `llms-full.txt`, `SKILL.md`, `AGENTS.md`, and `home/ai-integration.mdx` to reflect new structure. Deleted stale one-time prompt files (`CLMM_DOCS_PROMPT.md`, `ORDERBOOK_DOCS_PROMPT.md`).

---

## Phase B — Content Alignment (Done)

### Prepared Instructions = Wrapper Methods (Done)
The "TypeScript Prepared Instructions" sections now only contain `ixWrapper*` methods. Non-wrapper `ix*` methods were removed from navigation and deleted. Changes made:
1. Banner at top of each overview: "Prepared instructions handle token account creation, SY minting/redeeming, and account setup automatically."
2. Only `ixWrapper*` methods remain in prepared instructions
3. Non-wrapper `ix*` method pages deleted (35 files: 6 orderbook + 20 CLMM + 9 yield-stripping)
4. Human-readable sidebar names (e.g., "Buy PT" instead of "ixWrapperBuyPt")
5. `AGENTS.md` and `llms.txt` updated to reflect wrapper-only structure

Remaining wrappers per protocol:
- **Orderbook** (5): Post Order, Market Order, Remove Order, Collect Interest, Withdraw Funds
- **CLMM** (5): Buy PT, Sell PT, Buy YT, Sell YT, Provide Liquidity
- **Yield Stripping** (2): Strip (`ixStripFromBase`), Merge (`ixMergeToBase`)

### Tab Reorder (Done)
Tabs reordered to: Home > Yield Stripping > Orderbook > CLMM

### Programs Page (Done)
New page at `home/programs.mdx` listing all 8 on-chain program addresses (3 core + 5 SY flavors) with SDK usage example. Added under a "Programs" group on the Home tab.

### Structured Frontmatter
Add machine-readable frontmatter to every instruction page:
```yaml
sdk_class: Orderbook
sdk_method: ixWrapperPostOffer
returns: "{ ix: TransactionInstruction, setupIxs: TransactionInstruction[] }"
category: prepared-instruction
protocol: orderbook
```
Agents can extract class, method, and return type without parsing prose.

---

## Phase C — Visual & Recipes

### Excalidraw Decision Diagrams
5 new diagrams at key decision points:

| Diagram | Purpose |
|---------|---------|
| `images/home/protocol-selection.excalidraw` | Which protocol? Yield Stripping vs CLMM vs Orderbook |
| `images/home/instruction-selection.excalidraw` | Which instruction? Raw vs wrapper, buy vs sell |
| `images/home/flavor-selection.excalidraw` | Which flavor? marginfi vs kamino vs jito vs perena vs generic |
| `images/clmm/clmm-lp-decision.excalidraw` | CLMM LP strategy: range, tick spacing, rebalancing |
| `images/orderbook/orderbook-offer-types.excalidraw` | Virtual vs non-virtual, sellYt vs buyYt, postOnly vs fillOrKill |

5 excalidraw files already exist under `images/yield-stripping/` (vault internals). New diagrams cover protocol-level decisions.

### Recipes Section
New top-level "Recipes" tab with copy-paste multi-step workflows:
- Fixed-rate lending in 3 steps
- Leveraged yield position
- Cross-market arbitrage bot
- Collect all accrued interest across positions

Content exists partially in `choosing-a-protocol.mdx` and `integration-guide.mdx` but isn't surfaced as standalone, agent-friendly recipes.

---

## Phase D — Agent Infrastructure

### MCP Server
Mintlify already supports `"mcp"` in contextual options (enabled in docs.json). Verify end-to-end. If insufficient, build custom MCP server exposing:
- `get_instruction(name)` → method signature, params, example
- `get_decision(goal)` → recommended protocol + method
- `get_program_ids()` → all program addresses

### Context7 Registration
Register `@exponent-labs/exponent-sdk` on Context7 so agents with Context7 tools can pull live docs.

### Executable Documentation CI
CI pipeline that extracts every TypeScript code block from `.mdx` files, compiles against the real SDK (`tsc --noEmit`), and fails if types don't match. Prevents stale examples.

---

## Phase E — Automation

### Auto-Generated Reference from Types
Script that reads the SDK's `.d.ts` files, extracts method signatures, and validates against documented parameter tables. CI flags drift when SDK changes.

### Error Reference
Page per protocol listing every on-chain error code, meaning, and fix. Extracted from IDL files (`exponent-idl`, `exponent-clmm-idl`, `exponent-orderbook-idl`).

---

## Phase E.5 — Verify QuoteDirection Enum Casing

### TODO: Check QuoteDirection enum member naming

The SDK defines `QuoteDirection` in `exponent-sdk/src/orderbook/types.ts` with SCREAMING_SNAKE_CASE members (`SY_TO_YT`, `SY_TO_PT`, etc.), but the docs use PascalCase (`SyToYt`, `SyToPt`, `PtToSy`). These would be `undefined` at runtime if SCREAMING_SNAKE_CASE is correct.

**Need to verify:** Which casing is actually correct at runtime. Check the compiled SDK output or run a quick test. If SCREAMING_SNAKE_CASE is correct, fix these files:
- `orderbook/sdk-quickstart.mdx` lines 55, 212
- `orderbook/typescript/read-functions/get-quote.mdx` line 21
- `orderbook/virtual-offers.mdx` line 81

---

## Phase F — Support Links Audit

### Fix Support Links Everywhere
Audit all pages for support/help/contact links and ensure they point to the correct destinations. Currently links may point to Mintlify defaults, placeholder URLs, or stale endpoints. Every support link across the docs should point to Exponent's actual support channels (Discord, GitHub issues, Twitter/X, etc.).

Deliverables:
1. Grep all `.mdx` and config files for support-related links (Discord, Twitter, GitHub, email, "contact", "support", "help", "community")
2. Identify any broken, placeholder, or Mintlify-default links
3. Replace all with correct Exponent support URLs
4. Verify footer links, navbar links, and in-page CTAs all resolve correctly

---

## Phase G — Package & Source Links

### NPM Packages & GitHub Repos
Add npm package links and GitHub repository URLs for all packages used in the SDK and docs. This gives developers and agents direct access to source code, changelogs, and issue trackers.

Packages to document:
- `@exponent-labs/exponent-sdk` — npm + GitHub
- `@exponent-labs/exponent-types` — npm + GitHub
- `@exponent-labs/exponent-idl` — npm + GitHub
- `@exponent-labs/exponent-clmm-idl` — npm + GitHub
- `@exponent-labs/exponent-orderbook-idl` — npm + GitHub
- Any other dependencies or peer dependencies referenced in the docs (e.g., `@solana/web3.js`, `@coral-xyz/anchor`, etc.)

Deliverables:
1. Add an "SDK & Packages" page (or section on existing Programs page) listing every npm package with links to its npmjs.com page and GitHub repo
2. Update `llms.txt` / `llms-full.txt` with package URLs
3. Update `SKILL.md` with package registry links so agents can look up versions and changelogs

---

## Source of Truth

The monorepo is always authoritative:
- SDK source: `packages/exponent-sdk/src/`
- SDK exports: `packages/exponent-sdk/src/index.ts`
- Types: `packages/exponent-types/src/`
- IDLs: `packages/exponent-idl/`, `packages/exponent-clmm-idl/`, `packages/exponent-orderbook-idl/`
- Environment/Program IDs: `packages/exponent-sdk/src/environment.ts`
- Codama clients: `clients/exponent-core/`, `clients/exponent-clmm/`, `clients/exponent-orderbook/`
