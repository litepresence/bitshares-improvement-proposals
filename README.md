# BitShares Improvement Proposals (BSIPs 1-9)

This repository holds the design specifications, whitepapers, lightpapers, and audit
workflow scripts for nine BitShares Improvement Proposals. Each BSIP is a parallel
addition to BitShares — none modifies legacy assets, markets, or settlement rules,
and none is activated. `bit*`, `HONEST.*`, and gateway UIAs keep running unchanged,
permanently.

## The nine proposals

| BSIP | Title | Track | Status | Wire tags |
|------|-------|-------|--------|-----------|
| BSIP-1 | Gamified Smartcoins — bonded oracle base + vault-backed tranched issuance | H (smartcoins) | Built + audited, NO_GO | 78-103 (Phase 1: 78-82 oracle, Phase 2: 83-103 bonds/vaults); objects 1.23.x-1.38.x |
| BSIP-2 | True Stealth Transfers — ring signatures + Bulletproofs+ + blind pool swaps | G (stealth) | Built + audited, NO_GO | 104-109 |
| BSIP-3 | Foreign-chain ACCS (atomic cross-chain settlement) | F (foreign chain) | Built + audited, NO_GO | 110-117 |
| BSIP-4 | N-Asset Pools — geometric-mean multi-asset liquidity pools | F/G/H config | Design only | TBD (design spec pending) |
| BSIP-5 | Open-Order Rewards — maker rebates from taker fees | F/G/H config | Design only | TBD (design spec pending) |
| BSIP-6 | StableSwap Pools — Curve-style invariant for pegged pairs | F/G/H config | Design only | No new tags (extension to existing pool ops) |
| BSIP-7 | Asymmetric Bounded Pools — capital-efficient bounded-range AMM | F/G/H config | Design only | No new tags (extension to existing pool ops) |
| BSIP-8 | PAMM Accounts — Percentage Allocation Management Module | H (smartcoins) | Built + audited, NO_GO | 133-136; object 1.46.x |
| BSIP-9 | SPV-Enabled XClaim ACCS — trustless foreign-chain settlement on UTXO chains | F (foreign chain) | Built + audited, NO_GO | 118 (`accs_fulfill_spv_operation`) |

## Legacy rule

Legacy assets (`bit*`, `HONEST.*`, gateway UIAs) keep their semantics, fees, and
settlement rules — no migration, no caps, no changes, permanently. The proposals add
parallel capabilities beside the existing books.

## Activation status

All nine proposals are **code-complete or design-complete; none is activated.**
Activation requires: specialist audit clearance, replay evidence, live soak, pilot
(≤$20k face, 90-day coverage for BSIP-1), committee hardfork dates, and witness
majority adoption. See `build/ACTIVATION-CHECKLIST.md` (in the bitshares-core tree)
for the open gate register.

## Repository contents

- `docs/bsips/` — the nine BSIP draft specifications (canonical drafts, unfiled)
- `docs/whitepapers/` — full whitepapers for BSIP-1 through BSIP-9
- `docs/lightpapers/` — short lightpapers for BSIP-1 through BSIP-9
- `skills/` — audit and workflow skills (44 skills, one per audit domain)

## Per-BSIP overviews

### BSIP-1 — Gamified Smartcoins (Track H)

A parallel hybrid smartcoin class: bonded price oracles plus vault-backed, tranched
issuance. Every producer posts a bond behind the price feed it publishes; a lie loses
that bond to whoever catches it, and pay is for liveness, never for agreement. New
dollar-tracking smartcoins arrive as two tranches paid in a fixed order — a senior
stable (`FX.USD`) redeemable at the bonded price, first in line on exits, and a junior
reserve coin (`FXR`) that absorbs volatility, earns protocol revenue, dilutes first up
to an `rmax` cap, and may go to zero. Coverage (bonds behind a promise divided by the
promise) is public and gates growth. Nothing turns on until a pilot under $20,000
proves coverage for 90 days and a vote passes.

26 new operations at wire tags 78-103. Shares the oracle/guard substrate byte-identical
with BSIP-3 across all configurations.

### BSIP-2 — True Stealth Transfers (Track G)

True stealth transfers: every private send hides among at least sixteen users, every
hidden amount is verified by the whole network instead of by a wallet, and a fee rule
(Rule R) aims the subsidy at the operations that grow the live anonymity set. Also
adds blind→blind pool swaps against the same public liquidity pools without touching
your account, and makes private sending the wallet default with one tap to send
publicly. Nothing existing is removed — legacy transfers, markets, and every protocol
total stay as they are.

Six operations at wire tags 104-109 behind one hardfork flag
(`HARDFORK_STEALTH_PASSED`). Specialist CLSAG/BP+ audit is an open external gate.

### BSIP-3 — Foreign-chain ACCS Settlement (Track F)

Replaces custodial gateway IOUs with escrow and bonds. An atomic cross-chain swap
completes both legs together or neither: a holder posts an on-chain offer selling any
BitShares asset for BTC, a BTC holder matches it and sends BTC from their own wallet,
and a bonded oracle watching Bitcoin reports payment to consensus; only then does the
chain release the escrowed assets. No vault custodies foreign coins, no committee
approves individual swaps, and a false published record forfeits a bond. The earlier
vault-model design (retired H3) was fully removed; only the transaction oracle and the
orderbook-settlement idea survived.

Eight operations at wire tags 110-117, three new object types, independently votable.

### BSIP-9 — SPV-Enabled XClaim ACCS (Track F)

Enhances BSIP-3's ACCS orderbook with Simplified Payment Verification in consensus
for UTXO chains whose headers are verifiable (Bitcoin, Litecoin, Dash, Dogecoin,
Bitcoin Cash). The seller submits a self-contained SPV proof — raw transaction,
block header, merkle path with position bits, header chain to a checkpoint — and
BitShares verifies proof-of-work, header linkage, merkle inclusion, and confirmation
depth directly in `do_evaluate`. No attestor, no oracle, no trusted observation for
those chains. Chains without SPV headers (Ethereum, Solana, Polkadot) keep the
bonded-attestor F1 path unchanged — BSIP-9 is dual-path: SPV primary, attestor
appeal and fallback.

No witness keys, no vault custody, no witness honey pot as locked value grows. The
active BitShares witness set is not asked to hold foreign-chain signing keys or
report foreign-chain tips.

One new operation at wire tag 118 (`accs_fulfill_spv_operation`), gateway extended
with `spv_capable` and `checkpoint` fields, hardfork gate `HARDFORK_SPV_ACCS_PASSED`,
new SPV library in `libraries/chain/spv/`. Code-complete and audited; gate NO_GO
pending specialist SPV review, testnet seasoning, and real SPV test vectors.

### BSIP-4 — N-Asset Pools (Track F/G/H config)

A parallel multi-asset pool class: one pool holds up to eight assets, the pool's share
token is a proportional claim on the whole basket, and pricing follows a geometric-mean
invariant computed from every token ratio at once. Each swap's fee is derived from how
imbalanced the trade leaves the pool, and two self-steering regulators move that fee
toward volatility targets with no scheduled parameter changes. Every pool publishes its
own time-and-volume weighted average price, computed in consensus from the pool's own
trades. Legacy two-asset constant-product pools stay untouched.

Design only — no code in the BitShares core branches yet; design spec, test vectors,
and wire pinning remain before any build.

### BSIP-5 — Open-Order Rewards (Track F/G/H config)

Resting an order earns nothing today; this pays makers. Each taker's fee funds a maker
rebate paid when the order fills, plus a provider pool distributed to open orders
weighted by three measured qualities: recency (capped at one hour), remaining quantity,
and proximity to market price. Gating, capping, and self-trade refusal keep rewards
unforfeitable and unpumpable. Participation is opt-in per asset; everything routed or
paid stays on the ledger. The empty-book rule lets rates or splits change only while
no rewards-enabled open orders rest on the affected market.

Design only — no code in the BitShares core branches yet.

### BSIP-6 — StableSwap Pools (Track F/G/H config)

A second pricing curve for liquidity pools: Curve's StableSwap invariant with an
amplification coefficient A, aimed at pegged pairs (dollar tokens, wrappings of the
same coin, bridged twins). The curve is flat near balance — a 10,000 swap in a million
unit pool returns 9,978 (99.8%) instead of ~9,900 from constant product — and degrades
toward constant product as the pool becomes lopsided, so a depeg still has a liquidity
floor. Participation is per-pool at creation time; no new operation tags (the curve
rides existing pool operations through typed extensions).

Design only — no code yet.

### BSIP-7 — Asymmetric Bounded Pools (Track F/G/H config)

A second pricing curve: `(x + xv)^w * (y + yv)^(1-w) = k`, with hard price bounds
`[P_low, P_high]` and a free holdings ratio `w`. Puts 100% of capital inside the
chosen range with zero tail waste, and lets the market maker skew the fund ratio (e.g.
80/20) rather than forcing 50/50. Implemented as a parallel pool class reusing existing
liquidity-pool scaffolding. Closed-form math with no Newton iteration. Constant-product
and StableSwap pools are untouched; CES curves and dynamic rho are out of scope.

Design only — no code yet.

### BSIP-8 — PAMM Accounts (Track H)

Forex-style Percentage Allocation Management Module: an investor pools funds under a
money manager, the manager trades a single master account, and profits and losses are
distributed in proportion to share of the pool. Built on BSIP-72 Tanks and Taps for
bounded, auditable fund custody plus BitShares custom active authorities for per-investor
withdrawal rights. Shares are a user-issued asset minted and burned at current NAV; a
rolling 365-entry NAV buffer makes every pool comparable on standardised metrics. The
manager cannot self-report NAV (P8-07); snapshots require a non-manager caller.

Four operations at wire tags 133-136, object ID 1.46.x (`pamm_pool`). Built and
audited; NO_GO pending activation gates.

## How to use this repo

1. Read the BSIP abstracts in `docs/bsips/` and the corresponding whitepaper/lightpaper.
2. Review the activation gate register (`build/ACTIVATION-CHECKLIST.md` in the
   bitshares-core tree) for open items per proposal.
3. None of these proposals is authorized for real funds. Activation requires a vote
   and committee-set hardfork dates.

## License

All specifications are placed in the public domain unless otherwise noted in a file.