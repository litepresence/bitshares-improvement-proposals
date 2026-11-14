# Skill: mechanism-interaction-matrix-audit

Use when a round must audit the highest-risk interaction between mechanisms that share
consensus state (counters, caps, budgets, fee pipelines, admission sets, aggregate
objects), including interactions that cross branch boundaries (H1/H2/G1). Recurring
seat: re-derive the roster and the peak intersection every round; the worked example
below is one instance, not the target. Distinct from `cross-leg-integration-audit`
(merge-time op/field collisions on merged code) and from lane seats (single-mechanism),
this seat audits reachable states of mechanisms firing together.

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (in order, every item)

1. **Roster.** Derive candidate mechanisms from three sources, recording which source
   nominated each: (a) constructs changed since the last full sweep (repair waves,
   feature tips); (b) money-touching paths (fees, issuance and burn, subsidy grants,
   pool and pot arithmetic); (c) open activation gates
   (`build/ACTIVATION-CHECKLIST.md`). Per mechanism record: name, anchor `file:line`,
   state read/written, invariant defended, owning branch.
2. **Select the intersection.** Score mechanism pairs by shared surface: same object
   fields, same counters, same caps or ceilings, same fee pipeline position, same
   admission/sampler input set. Record the scoring table. Audit the highest-scoring
   pairs; list the rest for the dispatcher. If the brief names a specific intersection,
   audit it AND the two next-highest pairs.
3. **Build the matrix.** Rows and columns are the roster. Every cell states: shared
   surface; the coupled invariant(s); classification: independent / ordered / coupled /
   conflicting; and the potentiality, that is the reachable state when both mechanisms
   fire, naming the outcome (who loses what, which counter saturates, which gate flips),
   not a bug list. A cell left unclassified is a FAIL of the seat.
4. **Audit coupled and conflicting cells.** Per cell write an adversarial vector: actor,
   operation sequence, and the invariant expected to break. Trace the code path across
   ALL involved branches; a cell verdict from one branch alone is invalid (the
   `d0a9c2c5` precedent: the G2 pool-swap router left `interval_routed` at 0, so Rule
   R's burn denominator read 100% and halted every further subsidy; neither mechanism's
   own tests saw the other's default). Falsify by reading; run a targeted test only if
   the brief allows (never a full gate from a seat). Verdict per cell: PASS / FAIL /
   QUALIFIED / NOT_COVERED.
5. **NOT-EXAMINED discipline.** The matrix file names every cell not audited and why
   (out of round scope, mechanism code absent at this tip, blocked on evidence). Silent
   gaps are a FAIL; a matrix without NOT-EXAMINED accounting is returned.
6. **Output.** Matrix to `build/audit-round-N/mechanism-interaction-matrix.md` (roster,
   scoring table, full matrix, per-cell verdicts and vectors). Seat JSON (template
   below) to `build/audit-round-N/mechanism-interaction-matrix-audit.json`. Findings use
   the standard severity scale; each FAIL names the cell, the vector, file:line on every
   involved branch, and the smallest concrete fix.

## Worked example (historical; re-derive)

H2 fee/band mechanics (`h2_issue_bands_spend`, senior burn, offset-pool diversion) × G1
stealth economics (op109 decoy sampler `R17B_MIN_DISTINCT_WINDOWS`, Rule R
grant/route/pot, joint fee cap `stealth_joint_fee_within_cap`). Cell types observed:

- Coupled, opposite-sign: pool-swap routing feeds the parity pot (`interval_granted` /
  `interval_routed`) and Rule R's burn-ratio alarm consumes them. G2 shipped with
  `interval_routed` never written, driving burn to 100% and halting grants (fixed
  `d0a9c2c5`). No single-mechanism suite could see it.
- Conflicting, open: the joint fee cap spans H2 diversion and stealth parity; op109's
  100% parity routing is a third parity contributor the cap does not count (MG-04
  family in `ACTIVATION-CHECKLIST.md`).
- Boundary: H2's local fee gate and G1's cap constants live in different headers; the
  merged-tree pair assert is the open joint-cap merge item.

## Red flags (stop and redo)

- A cell verdict from one branch alone.
- Interaction findings with no vector: "these could interfere" without the reachable
  state named is a not-filed note.
- Auditing only the named intersection when the roster exposes a higher-scoring pair
  (record why, or audit it).
- Matrix cells left blank or silently dropped.
- Spending the budget on single-mechanism depth; lane seats own that.
- Building or committing anything: this seat is read-only (AGENTS.md §11).

## Output

```json
{"skill":"mechanism-interaction-matrix-audit","auditor_model":"<exact model string; mandatory>","branch":"cross-branch: feature/track-b-h2 + feature/track-g1-stealth","tip":"<h2 sha> + <g1 sha>",
"findings":[{"id":"MIM-01","severity":"Critical|Important|Minor|Note","check":"<matrix cell>","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line on each branch; vector; falsification","fix":"<smallest concrete fix>"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO",
"conditions":["roster N mechanisms; cells N (coupled C, conflicting K, not_examined N); matrix build/audit-round-N/mechanism-interaction-matrix.md"]}
```

`branch` and `tip` list every branch and tip the matrix spans; one seat covers the
intersection, not one branch. The JSON follows `tooling/audit_director_schema.json`;
`auditor_model` is required or the dispatcher rejects the seat at aggregation time.
