<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-4 — Parallel N-asset liquidity pools (lightpaper)

> **Lightpaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Abstract

BitShares swap pools each hold two assets and price by constant product, so liquidity for bigger baskets sits scattered across chains of pairwise pools, and every pool charges a rate fixed by its creator. This proposal adds a parallel pool class. One pool holds up to eight assets, the pool's share token is a proportional claim on the whole basket, and pricing follows a geometric-mean invariant computed from every token ratio at once. Each swap's fee is derived from how imbalanced the trade leaves the pool, and two self-steering regulators move that fee toward volatility targets with no scheduled parameter changes. Every pool publishes its own time-and-volume weighted average price, computed in consensus from the pool's own trades. Legacy two-asset pools stay untouched. The pitch: deposit a basket once, and every asset inside it trades, pays, and publishes from one pool.

## What changes for you

- Legacy asset holders: nothing changes; your markets, balances, and fees are untouched.
- Liquidity providers: deposit up to eight assets into one pool and hold a share of the whole basket, with fees reinvested rather than collected and withdrawn.
- Traders: a venue where any two assets inside a pool price against each other from one shared state, with fees that shrink for balanced trades and grow for imbalanced ones.
- Witnesses: appended operations and new object types behind one hardfork date the committee sets or does not set.
- Gateway operators: untouched; pools against your tokens are ordinary pools against ordinary user-issued assets.
- Developers: new operations at fresh wire tags, new object types, and new read-only API calls for pool state and window readings.

## The design in one page

A pool prices with one number, the konstant K, a geometric mean of token ratios against genesis balances. Deposits mint shares equal to K growth; withdrawals burn shares and pay pro-rata balances. An exchange charges a fee from the fairness curve aequitas, which measures how imbalanced the trade leaves the pool, then settles the remainder in shares through the reciprocity rule, refused if negative. Two regulators, an exponential moving average over 500 observations and an average true range over 100, steer the fee toward 50 and 10 basis point targets with no scheduled changes. Fees split three ways: founder-style share to the committee account, promoter-style share to each deposited asset's issuer, remainder reinjected into the pool. Both named takes default to zero and are capped at 500 basis points. Every state-mutating operation updates three TVWAP accumulators over 15-minute, 1-hour, and 24-hour windows, measured in head-block time, so a flash loan cannot fabricate a price. Pools hold 2 to 8 assets; weights normalize to a sum of 100. Pool shares are visible user-issued assets, so blind liquidity positions are impossible by construction. Pools do not mint smartcoins and do not gate debt.

## Key numbers

- Pool assets: 2 to 8.
- Weight sum after normalization: 100.
- Basis denominator: 10,000.
- Drip protection floor: 3 units.
- Royalty default: 0 basis points each.
- Royalty cap: 500 basis points each.
- Governor EMA period: 500 observations.
- Governor ATR period: 100 observations.
- EMA spread target: 50 basis points.
- ATR spread target: 10 basis points.
- TVWAP windows: 15 minutes, 1 hour, 24 hours.
- Full exit: zero dust, accumulators reset.

## Risks

- The pool oracle can be bent across blocks by a patient attacker holding an imbalanced position, though it costs fees; consumers that price debt or derivatives off pool TVWAP inherit that risk.
- Thin pools facing large trades compute large fairness spreads; the amount-received bound and the share floor keep the trader whole or reject the trade, but the experience is expensive.
- At a ten-times single-token deviation the share accounting could err about 13 percent of the konstant; reaching that state costs at least 30 percent of pool value in fees.
- Committee and issuer royalty rates are a lever; the default is zero and the cap is small, and LPs can migrate to a new pool.
- Unbounded pool creation must be bounded by a creation ceiling and per-account caps in the design spec.

## Non-goals

- No migration or upgrade of existing two-asset pools.
- No pool prices as smartcoin collateral.
- No blind liquidity positions.
- No smartcoin minting or debt gating.
- No foreign-chain anything in a pool.
- No uncapped mint or burn.

## Copyright

This document is placed in the public domain.
path: /workspace/docs/lightpapers/bsip-4-n-asset-pools.md