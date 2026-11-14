---
name: pilot-coverage-audit
description: >
  Use when designing, executing, or auditing the PRG-06/ISS-03 pilot
  coverage gate: proving the H2 bonded vault + bonded oracle + tranched
  issuance system maintains all invariants for 90 days at HONEST-scale
  (≤$20k face). Triggers: writing the pilot metrics harness, reviewing
  90-day attestation logs, or closing the pilot gate in
  ACTIVATION-CHECKLIST.md.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Pilot Coverage Audit

## Overview

Objective 3: "Prove coverage on a HONEST-scale pilot (≤$20k) for 90 days
before any USD issuance." This is not "run it and watch." It is: define
the quantitative invariants, instrument daily measurement, automate
violation detection, and produce a 90-day attestation log that a human
partner can sign.

The pilot gate is **only** about proving the core H2 mechanics work:
bonded vault + bonded oracle + tranched issuance. E0/Rule R, specialist
audit, replay/soak, governance, slash-to-seizure are **separate gates**
(EE3-G01/G02/G03, PRG-04, PRG-02/03, GOV-02). Conflating them is the
pressure trap.

## When to Use

- Designing the pilot metrics harness (daily on-chain queries)
- Reviewing the 90-day attestation log before gate closure
- Auditing whether a "pilot proved coverage" claim is quantitatively
  substantiated
- Closing `ACTIVATION-CHECKLIST.md` PRG-06/ISS-03

Do NOT use for: E0/Rule R measurement (`cryptoeconomic-quant` +
`supply-invariant-audit`), specialist CLSAG/BP+ audit
(`external-audit-engagement`), replay/soak (`replay-soak-audit`),
governance artifacts (`governance-capture-audit`), or slash-to-seizure
(`hardfork-upgrade-safety-audit`).

## Rules (no exceptions)

1. **Coverage is five invariants, not a vibe.** The pilot proves:
   (a) Oracle liveness — bonded producers post feeds, zero slashes.
   (b) Vault solvency — collateralization ≥ the vault's `secure_ratio_bps`
   (genesis proposal 20000 = 200%) at every block, and the post-mint rmin
   gate `GRAPHENE_H2_ISSUE_RMIN_BPS = 13000` (130%) holds.
   (c) Offset pool primacy — pool absorbs all shortfalls; junior
   dilution only at rmax cap in single batch.
   (c) Tranche integrity — senior (FX) peg ±1% to bonded oracle; junior
   (FXR) absorbs residual.
   (d) Consensus liveness — zero halts; supply/7-bucket/fee-cap
   invariants hold.

2. **Daily attestation is mandatory, not optional.** A 90-day pilot
   without a **daily automated attestation log** (one entry per day, 90
   entries) is a decoration. The log must be machine-generated, not
   human-written.

3. **Pilot cap is a consensus gate, not a guideline.** $20,000 face is
   enforced at bind-time (vault register + issuance mint). The audit
   must verify the cap holds at every block, not just at day 90.

4. **Legacy coexistence is a simultaneous proof.** The pilot pair is
   HONEST-scale (user's corporate smartcoin). Legacy assets
   (`bit*`, `HONEST.*`, gateway UIAs) must credit 100% to
   `accumulated_fees` throughout the pilot. Any fee diversion = pilot
   FAIL.

5. **Rule R / E0 are separate gates.** The pilot proves *core coverage*
   (vault+oracle+issuance). Rule R moves ~0 BTS at production fees
   (F-02 OPEN, cryptoeconomic NO_GO). E0 measurement requires
   SUP-R-10 (rolling cumulative counters, not yet built). Do not
   conflate pilot success with Rule R activation.

6. **Evidence package = attestation log + certificates, not a report.**
   The gate closes on: 90 daily attestations (machine-generated),
   invariant violation report (zero violations), pilot cap compliance
   proof, legacy coexistence proof, offset pool solvency certificate,
   oracle liveness certificate, human partner sign-off.

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (every item, in order)

### Pilot Design (pre-launch)

1. **Quantitative invariant definition.** Write `build/pilot/invariants.md`
   with the five invariants + code thresholds. Construct names first; line
   numbers must be re-derived at the tip being cited (raw line pins rot on
   every insert — re-pin before consuming this list):
   - rmin gate `GRAPHENE_H2_ISSUE_RMIN_BPS = 13000` (`issuance_ops.hpp:35`),
     enforced post-mint by `h2_issue_meets_rmin` (def `issuance.cpp:171`,
     invoked mint `:1061` / redeem `:1406` / exit `:1780` @ H2 `efcfd0ce`). Distinct from
     the per-vault `secure_ratio_bps` op field (`vault_ops.hpp:75`, default 0,
     must exceed 10000).
   - `rmax_cap` = `GRAPHENE_H2_WINDDOWN_RMAX_JUNIOR_X = 2`
     (`vault_lifecycle_ops.hpp:40`) — junior dilution ceiling
   - `pilot_cap` = $20,000 face, bind-time gate: `h2_issue_pilot_cap_face`
     (def `issuance.cpp:839`, called `:727` / `:1132` @ `efcfd0ce`) with
     `FC_ASSERT( … <= pilot_cap )` at `vault_evaluator.cpp:123-132` (bind);
     the mint leg refuses via the same helper
   - 7-bucket spend order: `GRAPHENE_H2_ISSUE_N_BANDS = 7`
     (`issuance_ops.hpp:83`), spent center-out by `h2_issue_bands_spend`
     (def `vault_lifecycle_evaluator.cpp:55`, decl `issuance_evaluator.hpp:259`;
     call sites `issuance.cpp:664` / `:1472-1473` / `:1836` @ `efcfd0ce`);
     `db_maint.cpp:1621-1636` (re-derive) is the H2 maintenance *sweep*
     order, not the band partition
   - Joint fee cap: `parity + h2 ≤ 7000 bps` — `stealth_joint_fee_within_cap`
     (`stealth_evaluator.cpp:1653` @ `8de355a3`), breach writes
     `alarms.joint_cap_breach` (`:1788`); the `:1330` def is
     `stealth_subsidy_output_count`, not the cap
   Cite construct + file:line for each, re-derived at the audited tip.

2. **Daily metrics harness.** Script `tooling/pilot_daily_metrics.py` that
   queries and emits JSON for all 8 metrics:
   - Vault collateralization ratio (per vault)
   - Bonded oracle producer bonds + slash events
   - Offset pool balance vs cumulative shortfall
   - Senior (FX) vs junior (FXR) supply
   - 7-bucket spend sequence compliance
   - Per-asset `parity_fee_share_opted_in` + fee routing
   - Pilot cap utilization (face value)
   - Oracle feed latency + dispute count
   Output: `build/pilot/attestations/day-<N>.json`

3. **Automated violation detection.** The harness must FAIL (exit non-zero)
   if ANY invariant is violated. No human triage — the script is the
   gate.

4. **Pilot cap enforcement verification.** Verify the bind-time gate:
   `vault_evaluator.cpp` (vault register — the pilot-cap `FC_ASSERT` block at
   `:123-132` @ `efcfd0ce`) and the issuance mint path
   (`h2_issue_pilot_cap_face`, `issuance.cpp:839`, called `:727`/`:1132`)
   refuse any
   operation that would push book face over $20k. There is no
   `vault_operation_evaluator.cpp` / `issuance_evaluator.cpp` pair, and no
   `issued_face` field — the liability mirror is `vault_object.outstanding`
   (`vault_object.hpp:57`). Cite the `FC_ASSERT` lines re-derived at the tip.

### Pilot Execution (90 days)

5. **Daily attestation log.** 90 consecutive days of `day-<N>.json` in
   `build/pilot/attestations/`. Each entry signed by pilot operator
   (GPG or SSH key). No gaps — a missing day = pilot FAIL.

6. **Invariant violation report.** Single document:
   `build/pilot/invariant-violations.md` listing zero violations, or
   documented false positives with root cause and fix. Any undocumented
   violation = pilot FAIL.

7. **Pilot cap compliance proof.** Off-chain verification that
   `vault_object.outstanding` (the liability mirror — there is no
   `issued_face` field; `vault_object.hpp:57`) summed across the pair stays
   within the $20k face cap at all blocks during pilot. Cite
   the consensus bind-time `FC_ASSERT` + off-chain query script.

8. **Legacy coexistence proof.** Fee routing audit showing `bit*`,
   `HONEST.*`, gateway UIAs receive 100% to `accumulated_fees` (no
   parity fee share opt-in). Cite the routing gate
   `parity_fee_share_opted_in` (`stealth_evaluator.cpp:98` @ `06dcd808` —
   the `:94-98` range lands in the doc comment above it) + `asset_ops.hpp:55`
   (optional flag absent by default).

9. **Offset pool solvency certificate.** Final offset pool balance ≥
   max cumulative shortfall observed during pilot. Cite
   `offset_pool_object {pair, balance, claims}` (reflect `issuance.cpp:1912`;
   object `issuance_object.hpp`, re-derive) and
   `reserve_object.uncovered_shortfall`. (`asset_object.offset_pool_balance` /
   `asset_object.cumulative_shortfall` do not exist — PC-R17-07.)

10. **Oracle liveness certificate.** All bonded producers posted feeds
    within SLA; zero slash events. Cite `bond_object.bond_amount` +
    `bond_object.slashed` (there is no slash history) and
    `oracle_object.submissions` / `oracle_object.current_value_time`.
    (`oracle_object.bond_amount` / `.slash_history` / `.last_feed_time` do
    not exist — PC-R17-07.)

### Gate Closure

11. **Human partner sign-off.** `GATE-CLOSE-PILOT.md` signed by the
    HONEST brand admin (owner of the pilot pair): "Pilot proved coverage
    for HONEST-scale pair — all invariants held for 90 days, zero
    violations, cap respected, legacy coexistence verified."

12. **Committee HF dates set.** Replace dummy dates (2030-01-01 etc.)
    with real dates in `hardfork.d/*.hf` files. BSIP-1/2 filed + worker
    vote artifacts collected.

## Output

```json
{"skill":"pilot-coverage-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","pilot_pair":"<asset_symbol>","start_date":"<ISO>","end_date":"<ISO>",
 "invariants":{"oracle_liveness":true,"vault_solvency":true,"offset_pool_primacy":true,
  "tranche_integrity":true,"consensus_liveness":true},
 "attestations":{"count":90,"path":"build/pilot/attestations/","gaps":[]},
 "violations":0,"pilot_cap_usd":20000,"legacy_coexistence":true,
 "offset_pool_solvent":true,"oracle_liveness":true,
 "human_signoff":"GATE-CLOSE-PILOT.md","gate":"GO|GO_WITH_CONDITIONS|NO_GO",
 "conditions":["..."],"not_covered":["E0/Rule R (separate gate)","specialist audit (separate gate)",
  "replay/soak (separate gate)","governance artifacts (separate gate)",
  "slash-to-seizure (separate gate)"]}
```

Severity: `Critical` = missing daily attestation, any invariant violation, pilot cap breach, legacy fee diversion, or gate closed without human sign-off. `Important` = metric gap in attestation, offset pool near exhaustion, oracle feed latency SLA breach. `Minor`/`Note` = doc deltas.

## Rationalizations (from RED baseline 2026-10-12)

| Excuse | Reality |
|---|---|
| "90 days is long; just run it and watch" | A pilot without daily automated attestation is a decoration. 90 entries or NO_GO. |
| "E0/Rule R must work for the pilot to count" | Pilot proves CORE COVERAGE (vault+oracle+issuance). Rule R is economically inert (F-02 OPEN). E0 needs SUP-R-10 (not built). Separate gates. |
| "Mainnet replay/soak first, then pilot" | PRG-02/03 are separate NO_GO gates (human partner owned). Pilot can run on testnet with scripted adversarial vectors. Conflating delays the 90-day clock. |
| "Specialist CLSAG/BP+ audit for op108/109" | Pilot uses H2+H1+G1 (ops 104-107). op108/109 are G2 pool swaps — not required for core coverage. Track as separate gate. |
| "Pilot cap $20k makes capture economically irrational (GOV-02)" | FALSE at the frozen tips (PC-R17-02): smallness of the FACE does not bound loss — the bonds + offset pool are the seizeable value (~$121,250 at-risk for a 5-vault/5-producer/$4k-seed pilot vs the $20k face; capture ≥ $837.50 + ~$795.63 freeze; ~16.8× on the $13,400 minimum bond). Size the pilot on defended value and publish the attack-cost table. |
| "90-day wait before any analysis" | Instrument for early signal (day 30/60 checks with margin). Blind waiting is not an audit. |

## Red Flags — STOP and document

- Pilot launch without `invariants.md` citing file:line thresholds.
- Any day missing from the 90-day attestation log.
- Any invariant violation not documented in `invariant-violations.md`.
- Pilot face exceeds $20k at any block (consensus should prevent; off-chain must verify).
- Legacy asset fee diversion detected (parity fee share opted in on legacy asset).
- Gate closed without `GATE-CLOSE-PILOT.md` signed by HONEST brand admin.
- Conflating pilot success with Rule R/E0/specialist/replay/soak/governance/slash-to-seizure gates.

**All of these mean: the pilot gate is NOT closed. Document the gap.**