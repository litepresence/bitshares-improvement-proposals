# Skill: complexity-audit

Algorithm efficiency audit. Every consensus path, sweep loop, per-block operation, and
admission control path carries an algorithmic cost. Bitwise-inefficient choices, nested loops
over candidate sets, quadratic joins, unpruned growth vectors, and O(n²) metadata scans all
steal headroom from honest participants. The question is not just "does it work?" but "does it
work within budget at maximum legitimate load?" — and "is there a simpler algorithm that does
the same thing with fewer operations?"

## Checklist (every item, in order)

1. **Operation inventory.** Every consensus-critical operation and its per-invocation cost:
   ticket mint/redeem, voucher issue/settle, chain join/revoke, settle-finalize, dispute-open,
   challenge-open, grant-vote, slash-distribute. Each gets: algorithmic complexity in terms of
   input parameters (e.g., O(n log n) where n = live_tickets), constant factors, hidden
   constants (sort vs scan vs binary search), and whether the same result can be computed
   incrementally instead of recomputed.

2. **State bloat vectors.** Every unbounded or super-linear structure: ticket_vouchers count,
   dispute registry size, challenge registry size, exit queue length, paused ticket count.
   Are they capped? Is there a prune/sweep? Does growth scale linearly with honest usage or
   with adversarial input? A structure that grows linearly with usage is acceptable if
   capped; one that grows quadratically is a finding.

3. **Per-block aggregation cost.** Every maintenance sweep, every per-block aggregation,
   every rollup over the full state: total cost per block as a function of state size,
   number of participants, and ticket count. Does the block-processing budget scale
   linearly? Are there sweeps that scale quadratically with number of tickets or voters?

4. **Algorithmic simplicity audit.** Every non-trivial computation: is there a simpler
   algorithm that produces the same result with fewer operations? Sorting vs linear scan?
   Heap vs full sort? Single-pass vs multiple passes? Incremental update vs full
   recomputation? The simplest algorithm that satisfies the security property is the right
   one — complexity without justification is a finding.

5. **Adversarial input analysis.** Worst-case input for each evaluator: an adversary who
   controls which ticket to fund, which dispute to open, which challenge to issue. What is
   the worst-case cost to honest nodes? Is there a gas/cost limit? Is there a denial-of-service
   vector via fee-cheap operations that force O(n²) validation?

6. **Symmetry of work.** Does the cost to create a position match the cost to resolve it?
   Does opening a dispute cost the challenger as much as it costs defenders to respond?
   Does exiting cost as much as entering? Asymmetry where one side pays O(1) and the other
   pays O(n) is a finding.

7. **Fee/economic alignment.** Are computation costs priced? Does a costly computation cost
   the same as a cheap one? Is there a gas metering model? Do op fees reflect worst-case
   compute cost? Mismatch between economic cost and computational cost is a finding.

## Output

Table of each consensus operation with: operation name, algorithmic complexity (best/worst
case), input parameter, constant factor notes, invariant maintained, simpler alternative
(if exists), adversarial worst case, fee pricing (match/mismatch), verdict.

## Machine-Readable Output (required)

```json
{"skill":"complexity-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"CA-01","severity":"Critical|Important|Minor|Note","check":"algorithmic-complexity|state-bloat|per-block-cost|simpler-algorithm|adversarial-input|asymmetry|fee-pricing","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["evaluators with no complexity analysis"]}
```

Severity: `Critical` = algorithmic complexity that enables denial-of-service or state exhaustion at honest usage levels. `Important` = algorithmic complexity that enables denial-of-service at scale, or missing complexity analysis. `Minor`/`Note` = suboptimal but not exploitable algorithms, missing complexity documentation.