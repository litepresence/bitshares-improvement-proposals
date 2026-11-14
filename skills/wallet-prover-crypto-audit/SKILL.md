---
name: wallet-prover-crypto-audit
description: >
  Use when auditing the G2 wallet prover (`libraries/chain/stealth_prover.{hpp,cpp}`)
  for cryptographic safety: secret handling, RNG, zeroization, constant-time
  discipline, key derivation, memory safety. Triggers: reviewing prover
  code changes, auditing off-chain secret management, or closing the
  prover safety gap (no existing skill covers this).
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Wallet Prover Cryptographic Audit

## Overview

The G2 wallet prover (`stealth_prover.cpp`) is NEW code handling:
Schnorr nonces (`k`), pseudo-output blinding (`b_P`), `RAND_bytes` RNG,
zeroization after use, constant-time discipline. This skill audits
off-chain secret management — the only skill that does. Prover bugs =
user fund loss (not chain halt), but prover is the only way honest
users construct valid transactions.

## When to Use

- Reviewing `stealth_prover.{hpp,cpp}` changes
- Auditing off-chain secret handling (nonces, blinding factors, RNG)
- Closing the prover safety gap (no existing skill covers this)

Do NOT use for: consensus verifier audit (`clsag-bp-plus-auditor`),
formal verification (`formal-verification-audit`), or side-channel
timing of consensus code (`side-channel-timing-audit`).

## Rules (no exceptions)

1. **No secret in observable output.** Secrets (`k`, `b_P`, derived
   keys) must never appear in: stdout/stderr, log files, coredumps,
   `assert()` messages, `FC_REFLECT` serialization, JSON RPC, or any
   debug instrumentation.

2. **No secret-dependent control flow.** Zero branches on secret bits:
   no `if (secret)`, `while (secret)`, `switch (secret)`,
   `secret ? a : b`. No secret-dependent loop counts.

3. **No secret-dependent memory access.** Zero table/array/pointer
   lookups indexed by secret: no `table[secret]`, `ptr[secret]`,
   `vec[secret]`. Use constant-time select (`ct_select`) or masked
   operations.

4. **Ephemeral secrets are single-use.** Schnorr nonce `k` generated
   fresh per signature. Reuse = private key recovery. Pseudo-output
   blind `b_P` single-use per transaction.

5. **Domain separation for all derivations.** Every HKDF expansion
   uses a unique, versioned label: `HKDF-Expand(PRK,
   "bts-stealth-prover/<purpose>/v1", L)`. No label reuse across
   `k`, `b_P`, recovery proof scalars.

6. **Zeroization survives compiler optimization.** `OPENSSL_cleanse`
   or `sodium_memzero` + compiler barrier (`atomic_signal_fence`,
   `asm volatile`). Never `memset` (dead-store eliminated).

7. **Error paths zeroize.** Every `return`, `throw`, early-exit,
   `FC_ASSERT` failure must zeroize secrets before unwinding. RAII
   wrappers required.

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (every item, in order)

### Secret Handling

1. **Scan for secret leaks.** Grep prover code for: `std::cout`,
   `std::cerr`, `ilog`, `elog`, `wlog`, `FC_LOG`, `std::to_string`,
   `boost::lexical_cast`, `fc::variant`, `fc::json::to_string`,
   `FC_REFLECT` on secret types. **Must find zero hits.**

2. **Secret-dependent branches.** Audit every `if`, `while`, `for`,
   `switch`, ternary in prover entry points — **use the production names at
   the audited tip, not this list**: at G1 `8de355a3` that is
   `stealth_prove_pool_ring` (stealth_prover.cpp:530, tx-level)
   / `prove_leg`, `stealth_prove_bp_plus_m1` (:1124) + `bp_prove_attempt`
   (:737), `stealth_prove_value_pin` (:1178), `stealth_sample_blind` (:505).
   There is **no production recovery prover** (only test helper
   `T_prove_recovery`, tests/tests/g1_stealth_tests.cpp:2569) and **no HKDF
   path** in the prover — record items 5/8 accordingly instead of hunting
   symbols that do not exist. Verify each condition is **public data only**
   (wire counts, public keys, asset IDs). No secret scalars in conditions.

3. **Secret-dependent memory access.** Audit every array/vector/table
   lookup: `table[idx]`, `ptr[idx]`, `vec[idx]`, `arr[idx]`. Verify
   `idx` is **public** (loop counter, public wire count). No secret
   scalar used as index.

4. **Single-use nonce enforcement.** `stealth_prove_pool_ring` /
   `prove_leg` draw nonces via `stealth_sample_blind`
   (stealth_prover.cpp:505, `RAND_bytes`-based at 8de355a3) — verify
   `RAND_bytes` called per call, no global nonce cache.
   `stealth_prove_value_pin` generates separate nonces — verify
   independence (distinct label if an HKDF path exists at the tip).

5. **Domain separation labels.** Every `HKDF-Expand` call uses unique
   label: `"bts-stealth-prover/ring-nonce/v1"`,
   `"bts-stealth-prover/pin-nonce/v1"`,
   `"bts-stealth-prover/pseudo-blind/v1"`,
   `"bts-stealth-prover/recovery-b/v1"`,
   `"bts-stealth-prover/recovery-v/v1"`. Verify no label reuse.

### RNG & Key Derivation

6. **RNG source & health.** `RAND_bytes` called directly (or
   `getrandom()`/`arc4random()` on Linux/BSD). Verify `RAND_status()`
   or `RAND_poll()` checked before first use; fail hard if entropy <
   256 bits. No `rand()`, `/dev/urandom` raw reads, homebrew PRNG.

7. **HKDF construction.** `HKDF-Extract(salt, IKM)` →
   `HKDF-Expand(PRK, label, L)` using `SHA-256` or `SHA-512`. Salt is
   public (asset ID + operation context) or empty. Verify `L` matches
   output length (32B for scalars).

8. **Forward secrecy.** Compromise of long-term wallet key ≠ compromise
   of past `k` or `b_P`. Ephemeral by design — verify no long-term key
   mixed into nonce derivation.

### Zeroization & Memory Safety

9. **Explicit zeroize.** Every secret buffer zeroized via
   `OPENSSL_cleanse(ptr, len)` or `sodium_memzero(ptr, len)`. Grep for
   `memset` — **must find zero hits** on secret buffers.

10. **Compiler barrier.** After `OPENSSL_cleanse`, verify
    `atomic_signal_fence(memory_order_seq_cst)` or `asm volatile(""
    ::: "memory")` to prevent dead-store elimination.

11. **No secret copies on heap.** Secrets in stack-allocated
    `std::array<uint8_t, 32>` or `secp256k1_scalar` with custom
    destructor. No `std::vector<uint8_t>`, `std::string`, `fc::sha256`
    holding secrets (heap fragmentation leaves copies).

12. **Stack allocation preferred.** If heap: `mlock()` + `mprotect
    (PROT_NONE)` after use. Verify via `objdump -d` that secret
    buffers are on stack or `mlock`ed.

13. **Error-path zeroization.** Every function returning secret
    buffers uses RAII wrapper (`secret_scalar`, `secret_point`,
    `secret_buffer`) with destructor that zeroizes. No `return` without
    cleanup. No exception throws without cleanup.

### Constant-Time Discipline

14. **No variable-time crypto primitives.** `secp256k1` /
    `libsodium` / OpenSSL 3.x (with constant-time patches) used for:
    scalar mul, point add, point decompression, `mod_inv`. No
    hand-rolled modular math.

15. **Constant-time compare for tags/keys.** `CRYPTO_memcmp` /
    `sodium_memcmp` for recovery proof tag, nullifier verification,
    range proof verification. No `==`, `memcmp`, `std::equal`.

16. **No timing side-channels in hot path.** Verify via `ctgrind`
    (Valgrind) or `dudect` on the production entry points at the tip
    (`stealth_prove_pool_ring`/`prove_leg`, `stealth_prove_bp_plus_m1`
    + `bp_prove_attempt`, `stealth_prove_value_pin`, `stealth_sample_blind`).
    p-value > 0.05 for all secret-dependent operations.

## Output

```json
{"skill":"wallet-prover-crypto-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","prover_tip":"<commit>",
 "secret_handling":{"leaks":0,"secret_branches":0,"secret_indices":0,
  "single_use_nonce":true,"domain_separation":true},
 "rng_derivation":{"rng_source":"RAND_bytes","health_check":true,
  "hkdf_labels_unique":true,"forward_secrecy":true},
 "zeroization":{"explicit_cleanse":true,"compiler_barrier":true,
  "no_heap_copies":true,"stack_or_mlock":true,"error_paths":true},
 "constant_time":{"ct_libs_only":true,"ct_compare":true,
  "ctgrind_clean":true,"dudect_p_value":0.05},
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],
 "not_covered":["hardware side-channels (Spectre, power/EM)",
  "compile-time CT guarantees (Vale, Jasmin)",
  "entropy source audit (HWRNG, TPM)"]}
```

Severity: `Critical` = secret leak, secret-dependent branch/index, missing zeroization, RNG failure, or domain separation violation. `Important` = compiler barrier missing, error-path cleanup gap, `ctgrind`/`dudect` not run. `Minor`/`Note` = doc deltas.

## Rationalizations (RED baseline 2026-10-12)

| Excuse | Reality |
|---|---|
| "Prover is off-chain, not consensus" | Prover bugs = user fund loss. Only way honest users construct valid txs. |
| "Tests pass" | Tests don't check secret leaks, zeroization, or timing side-channels. |
| "Zeroization is in the code" | `memset` optimized away; `OPENSSL_cleanse` + barrier required. |
| "Constant-time is hard, good enough is fine" | Schnorr nonce reuse = private key recovery. Branch on secret = timing leak. |

## Red Flags — STOP and fix

- Any `memset` on secret buffer (must be `OPENSSL_cleanse`).
- Any `if (secret_scalar)` or `table[secret_scalar]`.
- Any `RAND_bytes` without `RAND_status()` check.
- Any HKDF label reused across `k` / `b_P` / recovery scalars.
- Any `return`/`throw`/`FC_ASSERT` without secret zeroization.
- `ctgrind` or `dudect` not run on prover entry points.
- Prover compiled without ASan/MSan.

**All of these mean: the prover is NOT safe. NO_GO on G2 wallet.**