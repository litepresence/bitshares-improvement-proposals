---
name: chain-data-retrievability-audit
description: >
  Use when auditing whether "protocol totals permanently public" (AGENTS.md
  §4.4) is actually true: can a third party recompute all protocol totals
  from chain data alone using only registered objects and get_objects?
  Triggers: SUP-R-10 (interval_granted/routed cleared every maintenance
  interval), supply-invariant-audit item 9, cryptoeconomic-quant F-02,
  E0 measurement gate (EE3-G01/G02/G03), or any claim that protocol
  totals are publicly auditable.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Chain Data Retrievability Audit

## Overview

The program's invariant: "blind positions/transfers permitted; protocol
totals permanently public" (AGENTS.md §4.4). This skill audits whether
a third party can **actually** recompute every protocol total from chain
data alone — using only registered objects and `get_objects` — without
trusting any off-chain indexer, explorer, or API.

**Critical gap (SUP-R-10) — STATUS: IMPLEMENTED `8dc0ff75`, verify don't assume.** `interval_granted` / `interval_routed` in
`stealth_status_object` WERE cleared every maintenance interval
(cleared in `stealth_parity_maintenance`, `stealth_evaluator.cpp:1741-1742` at
`9d4b5d85`, invoked via `perform_chain_maintenance` in `db_maint.cpp` — re-grep
the tip, line numbers drift) with no rolling counter,
so the per-asset subsidy series was lost each interval. Commit `8dc0ff75`
added `cumulative_granted` / `cumulative_routed` (accumulate-before-clear,
never cleared). An auditor must verify the counters exist at the audited tip
(`git grep -n cumulative_granted -- libraries/chain/stealth*`) and FAIL the
old "lost forever" claim if the counters are present. Only the aggregate
worst-case `alarms.burn_rate_pct` survived pre-fix.

## When to Use

- Auditing SUP-R-10 fix (rolling append-only per-asset counters)
- Verifying `supply-invariant-audit` item 9 (third-party recompute)
- Closing E0 measurement gate (EE3-G01/G02/G03 require per-asset flow series)
- Any claim that "protocol totals are publicly auditable"

Do NOT use for: supply invariant audit itself (`supply-invariant-audit`),
cryptoeconomic quantification (`cryptoeconomic-quant`), or governance
audit (`governance-capture-audit`).

## Rules (no exceptions)

1. **Registered objects are the source of truth.** All protocol totals
   live in `FC_REFLECT`ed implementation objects queryable via
   `get_objects`. No off-chain computation is authoritative.

2. **Singleton objects at known IDs.** `stealth_status_object` (2.20),
   `reward_pool_object`, `tripwire_state_object`, etc. have stable
   object IDs. A third party can query them directly.

3. **Per-pair objects discoverable.** `oracle_object` (1.23),
   `reserve_object` (1.37), `vault_object` (1.25), etc. exist per
   asset-pair. A third party must be able to enumerate them — either
   via `get_objects` with known ID pattern, or via an index iteration
   API.

4. **Historical series must survive maintenance.** Any quantity that is
   cleared by `perform_chain_maintenance` (once per interval, ~daily)
   is **not permanently public** unless it is also accumulated into a
   never-cleared rolling counter.

5. **E0 measurement requires per-asset flow history.** Rule R's
   entropy economics (spec §8) needs per-asset, per-interval
   `granted[a,i]` and `routed[a,i]` to compute `ΔD_a` trends. If these
   are cleared, E0 cannot be measured from chain data.

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (every item, in order)

### Current-State Retrievability (Point-in-Time Totals)

1. **H1 oracle objects.** Verify `oracle_object` (space 1.23) fields:
   `current_value`, `consumed_value`, `history[]`, `submissions[]`,
   `pending_queue[]`, `guard_hits` retrievable via `get_objects` and
   `get_oracles` / `list_oracles` / `get_oracle_by_name` API endpoints.

2. **H2 reserve & vault objects.** Verify `reserve_object` (1.37):
   `bts_balance`, `stable_supply`, `fxr_supply`, `band_bts`,
   `band_stable`, `pending_vestings`, `exit_queue`, `uncovered_shortfall`.
   Verify `vault_object` (1.25): `outstanding`, `min_bond`,
   `forfeit_pool_balance`, `bound_oracle`. All queryable via
   `get_objects`.

3. **H2 offset pool & rewards.** Verify `offset_pool_object` (1.38):
   `balance`, `claims[]`. Verify `reward_pool_object` (1.32): `balance`,
   `paid_lifetime`, `h2_diversion_bps`.

4. **H2 tripwire/recovery/winddown.** Verify `tripwire_state_object`
   (1.34): `win_old`, `win_new`, `halt_stage`, `redemptions_halted`.
   `recovery_state_object` (1.35): `trigger_bps`, `new_debt_suspended`,
   `last_collateral`, `last_debt`. `winddown_vault_object` (1.30):
   `segregated_bond`, `total_claims`, `claimed_total`, `paid_total`,
   `senior_paid`, `junior_paid`.

5. **G1 stealth status (singleton).** Verify `stealth_status_object`
   (2.20): `blinded_supply_shadow`, `unspent_blind_count`,
   `parity_pool`, `interval_granted`, `interval_routed`, `alarms`,
   `deposit_windows`. Queryable via `get_objects` by ID.

6. **G1 blinded balances.** Verify `blinded_balance_object` (2.10):
   `commitment`, `owner`, `asset_id`, `created_block` — there is **no
   `amount` field by design** (CDR-R11-06: the blinding is the point);
   `spent` is vestigial (whole-ring spent flag retired; explicit spends
   delete the object, and `get_stealth_outputs` `unspent_only` filters on
   the retired flag). Enumerable via `get_stealth_outputs` or `get_objects`.
   (2.19 is `stealth_nullifier` — querying 2.19.x "for blinded balances"
   silently returns nullifier objects, CDR-14. Verified at `9d4b5d85`:
   `chain/types.hpp` 2.10 blinded / 2.19 nullifier / 2.20 status — re-grep tip.)

7. **G1 nullifiers.** Verify `stealth_nullifier_object` (2.19):
   `key_image`, `proof_version`, `created`. Queryable via `get_objects`.

8. **Maintenance money-movement per-party records.** For every maintenance-side money
   movement (exit-queue batch settlement, offset-pool absorb, wind-down payouts, vesting
   releases), verify a per-party record survives at head — requester, face, net, block —
   or document replay/sampling as the official audit answer. Round-17 CDR-R17-01: batch
   exit settlement leaves no per-request payout record on the reserve; totals reconstruct,
   individual settlements do not.
9. **Seasoning and terminal-vault provenance.** Verify `seasoning_record_object`
   (1.36.x; retrievable: `seasoning_object.hpp:47-61`, FC `seasoning.cpp:331`) and the
   terminal-vault / settled-pair re-registration provenance (`vault_evaluator.cpp:217`)
   are queryable at head — which pair, which registration epoch, what settled, what was
   paid. A settled pair whose history is not retrievable cannot be audited after the
   fact. Checklist items 3/4 omit both (CDR SKILL-INAD-2).
10. **Supply reconciliation.** Verify `stealth_reconcile_supplies`
   (def `stealth_evaluator.cpp:256`, decl `stealth_evaluator.hpp:206` at
   `9d4b5d85` — the old `stealth_evaluator.cpp:194-228` pin is STALE;
   re-grep the tip: `git grep -n stealth_reconcile_supplies -- libraries/chain/`)
   compares `blinded_supply_shadow`
    vs `confidential_supply` exact integer match. Tripwire on divergence
    is consensus-enforced.

### Historical Series Survivability (Maintenance-Interval Clearing)

11. **Identify all cleared fields.** Grep `stealth_evaluator.cpp`,
   `issuance.cpp`, `db_maint.cpp` for `.clear()` calls in maintenance
   hooks. Document each: object, field, clearing function, interval.

   **Known cleared (SUP-R-10 — line numbers at `9d4b5d85`, re-grep tip):**
   - `stealth_status_object.interval_granted` (`stealth_evaluator.cpp:1742`, in `stealth_parity_maintenance` at :1674)
   - `stealth_status_object.interval_routed` (`stealth_evaluator.cpp:1741`, same fn)
   - `stealth_status_object.blind_window_counts` (pruned to 301 windows, :1749-1750)

12. **Assess survivability.** For each cleared field, determine:
    - Is it accumulated into a never-cleared rolling counter?
    - Is the aggregate `alarms.burn_rate_pct` (worst-case across assets)
      sufficient for any audit purpose?
    - Does any activation gate (E0, Rule R, governance) depend on the
      per-asset historical series?

    **SUP-R-10 finding (verify at tip):** pre-`8dc0ff75` `interval_granted`/`interval_routed` had NO
    rolling counter and the per-asset subsidy series was **lost forever** each
    interval. Post-`8dc0ff75` the rolling counters exist — do NOT re-file the
    "lost forever" claim without grepping the tip first.

13. **Define the fix.** For every per-asset flow series required by an
    activation gate, specify: rolling counter name, object field, update
    logic (append at each interval, never clear), and API exposure.

    **Required fix (SUP-R-10):**
    - `stealth_status_object.cumulative_granted[asset_id]` — append
      `interval_granted` at maintenance, never clear
    - `stealth_status_object.cumulative_routed[asset_id]` — append
      `interval_routed` at maintenance, never clear
    - Expose via `get_objects` on `stealth_status_object` (singleton)

### Third-Party Recompute Feasibility

14. **API coverage test.** For each required object, verify:
    - Can a third party discover its object ID? (singleton = known ID;
      per-pair = enumerable via index or API)
    - Can they query it via `get_objects` without auth?
    - Are all fields in the `FC_REFLECT` definition?

    **Gaps found:** No dedicated API for `reserve_object`, `vault_object`,
    `offset_pool_object`, `reward_pool_object`, `tripwire_state_object`,
    `recovery_state_object`, `winddown_vault_object`, `stealth_status_object`
    fields. Must use generic `get_objects` with known IDs.

15. **Indexer independence.** A third party running **only** `witness_node`
    + `cli_wallet` + `database_api` (no custom indexer) must be able to
    recompute all totals. Test by: fresh sync → `get_objects` for all
    required IDs → recompute → match on-chain totals.

16. **E0 measurement feasibility.** Rule R spec §8 requires per-asset
    `ΔD_a` trend computed from: deposit/exit rates, grant/route flow
    series, effective depositor count. Verify:
    - `deposit_windows` (1024 events/asset, Phase-1) → gives `E_dep`
    - `cumulative_granted`/`cumulative_routed` (SUP-R-10 fix) → gives
      per-asset flow series
    - `parity_pool` per asset → gives subsidy pot state
    All three must be queryable from chain data alone.

## Output

```json
{"skill":"chain-data-retrievability-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","phase":"current-state|historical|recompute|all",
 "current_state":{"h1_oracles":true,"h2_reserve_vault":true,
  "h2_offset_rewards":true,"h2_tripwire_recovery_winddown":true,
  "g1_stealth_status":true,"g1_blinded_balances":true,
  "g1_nullifiers":true,"supply_reconcile":true},
 "historical_series":{"cleared_fields":["interval_granted",
  "interval_routed","blind_window_counts"],
  "rolling_counters":["cumulative_granted","cumulative_routed"],
  "survivability":"VERIFY_AT_TIP (pre-8dc0ff75: false, series lost each interval; post-8dc0ff75: flow leg feasible via cumulative_* — grep tip for cumulative_granted; pilot-deposit leg still open, no pilot data yet — see SUP-R-10 history)","e0_feasible":"VERIFY_AT_TIP (flow series feasible post-8dc0ff75; full E0 still gated on pilot data + EE3-G01/G02/G03)"},
 "recompute":{"api_coverage":0.6,"indexer_independent":true,
  "e0_measurement":"VERIFY_AT_TIP (was NOT_FEASIBLE_WITHOUT_SUP_R_10 pre-fix; post-8dc0ff75 flow leg feasible, pilot leg open)"},
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],
 "not_covered":["operation history (not needed for totals)"]}
```

Severity: `Critical` = point-in-time total not queryable, SUP-R-10 fix not implemented, E0 measurement infeasible from chain data. `Important` = dedicated API missing for key objects (must use generic `get_objects`), rolling counter not exposed. `Minor`/`Note` = doc deltas.

## Rationalizations (RED baseline 2026-10-12)

| Excuse | Reality |
|---|---|
| "Objects are in the DB, anyone can query" | True for current state. False for historical series cleared by maintenance (SUP-R-10). |
| "API endpoints exist" | Only 6 H1 endpoints + `get_stealth_outputs`. H2/G1 objects require `get_objects` with known IDs — no discovery API. |
| "Recompute is just math" | The math is trivial. The data must be *retrievable*. SUP-R-10 clears the data. |
| "No budget for indexer work" | `get_objects` is built-in. No indexer needed if objects are queryable. The fix is a rolling counter in an existing object. |

## Red Flags — STOP and fix

- SUP-R-10 counters absent at the audited tip (no `cumulative_granted`/`cumulative_routed` — grep tip first; post-`8dc0ff75` this flag must NOT fire).
- E0 gate claimed measurable but per-asset flow series lost every interval.
- `stealth_status_object` fields not exposed via any API (must use `get_objects` with known singleton ID).
- H2 objects (`reserve`, `vault`, `offset_pool`, `reward_pool`) have no discovery API.
- Third-party recompute test not run on fresh sync.

**All of these mean: "protocol totals permanently public" is NOT fully true. SUP-R-10 and API gaps must close.**