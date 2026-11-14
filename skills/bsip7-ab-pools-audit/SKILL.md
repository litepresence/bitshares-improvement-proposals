---
name: bsip7-ab-pools-audit
description: >
  Use when auditing the BSIP-7 Asymmetric Bounded (AB) pool work: typed
  extension (curve_type=2, w, P_low, P_high), virtual offsets, closed-form
  Balancer math, single-sided withdrawal, rounding, range behaviour.
  Vantage: zero tail waste with exact bounds and pool-favouring rounding.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Skill: bsip7-ab-pools-audit

Audit BSIP-7 against its BSIP and design. Vantage: closed-form concentrated
liquidity with exact bounds and no unbounded state.

## Checklist (in order, every item)

1. **Extension opt-in.** `curve_type = 2` with AB parameters at creation;
   omission produces the old bytes. Constant-product and StableSwap pools
   unaffected.
2. **Curve.** Invariant `(x+xv)^w * (y+yv)^(1-w) = k`; price formula
   `p = (w/(1-w)) * (y+yv)/(x+xv)`. Inside [P_low, P_high] continuous;
   outside single-sided.
3. **Virtual offsets.** Derived from (w, P_low, P_high) so the curve hits the
   axes exactly at the bounds. Verify the derivation and exact bound hits.
4. **Closed-form math.** Swaps are Balancer math with (x+xv), (y+yv)
   substituted; no Newton iteration; no floats.
5. **Single-sided withdrawal.** Permitted on AB pools, charged half the swap
   fee; refused on constant-product pools.
6. **Rounding.** Every step favours the pool; grinding confirms.
7. **State bounds.** No new object type; scalar params only; pool count caps
   inherited from BSIP-4.
8. **Hardfork gating.** `HARDFORK_AB_POOL_PASSED` gates extension, curve
   branch, and single-sided withdrawal; fail-closed at apply.
9. **Wire pinning.** Pool fields and extensions append and hold stable across
   builds with and without the flag.
10. **Test vectors.** Range single-sided, rounding grind favours pool,
    single-sided withdrawal refusal/allowance, wire pinning.

## Output

Numbered PASS/FAIL/QUALIFIED list with `file:line` evidence. Every FAIL gets
the smallest concrete fix. Ambiguity is a finding.

## Machine-Readable Output (required)

```json
{"skill":"bsip7-ab-pools-audit","auditor_model":"<exact model string — mandatory>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"AB-01","severity":"Critical|Important|Minor|Note","check":"extension|curve|virtual-offsets|closed-form|single-sided-withdrawal|rounding|state-bounds|gating|wire-pinning|test-vectors","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```