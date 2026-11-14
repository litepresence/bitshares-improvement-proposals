# Skill: production-readiness-gate

Use when deciding whether a protocol-upgrade branch is ready to ship or activate, or when defining what "done" means for a consensus workstream. Vantage: ship gate — evidence over assertion.

## Checklist (all must hold; one NO-GO anywhere blocks the ship)

1. **Tip-green full binary.** The exact tip commit (not an ancestor) has a full
   `chain_test` run: exit 0 AND both required strings greppable in the log on disk —
   `rg -c 'No errors detected'` (must be >=1) and `rg -c 'Entering test case'` (the
   case count). Suite-only runs do not qualify. Do NOT require an assert total: it
   varies run to run. Note Boost's `--report_level=detailed` SUPPRESSES the
   'No errors detected' line, so a log captured with that override fails this gate
   even on green code. Re-run after every consensus-touching commit, including
   comment-only header edits (headers recompile dependents).
2. **BSIP-accurate.** The BSIP describes the tip, not a stale tree: op tags,
   object IDs, hardfork dates, test counts, and every "NOT built" fence must
   match `git log`/code. A BSIP claiming unbuilt behavior is a NO-GO.
   **Re-pin rule (added 2026-10-15 after round-9 drift): after ANY commit past
   the audited tip, tip+count+log pins are FAIL until re-measured at the new
   HEAD — cite the gate log containing both strings for the NEW tip. Round-9
   went stale this way (H2 `0e6ea947`/785 → `9e73ad8b`/802 → `6448347f` with no
   exact-tip log; G1 `06dcd808`/827 → `a2dd340d`/845 → `9d4b5d85`; BSIP 718→719
   typo). The accepted G1 citation form is `Running N` + `No errors detected`
   with zero `Entering test case` lines (EE-4 form) — record it as QUALIFIED,
   do not re-litigate.**
3. **Zero deferred code items.** Every MUST-FIX/Important audit finding is closed
   in code with a test, or narrowed out of the voted scope in BSIP text with a
   named follow-up phase. "Pre-activation gate" is a promise, not a close —
   count promises separately and require an owner + worker for each.
4. **Reviewed commits.** Every commit on the ship branch passed review (diff read,
   mechanisms spot-checked, test evidence verified on disk, never self-reports
   alone). Unreviewed commits do not ship.
5. **Replay + soak.** Mainnet replay green on the tip binary; private testnet soak
   recorded (attacks attempted, halts exercised, not just happy paths).
6. **Activation artifacts.** Hardfork dates set by governance (placeholders are
   not dates); fee schedule voted; Discussion thread open; worker scoped with
   gate-before-funding and kill criteria.
7. **External gates honestly listed.** Specialist audits, testnet-with-strangers,
   exchange/integrator sign-off, legal review — each named with owner and status.
   An unnamed external dependency is a NO-GO.

## Output

GO / NO_GO with the exact failing checklist item, its `file:line` or missing
artifact, and the smallest close. Partial credit is not given: a branch is
shippable or it is not.

**Vocabulary correction (2026-10-07).** The line above is the historical binary
template, and it cannot express this program's real two-axis state. The program
legitimately uses `GO_WITH_CONDITIONS` — code-complete with named, owned, unclosed
conditions — and today code is `GO_WITH_CONDITIONS` while activation is `NO_GO` on
human/external gates. Use `GO | GO_WITH_CONDITIONS | NO_GO`. Do **not** file a false
NO_GO on green code in order to fit a two-valued template; that is a worse failure
than an imprecise label.

## Citation rule (added 2026-10-07 after three hand-cited numbers failed to grep)

Every number, log filename, and status claim in this program must be **greppable in the
artifact it cites**. This is not stylistic: three separate claims in one session were
written by hand and were wrong —

1. an assert total cited against a log that did not contain it (it came from a
   `--report_level=short` run whose output was never saved);
2. a regenerated log that dropped `*** No errors detected` because
   `--report_level=detailed` suppresses it, leaving a green tip looking unproven;
3. a `python3` edit whose assertion failed while the shell continued, so the ledger
   committed a line claiming an edit that had never landed.

All three were caught only because a later step re-verified instead of trusting. Rules:

- **Cite the case count, never an assert total.** Assert totals vary run to run
  (observed 10957970 and 10960350 on two clean runs of the same tip) because some
  suites assert on timing and budget distributions.
- **Both strings, one log.** `rg -c 'No errors detected'` >= 1 AND
  `rg -c 'Entering test case'` = the case count, from the SAME named log.
- **Never regenerate a cited log with `--report_level=detailed`** — it drops the
  no-errors line.
- **If an edit's assertion fails, the edit did not happen.** Fix the edit and re-run it;
  do not let a later shell command record that it landed.
- **A recorded finding must be re-verified against current code before it is re-filed.**
  Findings in this program have gone stale behind the very skill that recorded them
  (see `protocol-claim-truthfulness-audit`'s burn-first entry and the refuted 5%-skim
  example, both corrected 2026-10-07).

## Common mistakes

- Counting a suite run as a full-binary run; trusting Green-then-touched-tip.
- BSIP text describing last week's tree ("wire order fixed" while code still
  violates it).
- Deferring load-bearing economics (caps, backstops) to "activation" with no
  worker or owner.
- Shipping a dirty tree (uncommitted deltas ride along unreviewed).

## Machine-Readable Output (required)

The two-axis GO/NO_GO above is the verdict. Emit it machine-readable so
`audit-director` and the activation checklist can consume it:

```json
{"skill":"production-readiness-gate","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "code_axis":"GO|GO_WITH_CONDITIONS|NO_GO","activation_axis":"GO|GO_WITH_CONDITIONS|NO_GO",
 "findings":[{"id":"PRG-01","severity":"Critical|Important|Minor|Note","check":"checklist-item","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line or artifact","fix":"smallest concrete close"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["items not checked"]}
```

Severity: `Critical` = an item whose failure ships money-losing or
chain-forking code. `Important` = must close before activation.
`Minor`/`Note` = cleanup. Evidence for a full-binary item must cite the
log by name with the greppable counts taken from that SAME log.
