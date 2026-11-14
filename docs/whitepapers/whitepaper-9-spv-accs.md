<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-9 — SPV-Enabled XClaim ACCS: Trustless Settlement on UTXO Chains

> **Whitepaper**
> **Version 1.0 — November 2026**
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

BitShares users who want real BTC today must either trust a third-party issuer whose token is an IOU, or trust a bonded attestor who swears under penalty that a Bitcoin payment happened — and whose word consensus never verifies. BSIP-9 replaces attestation with verification where the technology allows it: on UTXO chains whose headers can be verified inside consensus (Bitcoin, Litecoin, Dash, Dogecoin, Bitcoin Cash), the seller submits a self-contained SPV proof — the transaction, its merkle path, and block headers chained to a checkpoint — and BitShares verifies proof-of-work, header linkage, merkle inclusion, and confirmation depth directly in `do_evaluate`. No attestor, no oracle, no trusted observation.

Chains without SPV-friendly headers (Ethereum, Solana, Polkadot, and others) keep the existing bonded-attestor F1 oracle path unchanged. BSIP-9 is dual-path: SPV primary for UTXO chains, attestor appeal and fallback for all chains. No witness keys, no vault custody, no witness honey pot as locked value grows.

BSIP-9 is part of the same independently votable program as BSIP-1 (bonded price oracle and smartcoins), BSIP-2 (stealth transfers), and BSIP-3 (bonded ACCS settlement). It enhances BSIP-3's ACCS orderbook without replacing it: every new field appends last, every new operation is hardfork-gated, and the wire format stays stable across every combination of approved proposals.

Status, stated plainly. The code is implemented but not activated. Activation requires the full gate register at `build/ACTIVATION-CHECKLIST.md` to close, including testnet seasoning and specialist review. Nothing changes for any existing holder, trader, or witness until activation.

## 2. Specification

### 2.1 Track structure and activation gates

The registry is pinned across all BSIP configurations: operation tags 0 through 77 are the existing chain operations, then the price oracle and smartcoin proposals, then stealth transfers, then BSIP-3 (ACCS) at tags 110 through 117. BSIP-9 owns tag 118 (`accs_fulfill_spv_operation`). Every branch declares every operation, object id, and hardfork name, so ids never shift with the approved set; an operation whose proposal was not approved has no evaluator registered and fails closed at apply.

BSIP-9 receives its own hardfork gate (`HARDFORK_SPV_ACCS_PASSED`) and activates strictly after the F_ACCS gate. Its evaluator asserts the SPV gate passed, the oracle gate passed, and the fail-closed ordering check. A schedule placing SPV before F_ACCS is refused rather than clamped.

### 2.2 The SPV chain abstraction layer

`libraries/chain/spv/` is a new chain-agnostic SPV layer. It exposes a pure C++ interface with no BitShares consensus dependence:

- **Header verification** — verify proof-of-work, link to parent, anchor to checkpoint
- **Difficulty adjustment** — retarget math per chain
- **Merkle path verification** — prove a transaction is inside a block header
- **Confirmation counting** — count blocks deep since the transaction's block

Chain-specific parameters live in a small parameter struct (magic bytes, proof-of-work algorithm, target block time, confirmation depth, checkpoint). All SPV logic is chain-agnostic; only the parameters and byte layouts differ per chain.

### 2.3 Chain implementations

| Chain | PoW algorithm | Target block time | Confirmation depth |
|-------|--------------|-------------------|-------------------|
| Bitcoin | SHA-256d | ~10 minutes | 6 |
| Litecoin | scrypt | ~2.5 minutes | 12 |
| Dash | X11 | ~2.5 minutes | 12 |
| Dogecoin | scrypt | ~1 minute | 24 |
| Bitcoin Cash | SHA-256d | ~10 minutes | 6 |

Each chain is implemented as a parameter struct plus, where needed, chain-specific byte layout handling. No new verification logic is added per chain beyond the parameter struct — DRY from onset.

Additional UTXO chains can be added as new chain parameter structs with no new verification logic. Non-UTXO chains (Ethereum, Solana, Polkadot, etc.) remain out of scope for SPV and stay on the bonded-attestor path.

### 2.4 Gateway declaration with SPV capability

`accs_declare_gateway_operation` is extended (appended-last) with an optional `spv_capable` flag and, when set, a checkpoint header hash for the foreign chain. Committee members declare whether a gateway chain accepts SPV proofs. For non-SPV chains, the flag is false and the existing F1 oracle path is used unchanged.

### 2.5 Dual-path fulfillment

`accs_fulfill_operation` (tag 116) remains the attestor path: the attestor publishes the foreign-chain event to the F1 foreign-tx oracle, and fulfillment matches against the promoted oracle value by `foreign_tx_id` equality. Required for non-SPV chains; optional fallback for UTXO chains.

`accs_fulfill_spv_operation` (tag 118) is the new SPV path. The seller submits a self-contained SPV proof with the fulfillment transaction:

- The foreign-chain transaction sending funds to the buyer's declared address
- A merkle path proving the transaction is inside a block
- Block headers chained from that block back to the gateway's declared checkpoint
- Proof-of-work valid on each header
- Confirmation depth at or above the per-chain minimum

BitShares verifies the proof in `do_evaluate` (never `validate()`). If valid, escrow is released to the buyer and the griefing bond is returned to the seller. No attestor, no oracle, no trusted observation.

The attestor path is retained as appeal and fallback: if a conflict arises between an SPV proof and attestor evidence, the challenge window allows bonded attestor testimony to override. This gives economic recourse where SPV itself has none.

### 2.6 Checkpoint anchoring

Every SPV-capable gateway declares a checkpoint: a trusted block header hash for the foreign chain at gateway declaration time. All SPV proofs must chain back to this checkpoint. The checkpoint may be advanced by a subsequent gateway declaration (committee operation), never rolled back.

### 2.7 Replay protection (unchanged)

`accs_consumed_tx_object` records one row per fulfilled transaction, keyed (chain_id, foreign_asset_id, tx). Rows are durable, never edited or removed, capped at 65536 per pair. This applies identically to both fulfillment paths — an SPV-proven transaction cannot be fulfilled twice.

### 2.8 Escrow, griefing bond, SLA (unchanged)

All ACCS economics from the F series carry over unchanged:
- Seller locks base asset in escrow at offer creation
- Buyer posts griefing bond at match
- SLA deadline set at match; griefing claim on timeout
- Fulfillment releases escrow to buyer, returns bond to seller

## 3. Theory

The invariants and assumptions:

- SPV verification in `do_evaluate`, never `validate()`: PoW, header linkage, merkle inclusion, confirmation depth against checkpoint.
- SPV proofs submitted by the seller client-side; witnesses have no role in SPV verification or withdrawal signing.
- Checkpoint declared per gateway, advanced only by committee, never rolled back.
- Confirmation minimums are protocol constants, not governance parameters.
- SPV path and attestor path share the same replay protection (`accs_consumed_tx_object`).
- Attestor path retained as appeal and fallback within the challenge window.
- Replay protection is instance-independent; gateway redeclaration cannot clear it.
- State growth is bounded by caps.
- No in-consensus SPV tip reporting by witnesses.
- Legacy `bit*`, `HONEST.*`, and gateway UIAs untouched.

## 4. Exploit analysis

- **False SPV proof from 51% hash power:** a majority miner on a UTXO chain can build a valid-looking chain containing a lock transaction that never happened on the real network. Consensus verifies the proof structurally and accepts it. There is no bond to slash and no economic recourse. Mitigated by per-chain confirmation depth and checkpoint anchoring; the attestor appeal path provides limited recourse within the challenge window.
- **Checkpoint poisoning:** if the declared checkpoint header is wrong or corrupted, every SPV proof chains from a false anchor. Mitigated by committee-gated checkpoint updates and rollback prevention.
- **Replay via gateway redeclaration:** addressed by the instance-independent append-only `accs_consumed_tx_object`, shared between both fulfillment paths.
- **State bloat:** capped at 65536 consumed records per pair; offer caps unchanged from F2.
- **Happy-path-only testing:** defenses asserted but not demonstrated; attack tests required before activation.
- **Governance capture on non-SPV chains:** the bonded-attestor path remains for ETH/SOL/etc.; committee-declared gateways and oracle binding are the ceiling.
- **Recipient non-verification (SPV path):** the SPV proof verifies the transaction is buried in a valid chain but consensus does not check that the output pays to the seller's declared address unless that check is added; this is a known limit.
- **Foreign chain reorganizations:** confirmation minimums price but do not eliminate reorg risk.
- **Client-side SPV burden:** the seller must assemble headers and merkle paths client-side; a compromised SPV node or explorer can hand the seller a structurally valid proof on an attacker-controlled chain.

## 5. Gas and cost analysis

Operations are consensus objects, not contract calls, so cost is measured in transaction fees and block weight. The SPV proof adds roughly 1–2 KB to a Bitcoin fulfillment transaction (6 block headers at 80 bytes each, plus merkle path and transaction bytes). Larger than a simple hash but within normal transaction size bounds.

SPV verification cost in `do_evaluate`: one double-SHA-256 per header in the chain, one merkle path verification with at most 2^depth candidate hashes (depth ~10-15 for Bitcoin, so ~32768 candidates worst case), plus PoW checks. Bounded and predictable.

SPV is client-side assembled, so no consensus burden for header collection. Attestors remain paid on non-SPV chains as before.

## 6. Error code registry

New BSIP-9 errors occupy a reserved range appended after the existing registry. Each error maps to one refusal condition: gateway not SPV-capable, no checkpoint declared, invalid checkpoint format, SPV chain id mismatch, header chain empty, first header not tx block header, merkle path failure, header chain link or PoW invalid, header chain does not anchor to checkpoint, insufficient confirmations, raw transaction empty, tx block header too short, merkle path/left_bits length mismatch, chain id out of range, replayed transaction, consumed-tx set at cap, SLA deadline exceeded, offer not matched, seller mismatch, SPV path on non-SPV chain. Legacy error codes keep their numbers.

## 7. Admin architecture

Admin is split between consensus and committee. Consensus owns offers, escrow, matching, SPV verification, fulfillment, and claims. The committee owns gateway declaration and deactivation (including `spv_capable` and `checkpoint`), oracle binding for the attestor path, and fee parameters. No admin key can move escrow by procedure alone.

Witnesses have no SPV role: they neither submit tips nor hold foreign-chain keys. The SPV proof is submitted by the seller client-side.

## 8. Max offer and set sizes

Offer lists and consumed-transaction records must be capped per account and per orderbook (unchanged from F2). Terminal offers remain for audit after removal from lists. One offer carries one base asset. Matching is first in line by creation block. The service-level window is 86,400 blocks. The terminal cleanup age is 100,800 blocks. The minimum offer lifetime is 100 blocks. Consumed-tx set capped at 65536 per pair.

## 9. Oracle windows

The SPV path does not use the oracle. The attestor path (retained for non-SPV chains and as appeal) reuses the price oracle's guard machinery: delay queue before promotion, freezes, rate limits, promotion proofs, and a challenge path. Confirmation minimums are 6 (BTC/BCH), 12 (LTC/Dash), and 24 (DOGE) blocks.

## 10. Glossary

- **ACCS:** atomic cross-chain swap.
- **SPV:** Simplified Payment Verification — proving a transaction is included in a block without downloading the full chain.
- **Checkpoint:** a trusted block header hash that anchors SPV proofs; weak subjectivity anchor.
- **SPV proof:** self-contained proof (transaction, merkle path, header chain) submitted by the seller.
- **Dual-path fulfillment:** SPV primary for UTXO chains, attestor appeal/fallback for all chains.
- **Recipient non-verification:** consensus cannot verify that the foreign payment reached the seller's stated address without an explicit output check.
- **Witness honey pot:** witness-held foreign-chain signing keys that become a theft target as locked value grows; eliminated by BSIP-9's design.

## 11. Provenance

This whitepaper builds on the F series design: `docs/superpowers/specs/2026-10-20-f1-foreign-chain-tx-oracle.md` and `docs/superpowers/specs/2026-10-20-f2-accs-orderbook-settlement.md`. The BSIP-9 design spec is `docs/superpowers/specs/2026-11-13-bsip9-spv-accs.md` and the build plan is `docs/superpowers/plans/2026-11-13-bsip9-spv-accs-build.md`. Implementation is under `worktrees/track-b-h9-spv-accs/` (branch `feature/track-b-h9-spv-accs`).

This document adapts concepts from the Prism and Reciprocity reference tree (`contracts/`) and from XCLAIM (eprint 2018/643), interBTC, and tBTC/Keep. The SPV verification approach follows the light-client pattern: prove inclusion in a verified header chain rather than trusting an off-chain reporter.

## 12. Test cases

- Tag stability: tag 118 (`accs_fulfill_spv_operation`) holds across all BSIP configurations.
- Gate ordering refusal: SPV gate after F_ACCS gate; inverted schedule refused.
- SPV fulfill success per chain: BTC, LTC, DASH, DOGE, BCH with real testnet/mainnet tx hashes and merkle paths.
- SPV bad merkle root refused.
- SPV wrong recipient refused (if output check added).
- SPV insufficient confirmations refused.
- SPV replay refused via `accs_consumed_tx_object`.
- Attestor path on non-SPV chain (ETH/SOL) unchanged.
- SPV path on non-SPV chain refused.
- Gateway declaration with `spv_capable` flag and checkpoint.
- Checkpoint declared, advanced by committee, rollback refused.
- Confirmation minimums as protocol constants (6, 12, 12, 24, 6).
- Per-chain address validation (BTC P2SH/P2WSH, LTC/DOGE/BCH P2SH, Dash P2SH).
- `do_evaluate` (not `validate()`) for all SPV checks.
- Wire pinning across builds with and without the SPV hardfork flag.
- Attack tests: tampered header, wrong merkle root, orphan header, checkpoint mismatch, insufficient confirmations.
- No witness involvement in SPV path.
- Legacy assets untouched.

# Copyright

This document is placed in the public domain.
