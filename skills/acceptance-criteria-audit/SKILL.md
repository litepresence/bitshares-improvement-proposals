# Skill: acceptance-criteria-audit

Gates nobody can verify are decorations. Every acceptance criterion, kill switch, review, and
"proven in production" claim must be measurable by a third party with chain data alone.

## Checklist (every item, in order)

1. **Gate inventory.** List every gate: phase transitions, coverage proofs, soak periods,
   drill requirements, review cadences, funding kill criteria, turnstile completion, sunset
   lapses. Quote each gate's exact current wording.
2. **Measurability test.** For each gate: what query, against what object, returns pass/fail?
   Name the API/objects/fields. A gate answerable only by committee judgment is a finding —
   convert to data or label it a vote, never both at once.
3. **Threshold justification.** Why 90 days and not 30 or 180? Why this coverage level, this
   drill count, this report cadence? Inherited numbers need re-derivation; round numbers need
   suspicion. Unjustified thresholds get a derivation demand, not a pass.
4. **Who measures, who disputes.** The measurer and the measured must differ. If the team
   building the phase also certifies its gate, require independent verification (auditor,
   challenger bounty, public dashboard query). A self-certified gate is a finding.
5. **Failure semantics.** Gate missed: exactly what happens, automatically or by whose hand,
   within how many blocks? "Funding stops" needs the funding tap identified; "winds down"
   needs the trigger op named; "does not launch" needs the launch transaction specified as
   blocked-by-what-rule. Vague consequences are findings.
6. **Gaming resistance.** How would a team fake passing each gate (wash coverage with
   friendly bonds, drill with cooperating keys, report favorable windows)? For each game,
   name the anti-game (independent bonds, adversarial drill red-team, full-window reporting).
   Un-gameable-by-design beats monitored; monitored beats trusted.

## Output

Per-gate verdict (VERIFIABLE/NEEDS-WORK/DECORATIVE) with the query that decides it, the
independent verifier, the failure consequence with trigger, and the anti-game. Gates are the
project's immune system — audit them like it.

## Machine-Readable Output (required)

```json
{"skill":"acceptance-criteria-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"AC-01","severity":"Critical|Important|Minor|Note","check":"verifiability|independent-verifier|failure-consequence|anti-game","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["gates with no deciding query"]}
```

Severity: `Critical` = a DECORATIVE gate — one whose failure cannot be
observed, so it cannot gate anything. `Important` = NEEDS-WORK: decidable
only by the party that would fail it. `Minor`/`Note` = wording and doc
deltas.
