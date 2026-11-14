<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-7 — Asymmetric Bounded (AB) pools for volatile pairs

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

Grid market makers concentrate capital by hand, with refill loops, reconcile machinery, and a running bot process. The AB pool gets that concentration as a property of the curve: a pool that puts all capital where the market trades, with no gaps, no refill, and no bot dependency. The cost is higher impermanent loss per unit of capital and a static range that needs re-anchoring when price exits it.

The underlying problem is capital efficiency. A constant-product pool prices across the whole positive line, so a pool holding 1,000,000 on each side quotes a tradeable price from the first unit to the last, but the capital far from the market price never gets used: a pool centered at 1.00 that trades between 0.99 and 1.01 leaves almost all of its depth untouched, and that depth earns no fee. Concentrating capital into the traded range multiplies the effective depth for the same money, which is why every serious market maker does it, and why doing it by hand is the norm on chains without a concentrated curve.

The program already ships a concentrated-ish answer in BSIP-4, the N-asset geometric-mean pool, whose arbitrage equilibrium settles each asset at its weight but still spreads capital across the whole positive line. BSIP-6's StableSwap flattens the middle for pegged pairs but is still unbounded. Neither puts capital inside a hard range. The AB pool is the primitive that does: a pool whose capital exists only inside `[P_low, P_high]`, and whose price function is zero outside that range, so a trade that would push price out of range simply cannot complete in that direction.

The cost is stated plainly, because it is real. Concentration raises impermanent loss per unit of capital: the same price move that a constant-product pool absorbs across its whole curve, an AB pool absorbs inside a narrow range, so the provider loses more per dollar when price exits. And the range is static: when price leaves `[P_low, P_high]` the pool goes single-sided and stops earning on the empty side until it is re-anchored, which means a new pool or a new creation. These are the tradeoffs the design accepts in exchange for putting 100 percent of capital where it trades.

This proposal adds a second pricing curve for liquidity pools, the Asymmetric Bounded (AB) pool: `(x + xv)^w * (y + yv)^(1-w) = k`, with hard price bounds `[P_low, P_high]` and a free holdings ratio `w`. The curve puts 100 percent of capital inside the chosen range with zero tail waste by construction of the curve, and lets the market maker skew the fund ratio (for example 80/20) rather than forcing 50/50. It is implemented as a parallel pool class reusing the existing liquidity-pool scaffolding. Constant-product and StableSwap pools are untouched. CES curves and dynamic rho are explicitly out of scope. The virtual offsets `xv` and `yv` are computed at creation from `(w, P_low, P_high)` so the invariant hits the axes exactly at the bounds, and swaps are Balancer math with `(x + xv)` and `(y + yv)` substituted, closed form with no Newton iteration. The pitch: every unit of capital prices inside the range where the market actually trades, and the math is auditable by hand.

Status, stated plainly. The proposal is a design. No code exists today. The design numbers are measured on the pure functions of the proposed curve, to be re-verified at implementation. The existing constant-product and StableSwap pools, their operations, their objects, and their fee accounting are untouched, and nothing here activates anything.

## 2. Specification

Consensus law throughout, matching the program's rules: no floats, no `unordered_*` containers, head-block time only; operations and extension fields appended, never inserted; every new curve branch hardfork-gated in its evaluator and in the proposal hardfork visitor; bounded state; 128-bit comparison with declared rounding.

### 2.1 Opt-in at creation

A pool opts into the AB curve at creation through a typed extension carrying `curve_type = 2` and the AB parameters: a free holdings ratio `w`, a lower price bound `P_low`, and an upper price bound `P_high`. A creation that omits the extension produces exactly the bytes it always did. Constant-product (`curve_type = 0`) and StableSwap (`curve_type = 1`) pools are unaffected.

Field-level detail, as proposed and to be fixed by the design spec:

- `curve_type`: an enumeration appended to the pool object, default 0. Value 2 selects the AB curve. No other values are valid at creation; an unknown value refuses at validate.
- `w`: a fixed-point fraction representing the fund ratio, carried in basis-point units (0 to 10000), strictly between 0 and 10000. `w = 5000` is the symmetric case; `w = 8000` is an 80/20 skew. Values of 0 or 10000 refuse at validate, since they degenerate the curve into a single-asset pool.
- `P_low`, `P_high`: price bounds, carried as fixed-point integers with a denominator fixed by the design spec, with `P_low < P_high` strictly. A bound of zero or an inverted range refuses at validate.
- The extension is an appended struct on the pool-creation operation, so a pre-fork client that does not know the field still deserialises the operation and refuses the curve branch at apply when the flag has not passed.

Because participation is per-pool and at creation time, a pool's curve, range, and fund ratio never change after creation. Re-anchoring requires a new pool, which keeps the invariant auditable: the parameters a pool was created with are the parameters it trades on for its whole life.

### 2.2 The curve

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

### 2.3 Single-sided withdrawal

Permitted on AB pools, charged half the swap fee (`n / (4 * (n - 1)) = 1/2` for n = 2), identical to BSIP-6's single-sided-withdrawal fee. Refused on constant-product pools. The withdrawal computes the share of the pool's invariant attributable to the withdrawn side, applies the half-fee to the imbalanced portion, and pays out the remainder in the requested asset.

The refusal on constant-product pools is at validate, with a message naming the curve type, so a client learns immediately. The reason is the same as in BSIP-6: on a constant-product pool the single-sided payout has no bounded price, and would let a provider extract value at the expense of the remaining side.

### 2.4 Rounding

Every rounding step favours the pool, identical to BSIP-6's rounding rule. Concretely:

- The output amount is rounded down, so the trader receives slightly less than the exact curve quote.
- The fee is computed on the rounded output and rounded up, so the pool keeps the fractional dust.
- Single-sided-withdrawal fees are rounded up.
- The virtual offsets are computed with rounding that understates the pool's value, so the stored invariant is conservative.

A grinding harness verifies that no sequence of small trades can extract value from the pool through rounding, identical in spirit to BSIP-6's harness. If any combination fails, the rounding direction is wrong and the implementation is not ready.

### 2.5 Hardfork gate

One flag, `HARDFORK_AB_POOL_PASSED`, gates the extension, the curve branch, and single-sided withdrawal. Evaluators refuse at apply when the flag has not passed, using the established fail-closed pattern. The flag is a dummy date in-tree, proposed as a committee-set parameter, and the gate is all-or-nothing.

### 2.6 Out of scope by design

- CES curves are not proposed. They are a known family with severe impermanent-loss tradeoffs and are redundant with StableSwap for pegged pairs and with the AB pool for volatile pairs.
- Dynamic rho is not proposed. It is a heuristic with high audit surface and no clear improvement over fixed parameters set at creation.
- Generalisation to n > 2 is not proposed. The AB curve assumes two assets; a multi-asset concentrated curve is a different product.
- No migration of existing pools. A pool's curve type, range, and fund ratio are immutable.
- No new operation tags, no new object types, no new API beyond what the existing pool operations and queries already expose.

## 3. Theory

The invariants the implementation must preserve:

- No floats anywhere in consensus; all math is integer or 128-bit fixed point.
- Every new field and extension is appended, never inserted, and hardfork-gated in its evaluator and in the proposal visitor.
- State is bounded: one appended field on the pool object (`curve_type`) and one appended struct on the creation operation (w, P_low, P_high, and the derived offsets). No new object type, no new per-trade state, no new per-pool accumulator beyond what the existing pool already carries.
- A pool's curve type, range, and fund ratio are immutable after creation.
- Rounding favours the pool on every step.
- Legacy pools are untouched by construction.

Overflow discipline: the virtual offsets are 128-bit integers, and a parameter set that overflows the offset computation refuses at creation rather than wrapping. The offsets are derived in closed form, so every node computes the same offsets from the same inputs, with no iteration and no convergence question.

## 4. Exploit analysis

- Concentration increases impermanent loss per unit of capital. The same price move that a constant-product pool absorbs across its whole curve, an AB pool absorbs inside a narrow range, so the provider loses more per dollar when price exits. This is the tradeoff the design accepts, and it is stated rather than hidden.
- AB pools go single-sided outside their range; re-anchoring is needed. A pool that has gone single-sided earns on only one side until it is re-anchored, which means a new pool. Providers must monitor range exits, or accept the single-sided state.
- Virtual offset derivation must be verified to hit exact bounds. If the offsets are off by one unit, the pool's capital is not exactly zero at the bound, and tail waste reappears. The derivation is closed form and the reference implementation verifies it, but the audit must check it independently.
- Rounding must favour the pool on every step. A rounding direction that favours the trader is a drain, identical to the StableSwap risk.
- State bloat is bounded: no new object type, scalar parameters only. The pool object gains a few appended fields; no new per-trade state, no new accumulators beyond what the existing pool already carries.
- Re-anchoring cost: a pool that exits its range requires a new pool, which fragments liquidity across versions of the same pair. Providers must plan ranges with this in mind, and the pilot should measure how often re-anchoring is needed in practice.
- Audit surface of the offset derivation: the closed-form derivation is short but must be verified independently; an off-by-one in the offsets re-introduces tail waste. The audit must check the derivation, not just the code.
- Parameter sets that overflow the offset computation refuse at creation, so an absurd range cannot corrupt the 128-bit math.

## 5. Gas and cost analysis

Operations are consensus objects, not smart contract calls, so cost is measured in transaction fees and block weight rather than gas. The AB curve is cheaper to evaluate than StableSwap: swaps evaluate the Balancer expression in closed form with no Newton iteration and no convergence cap to hit. The cost is a fixed amount of 128-bit integer work per swap, independent of pool size. Single-sided withdrawal adds one fee computation on top of the existing withdrawal path. Creation computes the virtual offsets once, in closed form.

## 6. Error code registry

New BSIP-7 errors occupy a reserved range appended after the existing registry. Each error maps to one refusal condition: unknown curve type, fund ratio out of range (0 or 10000), inverted or zero price bound, offset overflow at creation, swap pushing price outside the range, single-sided pool refusing the emptying direction, single-sided withdrawal on a constant-product pool, hardfork gate not passed. Legacy error codes keep their numbers.

## 7. Admin architecture

Admin is split between consensus and the pool creator. Consensus owns the curve evaluation, the offset derivation, the rounding, and the range enforcement. The pool creator chooses `w`, `P_low`, and `P_high` at creation, and those choices are immutable thereafter. No admin key can change a pool's range or fund ratio after creation; re-anchoring requires a new pool. The hardfork gate is a committee-set date, not a per-pool parameter.

## 8. Max pool and set sizes

- Pool assets: exactly two.
- Curve types: 0 (constant product), 1 (StableSwap, BSIP-6), 2 (AB). No other values valid at creation.
- Fund ratio `w`: 1 to 9999 basis points (0 and 10000 refuse).
- Price bounds: `P_low < P_high` strictly, neither zero.
- Virtual offsets: 128-bit integers, computed once at creation.
- State growth: one appended field on the pool object, one appended struct on the creation operation.

## 9. Curve parameters

The AB pool has no oracle window: it is a pure curve with no external feed. The only time-dependent behaviour is the swap itself, evaluated at the block in which it is included. Range exit and single-sided state are evaluated from balances, not from time. All behaviour runs on head-block time where any time is involved.

## 10. Glossary

- AB pool: Asymmetric Bounded pool, `(x + xv)^w * (y + yv)^(1-w) = k`.
- w: free holdings ratio, basis points, fund ratio skew.
- P_low, P_high: price bounds; capital prices only inside them.
- xv, yv: virtual offsets, computed at creation, hit the axes at the bounds.
- Zero tail waste: no capital priced outside the range.
- Single-sided: pool with one balance zero, quotes only the refilling direction.
- Re-anchoring: creating a new pool when price exits the range.
- Closed form: no Newton iteration, no convergence cap.

## 11. Provenance

This whitepaper adapts the structure of the Prism and Reciprocity whitepapers in the salvaged reference tree. The concepts are original to the BitShares Track B program. The design detail lives in `docs/superpowers/specs/` (to be filed) and the BSIP draft at `docs/bsips/bsip-7-concentrated-liquidity.md`. No reference implementation exists yet; the curve numbers are measured on the pure functions of the proposed curve.

## 12. Test cases

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
- Offset overflow refusal: a parameter set that overflows the offset computation refuses at creation.
- Single-sided refusal: a trade that would push price outside the range refuses at validate with the bound named.
- Immutability: a pool's `curve_type`, `w`, `P_low`, `P_high`, and offsets cannot be changed after creation.
- Single-sided state reporting: a pool whose balance on one side is zero reports single-sided true.

# Copyright

This document is placed in the public domain.