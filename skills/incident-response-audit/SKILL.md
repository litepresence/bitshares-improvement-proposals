# Skill: incident-response-audit

Assume the exploit is already happening. The design is judged by the first hour, not the
whitepaper.

## Checklist (every item, in order)

1. **Detection.** For each attack class in the threat model: what signal fires, where is it
   computed (consensus object vs off-chain monitor), who watches it, and what is the
   detection latency in blocks? An attack with no tripwire is a finding.
2. **First-hour playbook.** Halt/freeze/pause inventory: exact op or key, who holds it, quorum
   and window, what it stops and what it does NOT stop (e.g. halting mint while redemptions
   drain). Write the play-by-play for the top three scenarios; gaps become checklist items.
3. **Blast-radius bounds.** Per incident: maximum extractable value before each tripwire fires
   (caps × delay windows × throughput). If any incident is unbounded before human action,
   name it and bound it or file accepted risk.
4. **Recovery path.** Post-halt: how does the system resume (conditions, who decides, what
   proves safety), how are victims made whole (which fund, what haircut rule, what order),
   and what happens to positions opened in bad faith during the incident (revert vs absorb —
   pick one per incident class; "decide later" is a finding).
5. **Communication.** Who announces, through what channel, with what pre-drafted content?
   Silence favors the attacker. Require named channels and draft statements before launch.
6. **Drills.** Which scenarios get testnet rehearsals with the actual key-holders before
   mainnet activation? Undrilled halt keys are decorative.
7. **Post-mortem commitment.** What data is preserved for analysis (and for how long), who
   writes the report, and what governance change it can trigger. No post-mortem rule means
   the same incident twice.
8. **Runbook currency.** Every playbook row must be re-derived at the audited tip: a playbook
   that asserts behavior the code does not have (e.g. an auto-unhalt that does not exist,
   a seizure that has not shipped) is a finding in itself, not a doc note.
9. **Containment blast radius.** For each containment lever (`freeze_blind`, `pair_freeze`,
   oracle unfreeze, vault holds), measure the REAL blast radius across every pre-existing
   class and op the lever touches — not the intended scope. `pair_freeze` on live debt halts
   mints AND exits; `freeze_blind` halts legacy op39/40/41 chain-wide. Intended-scope claims
   are tested, never assumed.
10. **Abort drill.** A stopped-chain / ordered-abort / rollback procedure must exist in every
   playbook with a per-step authority column (who can execute each step today). Missing =
   finding; a drill row that asserts non-existent behavior is Important-or-worse severity.

## Output

Per-scenario playbooks (detect → halt → bound → recover → communicate), unbounded-extraction
findings, missing drills, and the pre-launch incident kit list.

## Machine-Readable Output (required)

```json
{"skill":"incident-response-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"IR-01","severity":"Critical|Important|Minor|Note","check":"scenario|unbounded-extraction|drill|kit","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["scenarios with no playbook"]}
```

Severity: `Critical` = an extraction path with no bound and no halt.
`Important` = a scenario whose playbook names no halt, or a drill that has
never been run. `Minor`/`Note` = kit and doc deltas.
