<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-2 — True stealth transfers, blind pool swaps, and fee rules (lightpaper)

> **Lightpaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Abstract

BitShares can already move value privately, but today's blind transfers hide only the amount and cost about twelve times a fair transfer. This proposal adds true stealth transfers that hide among at least sixteen users, with every hidden amount checked by the whole network. It adds blind pool swaps through the same public pools and fee rules that aim the subsidy at anonymity-set growth. Nothing existing is removed. Legacy transfers, markets, and protocol totals stay public.

## What changes for you

- Legacy holders: nothing changes.
- Traders: markets, pools, prices, margin, and settlement stay public; you gain a private way to swap through the same pools.
- Witnesses: six operations and two new state kinds behind one hardfork date.
- Gateway operators: deposits and withdrawals stay transparent.
- Developers: six operations at tags 104 to 109, appended state.

## The design in one page

A stealth transfer hides value in a Pedersen commitment, proves authorization with a ring signature, bounds amounts with a Bulletproofs+ range proof, and prevents double spends with a nullifier. Consensus verifies everything. Rings carry 16 to 32 members. Range proofs are 592 bytes. Decoys must be 24 hours old. An asset needs 11 live blinded outputs before ring spends. Blind work is capped at 750000 microseconds per block. A per-asset blinded supply counter reconciles every maintenance interval. Version bytes and a one-year dual-verify window allow safe upgrades. Operation 109 swaps blinded assets against the same constant-product pool with a six-link soundness chain including a 65-byte Schnorr pin. Rule R aims the subsidy at anonymity-set growth without new parameters.

## Key numbers

- Ring members: 16 to 32.
- Range proof: 592 bytes.
- Signature: 32 + 96·n bytes.
- Decoy minimum age: 24 hours.
- Window skew limit: 40 percent, 25 percent for small sets.
- Thin-set floor: 11 live blinded outputs.
- Verification budget: 750000 microseconds per block.
- Dual-verify window: 1 year.

## Risks

- A watch-only key must not be able to spend; the consensus half is open.
- The fee subsidy is inert at current fee settings.
- No clean specialist review means no activation.
- Verification cost on ordinary hardware is not yet measured.
- A verification bug could create coins; the halt ships first.
- Anonymity sets are load dependent.
- Privacy covers one leg, not the whole journey.
- Stealth swap fees bypass issuers pending a committee vote.
- Discrete-log assumptions fall to a quantum computer.

## Permanent non-goals

- Protocol totals stay public.
- Order-book blindness is permanent.
- Blind pool deposits and withdrawals are impossible by construction.
- Mandatory blindness is rejected.

## Copyright

This document is placed in the public domain.
