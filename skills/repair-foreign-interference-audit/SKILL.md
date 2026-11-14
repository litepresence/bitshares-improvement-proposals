# Skill: repair-foreign-interference-audit

Use when a repair wave has landed on any Track B branch (`feature/track-b-h1`,
`feature/track-b-h2`, `feature/track-g1-stealth`) and the untouched surface around those
repairs must be re-swept for interference. Recurring seat: re-dispatch after every repair
wave, per branch, alongside the lane re-audits, never instead of them. Lane seats audit
what the repair touched; this seat audits what the repair reached.

**Violating the letter of these rules is violating the spirit of the rules.**

## Scope

- IN SCOPE: everything NOT modified by the repair delta whose correctness depends on
  constructs the delta changed. Consumers, writers, serialization readers, invariant
  aggregates, and docs/pins/fixtures that assert the old behavior.
- OUT OF SCOPE: the repaired lanes themselves. Lane re-audit seats own those. A defect
  inside a repaired lane gets one filed line and a deferral note, not a deep-dive.
- Re-derive every pin (base, tip, commit ids, counts) from `git log` / `git diff` in the
  worktree at dispatch time. Pins rot: AGENTS.md §11 records the CLSAG overlay census
  moving 174 → 180 → 188, and round-16's stale `db_market.cpp` pin fed a false Critical
  candidate. A pin quoted in a skill, brief, or doc is a claim, not evidence.

## Checklist (in order, every item)

1. **Derive the delta.** In the target worktree: `git log --oneline <base>..<tip>` and
   `git diff --stat <base>..<tip>`, then read every repair commit's diff. Record base,
   tip, commit ids, file count, and insert/delete totals in the report. Inventory the
   changed constructs: functions or evaluators added, moved, or re-scoped; constants and
   object fields changed; invariants re-partitioned (spend orders, caps, gates,
   denominators, admission sets); tests and fixtures added; comments that carry pins.
2. **Build the untouched-surface manifest.** For EVERY changed construct, find what
   depends on it outside the delta and class each item:
   - STALE-ASSUMPTION: a doc, skill, spec, test, fixture, or comment asserting the OLD
     behavior (old constant, old line pin, old function name, old reachability claim).
   - BEHAVIORAL-COUPLING: a caller, consumer, or writer outside the changed files whose
     correctness depends on the changed semantics (state-machine transitions, gate
     outcomes, denominators, return-value shape).
   - INVARIANT-INTERFERENCE: a consensus aggregate spanning changed and unchanged code
     (band partitions, caps, ratios, fee splits, work budgets, pot denominators,
     admission sets). Check both sides of every aggregate.
   Each item records `file:line | class | the changed construct it depends on | why it
   could break`. The manifest is the deliverable; an intuition-ordered sweep is not this
   seat.
3. **Sweep the manifest.** Per item, read the path end to end on every affected branch
   and attempt falsification: name the failure state, then try to reach it or refute it
   by reading. Run a targeted test only if the brief allows (never a full gate from a
   seat). Verdict per item: PASS / FAIL / QUALIFIED / NOT_COVERED. A claimed defect
   without a stated falsification attempt is not filed; a passed item without one is not
   PASS.
4. **Doc-foreign sweep.** Repairs on one branch can stale pins in sibling worktrees, in
   `/workspace/skills/`, in specs/plans, and in the CLSAG/BP+ overlay. Grep each changed
   constant, function, and field name across the other worktrees and `/workspace` docs;
   file hits as STALE-ASSUMPTION. Read sibling code only for its references to the
   changed construct; do not re-audit sibling code.
5. **Coverage ledger (mandatory; a seat without it is returned).** Table: manifest item →
   swept yes/no → verdict → evidence. Report enumerated / swept / NOT_EXAMINED counts,
   with a per-item reason for every NOT_EXAMINED. The dispatcher reads the ledger before
   the findings and will not accept a "no issues" report with silent gaps.
6. **Output.** One seat JSON (template below) to
   `build/audit-round-N/repair-foreign-interference-audit.json`, branch-suffixed
   (`-h1`/`-h2`/`-g1`) when dispatched per branch. Ledger to
   `build/audit-round-N/repair-foreign-interference-ledger[-branch].md`. Findings use the
   standard severity scale; every FAIL carries file:line, the falsification attempt, and
   the smallest concrete fix. Record manifest counts and the ledger path in `conditions`.

## Failure shapes this seat exists to catch (historical; re-derive per round)

- ORC-02 (round-17 fix on H1 `46afd123`, refined at round-18 `45a4666e`): the first fix
  changed guard-bucket arithmetic (bucket ceiling) and only the in-lane re-audit caught
  the refinement a round later. A foreign-surface seat sweeps the untouched consumers of
  the changed arithmetic in the same round.
- The r17/r18 repair wave itself (H2 `09ed7242`: +1518/-67 across 20 files): it landed
  with lane re-audits only, so the untouched surface was never systematically swept.
  This seat is that sweep.
- Pin rot: round-16's stale `db_market.cpp` pin and round-17's stale overlay count (174
  vs 188) both fed seats wrong numbers. Both are STALE-ASSUMPTION class.

## Red flags (stop and redo)

- Sweeping the repaired lanes instead of the surface.
- A manifest without sweeps, or findings without a falsification attempt.
- "No issues found" with no coverage ledger, or a ledger with silent gaps.
- Trusting a pin (this file included) instead of re-deriving it.
- Filing a sibling-branch code defect instead of the stale reference to it.
- Building or committing anything: this seat is read-only, and a post-dispatch commit
  from an auditor is a failed dispatch (AGENTS.md §11).

## Output

```json
{"skill":"repair-foreign-interference-audit","auditor_model":"<exact model string; mandatory, AGENTS.md §11>","branch":"feature/track-b-h2","tip":"<sha>",
"findings":[{"id":"RFI-01","severity":"Critical|Important|Minor|Note","check":"<construct swept>","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line; falsification: <what was tried>","fix":"<smallest concrete fix>"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO",
"conditions":["delta <base>..<tip>; manifest N items (swept N, not_examined N); ledger build/audit-round-N/repair-foreign-interference-ledger.md"]}
```

The JSON follows `tooling/audit_director_schema.json`; `auditor_model` is required or the
dispatcher rejects the seat at aggregation time (AGENTS.md §11).
