<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-3 — Bonded cross-chain settlement: ACCS (lightpaper)

> **Lightpaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Abstract

BitShares users who want real BTC must trust a third-party issuer. This proposal replaces that promise with escrow and bonds. A holder posts an on-chain offer selling any BitShares asset for BTC, a BTC holder matches it and sends BTC from their own wallet, and a bonded oracle that watches Bitcoin reports that payment to consensus; only then does the chain release the seller's escrowed assets to the buyer. No vault custodies foreign coins, no committee approves individual swaps, and a false published record forfeits a bond.

## What changes for you

- Legacy asset holders: nothing changes; you may also sell directly for BTC through an escrowed offer.
- Traders: a new on-chain orderbook per asset against each declared foreign chain, price fixed at match, settlement within a published deadline.
- Witnesses: eight appended operations and three new object types behind their own hardfork gate.
- Gateway operators: existing IOUs untouched; you may run attestors.
- Developers: operations at tags 110 to 117, appended fields.
- BTC holders: send from your own wallet, receive BitShares assets directly.

## The design in one page

the transaction oracle is the foreign-chain transaction oracle: one oracle series kind for foreign transactions, published by a new operation, carrying chain, transaction id, event kind, amount, foreign asset index, foreign block height, and confirmation count. Confirmation minimums are protocol constants: 6 Bitcoin, 12 Ethereum, 32 Solana. Producers post one bond covering both price and transaction series. the settlement layer is the settlement layer: committee-declared gateways, lazily created orderbooks keyed by base asset, foreign chain, and foreign asset, offers with escrow, griefing bonds, and service-level deadlines. Seven operations implement create, cancel, match, fulfill, claim griefing, declare gateway, and deactivate gateway. Settlement is by equality against the promoted foreign record. Attestors are client-side and bond-backed.

## Key numbers

- Operation tags: 110 to 117.
- Bitcoin confirmation minimum: 6 blocks.
- Ethereum confirmation minimum: 12 blocks.
- Solana confirmation minimum: 32 blocks.
- Minimum offer lifetime: 100 blocks.
- Service-level window: 86,400 blocks.
- Minimum griefing bond: 8,375 cents' worth of base asset.
- Terminal cleanup age: 100,800 blocks.

## Risks

- Replay guard can be reset by gateway redeclaration; repair moves memory to an append-only record.
- Offer lists and consumed-transaction record have no caps; repair adds caps.
- Settlement suite proves happy paths, not attacks.
- Governance capture is a priced ceiling: committee declares corridors and binds oracles.
- Recipient term cannot be checked by consensus; bond is the only backstop.
- Oracle guard delay and swap service window must be ordered before activation.
- Foreign chain reorganizations can invalidate promoted payments.

## Non-goals

- No in-consensus verification of foreign payments.
- No per-trade committee approval.
- No vault custody of foreign coins.
- No two-sided swap in one transaction.
- No partial claims or multi-leg offers.
- No price-time priority matching.

## Copyright

This document is placed in the public domain.
