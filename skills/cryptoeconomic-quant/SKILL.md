# Skill: cryptoeconomic-quant

Replace hand-waving about incentives with numbers. Every bond, reward, fee, and cap in the
design must survive contact with a spreadsheet.

## Checklist (every item, in order)

1. **Inventory the money.** List every bonded amount, reward flow, fee route, forfeit
   destination, and coverage threshold. Anything without a number AND a unit is a finding.
2. **Cost-of-corruption vs profit-from-corruption.** For each attack worth attempting, compute
   CoC (bonds forfeited, bribes required, time locked) against PfC (debt stealable, redemptions
   extractable, short profits) using live chain numbers where they exist (supplies, depths,
   market caps — query the node, don't use aggregators). CoC > PfC must hold with margin, and
   the margin must survive a stated drawdown (e.g. −70% collateral) — recompute post-crash.
3. **Reward sustainability.** Liveness APY at expected participation: fee inflow per interval ÷
   bonded base. If APY doesn't beat opportunity cost, the set stays empty — say so and name
   the subsidy required and its source.
4. **Spam/griefing ledger.** Cost to file one malicious challenge, dispute round, or vote
   vs cost to process it (voter attention, block space, committee time). Attacker budget for
   sustained spam vs defender capacity. If spam is cheaper than defense, require: minimum
   bond floors, cooldowns, concurrency caps, fast-reject with forfeit.
5. **Self-dealing audit.** For every winner-takes-forfeit flow, ask: can one authority be both
   sides? If yes without a ban, compute the cartel's expected value and kill the incentive
   (finder-fee-only winnings, burn routing).
6. **Correlated-crash recompute.** Re-run (2) with all BTS-denominated values cut 70% and all
   exit doors crowded. Name exactly which guarantees invert and what rule fires (halt, haircut,
   wind-down) — "transparency" is not a rule.
7. **Staging economics.** H1-without-H2: what is attackable, for how long, at what cost?
   Put the threat model and the expiry in consensus or publish it as accepted risk.

## Output

A ledger: each flow with numbers, each attack with CoC/PfC and verdict, each gap with the
smallest numeric fix (floor, cap, rate, window). No adjectives where arithmetic belongs.

## Machine-Readable Output (required)

Numbers are the deliverable here, so they must be machine-readable or they
cannot be re-derived. Mark every number `measured | derived | speculative`.

```json
{"skill":"cryptoeconomic-quant","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"CQ-01","severity":"Critical|Important|Minor|Note","check":"flow|attack|gap","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"formula + inputs","fix":"smallest numeric fix (floor, cap, rate, window)","basis":"measured|derived|speculative"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["flows not quantified"]}
```

Severity: `Critical` = an attack whose CoC is finite and reachable under
production parameters, or a subsidy pot that is arithmetically empty
(`basis: derived`, with the arithmetic shown). `Important` = a mechanism
that is inert or defeasible under governance control. `Minor`/`Note` =
assumptions and doc deltas. **A subsidy or fee route that nets to zero is a
finding, not a passing result** — say the number.
