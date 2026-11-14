<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-7 — Asymmetric Bounded (AB) pools (lightpaper)

> **Lightpaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Abstract

Constant-product pools spread capital from zero to infinity, so most of it sits far from the market price and earns nothing. This proposal adds a second pricing curve for liquidity pools, the Asymmetric Bounded (AB) pool: `(x + xv)^w * (y + yv)^(1-w) = k`, with hard price bounds `[P_low, P_high]` and a free holdings ratio `w`. The curve puts 100 percent of capital inside the chosen range with zero tail waste by construction of the curve, and lets the market maker skew the fund ratio (for example 80/20) rather than forcing 50/50. It is implemented as a parallel pool class reusing the existing liquidity-pool scaffolding. Constant-product and StableSwap pools are untouched. CES curves and dynamic rho are explicitly out of scope. The virtual offsets `xv` and `yv` are computed at creation from `(w, P_low, P_high)` so the invariant hits the axes exactly at the bounds, and swaps are Balancer math with `(x + xv)` and `(y + yv)` substituted, closed form with no Newton iteration. The pitch: every unit of capital prices inside the range where the market actually trades, and the math is auditable by hand.

## What changes for you

- Legacy asset holders: nothing changes; no rule here can move, cap, or re-price legacy assets.
- Liquidity providers: you can create a pool that puts all of its capital inside a chosen price range with a chosen fund ratio, so the same money quotes deeper prices where the market actually trades, and you can exit from one side of an AB pool without taking the other.
- Traders: a new venue with tighter quotes inside its range, at the cost of the pool going single-sided when price leaves the range, which is a liquidity property you can read on screen before trading.
- Witnesses: no new duty; the curve runs in the evaluators you already run, behind one hardfork date the committee sets or does not set, and no new operation tags are introduced.
- Gateway operators: your IOUs are ordinary assets here too, eligible as pool assets unchanged.
- Developers: new fields append to the pool object and new typed extensions append to the existing pool operations, so no existing wire format changes, and the curve branch is selected by a creation-time flag.

## The design in one page

A pool opts into the AB curve at creation through a typed extension carrying `curve_type = 2` and the parameters: fund ratio `w` in basis points, lower bound `P_low`, upper bound `P_high`. Virtual offsets `xv` and `yv` are computed at creation so the curve hits the axes exactly at the bounds, giving zero tail waste. Swaps are Balancer math with `(x + xv)` and `(y + yv)` substituted, closed form with no Newton iteration. Single-sided withdrawal is allowed with the same half-fee as BSIP-6. A swap that would push price outside the range refuses at validate. One flag, `HARDFORK_AB_POOL_PASSED`, gates everything, all-or-nothing.

## Key numbers

- Curve types: 0 constant product, 1 StableSwap (BSIP-6), 2 AB.
- Assets per pool: 2.
- Fund ratio `w`: 1 to 9999 basis points (0 and 10000 refuse).
- Single-sided withdrawal fee: 1/2 of the swap fee for n = 2.
- Price bounds: `P_low < P_high` strictly, neither zero.
- Iteration: none; closed form.

## Risks

- Concentration increases impermanent loss per unit of capital; the provider loses more per dollar when price exits the range.
- AB pools go single-sided outside their range; re-anchoring means a new pool, fragmenting liquidity.
- Virtual offset derivation must hit exact bounds; an off-by-one re-introduces tail waste.
- Rounding must favour the pool on every step.
- Parameter sets that overflow the offset computation refuse at creation.
- State growth is bounded: scalar parameters only, no new object type.

## Non-goals

- No CES curves.
- No dynamic rho.
- No generalisation to n > 2.
- No migration of existing pools.
- No new operation tags, object types, or API.

## Copyright

This document is placed in the public domain.