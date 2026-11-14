# Skill: parameter-sensitivity-audit

Constants are guesses until stress-tested. Vary every number in the design and find which ones
are load-bearing, which are cosmetic, and which are cliffs.

## Checklist (every item, in order)

1. **Inventory every constant.** Bonds, floors, caps, ratios, windows, delays, fees, splits,
   thresholds, quorum sizes. Each gets: current proposal, unit, and basis (measured, ported
   from paper X, or guessed — label honestly).
2. **One-way sensitivity.** Vary each ±50% (and 10x/0.1x for fees and caps) holding others
   fixed. For each variation: does any guarantee invert (coverage <1, CoC<PfC, halt untriggered,
   spam profitable)? Mark CLIFF (inverts), SLOPE (degrades gracefully), or FLAT (cosmetic).
3. **Joint stress.** Vary correlated sets together the way reality does (collateral −70% AND
   exit demand ×3 AND spam +10x; fee volume −90% AND BTS −50%). Single-parameter analysis
   lies about crashes; report the joint cases.
4. **Basis check.** For every ported constant (Liquity 110%/150%, Tellor doubling, Maker 1h,
   THORChain 2/3): does the original context match ours (block time, finality, liquidity,
   volatility)? A port without re-derivation is a finding — re-derive or re-label as guess.
5. **Tuning authority.** Who may change each constant post-launch, by what vote, with what
   timelock, within what bounds? An untunable cliff is tech debt; an unconstrained tunable
   is governance capture surface. Each constant needs its mutability box stated.
6. **Kill bands.** For each CLIFF constant, define the monitoring band: green/amber/red values
   with the action each band triggers. Constants without bands are unmonitored single points
   of failure.
7. **Cross-parameter validity.** For every pair/triple where one constant can be legally set
   relative to another (owner-set vs committee-set, e.g. `guard_delay` vs the pending-queue
   capacity), enumerate the invalid regions and the guard that catches each — or file the
   region where the combination silently breaks a guarantee (round-17 PS-R17-01: delay ==
   16×bucket evicts every ripe sample; no guard existed).

## Output

Table: constant | proposal | basis | sensitivity class | joint-stress verdict | mutability |
kill bands. Plus the ranked list of the three constants most likely to kill the system if
guessed wrong.

## Machine-Readable Output (required)

```json
{"skill":"parameter-sensitivity-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"PS-01","severity":"Critical|Important|Minor|Note","check":"inventory|basis|joint-stress|mutability|kill-band|cross-parameter","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line or constant","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["constants with no basis"]}
```

Severity: `Critical` = a kill-band where the system silently fails open at
plausible values. `Important` = joint stress that only survives one-leg-at-
a-time testing. `Minor`/`Note` = basis and doc deltas. An unmeasured
constant is `NOT_COVERED`, never "safe".
