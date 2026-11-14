---
name: bsip4-n-asset-pools-audit
description: >
  Use when auditing the BSIP-4 n-asset liquidity pool work: object model,
  operation set, konstant invariant, aequitas fee, dual EMA/ATR governor,
  TVWAP accumulators, share accounting, royalty claims, bounded state.
  Vantage: integer math correctness and bounded consensus growth.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Skill: bsip4-n-asset-pools-audit

Audit BSIP-4 against its BSIP and design spec. Vantage: the pool's shared state
must price any pair of its own assets from one invariant, with no float and no
unbounded growth.

## Checklist (in order, every item)

1. **Op/object inventory.** Confirm 6 operations and 2 new object types
   (`reciprocity_pool_object`, `reciprocity_factory_object`), appended ids that
   do not collide with H (78–103), F (110–117), or G (104–109).
2. **Wire append-only.** New fields append last; market-issued assets refused as
   pool shares; creation that omits extensions produces the old bytes.
3. **Hardfork gating.** One flag gates all six operations; evaluators refuse at
   apply when the flag has not passed.
4. **Invariant math.** Konstant via Padé log-sum-exp; aequitas quadratic spread;
   no floats, no libm; uint128 with overflow checks; 256-bit conversion where
   needed; division/rounding direction declared and favours the pool.
5. **Share accounting.** Deposit mints shares equal to konstant growth; genesis
   balances set on first capitalization; full-exit re-deposit re-initializes;
   withdraw burns shares and pays pro-rata balances. First-depositor / empty-pool
   attack checked.
6. **Fee and royalty conservation.** Per-token fee accumulators; founder/promoter
   claim routes (committee account / token issuer); royalty bounds 0–500 bps;
   conservation identity holds exactly.
7. **Governor.** Dual EMA/ATR with fractional remainders; periods 500/100;
   targets 50 bps / 10; max 100. Verify no runaway or negative state.
8. **Pool oracle.** TVWAP accumulators for 3 windows (15m/1h/24h); zero-data
   window reverts; inside one block accumulators move zero.
9. **Bounded state.** 2–8 assets, 8 balances, 8 genesis entries, 13 fixed oracle
   fields; pool count ceiling and per-account caps filed and enforced; open
   growth surface documented.
10. **Legacy coexistence.** Legacy two-asset constant-product pools untouched;
    no migration, no repricing.
11. **API surface.** Read-only query methods appended; names and counts verified.
12. **Test vectors.** Invariant homogeneity, fee bounds (≤3 units taken
    entirely), split underflow pays each recipient ≥1, reciprocity refusal on
    negative settlement, slippage floor, full exit resets accumulators but keeps
    once-initialized flag, weight GCD normalization to 100, oracle windows,
    sandwich net loss, approximation-error bound, flash no movement, wire
    pinning.

## Output

Numbered PASS/FAIL/QUALIFIED list with `file:line` evidence. Every FAIL gets
the smallest concrete fix. Ambiguity is a finding.

## Machine-Readable Output (required)

```json
{"skill":"bsip4-n-asset-pools-audit","auditor_model":"<exact model string — mandatory>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"P4-01","severity":"Critical|Important|Minor|Note","check":"op-inventory|wire|gating|invariant-math|share-accounting|fee-royalty|governor|pool-oracle|bounded-state|legacy-coexistence|api|test-vectors","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```