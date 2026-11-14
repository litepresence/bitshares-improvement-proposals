---
name: formal-verification-audit
description: >
  Use when auditing the proof-to-implementation correspondence,
  machine-checked verification, or compositional soundness of the
  stealth-transfer + stealth-pool-swap cryptography (CLSAG, BP+, Schnorr
  value-pin, conservation kernel, nullifier set). Triggers: G1-05
  specialist gate review, kernel-proof.md verification, or any claim that
  a formal proof covers the implementation.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Formal Verification Audit

## Overview

The kernel soundness proof (`2026-10-10-stealth-pool-swaps-kernel-proof.md`)
explicitly states: "Not machine-checked." It proves Lemmas 1, 3, 4;
cites Lemmas 2, 5, 6 as obligations (M-leg 3-response soundness, BP+
soundness, nullifier mechanism). This skill audits: (a) wire-format
fidelity between proof and implementation, (b) machine-checked
verification of the *verifier* code (not prover), and (c) compositional
soundness via assume-guarantee contracts.

## When to Use

- Reviewing the G1-05 specialist audit gate requirements
- Verifying the `2026-10-10-stealth-pool-swaps-kernel-proof.md` against `stealth_evaluator.cpp`
- Auditing proof-to-implementation correspondence for CLSAG, BP+, Schnorr pin, kernel, nullifier
- Any claim that a formal proof covers the stealth branch implementation

Do NOT use for: implementation audit of primitives
(`clsag-bp-plus-auditor`), wallet prover audit (`wallet-prover-crypto-audit`),
or side-channel timing (`side-channel-timing-audit`).

## Rules (no exceptions)

1. **Verifier is the TCB, not prover.** Machine-checked verification
   targets the *consensus verifier* (`stealth_evaluator.cpp`,
   `stealth_proof.cpp` verifier functions). Prover bugs = failed tx,
   not chain halt. Prover correctness is a separate wallet audit.

2. **Wire-format is the contract.** Every scalar, point, and domain
   separator in the proof spec must map 1:1 to the serialized bytes in
   the evaluator. Endianness, curve encoding, hash absorption order —
   a single byte mismatch = soundness break.

3. **Fiat-Shamir transcript is non-negotiable.** The hash chain
   `H(pp || stmt || com_1 || ... || com_n)` must use the same hash
   function, same domain separators, same absorption order as the
   proof spec. Verify byte-for-byte against `stealth_challenge`,
   `stealth_ring_message`, `stealth_pool_message`, `stealth_bp_hash`.

4. **Assume-guarantee composition.** Each component (CLSAG, BP+, pin,
   kernel, nullifier) has an assumption (RO model, DDH, discrete log)
   and a guarantee (unforgeability, range soundness, value binding).
   The audit must prove the *glue*: the output of one is the exact
   input to the next — no coercion, truncation, or reinterpretation.

5. **Nullifier insertion is atomic with kernel verification.** The
   state machine `note_created → note_spent(nullifier) →
   kernel_verified → nullifier_inserted` has no interleaving. This
   must be proven by code inspection of `do_evaluate`/`do_apply`.

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (every item, in order)

### Wire-Format Fidelity

1. **CLSAG 3-response structure.** Verify `stealth_verify_ring`
   (`stealth_proof.cpp:555` at 8de355a3) parses: `c_0` (32B), then per
   member `s_{i,b}`, `s_{i,v}`, `s_{i,d}` (3×32B). NOTE: the wire gate is
   exactly `in.ring_sig.size() != 32 + 96 * n` (:567); `M_i`/`L_i`/`R_i`
   are NOT wire-parsed — `D_i = P − Q_i` is computed (:606-610) and the
   M/L/R legs are EC multiexps; duplicate-member check is point-based
   `EC_POINT_cmp` (:594-596). Match to `stealth_proof.hpp` serialization
   and `stealth_evaluator.cpp` ring-sig bytes.

2. **BP+ chained transcript (R9C).** Verify `stealth_verify_bp_plus`
   (:982) uses: `state_0 = z`, `e_j = H("G1BPr", state_{j-1}, L_j,
   R_j)`, `state_j = e_j`, `e = H("G1BPf", state_last, A1, B)`. Match
   to `stealth_bp_hash` (line 1067) labels `G1BPy/G1BPz/G1BPr/G1BPf`.

3. **G2RM ring message.** Verify `stealth_pool_message` (line 475) preimage:
   `"G2RM" ‖ version ‖ LE64(n) ‖ LE64(m) ‖ asset_A(varint) ‖ pool(varint)
   ‖ amount(LE64) ‖ min(LE64) ‖ image33 ‖ ring[33n] ‖ pseudo[33] ‖
   outputs[33m] ‖ F(LE64)`. Match to `stealth_proof.hpp` encoding table.

4. **G2VP Schnorr pin challenge.** Verify `stealth_verify_value_pin`
   (:863) computes `e = H("G2VP", P(33), amount(LE64), pool(varint),
   T(33))`. Match to `stealth_proof.hpp` encoding table.

5. **Nullifier derivation.** Verify `nullifier_key`
   (`stealth_evaluator.cpp:109-115`) computes `N = sha256(image33)` —
   33 raw image bytes, **no version byte, no spend_key** (the earlier
   `sha256(version_byte || spend_key)` pin and "stealth_proof.hpp eqns
   182-185" were wrong: the header is
   `libraries/chain/include/graphene/chain/stealth_proof.hpp`, 352 lines).
   Per-version images `I_v = b*J1_v + v*J2_v` — prover
   `stealth_prover.cpp:225-232`, recovery verifier checks the same linear
   map (`stealth_proof.cpp:803-806`); every spend path records
   `sha256(I)` (FORMAL-14-03; the byte-distinct-encoding residual was
   empirically refuted at round 17, FV-R17-04).

### Machine-Checked Verification (minimum viable)

6. **F* / Low* for verifier functions.** Target the five actual verifier
   functions: `stealth_verify_ring` (:555), `stealth_verify_bp_plus`
   (:982), `stealth_verify_value_pin` (:863), `stealth_verify_recovery_proof`
   (:710), `stealth_verify_excess` (:814) in `stealth_proof.cpp`, plus
   their evaluator entry points (pin invoke `stealth_evaluator.cpp:2336`,
   link-4 :2371-2373, kernel :2438, nullifier :2420-2422). Do NOT name
   `verify_clsag`/`verify_nullifier_uniqueness` — those do not exist.
   Prove: memory safety, no arithmetic overflow/underflow, transcript
   absorption order fixed, no early-exit on secret data.

7. **Coq + Fiat-Crypto for field arithmetic.** Port `secp256k1`
   scalar mul, point decompression, `mod_inv` to Coq. Prove
   equivalence to spec. Leverage existing Fiat-Crypto `curve25519`
   library adapted to secp256k1 order.

8. **Lean 4 for kernel soundness (Lemmas 1, 3, 4).** Formalize:
   - Lemma 1: Schnorr value-pin `s·G = T + e·(P − a·H)` ⇒ `P = a·H + b·G`
   - Lemma 3: Kernel `excess·G + F·H + C_out − P = 0` ⇒ `v_out = a − F`
   - Lemma 4: Link-4 `a − F == pool_pays − f_taker` ⇒ `v_out = pool_pays − f_taker`
   Assume CLSAG/BP+ axioms (Lemmas 2, 5, 6).

### Compositional Soundness (Assume-Guarantee)

9. **CLSAG guarantee → nullifier linkage.** Ring sig `c_i = H(msg)`,
   `s_i = k_i + c_i·x_i`. Nullifier `N = H(x_i || "nullifier")`. Prove:
   same `x_i` used in both; ring member `P_i = x_i·G` matches nullifier
   input; no equivocation possible.

10. **BP+ guarantee → kernel value binding.** BP+ proves `V = v·G + r·H`
    with `v ∈ [0, 2^64)`. Kernel uses same `v` in `excess·G + F·H + V_out −
    P = 0`. Prove: no coercion of `v` between BP+ and kernel (same
    integer width, same endianness, no truncation).

11. **Pin → kernel value binding.** Schnorr pin binds `P = a·H + b_P·G`
    with `a = amount_to_sell`. Kernel uses `P` in excess equation.
    Prove: `a` in pin is identical to `a` in kernel (same LE64 wire).

12. **Nullifier atomicity with kernel.** In `do_evaluate`/`do_apply`:
    ring verify → excess kernel → nullifier check → nullifier insert.
    Prove: no code path reaches nullifier insert without both ring
    verify AND kernel verify succeeding.

## Output

```json
{"skill":"formal-verification-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","phase":"wire-format|machine-checked|compositional|all",
 "wire_format":{"clsag":"PASS|FAIL","bpplus":"PASS|FAIL",
  "g2rm":"PASS|FAIL","g2vp":"PASS|FAIL","nullifier":"PASS|FAIL"},
 "machine_checked":{"fstar_verifier":"PASS|FAIL","coq_arithmetic":"PASS|FAIL",
  "lean_kernel":"PASS|FAIL"},
 "compositional":{"clsag_nullifier":"PASS|FAIL","bpplus_kernel":"PASS|FAIL",
  "pin_kernel":"PASS|FAIL","nullifier_atomic":"PASS|FAIL"},
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],
 "not_covered":["prover correctness (off-chain)","network adversary model",
  "hardware side-channels"]}
```

Severity: `Critical` = wire-format mismatch, machine-checked gap on verifier TCB, compositional glue missing, or atomicity broken. `Important` = F*/Coq/Lean not yet complete but wire-format OK. `Minor`/`Note` = doc deltas.

## Rationalizations (RED baseline 2026-10-12)

| Excuse | Reality |
|---|---|
| "The proof doc looks right" | A proof doc is a claim. Wire-format fidelity must be byte-verified against the evaluator. |
| "Machine-checked proofs take months" | Minimum viable: F* on verifier (~2k lines) + Coq on arithmetic + Lean on kernel (~200 lines math). Weeks, not months. Prover excluded. |
| "Internal audits 1-8 covered the code" | They audited implementation, not proof-to-code correspondence. The gap is exactly the G1-05 specialist gate. |
| "Just get the specialist to sign off" | Specialist sign-off without machine-checked evidence is a vibe, not a gate. G1-05 requires formal proof review. |

## Red Flags — STOP and verify

- Wire-format check done by "reading the spec" not byte-comparing to evaluator.
- Machine-checked verification targets prover, not verifier.
- Fiat-Shamir transcript labels paraphrased, not byte-matched.
- BP+ chained transcript (R9C) not verified against `stealth_bp_hash`.
- Nullifier insertion not proven atomic with kernel verify.
- Compositional audit assumes CLSAG/BP+ axioms without citing specialist sign-off.

**All of these mean: the formal verification is NOT complete. NO_GO on G1-05.**