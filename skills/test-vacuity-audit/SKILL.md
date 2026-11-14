# Skill: test-vacuity-audit

Use when checking whether green suites actually prove what the BSIPs claim. Vantage: hostile mutation — for each claimed defense, would any test fail if the defense were deleted?

## Checklist (in order, every item)

1. **Claim inventory.** List every defense the BSIP asserts (rate-limit hold, delay-then-consume, freeze halts margin, drift yields no price, legacy byte-identical, bond-gated mint/debt, fail-closed freeze on missed deadline, self-dealing ban, cooldown/concurrency caps, ring shape + decoy policy, BP+/conservation reject, double-spend reject, supply-reconcile halt, version sunset). Anything claimed without a test = FAIL (untested claim).
2. **Negative-case presence.** For each defense, name the test that feeds the attack input (bid>ask publish, 6% jump, early consumption, drifted mapping, colluding majority, tampered proof byte, duplicate nullifier, over-cap subscriber, pre-fork smuggled op, skewed ring, thin pool). Missing attack input = FAIL.
3. **Removal reasoning.** For the five load-bearing defenses (drift fail-closed, fail-closed arbitration, supply reconciliation, **band-partition conservation, quantile-rmax guard**), reason by inspection: which exact assertion breaks if the enforcement line is deleted? "Suite stays green" = Critical vacuity. Round-17: the only enabled rmax refusal vector was refused by an earlier gate — the rmax `FC_ASSERT` is mutation-dead until a true vector exists (VAC-R17-10).
4. **Coincident vector.** The both-legs-breach test vector (BSIP-1 acceptance gate) must exist as a test, not a paragraph. Absent = Important.
5. **H2b vacuity follow-up.** The negative-case vacuity flag is in **round-2** `build/audit-round-2/v.json` as **V-08** (not round-1); cap-3 headroom was flagged in the H2b report. Confirm closed or re-file with the exact thin case.
6. **Budget honesty.** WARN (not hard gate) with both ring ends benchmarked after the ring benchmark repair; no silent `exit 0` that a reader mistakes for fit. **Do not cite `REPAIR-03` here** — that ID is a cross-round collision: round-8 `REPAIR-03` is `MG-03` (merged maintenance order, still open), not the G1 ring-16 benchmark.
7. **Evidence-artefact vacuity (added round-16; this is the item that matters most).** A claim can be "verified" by something that is not a test at all: a rig, a census, an overlay pin table, a ledger line, a playbook row. For each such artefact ask **what does it measure, and does that quantity go to the right value when the defect is present?**
   - Round-16 exemplar: PRV-R11-01's sampler fix was signed off on a rig reporting `window_hit_rate ≈ 0.062` (~1/16, i.e. chance). The rig's attacker hunted the unique *doubled* window — which, after the fix, can only ever be a decoy. So the metric reads **0 where the attack succeeds** and 1/16 only in the one shape family where the real attack also gives 1/16. The rig was only ever run at 16+ distinct windows. The fix was declared verified by a measurement that could not distinguish fixed from defective.
   - Same round: the overlay pin census reported 174/174 green while holding **zero rows** for `stealth_pool_builder.cpp` — the file the repair lives in — and the activation checklist cited that green census as the closed gap. A census proves only what it has rows for.
   - Same round: `wallet_sampler_matches_consensus`, the test that mirrors the consensus sampler, **asserted the leaking structure** (`kv.first != realW`, `perW.size()==4`). A test that pins a defect is worse than no test: it makes the repair look like a regression.
   - Procedure: for each verification artefact, (a) state the quantity it measures, (b) state the defect, (c) give the value of that quantity **when the defect is present**, (d) confirm the artefact was exercised at that value. If (d) is no, the verification is vacuous regardless of how green it is.
8. **Skip-gate honesty.** Every skipped/disabled test needs its justification re-derived, not inherited: a skip justified by "unreachable under the $20k ceiling" is false for any path whose reachability does not depend on face size. Round-16: both H2 PILOT-NARROWED skips were justified on face size, and one of them hid the only reserve-bearing wind-down accounting — exactly where the chain-halt Critical lived.
9. **Name vs. scope.** A skill (or a census, or a rig) that names a *file* or a *metric* is making a claim about *scope*. Check the scope: does the artefact's coverage include the file the most recent repair changed? Round-16's census had a STEMS entry (path resolution) for the sampler file and still zero rows — path resolution is not coverage.
10. **Recovery/reopen endpoint honesty.** For every recovery/reopen test, identify the FIRST hazardous transaction after recovery (the next writer, sweep, or claim) and assert it executes in-test. An endpoint assertion that holds only because the hazardous next writer was omitted is vacuity: round-17's two `h2_09` recovery tests stop one transaction short of the post-reopen writer that breaks consensus (VAC-R17-11).

## Output (JSON, required)

```json
{"skill":"test-vacuity-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"all three worktrees","tip":"45a4666e+efcfd0ce+8de355a3",
"findings":[{"id":"V-01","severity":"Critical|Important|Minor|Note","check":"claim-inventory","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix (usually: add the attack-input test)"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["..."],"next_critical":"<the next Critical on this axis>"}
```

Put substantive content in `findings[]`, **not** in `conditions[]`. Round-16's
pilot-coverage seat filed eight findings — including its chain-halt Critical — into
`conditions[]`; the JSON validated and the aggregate tool counted **zero** findings
for that seat.

A green suite that cannot go red is a decoration. Name the test or file the gap.
