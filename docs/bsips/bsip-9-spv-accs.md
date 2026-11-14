    BSIP: (number assigned at filing; house label BSIP-9)
    Title: SPV-Enabled XClaim ACCS — Trustless Settlement on UTXO Chains
    Authors: litepresence (HONEST)
    Status: Draft
    Type: Protocol
    Created: 2026-11-13

# Abstract

BitShares' ACCS orderbook lets users trade BTS for foreign-chain assets through escrowed offers, griefing bonds, and SLA timeouts. Today every foreign-chain transaction must be attested by a bonded oracle publisher that BitShares never verifies in consensus — a trusted middleman for every chain. BSIP-9 replaces the attestor with SPV verification in BitShares consensus for UTXO chains that support simplified payment verification: the seller submits a self-contained SPV proof (transaction, merkle path, block headers) with the BitShares fulfillment transaction, and consensus verifies it directly. Chains without SPV (Ethereum, Solana, etc.) keep the existing bonded-attestor F1 path unchanged. No witness keys, no vault custody, no witness honey pot — just ACCS escrow economics with trustless verification where the technology allows it.

Pitch: real BTC on BitShares, settled by SPV proofs where headers are verifiable, bonded attestors where they are not.

# Motivation

ACCS settlement currently requires a foreign-chain transaction oracle: a bonded attetsor publishes lock/unlock events as BitShares memos, and consensus matches fulfillment against the promoted oracle value by transaction-id equality. The oracle is never verified in consensus. This works but leaves trust in the attestor for every chain, including chains where SPV could eliminate it entirely.

For UTXO chains with SPV-friendly headers (Bitcoin, Litecoin, Dash, Dogecoin, Bitcoin Cash), a self-contained SPV proof can prove that a foreign-chain transaction locked funds to the seller's address, buried deep enough, without trusting any attestor. BitShares can verify proof-of-work, merkle paths, and confirmation depth natively. For chains without SPV (account-based chains with no light-client header relay), the bonded-attestor model remains the only option.

BSIP-9 makes fulfillment dual-path: SPV-in-consensus where available, bonded-attestor where not. The SPV path is primary for normal fulfillment; the attestor path is retained as an appeal and fallback mechanism, not as a co-required second path. The ACCS escrow, griefing bond, SLA, and replay-protection machinery are reused unchanged from the F series. No witness is asked to hold foreign-chain signing keys, eliminating the witness honey pot that a vault model would create as locked value grows.

There is a hard ceiling behind this design, and BSIP-9 is shaped by it. BitShares has no general smart contracts, so no proof of a Bitcoin payment can ever be fully enforced inside consensus without external input: SPV verification assumes honest-majority hash power and a trusted checkpoint, and withdrawals still require off-chain signatures. SPV is trust-minimized, not trustless. The attestor appeal path provides economic recourse where SPV has none.

# What changes for you

- **Legacy asset holders (`bit*`, `HONEST.*`, gateway UIAs):** nothing changes; legacy assets keep their semantics, fees, and settlement rules.
- **Traders:** BTC/LTC/DASH/DOGE/BCH ACCS pairs settle with SPV-verified proofs instead of trusting an attestor; ETH/SOL pairs continue using bonded attestors unchanged.
- **Witnesses:** no new responsibility. Witnesses do not submit SPV tips, hold foreign-chain keys, or sign withdrawals.
- **Gateway operators:** UTXO chains no longer require an attestor for fulfillment; non-UTXO chains keep the bonded-attestor model. Existing gateway UIAs untouched.
- **Developers:** new SPV library (`libraries/chain/spv/`), one new operation (tag 118), gateway extended with `spv_capable` and `checkpoint`, hardfork gate `HARDFORK_SPV_ACCS_PASSED`.
- **BTC holders:** send coins from your own wallet; settlement verified against Bitcoin headers, not an attestor's word.

# Specifications

## 1. Track structure and activation gates

The registry is pinned across all BSIP configurations: operation tags 0 through 77 are existing chain operations, then the price oracle and smartcoin proposals, then stealth transfers, then BSIP-3 (ACCS) at tags 110 through 117. BSIP-9 owns tag 118 (`accs_fulfill_spv_operation`). Every branch declares every operation, object id, and hardfork name, so ids never shift with the approved set; an operation whose proposal was not approved has no evaluator registered and fails closed at apply.

BSIP-9 receives its own hardfork gate (`HARDFORK_SPV_ACCS_PASSED`) in `libraries/chain/hardfork.d/SPV_ACCS.hf` and activates strictly after the F_ACCS gate. Its evaluator asserts the SPV gate passed, the oracle gate passed, and the fail-closed ordering check. A schedule placing SPV before F_ACCS is refused rather than clamped.

## 2. The SPV chain abstraction layer

`libraries/chain/spv/` is a new chain-agnostic SPV layer. It exposes a pure C++ interface with no BitShares consensus dependence:

- Header verification — verify proof-of-work, link to parent, anchor to checkpoint
- Difficulty adjustment — retarget math per chain
- Merkle path verification — prove a transaction is inside a block header
- Confirmation counting — count blocks deep since the transaction's block

Chain-specific parameters live in a small parameter struct (magic bytes, proof-of-work algorithm, target block time, confirmation depth, checkpoint). All SPV logic is chain-agnostic; only the parameters and byte layouts differ per chain.

## 3. Chain implementations (phase 1)

| Chain | PoW algorithm | Target block time | Confirmation depth |
|-------|--------------|-------------------|-------------------|
| Bitcoin | SHA-256d | ~10 minutes | 6 |
| Litecoin | scrypt | ~2.5 minutes | 12 |
| Dash | X11 | ~2.5 minutes | 12 |
| Dogecoin | scrypt | ~1 minute | 24 |
| Bitcoin Cash | SHA-256d | ~10 minutes | 6 |

Each chain is a parameter struct plus byte-layout handling. No new verification logic per chain — DRY from onset.

## 4. Gateway declaration with SPV capability

`accs_declare_gateway_operation` is extended (appended-last) with `spv_capable` flag and `checkpoint` header hash. Committee members declare whether a gateway chain accepts SPV proofs. For non-SPV chains, the flag is false and the F1 oracle path is used unchanged.

## 5. Dual-path fulfillment

### Attestor path (unchanged, all chains)

The attestor publishes the foreign-chain event to the F1 foreign-tx oracle. Fulfillment matches against the promoted oracle value by `foreign_tx_id` equality, exactly as today. Required for non-SPV chains; optional fallback for UTXO chains.

### SPV path (primary, UTXO chains only)

The seller submits a self-contained SPV proof with the fulfillment transaction:

- The foreign-chain transaction sending funds to the buyer's declared address
- A merkle path proving the transaction is inside a block
- Block headers chained from that block back to the gateway's declared checkpoint
- Proof-of-work valid on each header
- Confirmation depth at or above the per-chain minimum

BitShares verifies the proof in `do_evaluate` (never `validate()`). If valid, escrow is released to the buyer and the griefing bond is returned to the seller.

## 6. Checkpoint anchoring

Every SPV-capable gateway declares a checkpoint: a trusted block header hash for the foreign chain at gateway declaration time. All SPV proofs must chain back to this checkpoint. The checkpoint may be advanced by a subsequent gateway declaration (committee operation), never rolled back.

## 7. Replay protection (unchanged)

`accs_consumed_tx_object` records one row per fulfilled transaction, keyed (chain_id, foreign_asset_id, tx). Rows are durable, never edited or removed, capped at 65536 per pair. Applies identically to both fulfillment paths.

## 8. Escrow, griefing bond, SLA (unchanged)

All ACCS economics from the F series carry over unchanged: seller locks base asset in escrow at offer creation; buyer posts griefing bond at match; SLA deadline set at match; fulfillment releases escrow to buyer and returns bond to seller.

# Rationale

- **SPV where it exists, attestor where it doesn't:** The five UTXO chains have SPV-friendly headers; Ethereum, Solana, and Polkadot do not. Using SPV where available removes trust without forcing a single model on all chains.
- **SPV primary, attestor appeal:** The SPV path handles normal fulfillment without an attestor. The attestor path remains as appeal and fallback, providing economic recourse (bond slashing) where SPV has none. Requiring both would defeat the purpose; dropping the attestor entirely would remove recourse.
- **No witness keys:** The SPV proof is submitted by the seller with the BitShares fulfillment transaction. Witnesses are never asked to hold foreign-chain signing keys or report foreign-chain tips. This eliminates the witness honey pot that a vault model creates as locked value grows.
- **Reuse F substrate:** Escrow, griefing bond, SLA, replay protection, gateway declaration — all audited and gated in the F series. BSIP-9 adds SPV verification and a dual-path fulfill, nothing more.
- **DRY from onset:** one abstract SPV layer serves all five UTXO chains with only parameter struct changes.

# Risks and failure modes

- **SPV is trust-minimized, not trustless:** SPV assumes honest majority hash power. A 51% miner can build a fake chain with valid proofs; there is no bond to slash and no economic recourse. The attestor appeal path provides limited recourse within the challenge window.
- **SPV reorg:** A deep foreign-chain reorg could invalidate a verified deposit. Mitigated by per-chain confirmation depth and checkpoint anchoring. Challenge windows should exceed confirmation depth.
- **Client-side SPV burden:** The seller must assemble SPV headers and merkle paths client-side (run an SPV node or query one). This is client-side software, not consensus code.
- **Checkpoint trust:** The starting header is trusted. If corrupted, SPV proofs chain from a false anchor. Mitigated by hardfork-gated checkpoint updates.
- **Attestor path remains for non-SPV chains:** Chains without SPV keep the bonded-attestor trust model. This is a known, accepted limitation.
- **Proof size:** A BTC SPV proof with 6 confirmations is roughly 1–2 KB. Fits in a BitShares transaction but larger than a simple hash.
- **Legacy assets untouched:** `bit*`, `HONEST.*`, and gateway UIAs keep their semantics, fees, and settlement rules; no rule in this design can move, cap, or re-price them.

# Backwards compatibility

- Nothing breaks for existing holders, traders, or witnesses. Legacy ACCS fulfillment via attestor executes unchanged code.
- All new fields are appended, never inserted, and every new op/field is hardfork-gated in the evaluators and the proposal visitor.
- Non-UTXO chains are unaffected: the `spv_capable` flag defaults to false, and the F1 oracle path remains the only path for those chains.
- Wallets and tools that do not know SPV proofs keep working on legacy paths; SPV proofs appear only behind the SPV fulfillment hardfork gate.

# Test cases

- Tag stability: tag 118 (`accs_fulfill_spv_operation`) holds across all BSIP configurations.
- Gate ordering refusal: SPV gate after F_ACCS gate; inverted schedule refused.
- SPV fulfill success per chain: BTC, LTC, DASH, DOGE, BCH with real testnet/mainnet tx hashes and merkle paths.
- SPV bad merkle root refused.
- SPV insufficient confirmations refused.
- SPV replay refused via `accs_consumed_tx_object`.
- Attestor path on non-SPV chain (ETH/SOL) unchanged.
- SPV path on non-SPV chain refused.
- Gateway declaration with `spv_capable` flag and checkpoint.
- Checkpoint declared, advanced by committee, rollback refused.
- Confirmation minimums as protocol constants (6, 12, 12, 24, 6).
- Per-chain address validation.
- `do_evaluate` (not `validate()`) for all SPV checks.
- Wire pinning across builds with and without the SPV hardfork flag.
- Attack tests: tampered header, wrong merkle root, orphan header, checkpoint mismatch, insufficient confirmations.
- No witness involvement in SPV path.
- Legacy assets untouched.

# Reference implementation

Branch: `feature/track-b-h9-spv-accs`. Design spec: `docs/superpowers/specs/2026-11-13-bsip9-spv-accs.md`. Build plan: `docs/superpowers/plans/2026-11-13-bsip9-spv-accs-build.md`. Implementation status, open findings, and activation gates are tracked in `build/ACTIVATION-CHECKLIST.md`. New files: `libraries/chain/spv/` (7 headers + 7 cpp files). Modified files: `accs_ops.hpp`, `accs_ops.cpp`, `accs_object.hpp`, `accs_object.cpp`, `accs_evaluator.cpp`, `accs_evaluator.hpp`, `hardfork_visitor.hpp`, `db_init.cpp`, `hardfork.d/SPV_ACCS.hf`, `operations.hpp`, `oracle.hpp`, `tests/tests/accs_tests.cpp`.

# Discussion

- Rejected alternatives: vault custody model (retired as H3, 2026-10-24); witness-level SPV tip reporting (rejected — creates witness honey pot); requiring both SPV and attestor paths (defeats the purpose).
- Contested threads: hardfork dates, confirmation depths per chain, checkpoint update procedure, challenge window length relative to confirmation depth.
- Audit must-close list: `build/audit-round-bsip9-spv-accs/` seats (bsip3-xclaim-accs-audit, consensus-determinism-audit, spv-light-client-audit, cpp-code-quality-audit, crypto-agility-audit, track-independence-audit). Activation remains unauthorized until every row of `build/ACTIVATION-CHECKLIST.md` clears.

# See Also

- BSIP-3: `docs/bsips/bsip-3-xclaim-accs.md`
- F1 oracle spec: `docs/superpowers/specs/2026-10-20-f1-foreign-chain-tx-oracle.md`
- F2 ACCS spec: `docs/superpowers/specs/2026-10-20-f2-accs-orderbook-settlement.md`
- H3 retired design: `docs/superpowers/specs/2026-10-20-h3-bitcoin-accs-foreign-chain-vaults-design.md`
- Split design: `docs/superpowers/specs/2026-10-24-three-track-split-design.md`
- XCLAIM (eprint 2018/643); interBTC; tBTC/Keep; SPV (nakamoto consensus light-client).
- Activation gates: `build/ACTIVATION-CHECKLIST.md`

# Copyright

This document is placed in the public domain.
