<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-6 — StableSwap pools for pegged pairs (lightpaper)

> **Lightpaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Abstract

Constant-product pools charge slippage on trades whose price should not move at all. This proposal adds a second pricing curve for liquidity pools, Curve's StableSwap invariant with an amplification coefficient A, aimed at pegged pairs: dollar tokens, wrappings of the same coin, bridged twins. The curve is flat near balance by construction of the curve, so a 10,000 swap in a million-unit pool returns 9,978 (99.8 percent) instead of the roughly 9,900 a constant-product pool would quote, and it degrades toward constant product as the pool becomes lopsided, so a depeg still has a liquidity floor. Participation is per-pool at creation time; constant-product pools are untouched. No new operation tags are needed: the curve rides the existing pool operations through typed extensions. The amplification coefficient A is fixed at creation, the pool holds exactly two assets, and every rounding step favours the pool. The pitch: pegged pairs trade at the price they are supposed to trade at, and the safety net appears only when something has gone wrong.

## What changes for you

- Legacy asset holders: nothing changes; no rule here can move, cap, or re-price legacy assets.
- Liquidity providers: you can create a pool whose curve is flat near balance for a pegged pair, so the same capital quotes tighter prices for the same trade size, and you can withdraw from one side of a stable pool without taking the other.
- Traders: a new venue where pegged pairs trade near parity with slippage measured in basis points rather than percent, while a depeg still leaves a liquidity floor instead of a cliff.
- Witnesses: no new duty; the curve runs in the evaluators you already run, behind one hardfork date the committee sets or does not set, and no new operation tags are introduced.
- Gateway operators: your IOUs are ordinary assets here too, eligible as pool assets unchanged.
- Developers: new fields append to the pool object and new typed extensions append to the existing pool operations, so no existing wire format changes, and the curve branch is selected by a creation-time flag.

## The design in one page

A pool opts into StableSwap at creation through a typed extension carrying `curve_type = 1` and an amplification coefficient A. The invariant is Curve's StableSwap equation for two assets, with D as the stored invariant and A interpolating between constant product (A to zero) and constant sum (A to infinity). `compute_d` solves for D by Newton iteration; `compute_new_y` inverts for the output side. Both work in 128-bit integers. Deposits off ratio pay an imbalance fee of half the swap fee on the excess side. Single-sided withdrawal is allowed on stable pools with the same half-fee, and refused on constant-product pools. Every rounding step favours the pool. One flag, `HARDFORK_STABLESWAP_PASSED`, gates everything, and the gate is all-or-nothing.

## Key numbers

- Curve types: 0 constant product, 1 StableSwap, 2 AB (BSIP-7).
- Assets per pool: 2.
- Imbalance fee scaling: 1/2 of the swap fee for n = 2.
- Oracle delay: none; pure curve, no external feed.
- Amplification: positive 64-bit integer, ceiling filed by the design spec.
- Newton iteration cap: compile-time constant, filed by the design spec.

## Risks

- An absurd A behaves as constant sum until one side runs to zero; creation-time validation is the mitigation, and the ceiling is an open decision.
- Fixed A: re-tuning needs a new pool, which fragments liquidity across versions.
- n = 2 only; generalising needs its own design spec and audit.
- Depeg dynamics need professional pricing review before any pool holds material value.
- Rounding must favour the pool on every step; the grinding harness is the defence.
- State growth is bounded by construction: one appended field and one appended struct.

## Non-goals

- No generalisation to n > 2.
- No dynamic A updated by governance.
- No new operation tags, object types, or API.
- No migration of existing pools.
- No in-consensus verification of foreign payments (not applicable).

## Copyright

This document is placed in the public domain.