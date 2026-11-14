# Skill: consensus-determinism-audit

Audit proposed graphene consensus changes for determinism, arithmetic safety, and hardfork
cleanliness — the class of bug that forks the chain rather than merely losing money.

## Checklist (every item, in order)

1. **Operation inventory.** List every new op, new field on existing ops/ext structs, and every
   evaluator behavior change. Anything missing from the list is unaudited by definition.
2. **Arithmetic walk.** For each new computation: types (int64/uint128/256?), division and
   rounding direction, overflow behavior at max supply / max precision / adversarial inputs.
   Graphene consensus forbids floats; flag any float, any `double`, any libm call. Confirm
   select-vs-compute: does aggregation only ever *select* observed values?
3. **Determinism walk.** No wall-clock reads (only head-block time), no map iteration order
   dependence (`flat_map` ordered — confirm; `std::unordered_*` is a FAIL), no randomness,
   no locale dependence. Multi-index iteration order must be fully keyed.
4. **State growth.** Every new object/index: who can create entries, at what fee, with what cap?
   Unbounded per-user state (history, submissions, challenges) needs explicit ring/cap + fee.
5. **Hardfork gating.** Each new op and each appended extension field: rejected pre-fork in
   evaluators AND `proposal_operation_hardfork_visitor`. Extension fields appended, never
   inserted. Cite the `.hf` constant and both enforcement points.
6. **Cross-evaluator interactions.** New ops vs market/margin/settlement/vesting/HTLC/pool
   evaluators: enumerate shared objects touched and ordering assumptions (e.g. median refresh
   timing vs margin matching in the same block).
7. **Serialization.** `FC_REFLECT` coverage for every new struct; external serializer
   declarations; wire-format append-only after activation.

## Output

Numbered PASS/FAIL/QUALIFIED list with `file:line` evidence. Every FAIL gets the smallest
concrete fix. Ambiguity is a finding, not an excuse — mark it explicitly.

## Machine-Readable Output (required)

The prose list above is the human deliverable. THIS block is the machine
contract: `audit-director` concatenates findings across auditors and sorts
them, so a finding that exists only in prose never reaches the repair TODO
list. Emit both.

```json
{"skill":"consensus-determinism-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"CD-01","severity":"Critical|Important|Minor|Note","check":"op-inventory|arithmetic|determinism|state-growth|hardfork-gating|cross-evaluator|serialization","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["checklist items not reached"]}
```

Severity: `Critical` = chain-forking or consensus-divergence defect (float,
`unordered_*`, wall-clock, missing pre-fork rejection, inserted extension
field). `Important` = arithmetic or serialization gap reachable under
plausible inputs. `Minor`/`Note` = hardening and doc deltas.
