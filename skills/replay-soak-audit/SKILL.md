---
name: replay-soak-audit
description: >
  Use when designing, executing, or auditing the PRG-02 replay and PRG-03
  multi-hour soak activation gates. Triggers: preparing replay evidence,
  writing soak scenarios, validating soak outcomes, or closing the
  replay/soak checklist items in ACTIVATION-CHECKLIST.md.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Replay-Soak Audit

## Overview

PRG-02 (replay) and PRG-03 (soak) are the only activation gates that
produce **fresh empirical evidence** of consensus correctness on live
state. Running the tool is not the audit. The audit is: did the replay
cover the right range, did the soak fire the right failure modes, and
are the invariants machine-checked at every step?

## When to Use

- Preparing the evidence package for `ACTIVATION-CHECKLIST.md` PRG-02/PRG-03
- Writing the soak scenario harness
- Auditing a completed replay/soak run before gate closure
- Reviewing whether a "replay passed" claim actually proves consensus safety

Do NOT use for: unit/integration test design (`test-vacuity-audit`),
fuzzing methodology (separate concern), or benchmark design
(`production-readiness-gate`).

## Rules (no exceptions)

1. **Full chain or documented scope.** A replay that does not cover
   genesis-to-head must have a **written scope document** naming: start
   block, end block, which hardforks are included, which consensus
   rules are NOT re-verified, and why the skip is acceptable. "Time
   pressure" is not a reason — it is a finding.

2. **Soak = scripted adversarial scenarios, not uptime.** A soak that
   merely runs the node for N hours with no injected failures is a
   decoration. Every soak run must have a **scenario manifest** listing
   each failure mode to be triggered, the trigger transaction, the
   expected consensus outcome, and the invariant that proves it.

3. **Invariants are machine-checked, not human-read.** Every soak
   scenario and every replay block window must run the invariant checks
   (supply conservation, blind supply, vault solvency, oracle bonds,
   tranche accounting, pool invariant, nullifier monotonicity, HF
   gating) as **automated assertions** in the harness, not post-hoc
   inspection.

4. **Three binaries, three evidence trees.** H1, H2, and G1 tip
   binaries are **separate consensus rulesets**. A replay/soak on one
   does not cover another. Evidence must be produced per binary and
   cross-referenced.

5. **Evidence package is the deliverable, not the log.** A `chain_test`
   log or `witness_node` stdout does not close the gate. The gate
   closes on: `summary.json`, `invariant-checks.json`, `scenario-log.json`,
   `GATE-CLOSE.md` signed by the human partner.

6. **Pilot funds ≠ reduced rigor.** The ≤$20k pilot cap limits blast
   radius, not audit scope. A soak on pilot funds must fire the same
   scenarios as a full soak; it just uses smaller amounts.

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (every item, in order)

### Replay (PRG-02)

1. **Scope document.** Written before replay starts: `build/evidence/replay-scope-<tip>.md` with:
   - Mainnet block log source (URL, height range, sha256 of log file)
   - Start block (must be pre-all-hardforks or justify why not)
   - End block (head at replay start or justify stop height)
   - Tip binary commit hash + build checksum
   - Which hardforks fall in range (ORACLE, H2_BONDS, H2_GATEWAY, H2_PRICE, STEALTH)
   - Explicit list of consensus rules NOT re-verified (with justification)
2. **Execution.** `witness_node --replay-blockchain --blocks-dir <log> --stop-at-block <end>` per binary. Capture: exit code, duration, final head block/hash/timestamp, any crash/assert.
3. **State root verification.** Final state root must match a reference node (separate clean sync to same height) — or document why reference is unavailable.
4. **Invariant checks.** Automated checks every 100 blocks (configurable): supply conservation, blind supply, vault solvency, oracle bonds, tranche accounting, pool invariant, nullifier monotonicity, HF gating. Output: `invariant-checks.json`.
5. **Targeted test suite.** Post-replay `chain_test --run_test='h2_*,oracle_*,stealth_*'` on the replayed state. Output captured.
6. **Object count diff.** Account, asset, pool, order, HTLC, vault, oracle, stealth output counts vs reference node. Output: `object-counts.diff`.

### Soak (PRG-03)

7. **Scenario manifest.** Written before soak: `build/evidence/soak-manifest-<tip>.md` listing each scenario:
   - Name (e.g., `freeze_blind`, `coverage_halt`, `tripwire_ladder`, `challenge_expiry`, `coincident_atomicity`)
   - Trigger: exact transaction construction (which op, fields, values)
   - Expected consensus outcome (accept/reject, state transitions)
   - Invariant that proves the outcome (from invariant set above)
   - Injection cadence (blocks between triggers)
8. **Harness.** Scripted transaction submission (not manual). Logs: tx ID, block height, before/after state diff, invariant check result. Output: `scenario-log.json`.
9. **Resource monitoring.** RAM/CPU/disk per hour. Alert on growth >10%/hr. Output: `resource-usage.csv`.
10. **Continuous invariants.** Same invariant checks as replay, running every block during soak. Any FAIL = soak FAIL, immediate halt and investigation.
11. **Pilot funds tracking (if mainnet).** Deposits, withdrawals, fees, P&L per asset. Output: `pilot-funds-tracking.json`.

### Gate Closure

12. **Evidence package.** Per binary: `build/evidence/replay-<tip>-<date>/` + `build/evidence/soak-<tip>-<date>/` with all JSON artifacts.
13. **GATE-CLOSE.md** signed by human partner: "PRG-02/PRG-03 satisfied for <tip> — evidence reviewed, all invariants green, no undocumented skips."
14. **Checklist update.** `ACTIVATION-CHECKLIST.md` PRG-02/PRG-03 items marked `DONE` with evidence paths.

## Output

```json
{"skill":"replay-soak-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","tip":"<commit>","phase":"replay|soak|closure",
 "replay":{"scope_doc":"build/evidence/replay-scope-<tip>.md","binaries":["H1","H2","G1"],
  "invariant_checks":"build/evidence/replay-<tip>/invariant-checks.json",
  "state_root_match":true,"object_counts":"build/evidence/replay-<tip>/object-counts.diff",
  "targeted_suite":"build/evidence/replay-<tip>/chain_test.log"},
 "soak":{"manifest":"build/evidence/soak-manifest-<tip>.md",
  "scenario_log":"build/evidence/soak-<tip>/scenario-log.json",
  "invariants":"build/evidence/soak-<tip>/invariant-checks.json",
  "resource_usage":"build/evidence/soak-<tip>/resource-usage.csv",
  "pilot_funds":"build/evidence/soak-<tip>/pilot-funds-tracking.json"},
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],
 "not_covered":["hardforks/consensus-rules not in replay scope","scenarios not in soak manifest"]}
```

Severity: `Critical` = replay scope undocumented, soak has no manifest, invariants not machine-checked, or gate closed without `GATE-CLOSE.md`. `Important` = scenario missing from manifest, invariant check gap, resource monitoring absent. `Minor`/`Note` = doc deltas.

## Rationalizations (from RED baseline 2026-10-12)

| Excuse | Reality |
|---|---|
| "The tool exists, just run it" | The tool produces a log. The audit produces evidence. A log without invariant checks proves uptime, not consensus correctness. |
| "100k blocks is enough, full chain takes 72h" | A scope document saying "100k blocks, HFs X/Y/Z covered, rules A/B/C skipped because..." is the audit. No document = NO_GO. |
| "Testnet soak with scripted txs is equivalent to mainnet" | Pilot cap limits blast radius, not scenario scope. The manifest must include the same failure modes; only the amounts differ. |
| "Node stayed up for 4 hours, soak passed" | Uptime ≠ consensus validation. A soak without `freeze_blind`, `coverage_halt`, `tripwire_ladder`, `challenge_expiry`, `coincident_atomicity` scenarios is a decoration. |
| "We'll check invariants after" | Invariants must be checked **during** the run. A post-hoc check cannot catch a transient violation that self-healed. |
| "Three binaries, but H1/H2/G1 are similar" | They have DIFFERENT op sets, DIFFERENT hardforks, DIFFERENT consensus rules. One replay does not cover three. |

## Red Flags — STOP and document

- Running replay without a written scope document.
- Running soak without a written scenario manifest.
- Any invariant check that is not automated (human reads logs and says "looks good").
- Closing PRG-02 or PRG-03 without `GATE-CLOSE.md` signed by the human partner.
- A soak that runs but never triggers `freeze_blind`, `coverage_halt`, `tripwire_ladder`, `challenge_expiry`, or `coincident_atomicity`.
- Evidence package missing any of: `summary.json`, `invariant-checks.json`, `scenario-log.json`, `GATE-CLOSE.md`.

**All of these mean: the gate is NOT closed. Document the gap and re-run.**