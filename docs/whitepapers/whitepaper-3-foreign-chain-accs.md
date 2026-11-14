<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-3 — Bonded cross-chain settlement: an atomic cross-chain swap (ACCS) orderbook and a foreign-chain transaction oracle

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
8. Max offer and set sizes
9. Oracle windows
10. Glossary
11. Provenance
12. Test cases

---

## 1. Narrative

BitShares users who want real BTC must trust a third-party issuer: the token is an IOU and the coins sit in the issuer's wallet. This proposal replaces that promise with escrow and bonds. An atomic cross-chain swap completes both legs together or neither. A holder posts an on-chain offer selling any BitShares asset for BTC, a BTC holder matches it and sends BTC from their own wallet, and a bonded oracle that watches Bitcoin reports that payment to consensus; only then does the chain release the seller's escrowed assets to the buyer. No vault custodies foreign coins, no committee approves individual swaps, and a false published record forfeits a bond.

BitShares has no general smart contracts, so no proof of a Bitcoin payment can ever be verified inside consensus: no in-consensus light client, no SPV relay, no programmatic slashing of a foreign-chain liar. Anything that moves foreign value across this chain therefore rests on one of two foundations: custody, where somebody holds the coins for you, or attestation, where somebody posts a bond and swears under penalty that a payment happened. The BSIP-3 chooses attestation and prices it in bonds, disputes, and fail-closed rules.

The replacement works peer to peer. Alice holds a BitShares asset and wants real BTC. Bob holds real BTC and wants Alice's asset. Alice posts an offer on an on-chain orderbook, locking her asset in escrow. Bob matches the offer and posts a griefing bond, which is his own money held by the chain: Alice keeps it if he never pays and he gets it back when he delivers. Bob then sends BTC from his own wallet to the address Alice wrote in the offer. Bonded attestors report the payment. Consensus compares the report to the offer, and on an exact match Alice's escrowed asset releases to Bob and his bond returns. If Bob never pays, Alice claims his bond and gets her escrow back. No vault, no custodian, no per-swap committee vote. The only trust assumption in the flow is the attestor bond, stated openly rather than papered over.

BSIP-3 is one of three independently votable proposals. BSIP-2 is stealth transfers, BSIP-1 is the bonded price oracle and its smartcoins, and BSIP-3 is this proposal. The community may approve any one, any two, or all three, and the wire format is pinned so that a signed transaction means the same thing in every configuration. Nothing here needs the other proposals to be useful, and nothing here is switched on by their approval.

Status, stated plainly. The BSIP-3 is implemented and has been independently audited by two separate review efforts. Those audits found open defects that are being repaired, and the risks section names each one in plain words rather than summarizing the code as finished. Activation is not authorized. Nothing changes for anyone until every gate in the activation checklist closes. The first trading pair is BTC against BTS, and it must prove coverage in a pilot capped at $20,000 face value for 90 days of live observation before any USD issuance is allowed to depend on it.

## 2. Specification

### 2.1 BSIP structure and activation gates

The registry is pinned across all BSIP configurations: operation tags 0 through 77 are the existing chain operations, then the price oracle and smartcoin proposals, then stealth transfers, and the BSIP-3 owns tags 110 through 117. Every branch declares every operation, object id, and hardfork name, so ids never shift with the approved set; an operation whose proposal was not approved has no evaluator registered and fails closed at apply.

the transaction oracle keeps the oracle subsystem's gate, because the oracle subsystem is shared machinery that ships with whichever proposal needs it. the settlement layer receives its own gate, and its evaluators assert three things: that the settlement gate has passed, that the oracle gate has passed, and a fail-closed ordering check that refuses a chain schedule placing the settlement gate before the oracle gate. A schedule in which BSIP-3 activation precedes the oracle activation it depends on is refused rather than clamped.

### 2.2 the transaction oracle, the foreign-chain transaction oracle

the transaction oracle adds one oracle series kind for foreign transactions, distinct from price series, published by a new operation. Each published record carries the foreign chain identifier, a fixed-size foreign transaction id, an event kind of lock, unlock, or send, the amount in the foreign chain's smallest denomination, an index identifying the foreign asset (zero meaning the chain's native asset), the foreign block height at which the transaction confirmed, and the number of confirmations observed.

Each foreign chain has a minimum confirmation count, proposed at 6 for Bitcoin, 12 for Ethereum, and 32 for Solana. These are protocol constants, not governance parameters, so changing one requires a hardfork.

Producers post one bond against the oracle object, and that bond covers both price series and transaction series production, so a producer who lies about a transaction forfeits the same bond as a producer who lies about a price. The guard machinery is shared with the price oracle: a delay queue before promotion, freezes, rate limits, promotion proofs, and a challenge path where a sustained challenge slashes the bond and winds the producer down. A missed publication deadline freezes and reimburses rather than slashing, because liveness failure and dishonesty are different faults.

Consumption is by equality, not by ratio. A consumer matches a promoted record against a specific expected transaction: same transaction id, same amount, same foreign asset, event kind of send, and a confirmation count at or above the chain minimum. A match lets the consumer proceed; any mismatch stops it, and the deadline consequences fall where the consuming protocol puts them.

### 2.3 the settlement layer, the gateway, the orderbook, and the offer

A gateway in this design is not a custodian. It is a consensus record, created by a committee member, binding a foreign chain and optionally a foreign subtoken to an the transaction oracle oracle. The committee declares a Bitcoin gateway, or a USDT-on-Ethereum gateway, once per chain and foreign asset pair, and after that any BitShares base asset may open an orderbook against it. There is no per-base-asset committee approval, so the committee's power is to open and close corridors, not to approve trades. Deactivation blocks new offers while leaving existing ones alone.

An orderbook is keyed by base asset, foreign chain, and foreign asset, and is created lazily by the first offer against that triple. An offer records the seller, the base amount locked in escrow, the minimum foreign amount the seller wants, the seller's foreign-chain recipient address, an optional expiry block, and a state of open, matched, fulfilled, expired, or cancelled. The transitions are exactly: open to matched when a buyer matches, open to cancelled when the seller cancels, open to expired when the maintenance sweep passes the expiry, matched to fulfilled when a valid foreign transaction record is consumed, and matched to expired when the service-level deadline passes and the seller claims. Fulfilled, expired, and cancelled are terminal.

Seven operations implement the flow. Creating an offer deducts the base amount from the seller into escrow and validates that the chain has an active gateway and that the recipient address is well formed. Cancelling releases the escrow, and only the seller may cancel, and only while the offer is open. Matching requires the buyer to post a griefing bond at or above the minimum, sets the offer's service-level deadline to the current block plus a fixed window, and moves the offer to matched, at which point the seller can no longer cancel. Fulfilling releases the escrow to the buyer and returns his bond; it requires the promoted foreign record to match the offer on transaction id, amount, foreign asset, event kind, and confirmation count, and it is refused once the deadline has passed. Claiming the griefing bond transfers it to the seller and returns her escrow, and it is refused before the deadline. Declaring and deactivating gateways are the two committee operations.

Maintenance integration is small and explicit. Expired open offers release their escrow in the maintenance sweep. Service-level timeouts are never claimed automatically, because automated fee farming would follow, so the seller must claim explicitly. Terminal offers are removed from the orderbook's lists after an age, while the object itself remains for audit.

The constants, as proposed: a minimum offer lifetime of 100 blocks, a service-level window of 86,400 blocks, roughly three days at the chain's three-second block time, a minimum griefing bond of 8,375 USD cents' worth of the base asset, and a terminal cleanup age of 100,800 blocks.

### 2.4 One swap, end to end

Alice calls create offer, naming BTS as the base asset, Bitcoin as the chain, her own Bitcoin address as the recipient, and a minimum BTC amount as her price; her BTS leaves her balance and sits in escrow on the offer. Bob matches the offer and posts his griefing bond. Bob sends the agreed BTC from his wallet. Attestors running Bitcoin listeners observe the transaction, wait out the confirmation minimum, and publish a foreign transaction record. Once that record clears the oracle's delay queue and is promoted, anyone may call fulfill. Consensus compares the promoted record to the offer, finds every term equal, releases the BTS to Bob, returns his bond, and marks the offer fulfilled. If instead Bob never sends, or sends the wrong amount, the record will not match, the deadline passes, Alice calls claim griefing, and she leaves with her BTS plus Bob's bond.

One term of that comparison cannot be enforced by consensus, and the design says so rather than implying otherwise. The foreign transaction record has no recipient field, so consensus cannot verify that the BTC reached Alice's stated address; that leg is attested by the bond of whoever published the record, not verified by the chain. This is the attestor trust ceiling of the whole proposal, and it is permanent under the current object format.

### 2.5 Client-side attestor infrastructure

Attestors are client-side software, not consensus. For each supported foreign chain, an attestor runs a listener that watches foreign blocks for transactions to addresses named in open offers, extracts the transaction id, amount, recipient, and confirmation count, and publishes the record. Subtoken support adds event-log decoding on top, since a token transfer on Ethereum is a contract event rather than a native transfer. Consensus verifies none of this; the bond is what stands behind a published record.

Attestors are paid from production fees on the operations that consume their records, from bond yield shared with price oracle producers, and from a share of slashed bonds distributed to successful challengers. The fee split and yield rate are committee-governable parameters, not hardcoded constants. The attestor allowlist per foreign chain is a committee operation, and an attestor must post a bond, run the listener for the chains they declare, and publish within the deadline.

### 2.6 Out of scope by design

Matching is first in line by creation block, not price-time priority, and one offer carries one base asset. There are no partial claims, no multi-leg offers, no two-sided swap inside a single transaction: the reverse direction, foreign coins for a BitShares asset, is simply another offer from a different seller. Foreign chain fees are the buyer's problem, not the protocol's. Confirmation rules beyond the three named chains are not proposed here.

## 3. Theory

The invariants and assumptions:

- No in-consensus verification of foreign payments; attestation replaces verification.
- Consensus compares promoted records to offers by equality on transaction id, amount, foreign asset, event kind, and confirmation count.
- Escrow holds only native BitShares assets; no foreign custody.
- Bonds cover both price and transaction series production by one producer.
- A missed publication deadline freezes and reimburses rather than slashing.
- A sustained challenge slashes the bond and winds the producer down.
- Gateway deactivation blocks new offers while leaving existing ones alone.
- Offer state transitions are finite and terminal states are absorbing.
- Service-level timeouts are never claimed automatically.
- Terminal offers remain for audit after removal from orderbook lists.
- Replay protection must be instance-independent; redeclaration must not clear it.
- State growth must be bounded by caps.

## 4. Exploit analysis

- Replay via gateway redeclaration: a redeclared gateway starts with empty consumed-transaction memory, allowing one payment to fulfill twice. Repair moves memory to an append-only record.
- State bloat: uncapped offer lists and consumed records allow unbounded growth. Repair adds caps.
- Happy-path-only tests: defenses asserted but not demonstrated; attack tests are required before activation.
- Governance capture: committee declares corridors and binds oracles; bond size and challenge path are the backstop.
- Recipient non-verification: consensus cannot check the recipient; bond is the only backstop.
- Clock ordering: if the oracle guard delay exceeds the swap service window, every matched offer expires. Ordering must be fixed before activation.
- Foreign reorganization: confirmation minimums price but do not eliminate reorg risk.
- Attestor collusion: bonded and challengeable, but never verified.

## 5. Gas and cost analysis

Operations are consensus objects, not contract calls, so cost is measured in transaction fees and block weight. Seven operations per swap plus publication and promotion. Attestor payment comes from production fees, bond yield, and slashed-bond shares. The per-block verification budget of the BSIP-2 does not apply here. State growth is bounded by the caps under repair.

## 6. Error code registry

New BSIP-3 errors occupy a reserved range appended after the existing registry. Each error maps to one refusal condition: gateway not declared, gateway inactive, offer not found, not seller, offer not open, insufficient griefing bond, record not promoted, mismatch on transaction id, amount, foreign asset, event kind, or confirmation count, deadline passed, claim before deadline, expired, terminal, malformed recipient. Legacy error codes keep their numbers.

## 7. Admin architecture

Admin is split between consensus and committee. Consensus owns offers, escrow, matching, fulfillment, and claims. The committee owns gateway declaration and deactivation, oracle binding, attestor allowlist, fee split, and yield rate. No admin key can move escrow by procedure alone. The recipient non-verification is a permanent limit on what consensus can enforce.

## 8. Max offer and set sizes

Offer lists and consumed-transaction records must be capped per account and per orderbook. Terminal offers remain for audit after removal from lists. One offer carries one base asset. Matching is first in line by creation block. The service-level window is 86,400 blocks. The terminal cleanup age is 100,800 blocks. The minimum offer lifetime is 100 blocks.

## 9. Oracle windows

the transaction oracle reuses the price oracle's guard machinery: delay queue before promotion, freezes, rate limits, promotion proofs, and a challenge path. Confirmation minimums are 6, 12, and 32 blocks. The service-level window of 86,400 blocks must be ordered relative to the oracle guard delay before activation.

## 10. Glossary

- ACCS: atomic cross-chain swap.
- Gateway: a consensus record binding a foreign chain and optional subtoken to an the transaction oracle oracle.
- Orderbook: offers against one (base asset, foreign chain, foreign asset) triple.
- Offer: a seller's escrowed sale of a base asset for a foreign asset.
- Escrow: the chain holds the base asset until conditions are met.
- Griefing bond: the buyer's money held by the chain, forfeited on failure to pay.
- Service-level deadline: the block by which the buyer must deliver.
- Fulfill: release escrow to the buyer on a matching promoted record.
- Claim griefing: transfer the bond to the seller after the deadline.
- Attestor: a bonded off-chain operator who watches foreign chains and publishes records.
- Promotion: a record clears the oracle's delay queue and becomes consumable.
- Recipient non-verification: consensus cannot check the foreign payment's destination.

## 11. Provenance

This whitepaper adapts the structure of the Prism and Reciprocity whitepapers in the salvaged reference tree. The concepts are original to the BitShares Track B program. The design detail lives in `docs/superpowers/specs/2026-10-20-f1-foreign-chain-tx-oracle.md` and `docs/superpowers/specs/2026-10-20-f2-accs-orderbook-settlement.md`. The reference implementations live under `docs/reference/contracts-prism` and `docs/reference/contracts-reciprocity`.

## 12. Test cases

- Tag stability across all BSIP configurations.
- Gate ordering refusal.
- Foreign publish validation.
- Confirmation minimums as protocol constants.
- Consumption equality on all terms.
- Gateway declaration and deactivation.
- Offer lifecycle transitions.
- Fulfill match and deadline refusal.
- Claim timing refusal before deadline.
- Recipient non-verification documented.
- Maintenance sweep escrow release.
- Terminal cleanup after age.
- Constants as proposed.
- Registry completeness across branches.
- Wire pinning across builds.
- Replay protection instance-independent.
- State caps enforced.
- Attack tests for wrong amount, replay, early claim, sweep.

# Copyright

This document is placed in the public domain.
