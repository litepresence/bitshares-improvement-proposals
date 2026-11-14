# Skill: build-feasibility-audit

Ask whether the plan can actually be built, by whom, in what order, with what proof. A spec
no team can execute is a wish; find out now.

## Checklist (every item, in order)

1. **Work inventory.** Break the proposal into buildable units (new ops, evaluator changes,
   API/wallet changes, tests). Each unit gets: files touched, new consensus code vs
   API-only, estimated complexity (S/M/L with reasoning, never bare T-shirt sizes without it).
2. **Dependency graph.** Order units by dependency (what needs what live first). Name the
   critical path and anything falsely sequenced (parallelizable work serialized, or
   dependent work scheduled in parallel).
3. **Test burden.** For each consensus unit: unit tests, tag/wire-format stability tests,
   replay-against-mainnet requirement, adversarial test vectors (the spec's own attack
   section must have a matching test). Untestable-in-practice claims are findings.
4. **Upgrade logistics.** Witness adoption mechanics for each hardfork (lead time, version
   signaling, what happens to non-upgraded nodes), testnet soak duration, rollback story per
   phase. A phase with no rollback is a finding.
5. **Supply chain.** Every new cryptographic or consensus dependency (libraries, submodules,
   forks): maturity, audit history, build reproducibility, what happens if it rots. The `fc`
   submodule state in this tree is the cautionary example — check it.
6. **Team realism.** Skills required per phase (consensus C++, applied crypto, wallet/UI, QA)
   vs skills plausibly available. Name the hire-or-contract gap explicitly; a phase nobody on
   the team can review is unfundable.
7. **Stall analysis.** If phase N slips a year, what operates in the meantime and is it safe
   and honest to operate it? Every gap needs a threat model or a shutdown rule.

## Output

Build order with dependencies, per-unit complexity + test requirements, upgrade/rollback per
phase, supply-chain verdicts, staffing gaps, stall threat models. No "just implement it."

## Machine-Readable Output (required)

```json
{"skill":"build-feasibility-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"BF-01","severity":"Critical|Important|Minor|Note","check":"dependency|complexity|upgrade-rollback|supply-chain|staffing|stall","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line or command output","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["units not sized"]}
```

Severity: `Critical` = an unbuildable dependency order, or a consensus unit
whose rollback forks the chain. `Important` = a unit that cannot be tested
at its integration boundary. `Minor`/`Note` = staffing and doc deltas.
