<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-1 — Parallel bonded smartcoin class (lightpaper)

> **Lightpaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Abstract

BitShares trades like a modern exchange and trusts like an old one. This proposal adds a parallel asset class in which every claim states its capital. Bonded oracles publish prices with a delay and a rate limit. Bonded vaults custody backing behind locked bonds. New dollar-tracking smartcoins arrive as two tranches, a senior stable first and a junior reserve coin that absorbs losses up to a cap. Coverage is public and gates growth. Legacy assets stay untouched, permanently. Nothing turns on until a pilot under $20,000 proves coverage for 90 days and a vote passes.

## What changes for you

- Legacy asset holders: nothing changes, and no rule here can move, cap, or re-price legacy assets.
- Traders: new bonded pairs sit beside legacy books with prices that arrive one hour late and move at most five percent per interval.
- Witnesses: no new duty unless you choose to run a bonded oracle slot.
- Custodial token issuers: keep your token as is or move to bonded custody with on-screen coverage.
- Developers: 26 new operations at tags 78 to 103, appended objects, new API calls.

## The design in one page

Phase 1 is the bondable oracle base and spike guards: five operations at tags 78 to 82, per-oracle opt-in guards, one-hour delay, five percent rate limit, consensus expiry, six API calls. Phase 2 is bonds, disputes, and vaults: twenty-one operations at tags 83 to 103, USD-denominated bonds, a shared dispute rulebook, threshold vaults, mint at 200 percent secure ratio, coverage halt at 150 percent, wind-down at 15 percent haircut. Issuance runs on hybrid_mint and hybrid_redeem with a 130 percent floor, a per-block door cap, a 7-bucket absorption partition, and a cap function over depth. Seasoning runs 90 days on head-block time.

## Key numbers

- Oracle delay default: 1 hour, maximum 1 day.
- Oracle rate limit: 5 percent per interval.
- Secure ratio at genesis: 200 percent.
- Coverage halt: 150 percent.
- Wind-down haircut: 15 percent.
- Mint floor: 130 percent.
- Seasoning: 90 days.
- Pilot cap: $20,000 face for 90 days.

## Risks

- Legacy assets are not touched; losses stay inside the new class.
- Collusion can win votes; a missed vote freezes rather than acquits.
- BTS-correlated collateral crashes together until the mix widens.
- Silence is free; a challenger drought is possible.
- Custodial recovery is partial or zero by design.
- The junior token can go to zero.
- Phase 1 guards alone do not stop a colluding majority.
- Pilot scale can fail by design.

## Non-goals

- No uncapped mint or burn.
- No unbonded voted feeds as an end state.
- No rewards for agreement with the median.
- No blind supply totals.
- No automatic slash on deviation.
- No reputation-earned weights.
- No per-account caps.
- No mandated blindness.
- No verifier-model claims.

## Copyright

This document is placed in the public domain.
