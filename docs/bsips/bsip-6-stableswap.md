     BSIP: (number assigned at filing; house label BSIP-6)
     Title: StableSwap pools for pegged pairs: flat near the peg, constant product in a depeg
     Authors: litepresence (HONEST) (identity confirmation pending per AGENTS.md section 3)
     Status: Draft
     Type: Protocol
     Created: 2026-10-26

# Abstract

Constant-product pools charge slippage on trades whose price should not move at all. This proposal adds a second pricing curve for liquidity pools, Curve's StableSwap invariant with an amplification coefficient A, aimed at pegged pairs: dollar tokens, wrappings of the same coin, bridged twins. The curve is flat near balance by construction of the curve, so a 10,000 swap in a million-unit pool returns 9,978 (99.8%) instead of the ~9,900 a constant-product pool would quote, and it degrades toward constant product as the pool becomes lopsided, so a depeg still has a liquidity floor. Participation is per-pool at creation time; constant-product pools are untouched. No new operation tags are needed: the curve rides the existing pool operations through typed extensions. The amplification coefficient A is fixed at creation, the pool holds exactly two assets, and every rounding step favours the pool. The pitch: pegged pairs trade at the price they are supposed to trade at, and the safety net appears only when something has gone wrong.

# Motivation

Pegged pairs dominate stablecoin and bridged-asset markets, and on those pairs constant product is the wrong curve. The slippage it charges buys nothing: both trader and provider lose the spread for no economic reason, because the price was never supposed to move. Competitive stable-swap venues exist for exactly this reason. The fix is a curve whose whole point is to be flat where the pair is supposed to trade and to fall back to the existing curve only when something has gone wrong.

The numbers behind the gap are the same ones that frame the whole program. A 2026-09-26 snapshot from a public API node taken by `tooling/bts_chain_snapshot.py` at head block ~114706236 found order books holding about 15 dollars of depth per pair, so trading routes to the swap pools, and the pools disagree with each other by roughly 86 percent about the value of the chain's own reference assets, with the BTS/USD pool clearing 59.7 BTS per bitUSD while the BTS/CNY pool cleared 15.6 BTS per bitCNY, an implied cross-rate of about 3.8 CNY per USD (source: AGENTS.md, section 6, chain facts). Inside a pool the damage is smaller but still real: on a million-unit constant-product pool a 10,000 trade moves the price about one percent, so a trader buying a dollar token pays a dollar and a cent for a dollar that is supposed to cost a dollar. That cent is not a fee anyone collects; it is pure slippage, gone on the next arbitrage.

The cost lands hardest on the pairs that least need price discovery. A pool holding two wrappings of the same underlying coin, or two dollar tokens that track the same reference, has no fundamental price movement to price; the constant product curve imposes one anyway, because its whole construction assumes every trade shifts the ratio. The result is that pegged pairs pay for volatility they do not have, and the liquidity that would sit in them sits elsewhere or not at all.

StableSwap addresses this with a single change to the invariant. Near balance the curve is nearly flat, so a 10,000 swap in a million-unit pool returns 9,978, 99.8 percent, instead of the roughly 9,900 a constant-product pool quotes. As the pool becomes lopsided the curve steepens and approaches constant product, so a depeg still leaves a liquidity floor rather than a cliff. No keeper rebalances, no oracle feeds a target price, and no governance vote tunes an amplifier after the fact; the flatness is a property of the pair and of the curve, not of an operator.

Status, stated plainly. The proposal is a design. No code exists today, and the curve numbers are measured on the pure functions of the proposed invariant, a Python 3 reference implementation of the StableSwap equation run 2026-10-25 against the design's own arithmetic, to be re-verified at implementation. The existing constant-product pools, their operations, their objects, and their fee accounting are untouched, and nothing here activates anything.

# What changes for you

- If you hold `bit*`, `HONEST.*`, or gateway assets, nothing about your balances, markets, or fees changes, and no rule here can move, cap, or re-price them later either.
- If you are a liquidity provider, you can create a pool whose curve is flat near balance for a pegged pair, so the same capital quotes tighter prices for the same trade size, and you can withdraw from one side of a stable pool without taking the other.
- If you trade, you gain a venue where pegged pairs trade near parity with slippage measured in basis points rather than percent, while a depeg still leaves a liquidity floor instead of a cliff.
- If you witness, you gain no new duty: the curve runs in the evaluators you already run, behind one hardfork date the committee sets or does not set, and no new operation tags are introduced.
- If you operate a gateway, your IOUs are ordinary assets here too, eligible as pool assets unchanged, and a stable pool against your token is just a pool against a UIA.
- If you develop on BitShares, new fields append to the pool object and new typed extensions append to the existing pool operations, so no existing wire format changes, and the curve branch is selected by a creation-time flag.

# Specifications

This section is for developers and reviewers. Every formula cites the curve it comes from, and the full math lives in the design spec that must precede any build. Readers outside development can stop after the first paragraph of each subsection.

Consensus law throughout, matching the program's rules: no floats, no `unordered_*` containers, head-block time only; operations and extension fields appended, never inserted; every new operation and curve branch hardfork-gated in its evaluator and in the proposal hardfork visitor; bounded state; 128-bit comparison with declared rounding.

## 1. Opt-in at creation

A pool opts into StableSwap at creation through a typed extension carrying `curve_type = 1` and an amplification coefficient A, a positive integer. A creation that omits the extension produces exactly the bytes it always did, so existing creations stay valid, and constant-product pools (`curve_type = 0`, the default) are unaffected. The extension is appended to the existing pool-creation operation rather than introducing a new operation, so no new wire tag is consumed and no new object type is required.

Field-level detail, as proposed and to be fixed by the design spec:

- `curve_type`: an enumeration appended to the pool object, default 0 (constant product). Value 1 selects StableSwap. Value 2 selects the AB curve of BSIP-7. No other values are valid at creation; an unknown value refuses at validate.
- `amplification`: a 64-bit unsigned integer, present only when `curve_type = 1`, strictly greater than zero. Proposed validation refuses a value of zero and refuses any value that would overflow the 128-bit intermediate math of section 2; the exact ceiling is filed by the design spec.
- The extension itself is an appended struct on the pool-creation operation, so a pre-fork client that does not know the field still deserialises the operation and refuses the curve branch at apply when the flag has not passed.

Because participation is per-pool and at creation time, a pool's curve never changes after creation. Re-tuning requires a new pool, which keeps the invariant auditable: the curve a pool was created with is the curve it trades on for its whole life.

## 2. The curve

The invariant is Curve's StableSwap equation for two assets:

    A * n^n * sum(x_i) + D = A * D * n^n + D^(n+1) / (n^n * prod(x_i))

with n = 2, so n^n = 4 and the equation becomes

    4 * A * (x + y) + D = 4 * A * D + D^3 / (4 * x * y)

D replaces k as the stored invariant. At perfect balance D = x + y, so a balanced pool of 1,000,000 each has D = 2,000,000, a number with a plain meaning: D is the total balance the pool would hold if it were perfectly balanced at the current invariant. A interpolates between the two curves: A tending to zero degenerates to constant product, A tending to infinity approaches constant sum.

Two pure functions carry the curve:

- `compute_d(x, y, A)` solves the invariant for D by Newton iteration. It starts from D = x + y and iterates until the correction is zero or the iteration cap is reached. The iteration is deterministic: every node runs the same number of steps with the same integer rounding, so two nodes cannot disagree on D.
- `compute_new_y(x, y, A, x_new)` inverts the invariant for the other side given one side's new balance. It returns the balance of y that keeps D constant after x moves to x_new, which is the amount the trader receives (minus the fee).

Both work in 128-bit integers and never divide before multiplying, so two nodes cannot round differently and fork. Concretely:

- Every intermediate product is computed in 128-bit (`fc::uint128` / `__int128`) before any division.
- Division occurs only as the final step of a term, and the remainder is discarded in the direction that favours the pool (section 4).
- The Newton iteration cap is a compile-time constant; reaching it without convergence refuses the operation rather than returning a partial result.
- A zero balance on either side refuses the computation, since the product term divides by `x * y`; a pool with a zero side is not a tradable pool and must be re-capitalised or exited.

The fee mechanics of the pool are unchanged. The swap fee is charged on the output leg as it is today, and the StableSwap curve only changes how much output the pool quotes for a given input. The imbalanced-deposit and single-sided-withdrawal fees of section 3 are additional, charged on top of the existing fee schedule.

## 3. Imbalanced deposits and single-sided withdrawals

A deposit that does not match the pool's current ratio moves the pool along its curve and takes value from existing providers unless it pays for it. The imbalance fee is the swap fee scaled by `n / (4 * (n - 1))`, which for n = 2 is exactly half the swap fee. The fee is charged on the value of the excess side, the side whose deposit ratio exceeds the pool's current ratio, and is added to the pool's fee accumulator so it compounds into share value.

Single-sided withdrawal is the mirror image and carries the same fee. It exists so a provider who wants out of one side need not take the other and swap it back, paying the full swap fee twice. The withdrawal computes the share of the pool's invariant attributable to the withdrawn side, applies the half-fee to the imbalanced portion, and pays out the remainder in the requested asset.

Single-sided withdrawal is refused on constant-product pools, because there the payout has no bounded price: a constant-product pool's single-sided payout is unbounded in the limit and would let a provider extract value at the expense of the remaining side. On stable pools the payout is bounded by the curve's flat region and by the fee, so it is safe. The refusal is at validate, with a message naming the curve type, not at apply, so a client learns immediately.

## 4. Rounding

Every rounding step favours the pool. This is the single most important implementation rule in the proposal, and it is stated as a rule rather than as a hope. Concretely:

- `compute_d` rounds D down, so the stored invariant understates the pool's true value.
- `compute_new_y` rounds the output down, so the trader receives slightly less than the exact curve quote.
- The fee is computed on the rounded output and rounded up, so the pool keeps the fractional dust.
- Imbalanced-deposit and single-sided-withdrawal fees are rounded up.

Design intent, to be verified by a grinding harness at implementation: 200 fee-free A to B to A cycles leave the trader 200 units down and grow D from 2,000,000 to 2,000,200. The harness grinds every combination of small balances and small trade sizes and asserts that no sequence of trades can extract value from the pool through rounding. If any combination fails, the rounding direction is wrong and the implementation is not ready.

## 5. Hardfork gate

One flag, `HARDFORK_STABLESWAP_PASSED`, gates the extension, the curve branch, the imbalanced-deposit fee, and single-sided withdrawal together. Evaluators refuse at apply when the flag has not passed, using the established fail-closed pattern across the program's proposals. The flag is a dummy date in-tree, proposed as a committee-set parameter, and the gate is all-or-nothing: a build that has the curve branch but not the gate refuses every stable-pool operation.

## 6. Out of scope by design

- Generalisation to n > 2 is not proposed. The invariant formula supports arbitrary n, but the Newton iteration, the fee scaling, and the rounding all assume n = 2, and generalising needs its own design spec and its own audit.
- Dynamic A is not proposed. A is fixed at creation and never changes; re-tuning requires a new pool.
- No new operation tags, no new object types, no new API beyond what the existing pool operations and queries already expose. The curve is selected by a creation-time field and branched on in the evaluator.
- No migration of existing pools. A constant-product pool stays constant product for its whole life; its curve type is immutable.

## 7. Implementation notes

The curve is integer-only by construction. `compute_d` and `compute_new_y` use Newton iteration in 128-bit integers, and the iteration cap is a compile-time constant. No floating point enters consensus, and no `unordered_*` container holds pool state. Pool objects and their fields are appended in declaration order, and the extension struct is appended to the existing creation operation, so a pre-fork client still deserialises every operation it sees.

State is bounded. The proposal adds one appended field to the pool object (`curve_type`) and one appended struct to the creation operation (amplification, present only for `curve_type = 1`). No new object type, no new per-trade state, no new per-pool accumulator beyond what the existing pool already carries. A pool's curve type and amplification are immutable after creation, which keeps the invariant auditable for the pool's whole life.

Overflow is refused, not wrapped. Every intermediate product in `compute_d` and `compute_new_y` is computed in 128-bit before any division, and a parameter set or balance set that would overflow refuses at validate or at creation rather than producing a wrapped result. A zero balance on either side refuses the computation, since the product term divides by `x * y`; a pool with a zero side is not tradable and must be re-capitalised or exited.

The hardfork gate is all-or-nothing. `HARDFORK_STABLESWAP_PASSED` gates the extension, the curve branch, the imbalanced-deposit fee, and single-sided withdrawal together. Evaluators refuse at apply when the flag has not passed, using the established fail-closed pattern. A build that has the curve branch but not the gate refuses every stable-pool operation.

## 8. API and queries

No new API endpoints are introduced. The existing pool query calls are extended to return `curve_type` and, for stable pools, `amplification` and the stored invariant D. A client that does not know the fields ignores them and keeps working on constant-product pools. Single-sided withdrawal availability is reported as a boolean derived from `curve_type`, so a wallet can show or hide the option without hardcoding curve types.

## 9. Client-side considerations

Wallets and UIs should treat a stable pool as a distinct venue type for display purposes only: quote the slippage the curve actually returns, show the amplification coefficient, and warn when a trade would push the pool far enough along the curve that the quote diverges materially from parity. The on-chain behaviour is identical regardless of how the client labels it. Providers should be warned at creation that A is immutable and that an absurd A behaves as constant sum until one side runs to zero.

# Rationale

StableSwap is the right curve for pegged pairs because its flat region is a property of the pair, not of the market: no keeper is needed to rebalance, and on a depeg it degrades toward constant product, giving a liquidity floor for free. The alternative, a high-amplification CES curve, thins below constant product exactly when depth is needed most, because its concentration is achieved by shrinking the tails rather than by flattening the middle, so a depeg drains the pool faster than a constant-product pool would. StableSwap's flatness is in the middle, where pegged pairs trade, and its tails are the constant-product tails, where depegs resolve.

The cost of that choice is that A is fixed at creation and n is 2 only, both filed as open decisions. Fixed A is a feature rather than a limitation in this design: a tunable A is a governance knob that a captured committee can turn to subsidise a favoured pool, and the program's settled position is that parameters that affect pricing should be set once, in public, at creation, rather than adjusted later by vote. A pool whose A turns out wrong is a pool that gets less trade; it does not threaten the chain, because the curve degrades to constant product as imbalance grows, and a new pool with a better A can be created by anyone.

No new operation tags is a deliberate economy. The existing pool operations already carry creation, deposit, withdrawal, and swap, and the curve is a property of the pool object, not of the operation. Adding tags would double the operation count for no benefit and would fragment the wire-format pinning that the independent-vote property of the program depends on. The extension rides the existing operations, so a client that does not know the curve still deserialises every operation it sees.

Rounding favours the pool because the pool is the commons. A rounding direction that favours the trader can be ground down by repeated small trades, each extracting a fraction of a unit, and the grinding harness of section 4 is the evidence that the direction holds. The cost is that traders receive slightly less than the exact curve quote, which is already smaller than the constant-product quote, so the difference is negligible in practice and is the price of a pool that cannot be drained by rounding.

Rejected alternatives, each with its reason. A CES curve with dynamic concentration is rejected because its thin tails remove liquidity exactly when a depeg needs it, and dynamic rho is a heuristic with high audit surface. A tunable A updated by governance is rejected for the capture reason above. Generalising to n > 2 in the same proposal is rejected because the fee scaling and rounding all assume n = 2, and bundling the generalisation would delay the pegged-pair fix for a feature that has no demand yet. Migrating existing pools to the new curve is rejected because legacy stays untouched permanently and a parallel class costs nothing to defend.

# Risks and failure modes

- A has no upper bound in protocol. An absurd A behaves as constant sum until one side runs to zero. Numerically safe, because the 128-bit math and the zero-balance refusal keep the computation sound, but economically dangerous, because a constant-sum pool can be drained by a single large trade that exhausts the cheaper side. The mitigation is creation-time validation refusing A above a proposed ceiling, and the open decision is what that ceiling should be. A pool created with an absurd A is a pool whose provider chose it; the chain does not protect a provider from their own parameters.
- A is fixed at creation. Re-tuning needs a new pool, which is a friction but also the auditability property. A pool whose A is wrong loses trade to a better pool, which is the market correcting itself rather than a governance vote correcting it.
- n = 2 only. Generalising needs more work and its own audit. A three-asset stable pool is a different product with different fee scaling and different rounding, and bundling it here would delay the two-asset fix.
- Depeg dynamics are curve behaviour, not market behaviour. Real depeg speed and arbitrage timing need professional pricing review before any pool holds material value, and the pilot cap of $20,000 face value for 90 days applies to the first stable pool as it applies to every other new class in this program.
- Rounding error compounds. The grinding harness of section 4 is the defence, and it must run before the gate closes. A rounding direction that favours the trader, even by one unit, is a drain that 200 cycles turn into a measurable loss.
- State growth is bounded by construction: one appended field on the pool object and one appended struct on the creation operation. No new object type, no new per-trade state, no new per-pool accumulator beyond what the existing pool already carries.

# Backwards compatibility

No migration. Constant-product pools keep every behaviour they have, and a pool's curve type is immutable, so no existing pool can be re-typed after creation. New fields append to the pool object and new extensions append to the operations, so no existing wire format changes, and a wallet that does not know the field keeps working on constant-product pools. Where the flag has not passed, the extension and the curve branch refuse at apply, using the program's established fail-closed pattern. Existing UIAs, including gateway tokens, are eligible as pool assets unchanged, since a pool treats a UIA as an ordinary balance to custody and return.

# Test cases

Each row names behaviour to pin before any build, derived from what this document's specification already states, and the last rows are the properties the grinding harness and the audit depend on. The suite does not exist, and this section claims no recorded result; it claims what any implementation must refuse or match. Every row belongs to the design spec's assertion suite and to the wire-format pinning the Reference implementation section files.

- Curve convergence: `compute_d` and `compute_new_y` agree across nodes for A up to the proposed ceiling and for balances up to 128-bit range.
- Rounding grind: 200 fee-free A to B to A cycles grow the pool by 200 and shrink the trader by 200, for every combination the harness grinds.
- Imbalanced deposit: depositing 300/100 into a balanced 500/500 pool yields 1,994,277 share units, not 4,000,000, matching the measured design number.
- Single-sided withdrawal: refused on constant-product pools with the stated message; allowed and fee'd on stable pools.
- Zero-balance refusal: a pool with a zero side refuses the computation rather than dividing by zero.
- Newton cap: reaching the iteration cap without convergence refuses the operation.
- Extension pinning: a creation that omits the extension produces exactly the bytes it always did.
- Wire pinning: pool field and extensions append and hold stable across builds with and without the flag.
- Unknown curve type: a creation with an unknown `curve_type` refuses at validate.
- Zero amplification: a StableSwap creation with A = 0 refuses at validate.
- Immutability: a pool's `curve_type` and `amplification` cannot be changed after creation.
- Implementation note: no floating point enters consensus; the grinding harness runs in the test binary.
- Fail-closed gate: every stable-pool operation refuses at apply when `HARDFORK_STABLESWAP_PASSED` has not passed.
- Pre-fork deserialisation: a creation omitting the extension produces exactly the bytes it always did.
- Overflow refusal: a balance or parameter set that overflows the 128-bit intermediate refuses rather than wraps.
- Zero-side refusal: a pool with a zero balance refuses the computation rather than dividing by zero.

# Reference implementation

The proposal is a design. No code exists today. The design numbers are measured on the pure functions of the proposed curve: a Python 3 reference implementation of the StableSwap invariant, run 2026-10-25 against the design's own arithmetic, to be re-verified at implementation. No commit of a reference tree exists yet.

What still needs writing before any build: the design spec in `docs/superpowers/specs/` fixing the extension field order, the amplification ceiling, the Newton iteration cap, the grinding harness, and the wire-format pinning; the assertion suite including the rounding grind and the convergence tests; and the full-binary evidence gates every other proposal in this program has already run. None of that exists today, and this document claims none of it.

# Discussion

Contested threads, rejected alternatives, and the audit must-close list for this draft:

- Rejected alternatives and why: a CES curve with dynamic concentration, a tunable A updated by governance, generalisation to n > 2 in this proposal, and migration of existing pools. See the Rationale section for reasons.
- Contested threads: the amplification ceiling, whether n > 2 should be a follow-up proposal, the hardfork date, the pilot scale and whether the first stable pool is included in the $20,000 face cap, and whether professional pricing review is a gate or a recommendation.
- Audit must-close list: `build/audit-round-bsip-readiness/bsip-readiness-review.json` (BSIPR-01/03/12) and the per-BSIP rounds `build/audit-round-bsip{3..8}/`. Activation remains unauthorized until every row of `build/ACTIVATION-CHECKLIST.md` clears.

# Summary for Shareholders

- Cost: nothing to existing holders by construction. Legacy assets (`bit*`, `HONEST.*`, gateway UIAs) keep their semantics, fees, and settlement rules; no migration, no caps, no repricing.
- Who pays: new work is funded by opt-in fees, forfeits, pilot-capped issuance, and bonded producer stakes on the new asset class only, not by a standing BTS subsidy. Stable pools charge the same fee schedule as constant-product pools, so no new fee surface is created.
- What changes for existing holders: nothing, unless they opt into the new class. Every new field and extension is appended and hardfork-gated, so pre-fork nodes keep their current behaviour.
- Review date: any standing policy (parity share, fee schedule, subsidy) is re-reviewed against pilot evidence before activation; see `build/ACTIVATION-CHECKLIST.md`.

# See Also

- BSIP-1: `docs/bsips/bsip-1-gamified-smartcoins.md`
- BSIP-2: `docs/bsips/bsip-2-true-stealth.md`
- BSIP-3: `docs/bsips/bsip-3-xclaim-accs.md`
- BSIP-4: `docs/bsips/bsip-4-n-asset-pools.md`
- BSIP-5: `docs/bsips/bsip-5-open-order-rewards.md`
- BSIP-6: `docs/bsips/bsip-6-stableswap.md`
- BSIP-7: `docs/bsips/bsip-7-concentrated-liquidity.md`
- BSIP-8: `docs/bsips/bsip-8-pamm.md`
- Track map: `docs/TRACKS.md`
- Activation gates: `build/ACTIVATION-CHECKLIST.md`

# Copyright

This document is placed in the public domain.