---
name: spv-light-client-audit
description: >
  Use when auditing SPV (Simplified Payment Verification) / light-client
  verification constructs grafted into consensus: header-chain verification,
  proof-of-work difficulty checks, merkle-path authentication, checkpoint
  anchoring (weak subjectivity), and dual-path fulfillment where SPV coexists
  with a bonded-attestor fallback. Triggers: new SPV chain implementations,
  checkpoint trust models, confirmation-depth thresholds, SPV-to-attestor
  appeal paths, reorg risk pricing. Version: 1.0.0. License: MIT.
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Skill: SPV Light-Client Audit

## Overview

SPV verification in consensus is a novel construct for BitShares. It shifts
trust from a single attestor to honest-majority hash power plus checkpoint
integrity. Every construct must be re-derived from code — not inherited from
the design doc. The audit focuses on: can a false SPV proof be accepted?
Can a checkpoint be poisoned? Can the SPV path and attestor path disagree
unsafely? Is reorg risk priced honestly?

## When to Use

- New SPV chain implementation (Bitcoin SHA-256d, Litecoin scrypt, Dash X11,
  Dogecoin scrypt, Bitcoin Cash SHA-256d)
- Checkpoint anchoring / weak-subjectivity model
- Confirmation-depth thresholds per chain
- Dual-path fulfillment (SPV primary + attestor appeal)
- Reorg handling and false-finality claims
- Any SPV proof submitted in a `do_evaluate` money path

## Rules (no exceptions)

1. **SPV doc is a claim, not evidence.** Re-derive every verification step
   from code. Cite `file:line` for the equation enforced, not the spec section.
2. **Test the adversary, not the happy path.** Each check needs a refusal
   vector: tampered header, wrong merkle root, insufficient confirmations,
   orphan chain, checkpoint mismatch.
3. **Byte-compare transcripts.** Header byte layout (version, prev_hash,
   merkle_root, timestamp, bits, nonce positions) must match the chain's
   actual format byte-for-byte. No paraphrase acceptance.
4. **Checkpoint trust is explicit.** The checkpoint source, update mechanism,
   and rollback prevention must be coded, not assumed.
5. **Dual-path safety.** SPV and attestor paths must not create an unsafe
   disagreement: if both can fulfill the same offer, the conflict resolution
   (challenge window, appeal override) must be enforced in code.

## Checklist (every item, in order)

1. **Chain parameter correctness.** Each chain's magic bytes, PoW algorithm,
   target block time, confirmation depth, and checkpoint default match the
   chain's actual parameters. Confirm depths are constants, not governance
   parameters. Cite `file:line` for each chain.
2. **Header verification.** PoW difficulty check enforced: header hash below
   target. Parent linkage: header's prev_hash equals parent's hash. Cite the
   refusal vectors for invalid PoW and broken linkage.
3. **Merkle path authentication.** Merkle path proves the transaction is in
   the block's merkle root. Left/right position handling correct (or both
   orientations tried). Cite the refusal vector for a path that doesn't
   authenticate.
4. **SPV proof assembly.** Raw transaction, tx block header, merkle path,
   header chain to checkpoint all required. Missing any component refuses.
   Cite the refusal vector for each missing component.
5. **Checkpoint anchoring.** Every SPV proof chains back to the declared
   checkpoint. Checkpoint update is committee-gated, never rolled back.
   Cite the enforcement and the rollback-prevention line.
6. **Confirmation depth enforcement.** Proof refused below per-chain minimum.
   Depth checked against the checkpoint-anchored header count, not
   self-reported.
7. **Dual-path fulfillment.** SPV path and attestor path for the same offer:
   no double fulfillment (replay guard shared), conflict resolution via
   challenge window, appeal override enforceable.
8. **do_evaluate vs validate().** SPV verification in `do_evaluate` (money
   path), never `validate()` (stateless). Cite the file:line of each SPV
   check and confirm it is in `do_evaluate`.
9. **No witness honey pot.** Witnesses do not submit SPV tips, hold foreign
   chain keys, or participate in the SPV path. Confirm the SPV proof is
   submitted by the seller/user client-side.
10. **Test vectors.** Real BTC/LTC/DASH/DOGE/BCH transaction hashes and
    merkle paths (testnet and/or mainnet). Wrong chain, wrong recipient,
    insufficient confirmations, orphan header, checkpoint mismatch all have
    refusal vectors. A defense with no test is a finding.

## Output

Numbered PASS/FAIL/QUALIFIED list with `file:line` evidence. Every FAIL gets
the smallest concrete fix. Ambiguity is a finding.

```json
{"skill":"spv-light-client-audit","auditor_model":"<exact model string — mandatory>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"SPV-01","severity":"Critical|Important|Minor|Note","check":"chain-params|header-verify|merkle-path|proof-assembly|checkpoint|confirm-depth|dual-path|do-evaluate|no-witness-honeypot|test-vectors","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["checklist items not reached"]}
```

Severity: `Critical` = false SPV proof accepted (forgery, inflation, double
spend via fake lock). `Important` = soundness break under plausible
conditions, or checkpoint trust not codified. `Minor`/`Note` = hardening and
doc deltas.

## Rationalizations (all STOP)

| Excuse | Reality |
|--------|---------|
| "SPV is standard, Bitcoin Core does it" | BitShares consensus is not Bitcoin Core; re-derive from code. |
| "Checkpoint is just a default" | Default checkpoint without update/rollback protection is a trust anchor. |
| "Attestor path is the backstop" | Backstop must be enforceable, not aspirational. Cite the override line. |
| "Happy-path test passes" | Soundness IS the refusal set. No refusal vectors = no verdict. |

## Red Flags — STOP and re-derive

- Citing a spec section as evidence for a code property.
- Checkpoint update without rollback prevention.
- SPV verification in `validate()` instead of `do_evaluate`.
- A verdict with zero refusal vectors.
