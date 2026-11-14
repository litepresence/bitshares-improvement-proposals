# Skill: governance-capture-audit

Assume the attacker votes. DPOS security analysis that ignores vote-buying is fiction; this
skill makes the politics explicit and prices it.

## Checklist (every item, in order)

1. **Map every vote-gated action.** Committee votes, witness elections, challenge resolutions,
   freeze/unfreeze keys, fee changes, ESM triggers. For each: quorum, threshold, window, and
   what happens on abstention or missed deadline (abstention must default to freeze, never to
   proceed — verify).
2. **Build the capture tree.** For each gate: cost to buy the deciding votes (stake to acquire
   at current depth + slippage, bribes at historical worker rates), cheapest path (buy vs
   bribe vs apathy-exploit), and what the captured gate yields — thefts it unlocks **and
   destructive-capture yields: seizure, forced wind-down, or termination count as funds loss**
   (round-17 GOV-R17-01 was found there). Compare yield to cost — a profitable capture is a
   scheduled event, not a risk. **Composition rule: a denial-only gate plus a consequential
   executor operation composes into a new severity** — price the composed pair, not each half
   (GOV-R17-02: sybil-denied quorum converts freeze-by-default into seizure-by-default with
   GOV-R17-01).
3. **Minority protections.** For every majority gate, name the minority trigger that bypasses
   it (ESM-style freeze, pair-level settlement, validator veto). A gate with no minority
   override is a finding.
4. **Cartel history check.** Map each gate onto the chain's actual capture episodes
   (fake-feed marginer protection, etc.): would this design's rules have changed the outcome,
   or would the same votes produce the same result through the new machinery? If the latter,
   say so and add the rule that would have mattered.
5. **Arbitration liveness.** Who votes, in what window, with what quorum — and what executes
   automatically when they don't. Every human step needs a timeout default; every timeout
   default must fail closed. List them in a table or flag the missing ones.
6. **Fork-threat credibility.** If "threat of fork" is cited as a backstop, price it: who
   coordinates, what stake follows, what breaks. A threat nobody can execute is decoration.

## Additional gates (BSIP-3, BSIP-8)

7. **Gateway and attestor capture (BSIP-3).** Committee declares/deactivates
   gateways, binds each to an oracle, and maintains the attestor allowlist. Price
   the capture of that gate and whether a minority can override; a captured
   committee can bind a colluding attestor set and release escrow to a friend.
8. **Manager capture (BSIP-8).** Manager controls trading tap within bounds but
   cannot lower the minimum tank level or shorten withdrawal delays; emergency
   tap requires supermajority. Price capture of the manager key and the
   emergency authority.

## Output

Per-gate verdicts (SAFE/CAPTURABLE-AT-$X/UNBOUNDED), missing minority overrides, timeout
defaults table, and the cartel-history replay result. Adversarial, not agreeable.

## Machine-Readable Output (required)

```json
{"skill":"governance-capture-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"GOV-01","severity":"Critical|Important|Minor|Note","check":"gate-capture|minority-override|timeout-default|cartel-replay","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["gates not exercised"]}
```

Severity (aligned with the program round discipline; the round brief's rule
overrides this line when they conflict — state both readings): `Critical` =
funds/consensus/invariant loss reachable by an unprivileged or lightly-privileged
actor with no minority override (a seat may price the capture in votes as
evidence, but reachability governs). `Important` = capture cost is open-ended,
or a default timeout favors one coalition. `Minor`/`Note` = process and doc
deltas. A gate you could not price is `NOT_COVERED` — it is not SAFE.
