<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-8 — Forex-style PAMM accounts (lightpaper)

> **Lightpaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Abstract

PAMM (Percentage Allocation Management Module) is the dominant account model in retail forex: an investor pools funds under a money manager, the manager trades a single master account, and profits and losses are distributed to investors in proportion to their share of the pool. This proposal adds a first-class PAMM account type to BitShares by reusing two existing primitives: BSIP-72 Tanks and Taps for bounded, auditable fund custody, and BitShares custom active authorities for per-investor withdrawal and reporting rights. It does not touch legacy accounts, does not add a new VM, and does not change core asset or pool semantics. The tank holds all investor deposits plus accrued performance fees; taps encode the manager's trading limit, the performance-fee waterfall, the investor withdrawal right, and the emergency drain, each with its own flow limit and delay; shares are a user-issued asset minted and burned at current NAV; and a rolling 365-entry NAV buffer makes every pool comparable on standardised metrics. NAV is sourced from a BSIP-1 bonded oracle when available, and from manager self-report with a challenge window otherwise. The pitch: managed accounts on chain, with the manager's power to move money written on screen and enforced by consensus, rather than by a custodian's word.

## What changes for you

- Legacy asset holders: nothing changes; no rule here can move, cap, or re-price legacy assets.
- Managers: you can create a PAMM pool with a tap topology that bounds your trading power, accrue a performance fee only above a high-water mark, and publish a standardised performance record that investors can compare.
- Investors: you can deposit into a pool, hold a share denominated in the pool asset, withdraw subject to a delay and a per-investor limit, and retain a withdrawal right the manager cannot revoke.
- Platforms and auditors: you hold the emergency-tap authority at supermajority, and enforce reporting and large-flow review as authority predicates.
- Witnesses: no new duty; the proposal reuses primitives you already run, behind one hardfork date the committee sets or does not set.
- Developers: new operations append at fresh wire tags with new object types, old operations and fields stay where they are, and the tank and tap primitives are reused rather than duplicated.

## The design in one page

One tank per PAMM pool, denominated in the pool asset, holds all investor deposits plus accrued performance fees. Four tap types bound the flows: trading tap with periodic flow limit and minimum tank level floor; performance-fee tap that opens only above a high-water mark; withdrawal tap with delay and per-investor cumulative limit; and an emergency tap requiring a supermajority. Shares are a UIA minted and burned at NAV, with virtual offsets to prevent a first-depositor attack. Each investor gets a custom active authority on their withdrawal tap. A rolling 365-entry NAV buffer is appended by a platform-gated `pamm_snapshot`, using a bonded oracle or manager self-report with a challenge window. One flag, `HARDFORK_PAMM_PASSED`, gates everything.

## Key numbers

- NAV buffer: 365 entries per pool.
- Emergency tap threshold: proposed two-thirds, filed by the design spec.
- Virtual offsets: 1 unit of pool asset and 1 virtual share (proposed).
- Performance fee: accrues only above the high-water mark.
- Withdrawal delay: anti-run, filed by the design spec.
- Operations: 9, appended at fresh wire tags.

## Risks

- The manager can still trade badly; PAMM guarantees bounded custody and transparent reporting, not returns.
- NAV oracle risk: a manager who overstates NAV and is not challenged locks in a false record; the challenge window and platform gate are the defence.
- Run risk: withdrawals queue and settle at falling NAV; the delay spreads exits but does not change the arithmetic.
- Share dilution: virtual offsets prevent a first-depositor attack.
- Manager capture of the platform authority: mitigated by the supermajority threshold and on-chain visibility.
- State growth if pool creation is unbounded: mitigated by the creation fee and proposed caps.

## Non-goals

- No new VM.
- No new consensus math beyond share and NAV arithmetic.
- No foreign-chain attestation.
- No legacy account, asset, or pool touched.
- No blind form of anything; share balances and NAV are public.
- No manager-set NAV; the oracle or challenge window is the backstop.

## Copyright

This document is placed in the public domain.