<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-9 — SPV-Enabled XClaim ACCS: Trustless Settlement on UTXO Chains (lightpaper)

> **Lightpaper**
> **Version 1.0 — November 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Abstract

BSIP-9 enhances BitShares' ACCS orderbook settlement (BSIP-3) with SPV verification on UTXO chains. Where a foreign chain has verifiable headers (Bitcoin, Litecoin, Dash, Dogecoin, Bitcoin Cash), the seller submits an SPV proof — the transaction, its merkle path, and block headers chained to a checkpoint — and BitShares verifies proof-of-work, header linkage, merkle inclusion, and confirmation depth directly in consensus. No attestor, no oracle, no trusted observation for those chains. Chains without SPV (Ethereum, Solana, Polkadot) keep the bonded-attestor path unchanged. No witness keys, no vault custody, no witness honey pot as locked value grows.

## What changes for you

- **Legacy asset holders:** nothing changes; legacy assets (`bit*`, `HONEST.*`, gateway UIAs) keep their semantics, fees, and settlement rules.
- **Traders:** BTC/LTC/DASH/DOGE/BCH ACCS pairs settle with SPV-verified proofs instead of trusting an attestor; ETH/SOL pairs continue using bonded attestors unchanged.
- **Witnesses:** no new responsibility. Witnesses do not submit SPV tips, hold foreign-chain keys, or sign withdrawals.
- **Gateway operators:** UTXO chains no longer require an attestor for fulfillment; non-UTXO chains keep the bonded-attestor model. Existing gateway UIAs untouched.
- **Developers:** new SPV library (`libraries/chain/spv/`), one new operation (tag 118), gateway extended with `spv_capable` and `checkpoint`.
- **BTC holders:** send coins from your own wallet; settlement verified against Bitcoin headers, not an attestor's word.

## The design in one page

SPV verification in consensus: the seller submits raw transaction bytes, the tx block header, a merkle path with position bits, a header chain from the tx block to a checkpoint, and the chain id. Consensus verifies proof-of-work on each header, parent linkage through the header chain, merkle inclusion of the transaction in its block, and confirmation depth at or above the per-chain minimum — all inside `do_evaluate`, never `validate()`. If valid, escrow releases to the buyer and the griefing bond returns to the seller.

For chains without SPV headers, the existing F1 oracle path applies: a bonded attestor publishes the payment to the foreign-chain transaction oracle, and consensus matches by transaction-id equality as before.

## Key numbers

- Operation tag: 118 (`accs_fulfill_spv_operation`).
- SPV chains: Bitcoin (SHA-256d, 6 confirmations), Litecoin (scrypt, 12 confirmations), Dash (X11, 12 confirmations), Dogecoin (scrypt, 24 confirmations), Bitcoin Cash (SHA-256d, 6 confirmations).
- Confirmation minimums are protocol constants, not governance parameters.
- Checkpoint anchor: declared per gateway, advanced only by committee, never rolled back.

## Risks

- SPV is trust-minimized, not trustless: a 51% hash-power attacker on a UTXO chain can build a valid-looking but false chain, and consensus would accept it. There is no bond to slash on the foreign chain.
- Checkpoint trust: the starting header is trusted at gateway declaration; a corrupted checkpoint poisons every proof built from it.
- Client-side burden: the seller must assemble SPV headers and merkle paths client-side (run an SPV node or query one).
- Attestor path remains for non-SPV chains: Ethereum, Solana, Polkadot, and others keep bonded-attestor trust.
- Proof size: roughly 1–2 KB for a 6-confirmation Bitcoin proof, larger than a simple hash.

## Non-goals

- No witness keys for foreign chains.
- No vault custody of foreign coins.
- No two-sided swap in one transaction.
- No per-trade committee approval.
- No migration, caps, or changes to legacy `bit*`, `HONEST.*`, or gateway UIAs.

## Copyright

This document is placed in the public domain.
