# Skill: bsip-readiness-review

Review a BSIP draft the way a hostile-but-fair community will: shareholders protect capital,
developers protect the codebase, the cartel protects itself. Survive all three.

## Checklist (every item, in order)

1. **Template compliance.** Every required section present and non-thin: header fields with
   real-or-explicitly-unassigned values, Abstract, Motivation (with measured numbers, dated),
   Rational, Specifications (mechanisms specified, constants marked proposal), Discussion
   (contested threads named, rejected alternatives named, audit must-close list),
   Summary for Shareholders (costs, who pays, what changes for existing holders: ideally
   nothing), Copyright, See Also.
2. **Shareholder objections.** Steelman the three hardest no-votes: the large legacy holder
   ("repricing risk"), the gateway operator ("you're regulating me out"), the apathetic BTS
   holder ("why fund this"). Each must be answered in the text, not in the forum thread.
3. **Numbers with provenance.** Every figure carries source + date + method (node query vs
   aggregator; block height for chain data). Stale or unsourced numbers are findings.
4. **Scope discipline.** One BSIP, one system, one vote. Coupled legs share a BSIP only with
   phase gates and acceptance criteria; independent work (stealth vs economics) stays split.
   Flag anything that can slip independently bundled into a shared fate.
5. **No implied perfection.** Every accepted risk stated in-text (not in an audit annex nobody
   reads): what the design does NOT resist, what happens in the gap periods, what breaks first
   in a correlated crash. The cartel's game is implied perfection; deny it to them.
6. **Worker/funding sanity.** Costs scoped per phase, no phase funds the next before its gate
   passes. No standing rule without a review date: every ongoing policy (fee parity, reward
   rates, caps) carries its review cadence in-text.
7. **Tip-consistency check (MUST-FIX on mismatch).** Every commit id, tip, case
    count, or log filename cited anywhere in the BSIP must agree three ways:
    (a) the header pin, (b) every body-cited commit/log, (c) `git -C <worktree>
    rev-parse HEAD` + `git log --oneline -5` at audit time. Any divergence
    (header says one tip, body cites another, HEAD has moved comment-only or
    otherwise) is a MUST-FIX: either re-pin the document to the verified tip
    or label the stale pin SUPERSEDED-as-current (history only, with log
    filename). Banned: unqualified "N cases entered/passed" with no log
    filename — every case-count claim must name its `build/logs/*.log` plus
    the tip it ran at (e.g. "802 Entering/802 Leaving exit 0,
    `r9-verify-h2-full.log` at `9e73ad8b`"), so a re-run at a moved HEAD is
    distinguishable from stale evidence.

## Output

Punchlist with MUST-FIX/SHOULD-FIX/NIT, exact locations, and the correction. Then a
three-sentence community summary: what it does, what it costs, what could still go wrong.

## Machine-Readable Output (required)

```json
{"skill":"bsip-readiness-review","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"BSIP-01","severity":"Critical|Important|Minor|Note","check":"MUST-FIX|SHOULD-FIX|NIT","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"doc:line + code:line","fix":"exact replacement wording"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["sections not reviewed"]}
```

Severity: `Critical` = the BSIP describes behavior the chain does not
implement (use `protocol-claim-truthfulness-audit` to confirm the code
side first). `Important` = MUST-FIX blocking community review.
`Minor`/`Note` = SHOULD-FIX / NIT wording deltas.
