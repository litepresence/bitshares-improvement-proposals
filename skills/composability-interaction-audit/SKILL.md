# Skill: composability-interaction-audit

Systems fail at interfaces. Audit every place two designed components — or the design and
the legacy chain — touch, transact, or assume something about each other.

## Checklist (every item, in order)

1. **Interface inventory.** List every touchpoint: new↔new (vault leg × oracle leg × issuance
   × stealth), new↔legacy (bitUSD pools as tripwire inputs, BTS as bond/fee asset, HONEST
   pilot ops, pool-issued hubs as routing, legacy margin engine sharing BTS collateral),
   new↔off-chain (wallets, explorers, auditors, CEX listings).
2. **Assumption ledger.** At each interface, write what side A assumes about side B (liveness,
   honesty, precision, timing, value). Then break each assumption one at a time: stale feed
   into coverage math, depegged pool into tripwire, halted legacy market into shared BTS
   price, blind wallet into transparent challenge bond, delisted asset into wind-down exit.
3. **Shared-resource contention.** Block space, committee attention, worker funding, BTS
   collateral, pool liquidity — model two consumers spiking at once (both legs breaching
   while a blind halt fires; challenge spam during a wind-down vote). Who starves, and does
   starvation fail closed?
4. **Emergent incentives.** New strategies the combination enables that no component has
   alone: routing hybrid redemptions through pool-issued hubs to dodge caps; using gated-window
   conversions to feed offset-pool yield loops; staking FXR while operating a vault (self-
   insurance that isn't); arbitraging timelocked owner changes. Price each new strategy's
   extractable value; neutralize or bound the profitable ones.
5. **Legacy contamination paths.** Every way legacy failure enters the new system (shared BTS
   collateral crash, shared pool price confusion, shared committee capture, shared brand
   contagion) and every way new-system failure leaks out (wind-down BTS dumps hitting legacy
   margin calls, tripwire halts freezing shared markets, stigma repricing bitUSD). "Untouched"
   is a code property; contagion is a market property — audit the market half.
6. **Version skew.** H1/H2/G1/G2 live simultaneously in different states: enumerate the
   cross-version interactions (H1 pair beside H2 pair sharing an oracle; pre-turnstile and
   post-turnstile blind outputs interoperating) and confirm each is specified, not assumed.

## Output

Per-interface verdicts with broken-assumption consequences, contention outcomes, emergent
strategies with EV and neutralization, contamination paths both directions, version-skew
matrix. Interfaces are where the next critical lives — assume it until proven otherwise.

## Machine-Readable Output (required)

```json
{"skill":"composability-interaction-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"CI-01","severity":"Critical|Important|Minor|Note","check":"interface|assumption-break|contention|emergent-incentive|contamination|version-skew","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["interfaces not traced"]}
```

Severity: `Critical` = an interface that fails open, or an emergent
strategy with unbounded extractable value. `Important` = a broken
assumption that starves or misprices without failing closed. `Minor`/`Note`
= documentation and version-skew deltas. Pair every joint verdict with
`stealth-composition-audit` for crypto primitives and cite both.
