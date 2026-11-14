---
name: bsip6-stableswap-audit
description: >
  Use when auditing the BSIP-6 StableSwap curve work: typed extension
  (curve_type=1, amplification A), Newton D/y solvers in 128-bit integers,
  rounding that favours the pool, imbalanced-deposit fee, single-sided
  withdrawal, hardfork gating. Vantage: no fork from rounding differences.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Skill: bsip6-stableswap-audit

Audit BSIP-6 against its BSIP and design. Vantage: integer determinism and
rounding that favours the pool on every step.

## Checklist (in order, every item)

1. **Extension opt-in.** `curve_type = 1` with amplification A (positive) at
   creation; omission produces the old bytes. Constant-product pools
   (`curve_type = 0`) unaffected.
2. **Invariant.** Curve StableSwap equation, n = 2, with D as stored
   invariant. At perfect balance D = x + y.
3. **Solvers.** `compute_d` and `compute_new_y` by Newton iteration; 128-bit
   integers; never divide before multiplying; agreement across nodes for A up to
   uint64 max.
4. **Rounding.** Every rounding step favours the pool. Grinding: 200 fee-free
   A→B→A cycles leave the trader 200 units down and grow D from 2,000,000 to
   2,000,200.
5. **Imbalanced deposit.** Fee = swap fee × n/(4(n-1)) = half swap fee for n=2.
   Test: 300/100 into balanced 500/500 yields 1,994,277 share units, not
   4,000,000.
6. **Single-sided withdrawal.** Allowed and fee'd on stable pools; refused on
   constant-product pools with the stated message.
7. **Amplification bounds.** A has no protocol upper bound; document the
   economic danger of absurd A. A fixed at creation; re-tuning needs a new
   operation.
8. **Hardfork gating.** One flag gates extension, curve branch, imbalanced fee,
   and single-sided withdrawal; refuse at apply when not passed.
9. **Wire pinning.** Pool fields and extensions append and hold stable across
   builds with and without the flag.
10. **Test vectors.** Convergence, rounding grind, imbalanced deposit, single-
    sided withdrawal refusal/allowance, wire pinning.

## Output

Numbered PASS/FAIL/QUALIFIED list with `file:line` evidence. Every FAIL gets
the smallest concrete fix. Ambiguity is a finding.

## Machine-Readable Output (required)

```json
{"skill":"bsip6-stableswap-audit","auditor_model":"<exact model string — mandatory>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"SS-01","severity":"Critical|Important|Minor|Note","check":"extension|invariant|solvers|rounding|imbalanced-deposit|single-sided-withdrawal|amplification|gating|wire-pinning|test-vectors","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```