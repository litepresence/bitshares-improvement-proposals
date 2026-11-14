<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-6 — StableSwap pools for pegged pairs: flat near the peg, constant product in a depeg

> **Whitepaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Table of contents

1. Narrative
2. Specification
3. Theory
4. Exploit analysis
5. Gas and cost analysis
6. Error code registry
7. Admin architecture
8. Max pool and set sizes
9. Curve parameters
10. Glossary
11. Provenance
12. Test cases

---

## 1. Narrative

Pegged pairs dominate stablecoin and bridged-asset markets, and on those pairs constant product is the wrong curve. The slippage it charges buys nothing: both trader and provider lose the spread for no economic reason, because the price was never supposed to move. Competitive stable-swap venues exist for exactly this reason. The fix is a curve whose whole point is to be flat where the pair is supposed to trade and to fall back to the existing curve only when something has gone wrong.

BitShares trade machinery has never been the weak part. Deterministic order books, a margin engine, constant-product swap pools, and native hash-time-locked contracts compare well with what most chains run. The weak part is depth and trust. A 2026-09-26 snapshot from a public API node taken by `tooling/bts_chain_snapshot.py` at head block ~114706236 found order books holding about 15 dollars of depth per pair, so trading routes to the swap pools, and the pools disagree with each other by roughly 86 percent about the value of the chain's own reference assets, with the BTS/USD pool clearing 59.7 BTS per bitUSD while the BTS/CNY pool cleared 15.6 BTS per bitCNY, an implied cross-rate of about 3.8 CNY per USD. Inside a pool the damage is smaller but still real: on a million-unit constant-product pool a 10,000 trade moves the price about one percent, so a trader buying a dollar token pays a dollar and a cent for a dollar that is supposed to cost a dollar. That cent is not a fee anyone collects; it is pure slippage, gone on the next arbitrage.

The cost lands hardest on the pairs that least need price discovery. A pool holding two wrappings of the same underlying coin, or two dollar tokens that track the same reference, has no fundamental price movement to price; the constant product curve imposes one anyway, because its whole construction assumes every trade shifts the ratio. The result is that pegged pairs pay for volatility they do not have, and the liquidity that would sit in them sits elsewhere or not at all.

StableSwap addresses this with a single change to the invariant. Near balance the curve is nearly flat, so a 10,000 swap in a million-unit pool returns 9,978, 99.8 percent, instead of the roughly 9,900 a constant-product pool quotes. As the pool becomes lopsided the curve steepens and approaches constant product, so a depeg still leaves a liquidity floor rather than a cliff. No keeper rebalances, no oracle feeds a target price, and no governance vote tunes an amplifier after the fact; the flatness is a property of the pair and of the curve, not of an operator.

This proposal is one of several independently votable pool proposals. It adds a second pricing curve for liquidity pools, Curve's StableSwap invariant with an amplification coefficient A, aimed at pegged pairs: dollar tokens, wrappings of the same coin, bridged twins. Participation is per-pool at creation time; constant-product pools are untouched. No new operation tags are needed: the curve rides the existing pool operations through typed extensions. The amplification coefficient A is fixed at creation, the pool holds exactly two assets, and every rounding step favours the pool.

Status, stated plainly. The proposal is a design. No code exists today, and the curve numbers are measured on the pure functions of the proposed invariant, a Python 3 reference implementation of the StableSwap equation run 2026-10-25 against the design's own arithmetic, to be re-verified at implementation. The existing constant-product pools, their operations, their objects, and their fee accounting are untouched, and nothing here activates anything.

## 2. Specification

Consensus law throughout, matching the program's rules: no floats, no `unordered_*` containers, head-block time only; operations and extension fields appended, never inserted; every new curve branch hardfork-gated in its evaluator and in the proposal hardfork visitor; bounded state; 128-bit comparison with declared rounding.

### 2.1 Opt-in at creation

A pool opts into StableSwap at creation through a typed extension carrying `curve_type = 1` and an amplification coefficient A, a positive integer. A creation that omits the extension produces exactly the bytes it always did, so existing creations stay valid, and constant-product pools (`curve_type = 0`, the default) are unaffected. The extension is appended to the existing pool-creation operation rather than introducing a new operation, so no new wire tag is consumed and no new object type is required.

Field-level detail, as proposed and to be fixed by the design spec:

- `curve_type`: an enumeration appended to the pool object, default 0 (constant product). Value 1 selects StableSwap. Value 2 selects the AB curve of BSIP-7. No other values are valid at creation; an unknown value refuses at validate.
- `amplification`: a 64-bit unsigned integer, present only when `curve_type = 1`, strictly greater than zero. Proposed validation refuses a value of zero and refuses any value that would overflow the 128-bit intermediate math of section 2.2; the exact ceiling is filed by the design spec.
- The extension itself is an appended struct on the pool-creation operation, so a pre-fork client that does not know the field still deserialises the operation and refuses the curve branch at apply when the flag has not passed.

Because participation is per-pool and at creation time, a pool's curve never changes after creation. Re-tuning requires a new pool, which keeps the invariant auditable: the curve a pool was created with is the curve it trades on for its whole life.

### 2.2 The curve

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
- Division occurs only as the final step of a term, and the remainder is discarded in the direction that favours the pool (section 2.4).
- The Newton iteration cap is a compile-time constant; reaching it without convergence refuses the operation rather than returning a partial result.
- A zero balance on either side refuses the computation, since the product term divides by `x * y`; a pool with a zero side is not a tradable pool and must be re-capitalised or exited.

The fee mechanics of the pool are unchanged. The swap fee is charged on the output leg as it is today, and the StableSwap curve only changes how much output the pool quotes for a given input. The imbalanced-deposit and single-sided-withdrawal fees of section 2.3 are additional, charged on top of the existing fee schedule.

### 2.3 Imbalanced deposits and single-sided withdrawals

A deposit that does not match the pool's current ratio moves the pool along its curve and takes value from existing providers unless it pays for it. The imbalance fee is the swap fee scaled by `n / (4 * (n - 1))`, which for n = 2 is exactly half the swap fee. The fee is charged on the value of the excess side, the side whose deposit ratio exceeds the pool's current ratio, and is added to the pool's fee accumulator so it compounds into share value.

Single-sided withdrawal is the mirror image and carries the same fee. It exists so a provider who wants out of one side need not take the other and swap it back, paying the full swap fee twice. The withdrawal computes the share of the pool's invariant attributable to the withdrawn side, applies the half-fee to the imbalanced portion, and pays out the remainder in the requested asset.

Single-sided withdrawal is refused on constant-product pools, because there the payout has no bounded price: a constant-product pool's single-sided payout is unbounded in the limit and would let a provider extract value at the expense of the remaining side. On stable pools the payout is bounded by the curve's flat region and by the fee, so it is safe. The refusal is at validate, with a message naming the curve type, not at apply, so a client learns immediately.

### 2.4 Rounding

Every rounding step favours the pool. This is the single most important implementation rule in the proposal, and it is stated as a rule rather than as a hope. Concretely:

- `compute_d` rounds D down, so the stored invariant understates the pool's true value.
- `compute_new_y` rounds the output down, so the trader receives slightly less than the exact curve quote.
- The fee is computed on the rounded output and rounded up, so the pool keeps the fractional dust.
- Imbalanced-deposit and single-sided-withdrawal fees are rounded up.

Design intent, to be verified by a grinding harness at implementation: 200 fee-free A to B to A cycles leave the trader 200 units down and grow D from 2,000,000 to 2,000,200. The harness grinds every combination of small balances and small trade sizes and asserts that no sequence of trades can extract value from the pool through rounding. If any combination fails, the rounding direction is wrong and the implementation is not ready.

### 2.5 Hardfork gate

One flag, `HARDFORK_STABLESWAP_PASSED`, gates the extension, the curve branch, the imbalanced-deposit fee, and single-sided withdrawal together. Evaluators refuse at apply when the flag has not passed, using the established fail-closed pattern across the program's proposals. The flag is a dummy date in-tree, proposed as a committee-set parameter, and the gate is all-or-nothing: a build that has the curve branch but not the gate refuses every stable-pool operation.

### 2.6 Out of scope by design

- Generalisation to n > 2 is not proposed. The invariant formula supports arbitrary n, but the Newton iteration, the fee scaling, and the rounding all assume n = 2, and generalising needs its own design spec and its own audit.
- Dynamic A is not proposed. A is fixed at creation and never changes; re-tuning requires a new pool.
- No new operation tags, no new object types, no new API beyond what the existing pool operations and queries already expose. The curve is selected by a creation-time field and branched on in the evaluator.
- No migration of existing pools. A constant-product pool stays constant product for its whole life; its curve type is immutable.

## 3. Theory

The invariants the implementation must preserve:

- No floats anywhere in consensus; all math is integer or 128-bit fixed point.
- Every new field and extension is appended, never inserted, and hardfork-gated in its evaluator and in the proposal visitor.
- State is bounded: one appended field on the pool object and one appended struct on the creation operation. No new object type, no new per-trade state, no new per-pool accumulator beyond what the existing pool already carries.
- Every rounding step favours the pool; no trade sequence can extract value through rounding.
- A pool's curve type and amplification are immutable after creation.
- Legacy pools are untouched by construction.

Overflow discipline: 128-bit comparison for bond and coverage math, 256-bit conversion where a product of two 128-bit values is needed, and saturation on price conversion so a wrap becomes a refusal rather than a silent value. For the curve, every intermediate product in `compute_d` and `compute_new_y` is computed in 128-bit before any division, and a parameter set or balance set that would overflow refuses at validate or at creation rather than producing a wrapped result.

## 4. Exploit analysis

- Rounding drain: a rounding direction that favours the trader can be ground down by repeated small trades, each extracting a fraction of a unit. The grinding harness of section 2.4 is the defence, and it must run before the gate closes.
- Absurd amplification: an A with no upper bound behaves as constant sum until one side runs to zero. Numerically safe, because the 128-bit math and the zero-balance refusal keep the computation sound, but economically dangerous, because a constant-sum pool can be drained by a single large trade that exhausts the cheaper side. The mitigation is creation-time validation refusing A above a proposed ceiling, and the open decision is what that ceiling should be. A pool created with an absurd A is a pool whose provider chose it; the chain does not protect a provider from their own parameters.
- Fixed A friction: re-tuning needs a new pool. A pool whose A is wrong loses trade to a better pool, which is the market correcting itself rather than a governance vote correcting it.
- n = 2 only: generalising needs more work and its own audit. A three-asset stable pool is a different product with different fee scaling and different rounding, and bundling it here would delay the two-asset fix.
- Depeg dynamics are curve behaviour, not market behaviour. Real depeg speed and arbitrage timing need professional pricing review before any pool holds material value, and the pilot cap of $20,000 face value for 90 days applies to the first stable pool as it applies to every other new class in this program.
- State bloat is bounded by construction: one appended field on the pool object and one appended struct on the creation operation. No new object type, no new per-trade state, no new per-pool accumulator beyond what the existing pool already carries.

## 5. Gas and cost analysis

Operations are consensus objects, not smart contract calls, so cost is measured in transaction fees and block weight rather than gas. The curve adds no new operation and no new per-trade state: `compute_d` runs at creation and on imbalance, `compute_new_y` runs on every swap, and both are closed-form-ish Newton iterations bounded by a compile-time cap. The cost is witness time during swaps, which scales with the iteration count, not with pool size. Imbalanced deposits and single-sided withdrawals add one fee computation on top of the existing deposit and withdrawal paths.

## 6. Error code registry

New BSIP-6 errors occupy a reserved range appended after the existing registry. Each error maps to one refusal condition: unknown curve type, zero amplification, amplification overflow, zero balance on a side, Newton iteration cap reached without convergence, single-sided withdrawal on a constant-product pool, imbalanced deposit below the fee floor, hardfork gate not passed. Legacy error codes keep their numbers.

## 7. Admin architecture

Admin is split between consensus and the pool creator. Consensus owns the curve evaluation, the rounding, the imbalanced-deposit fee, and single-sided withdrawal gating. The pool creator chooses `curve_type` and A at creation, and those choices are immutable thereafter. No admin key can change a pool's curve or amplification after creation; re-tuning requires a new pool. The hardfork gate is a committee-set date, not a per-pool parameter.

## 8. Max pool and set sizes

- Pool assets: exactly two.
- Curve types: 0 (constant product), 1 (StableSwap), 2 (AB, BSIP-7). No other values valid at creation.
- Amplification: positive 64-bit integer, ceiling filed by the design spec.
- Newton iteration cap: compile-time constant, filed by the design spec.
- Imbalance fee scaling: `n / (4 * (n - 1))` = 1/2 for n = 2.
- State growth: one appended field on the pool object, one appended struct on the creation operation.

## 9. Curve parameters

StableSwap has no oracle window: it is a pure curve with no external feed. The only time-dependent behaviour is the swap itself, evaluated at the block in which it is included. The imbalanced-deposit and single-sided-withdrawal fees are charged at the transaction's block height. All windows, where any, run on head-block time.

## 10. Glossary

- StableSwap: Curve's StableSwap invariant with amplification A, flat near balance.
- A: amplification coefficient, fixed at creation, interpolates between constant product and constant sum.
- D: stored invariant, the total balance at perfect balance.
- compute_d: Newton solve for D.
- compute_new_y: inversion for the output side.
- Imbalanced deposit: a deposit off ratio, charged half the swap fee on the excess side.
- Single-sided withdrawal: withdrawal of one asset from a stable pool, same half-fee.
- Rounding grind: the harness that verifies no trade sequence extracts value.
- Zero tail waste: no capital priced outside the range (AB pools, BSIP-7).

## 11. Provenance

This whitepaper adapts the structure of the Prism and Reciprocity whitepapers in the salvaged reference tree. The concepts are original to the BitShares Track B program. The design detail lives in `docs/superpowers/specs/` (to be filed) and the BSIP draft at `docs/bsips/bsip-6-stableswap.md`. No reference implementation exists yet; the curve numbers are measured on a Python 3 reference implementation of the StableSwap invariant run 2026-10-25.

## 12. Test cases

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
- Overflow refusal: a balance or parameter set that overflows the 128-bit intermediate refuses rather than wraps.
- Fail-closed gate: every stable-pool operation refuses at apply when `HARDFORK_STABLESWAP_PASSED` has not passed.
- Pre-fork deserialisation: a creation omitting the extension produces exactly the bytes it always did.

# Copyright

This document is placed in the public domain.