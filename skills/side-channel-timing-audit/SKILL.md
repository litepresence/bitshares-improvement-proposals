---
name: side-channel-timing-audit
description: >
  Use when auditing timing side-channels, constant-time implementation,
  or benchmark methodology for PRG-05 (Ring-11/16 full-block loops on
  target hardware). Triggers: reviewing consensus hot paths
  (stealth_verify_ring, stealth_verify_bp_plus, stealth_verify_excess,
  stealth_verify_value_pin, stealth_verify_recovery_proof, hash_to_point)
  for variable-time operations, or validating target-hardware benchmark
  evidence.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Side-Channel Timing Audit

## Overview

The work-budget gate uses deterministic integer work units (750,000
units/block envelope), but consensus code runs on OpenSSL paths that
are **not constant-time** for secret scalars. PRG-05 explicitly
requires: "Ring-11 AND ring-16 full-block loops on 3s blocks with
headroom on TARGET hardware. Container numbers above are not target
hardware." This skill audits: (a) timing side-channels in consensus
hot paths, (b) constant-time implementation verification, and (c)
target-hardware benchmark methodology.

## When to Use

- Auditing `stealth_verify_ring`, `stealth_verify_bp_plus`,
  `stealth_verify_excess`, `stealth_verify_value_pin`,
  `stealth_verify_recovery_proof`, `hash_to_point` for variable-time
  operations
- Validating PRG-05 benchmark evidence on target hardware
- Reviewing `stealth_work_for_*` constants against empirical measurements

Do NOT use for: wallet prover timing (`wallet-prover-crypto-audit`),
formal verification of crypto (`formal-verification-audit`), or general
performance profiling.

## Rules (no exceptions)

1. **Consensus is single-threaded and observable.** BitShares block
   validation runs single-threaded. Wall-clock timing of an operation
   is observable by the network (block production timing). An attacker
   who crafts operations with secret-dependent verification time can:
   fingerprint their ops, bias block production (if witness), leak ring
   member position / value bits through latency.

2. **Integer work model is the gate; wall-clock is corroboration.**
   `stealth_work_for_*` functions compute **pure integer arithmetic** on
   public wire counts. This is the consensus gate. Wall-clock
   measurements only corroborate that the integer model is a safe upper
   bound on target hardware.

3. **OpenSSL default paths are variable-time.** `EC_POINTs_mul` with
   secret scalars, `BN_mod_inverse`, `BN_cmp`, `EC_POINT_oct2point`,
   `EC_POINT_is_on_curve` — all have data-dependent timing in default
   OpenSSL. This is not a bug; it's the library default. The audit must
   quantify the leakage and decide if it's acceptable.

4. **Target hardware ≠ CI container.** Container benchmarks measure
   noise (noisy neighbors, CPU steal, hyperthreading). PRG-05 explicitly
   rejects them. The benchmark must run on isolated, documented target
   hardware with fixed frequency, pinned core, disabled turbo.

5. **Nullifier lookup timing is public.** The `stealth_nullifier_index`
   is public state. Presence/absence is public knowledge. Timing leak
   reveals nothing secret — accept or use constant-time set.

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (every item, in order)

### Hot-Path Timing Analysis

1. **Dual ring verification (`stealth_verify_ring`, line 555).**
   - `EC_POINTs_mul` 3-term per member (L, R, M legs) with secret
     scalars `rb_i, rv_i, rd_i, c_i` — OpenSSL **variable-time** for
     secret scalars.
   - `scalar_parse_strict` (line 116): `BN_cmp` vs `secp_order()`
     — `BN_cmp` is variable-time.
   - `point_from_bytes` / `point_from_commit`: `EC_POINT_oct2point`
     + `EC_POINT_is_on_curve` + `EC_POINT_is_at_infinity` — all
     variable-time for invalid encodings (early exit).
   - Duplicate ring-member check: `EC_POINT_cmp` O(n²) — timing
     reveals which member matched.
   - **Quantify:** max timing variance across all scalar values.

2. **R15 recovery proof (`stealth_verify_recovery_proof`, line 710).**
   - 3× batched `EC_POINTs_mul` (2-term each) with secret scalars
     `s_b, s_v` (×3 legs).
   - `scalar_parse_strict` on 5 scalars — `BN_cmp` variable-time.
   - Challenge `c` derived from 163B wire; `c==0` early reject
     (probability 2⁻²⁵⁶ — acceptable).

3. **Conservation kernel (`stealth_verify_excess`, line 814).**
   - Loop over inputs/outputs (≤8 each): `EC_POINT_mul` +
     `EC_POINT_add` per commitment.
   - `EC_POINT_invert` per input.
   - Scalars `excess, fee` are **public** (wire values) — timing
     leakage acceptable.

4. **G2 Schnorr value-pin (`stealth_verify_value_pin`, line 863).**
   - `EC_POINT_mul` for `amount×H`, `e×Y`, `s×G`.
   - `EC_POINTs_mul` 2-term for RHS (`s×G = T + e×Y`).
   - `scalar_parse_strict` on `s` — `BN_cmp` variable-time.
   - Challenge `e==0` early reject (probability 2⁻²⁵⁶ — acceptable).

5. **Bulletproofs+ V2 (`stealth_verify_bp_plus`, line 982).**
   - **Chained transcript (R9C)**: 7 rounds × `stealth_bp_hash`
     (SHA-256) — hash is constant-time.
   - `mod_inv` per round challenge — `BN_mod_inverse` is **highly
     variable-time** (extended Euclid). Must use constant-time Fermat
     inversion (`x^(p-2) mod p`) for secp256k1 order.
   - `EC_POINT_point2oct` for V commitments (33B each).
   - Generator cache (64 `Gi/Hi`) — timing-only on first use.

6. **Hash-to-point (`hash_to_point`, line 232).**
   - Try-and-increment loop (1–256 iterations) — **variable loop
     count** leaks `domain‖tag` prefix entropy.
   - `BN_mod_exp` with `(p+1)/4` exponent (fixed) but input-dependent.
   - **Runs once at genesis** (static generators cached via
     `call_once`) — not in hot path. Acceptable if padded to 256.

### Constant-Time Verification

7. **Replace OpenSSL EC multiexp.** Use `secp256k1` library's
   `ecmult_multi` (constant-time) or custom Pippenger implementation.
   Verify via `ctgrind` (Valgrind) on `stealth_verify_ring` with secret
   scalars.

8. **Replace `BN_mod_inverse`.** Use constant-time Fermat inversion:
   `pow_mod(x, p-2, p)` for secp256k1 order (prime). Verify via
   `dudect` 10⁶ traces.

9. **Pad `hash_to_point` to 256 iterations.** Or use constant-time map
   (SWU). Verify loop count distribution is constant.

10. **Blind scalars in multiexp.** For `stealth_verify_ring` and
    `stealth_verify_bp_plus`, blind secret scalars: `s' = s + r·n`
    where `r` random, `n` curve order. Verify blinding in multiexp
    implementation.

11. **Constant-time point decompression.** Replace
    `EC_POINT_oct2point` + checks with constant-time decode (e.g.,
    `secp256k1_ec_pubkey_parse`).

12. **Constant-time scalar compare.** Replace `BN_cmp` with
    `CRYPTO_memcmp` or constant-time limb-wise compare.

### Target-Hardware Benchmark Methodology (PRG-05)

13. **Hardware specification.** Document exact CPU model, microcode,
    cache topology, frequency scaling (disable turbo), pinned core
    (`taskset`), disabled hyperthreading/ASLR.

14. **Workload enumeration.** All consensus-valid shapes:
    - `stealth_transfer`: inputs 1..8, ring 16..32, outputs 1..8, V1/V2
    - `turnstile_reproof`: 1-in/1-out + recovery proof
    - `stealth_explicit_spend`: inputs 1..8, outputs 0..8, recovery proofs
    - `stealth_pool_exchange`: ring 16..32, 1-in/1-out + value-pin + pool math
    - Legacy ops 39/40/41 (charged at BP+ rate)

15. **Measurement protocol.**
    - Warmup: 10k iterations (steady state)
    - Measurement: 100k iterations per shape
    - Metrics: min, p50, p99, p99.9, max (cycles, instructions, L1/L2/L3 misses, branch mispredicts)
    - 5+ reboots for microcode/cache variance

16. **Model validation.** For each component (ring-member, range-proof,
    kernel, pool-math, recovery-proof): measure in isolation, fit
    linear model `time = a × count + b`, verify residuals < 5%.

17. **Headroom calculation.** Target: 3s block time. Required:
    `max_measured_p99.9 × 1.5 < 750,000 µs`. If fails: raise work-unit
    constants or lower ring/output caps.

## Output

```json
{"skill":"side-channel-timing-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","target_hw":"<cpu_model>",
 "hot_paths":{"ring_verify":{"openssl_ecmult":"variable-time",
  "bn_cmp":"variable-time","point_decode":"variable-time",
  "dup_check":"O(n2) leak","ctgrind":"PASS|FAIL","blinded":true},
  "recovery_proof":{"ecmult":"variable-time","bn_cmp":"variable-time",
   "ctgrind":"PASS|FAIL"},
  "kernel":"public-scalars-ok","value_pin":{"ecmult":"variable-time",
   "bn_cmp":"variable-time","ctgrind":"PASS|FAIL"},
  "bpplus":{"mod_inv":"variable-time (BN_mod_inverse)",
   "fermats_inversion":"PASS|FAIL","ctgrind":"PASS|FAIL"},
  "hash_to_point":{"loop_count":"variable (1-256)","padded_to_256":true}},
 "benchmark":{"target_hw":"<cpu_model>","shapes_measured":["all"],
  "p999_max_us":0,"headroom_x1_5":true,"reboots":5},
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],
 "not_covered":["nullifier lookup (public state)","legacy op39/40/41",
  "BP+ m=2 (not wired)","genesis-time hash_to_point"]}
```

Severity (aligned with the program round discipline; the brief's rule overrides this line when they conflict): `Critical` = funds/consensus/invariant loss reachable through the timing surface itself. **Missing PRG-05 evidence is an external-gate NOT_COVERED row, never a code Critical** (round-17 SCT-R17-01 correction: missing dynamic evidence is a gate, not a defect). `Important` = variable-time path quantified but not fixed, benchmark methodology incomplete. `Minor`/`Note` = doc deltas.

## Rationalizations (RED baseline 2026-10-12)

| Excuse | Reality |
|---|---|
| "Work units are integer, not wall-clock" | True for consensus gate, but PRG-05 requires target-hardware corroboration that the integer model is a safe upper bound. |
| "Benchmarks pass in CI" | CI container noise invalidates wall-clock numbers. PRG-05 explicitly rejects container measurements. |
| "Target hardware not available yet" | The skill documents the methodology; gate closes when hardware exists and evidence is produced. |
| "Constant-time is already enforced by libraries" | OpenSSL default EC multiexp, mod_inv, BN_cmp are NOT constant-time for secret data. |

## Red Flags — STOP and quantify

- PRG-05 closed with only container benchmarks.
- `BN_mod_inverse` used in BP+ verify without constant-time replacement.
- `EC_POINTs_mul` with secret scalars not blinded or replaced.
- `hash_to_point` loop count not padded to constant.
- Work-unit constants (`WORK_US_PER_*`) derived from container measurements.
- No `ctgrind`/`dudect` run on consensus verifier hot paths.

**All of these mean: the timing audit is NOT complete. PRG-05 remains NO_GO.**