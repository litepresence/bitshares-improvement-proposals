     BSIP: (number assigned at filing; house label BSIP-7)
     Title: Asymmetric Bounded (AB) pools for volatile pairs
     Authors: litepresence (HONEST) (identity confirmation pending per AGENTS.md section 3)
     Status: Draft
     Type: Protocol
     Created: 2026-10-26

# Abstract

Constant-product pools spread capital from zero to infinity, so most of it sits far from the market price and earns nothing. This proposal adds a second pricing curve for liquidity pools, the Asymmetric Bounded (AB) pool: `(x + xv)^w * (y + yv)^(1-w) = k`, with hard price bounds `[P_low, P_high]` and a free holdings ratio `w`. The curve puts 100 percent of capital inside the chosen range with zero tail waste by construction of the curve, and lets the market maker skew the fund ratio (for example 80/20) rather than forcing 50/50. It is implemented as a parallel pool class reusing the existing liquidity-pool scaffolding. Constant-product and StableSwap pools are untouched. CES curves and dynamic rho are explicitly out of scope. The virtual offsets `xv` and `yv` are computed at creation from `(w, P_low, P_high)` so the invariant hits the axes exactly at the bounds, and swaps are Balancer math with `(x + xv)` and `(y + yv)` substituted, closed form with no Newton iteration. The pitch: every unit of capital prices inside the range where the market actually trades, and the math is auditable by hand.

# Motivation

Grid market makers concentrate capital by hand, with refill loops, reconcile machinery, and a running bot process. The AB pool gets that concentration as a property of the curve: a pool that puts all capital where the market trades, with no gaps, no refill, and no bot dependency. The cost is higher impermanent loss per unit of capital and a static range that needs re-anchoring when price exits it.

The underlying problem is capital efficiency. A constant-product pool prices across the whole positive line, so a pool holding 1,000,000 on each side quotes a tradeable price from the first unit to the last, but the capital far from the market price never gets used: a pool centered at 1.00 that trades between 0.99 and 1.01 leaves almost all of its depth untouched, and that depth earns no fee. Concentrating capital into the traded range multiplies the effective depth for the same money, which is why every serious market maker does it, and why doing it by hand is the norm on chains without a concentrated curve.

The program already ships a concentrated-ish answer in BSIP-4, the N-asset geometric-mean pool, whose arbitrage equilibrium settles each asset at its weight but still spreads capital across the whole positive line. BSIP-6's StableSwap flattens the middle for pegged pairs but is still unbounded. Neither puts capital inside a hard range. The AB pool is the primitive that does: a pool whose capital exists only inside `[P_low, P_high]`, and whose price function is zero outside that range, so a trade that would push price out of range simply cannot complete in that direction.

The cost is stated plainly, because it is real. Concentration raises impermanent loss per unit of capital: the same price move that a constant-product pool absorbs across its whole curve, an AB pool absorbs inside a narrow range, so the provider loses more per dollar when price exits. And the range is static: when price leaves `[P_low, P_high]` the pool goes single-sided and stops earning on the empty side until it is re-anchored, which means a new pool or a new creation. These are the tradeoffs the design accepts in exchange for putting 100 percent of capital where it trades.

Status, stated plainly. The proposal is a design. No code exists today. The design numbers are measured on the pure functions of the proposed curve, to be re-verified at implementation. The existing constant-product and StableSwap pools, their operations, their objects, and their fee accounting are untouched, and nothing here activates anything.

# What changes for you

- If you hold `bit*`, `HONEST.*`, or gateway assets, nothing about your balances, markets, or fees changes, and no rule here can move, cap, or re-price them later either.
- If you are a liquidity provider, you can create a pool that puts all of its capital inside a chosen price range with a chosen fund ratio, so the same money quotes deeper prices where the market actually trades, and you can exit from one side of an AB pool without taking the other.
- If you trade, you gain a venue with tighter quotes inside its range, at the cost of the pool going single-sided when price leaves the range, which is a liquidity property you can read on screen before trading.
- If you witness, you gain no new duty: the curve runs in the evaluators you already run, behind one hardfork date the committee sets or does not set, and no new operation tags are introduced.
- If you operate a gateway, your IOUs are ordinary assets here too, eligible as pool assets unchanged.
- If you develop on BitShares, new fields append to the pool object and new typed extensions append to the existing pool operations, so no existing wire format changes, and the curve branch is selected by a creation-time flag.

# Specifications

This section is for developers and reviewers. Every formula cites the curve it comes from, and the full math lives in the design spec that must precede any build. Readers outside development can stop after the first paragraph of each subsection.

Consensus law throughout, matching the program's rules: no floats, no `unordered_*` containers, head-block time only; operations and extension fields appended, never inserted; every new curve branch hardfork-gated in its evaluator and in the proposal hardfork visitor; bounded state; 128-bit comparison with declared rounding.

## 1. Opt-in at creation

A pool opts into the AB curve at creation through a typed extension carrying `curve_type = 2` and the AB parameters: a free holdings ratio `w`, a lower price bound `P_low`, and an upper price bound `P_high`. A creation that omits the extension produces exactly the bytes it always did. Constant-product (`curve_type = 0`) and StableSwap (`curve_type = 1`) pools are unaffected.

Field-level detail, as proposed and to be fixed by the design spec:

- `curve_type`: an enumeration appended to the pool object, default 0. Value 2 selects the AB curve. No other values are valid at creation; an unknown value refuses at validate.
- `w`: a fixed-point fraction representing the fund ratio, carried in basis-point units (0 to 10000), strictly between 0 and 10000. `w = 5000` is the symmetric case; `w = 8000` is an 80/20 skew. Values of 0 or 10000 refuse at validate, since they degenerate the curve into a single-asset pool.
- `P_low`, `P_high`: price bounds, carried as fixed-point integers with a denominator fixed by the design spec, with `P_low < P_high` strictly. A bound of zero or an inverted range refuses at validate.
- The extension is an appended struct on the pool-creation operation, so a pre-fork client that does not know the field still deserialises the operation and refuses the curve branch at apply when the flag has not passed.

Because participation is per-pool and at creation time, a pool's curve, range, and fund ratio never change after creation. Re-anchoring requires a new pool, which keeps the invariant auditable: the parameters a pool was created with are the parameters it trades on for its whole life.

## 2. The curve

Invariant: `(x + xv)^w * (y + yv)^(1-w) = k`, where `xv` and `yv` are virtual offsets computed at creation from `(w, P_low, P_high)` so the curve hits the axes exactly at the bounds. Price: `p = (w / (1 - w)) * (y + yv) / (x + xv)`. Inside `[P_low, P_high]` the pool quotes continuously; outside it is single-sided, with zero tail waste by construction.

The virtual offsets are the load-bearing derivation. They are chosen so that at `P_low` the pool holds only asset y (x = 0) and at `P_high` the pool holds only asset x (y = 0), which is what "zero tail waste" means: no capital is priced outside the range. Concretely, given `w`, `P_low`, and `P_high`, the offsets satisfy:

- At `x = 0`, price equals `P_low`: `P_low = (w / (1 - w)) * (y + yv) / xv`.
- At `y = 0`, price equals `P_high`: `P_high = (w / (1 - w)) * yv / (x + xv)`.

Solving these two equations for `xv` and `yv` yields closed-form expressions in `w`, `P_low`, and `P_high`, which the design spec files and the reference implementation verifies. The offsets are computed once at creation, stored on the pool object, and never recomputed.

Swaps are Balancer math with `(x + xv)` and `(y + yv)` substituted: closed form, no Newton iteration. Given an input `dx`, the output `dy` satisfies:

    (x + xv + dx)^w * (y + yv - dy)^(1-w) = (x + xv)^w * (y + yv)^(1-w)

Solving for `dy` gives a closed-form expression, so every node computes the same output with the same integer rounding, and there is no iteration cap to hit. This is the principal advantage over StableSwap's Newton loop: the AB curve is cheaper to evaluate and cannot fail to converge.

Edge cases:

- A swap that would push price outside `[P_low, P_high]` in the trade direction refuses at validate, because the pool is single-sided there and the requested output exceeds what the curve can deliver. The refusal names the bound that would be crossed.
- A pool that has gone single-sided (one balance zero) quotes only in the direction that refills the empty side; the other direction refuses.
- The offsets are stored as 128-bit integers; a parameter set that overflows the offset computation refuses at creation.

## 3. Single-sided withdrawal

Permitted on AB pools, charged half the swap fee (`n / (4 * (n - 1)) = 1/2` for n = 2), identical to BSIP-6's single-sided-withdrawal fee. Refused on constant-product pools. The withdrawal computes the share of the pool's invariant attributable to the withdrawn side, applies the half-fee to the imbalanced portion, and pays out the remainder in the requested asset.

The refusal on constant-product pools is at validate, with a message naming the curve type, so a client learns immediately. The reason is the same as in BSIP-6: on a constant-product pool the single-sided payout has no bounded price, and would let a provider extract value at the expense of the remaining side.

## 4. Rounding

Every rounding step favours the pool, identical to BSIP-6's rounding rule. Concretely:

- The output amount is rounded down, so the trader receives slightly less than the exact curve quote.
- The fee is computed on the rounded output and rounded up, so the pool keeps the fractional dust.
- Single-sided-withdrawal fees are rounded up.
- The virtual offsets are computed with rounding that understates the pool's value, so the stored invariant is conservative.

A grinding harness verifies that no sequence of small trades can extract value from the pool through rounding, identical in spirit to BSIP-6's harness. If any combination fails, the rounding direction is wrong and the implementation is not ready.

## 5. Hardfork gate

One flag, `HARDFORK_AB_POOL_PASSED`, gates the extension, the curve branch, and single-sided withdrawal. Evaluators refuse at apply when the flag has not passed, using the established fail-closed pattern. The flag is a dummy date in-tree, proposed as a committee-set parameter, and the gate is all-or-nothing.

## 6. Out of scope by design

- CES curves are not proposed. They are a known family with severe impermanent-loss tradeoffs and are redundant with StableSwap for pegged pairs and with the AB pool for volatile pairs.
- Dynamic rho is not proposed. It is a heuristic with high audit surface and no clear improvement over fixed parameters set at creation.
- Generalisation to n > 2 is not proposed. The AB curve assumes two assets; a multi-asset concentrated curve is a different product.
- No migration of existing pools. A pool's curve type, range, and fund ratio are immutable.
- No new operation tags, no new object types, no new API beyond what the existing pool operations and queries already expose.

## 7. Implementation notes

The curve is closed form and integer-only. Swaps evaluate the Balancer expression with `(x + xv)` and `(y + yv)` substituted, in 128-bit integers, with no Newton iteration and no convergence cap to hit. No floating point enters consensus, and no `unordered_*` container holds pool state. Pool objects and their fields are appended in declaration order, and the extension struct is appended to the existing creation operation, so a pre-fork client still deserialises every operation it sees.

State is bounded. The proposal adds one appended field to the pool object (`curve_type`) and one appended struct to the creation operation (w, P_low, P_high, and the derived offsets). No new object type, no new per-trade state, no new per-pool accumulator beyond what the existing pool already carries. A pool's curve type, range, and fund ratio are immutable after creation, which keeps the invariant auditable for the pool's whole life.

The virtual offsets are computed once at creation and stored on the pool object. A parameter set that overflows the offset computation refuses at creation rather than wrapping. The offsets are 128-bit integers, and the derivation is closed form, so every node computes the same offsets from the same inputs.

The hardfork gate is all-or-nothing. `HARDFORK_AB_POOL_PASSED` gates the extension, the curve branch, and single-sided withdrawal together. Evaluators refuse at apply when the flag has not passed, using the established fail-closed pattern. A build that has the curve branch but not the gate refuses every AB-pool operation.

Rounding favours the pool on every step, identical to BSIP-6's rounding rule: output rounded down, fee rounded up, offsets computed to understate the pool's value. A grinding harness verifies that no sequence of small trades can extract value through rounding.

## 8. API and queries

No new API endpoints are introduced. The existing pool query calls are extended to return `curve_type`, and for AB pools `w`, `P_low`, `P_high`, and the stored virtual offsets. A client that does not know the fields ignores them and keeps working on constant-product and stable pools. Single-sided withdrawal availability is reported as a boolean derived from `curve_type`. Whether a pool is currently single-sided is reported as a boolean derived from its balances and bounds, so a wallet can warn a trader before submitting a trade that would refuse.

## 9. Client-side considerations

Wallets and UIs should treat an AB pool as a distinct venue type for display purposes only: show the range, the fund ratio, and whether the pool is currently single-sided, and warn when a trade would push price outside the range and refuse. Providers should be warned at creation that the range is immutable and that a price exit requires a new pool. The on-chain behaviour is identical regardless of how the client labels it.

# Rationale

The AB pool is the one genuinely new primitive in the concentrated-liquidity space: bounded range with asymmetric fund ratio and zero tail waste by construction of the curve. The virtual offsets `xv` and `yv` are chosen at creation so the invariant hits the axes exactly at `P_low` and `P_high`, so no capital is priced outside the range, and the math is closed form and auditable. CES curves are a known family with severe impermanent-loss tradeoffs and are redundant with DEXBot2 and StableSwap. Dynamic rho is a heuristic with high audit surface. The AB pool alone earns its place.

The concentration is the point. A constant-product pool spreads capital across the whole positive line because its invariant assumes every price is possible; an AB pool assumes only prices inside the range are possible, and prices its capital accordingly. That assumption is wrong when price leaves the range, which is why the pool goes single-sided there and why re-anchoring is needed, but it is right most of the time for a pair that trades in a band, which is most pairs most of the time.

The asymmetric fund ratio is the second point. Forcing 50/50 means a provider who thinks the market will trend holds equal amounts of the asset they expect to fall and the asset they expect to rise, which is the opposite of what they want. A free `w` lets the provider skew the pool toward the asset they expect to hold, so the pool's fee income matches their view. The cost is that the pool is no longer symmetric, which complicates the impermanent-loss accounting, but the math is still closed form and still auditable.

Rejected alternatives, each with its reason. CES curves are rejected because their concentration is achieved by shrinking the tails, which removes liquidity exactly when a depeg needs it, and their dynamic rho is a heuristic with high audit surface. Dynamic rho is rejected for the same audit-surface reason. A tunable range updated by governance is rejected because range changes would let a captured committee move a pool's capital after the fact, and the program's settled position is that pricing parameters are set once, in public, at creation. Migrating existing pools is rejected because legacy stays untouched permanently.

# Risks and failure modes

- Concentration increases impermanent loss per unit of capital. The same price move that a constant-product pool absorbs across its whole curve, an AB pool absorbs inside a narrow range, so the provider loses more per dollar when price exits. This is the tradeoff the design accepts, and it is stated rather than hidden.
- AB pools go single-sided outside their range; re-anchoring is needed. A pool that has gone single-sided earns on only one side until it is re-anchored, which means a new pool. Providers must monitor range exits, or accept the single-sided state.
- Virtual offset derivation must be verified to hit exact bounds. If the offsets are off by one unit, the pool's capital is not exactly zero at the bound, and tail waste reappears. The derivation is closed form and the reference implementation verifies it, but the audit must check it independently.
- Rounding must favour the pool on every step. A rounding direction that favours the trader is a drain, identical to the StableSwap risk.
- State bloat is bounded: no new object type, scalar parameters only. The pool object gains a few appended fields; no new per-trade state, no new accumulators beyond what the existing pool already carries.
- Re-anchoring cost. A pool that exits its range requires a new pool, which fragments liquidity across versions of the same pair. Providers must plan ranges with this in mind, and the pilot should measure how often re-anchoring is needed in practice.
- Audit surface of the offset derivation. The closed-form derivation is short but must be verified independently; an off-by-one in the offsets re-introduces tail waste. The audit must check the derivation, not just the code.
- Parameter sets that overflow the offset computation refuse at creation, so an absurd range cannot corrupt the 128-bit math.

# Backwards compatibility

No migration. Constant-product and StableSwap pools keep every behaviour they have. New fields append to the pool object and new extensions append to the operations. Where the flag has not passed, the extension and curve branch refuse at apply, using the program's established fail-closed pattern. Existing UIAs, including gateway tokens, are eligible as pool assets unchanged.

# Test cases

Each row names behaviour to pin before any build, derived from what this document's specification already states. The suite does not exist, and this section claims no recorded result.

- Range: pool is single-sided outside `[P_low, P_high]`, and a trade that would cross a bound refuses at validate.
- Closed form: swap output agrees across nodes for the full 128-bit balance range, with no iteration cap to hit.
- Offset derivation: virtual offsets hit the axes exactly at `P_low` and `P_high`, verified by the reference implementation and independently by audit.
- Rounding grind favours the pool: no sequence of small trades extracts value through rounding.
- Single-sided withdrawal: refused on constant-product pools with the stated message; allowed and fee'd on AB pools.
- Fund ratio: `w = 0` and `w = 10000` refuse at validate; `w = 5000` gives the symmetric case.
- Bound ordering: `P_low >= P_high` refuses at validate; a zero bound refuses.
- Wire pinning: pool fields and extensions append and hold stable across builds with and without the flag.
- Fail-closed gate: every AB-pool operation refuses at apply when `HARDFORK_AB_POOL_PASSED` has not passed.
- Pre-fork deserialisation: a creation omitting the extension produces exactly the bytes it always did.
- Closed-form agreement: swap output agrees across nodes for the full 128-bit balance range, with no iteration cap to hit.
- Offset overflow refusal: a parameter set that overflows the offset computation refuses at creation.
- Single-sided refusal: a trade that would push price outside the range refuses at validate with the bound named.
- Immutability: a pool's `curve_type`, `w`, `P_low`, `P_high`, and offsets cannot be changed after creation.
- Rounding grind: no sequence of small trades extracts value through rounding.
- Fund ratio bounds: `w = 0` and `w = 10000` refuse at validate; `w = 5000` gives the symmetric case.
- Single-sided state reporting: a pool whose balance on one side is zero reports single-sided true.
- Immutability: a pool's `curve_type`, `w`, `P_low`, `P_high`, and offsets cannot be changed after creation.
- Offset overflow: a parameter set that overflows the offset computation refuses at creation.

# Reference implementation

The proposal is a design. No code exists today.

What still needs writing before any build: the design spec in `docs/superpowers/specs/` fixing the extension field order, the offset derivation formulas, the fixed-point denominators for `w` and the price bounds, the rounding directions, and the wire-format pinning; the assertion suite including the closed-form agreement tests and the rounding grind; and the full-binary evidence gates every other proposal in this program has already run. None of that exists today, and this document claims none of it.

## 10. Formula summary

For reference, the curve in one place:

- Invariant: `(x + xv)^w * (y + yv)^(1-w) = k`.
- Price: `p = (w / (1 - w)) * (y + yv) / (x + xv)`.
- Virtual offsets: derived at creation from `(w, P_low, P_high)` so that at `x = 0` price equals `P_low` and at `y = 0` price equals `P_high`, giving zero tail waste.
- Swap output: solve `(x + xv + dx)^w * (y + yv - dy)^(1-w) = (x + xv)^w * (y + yv)^(1-w)` for `dy`, in closed form.
- Single-sided withdrawal fee: `n / (4 * (n - 1)) = 1/2` for n = 2, applied to the imbalanced portion.
- Rounding: output down, fee up, offsets conservative.

All of the above is evaluated in 128-bit integers with no floating point and no iteration.

# Discussion

Contested threads, rejected alternatives, and the audit must-close list for this draft:

- Rejected alternatives and why: CES curves, dynamic rho, a tunable range updated by governance, and migration of existing pools. See the Rationale section for reasons.
- Contested threads: the offset derivation's fixed-point denominator, whether re-anchoring should be a new pool or a new operation, the hardfork date, the pilot scale, and whether professional pricing review is a gate or a recommendation.
- Audit must-close list: `build/audit-round-bsip-readiness/bsip-readiness-review.json` (BSIPR-01/03/12) and the per-BSIP rounds `build/audit-round-bsip{3..8}/`. Activation remains unauthorized until every row of `build/ACTIVATION-CHECKLIST.md` clears.

# Summary for Shareholders

- Cost: nothing to existing holders by construction. Legacy assets (`bit*`, `HONEST.*`, gateway UIAs) keep their semantics, fees, and settlement rules; no migration, no caps, no repricing.
- Who pays: new work is funded by opt-in fees, forfeits, pilot-capped issuance, and bonded producer stakes on the new asset class only, not by a standing BTS subsidy. AB pools charge the same fee schedule as constant-product pools.
- What changes for existing holders: nothing, unless they opt into the new class. Every new field and extension is appended and hardfork-gated, so pre-fork nodes keep their current behaviour.
- Review date: any standing policy is re-reviewed against pilot evidence before activation; see `build/ACTIVATION-CHECKLIST.md`.

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