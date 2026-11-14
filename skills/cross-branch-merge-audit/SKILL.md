# Skill: cross-branch-merge-audit

Use when auditing three FEATURE BRANCHES that are built in parallel and will eventually be
merged into one chain: `feature/track-b-h1` @ `45a4666e`, `feature/track-b-h2` @ `efcfd0ce`,
`feature/track-g1-stealth` @ `8de355a3` (round-18 repair tips; round-17 tips
`a9c5dfa6`/`0e6ea947`/`5a580663`, the specialist CLSAG/BP+ pin at the earlier G1 tip
`e7d978c3`, and pre-repair tips `32285c5d`/`91e866a3`/`bb6a216f` valid as history), all
descending from `develop @ fe7000ca`.

The program's model is that H1+H2 merge into BSIP-1 (phase-gated) and G1 merges separately
as BSIP-2 (independent schedule, independent audit, independent activation). Audit rounds
1–8 each ran against ONE tree at a time (round-8: 19 auditors, 71 findings, 0 Critical;
code GO_WITH_CONDITIONS, activation NO_GO) and never checked the trees against EACH OTHER.
This skill is the only place that merge feasibility is examined. Vantage: interface
betrayal between branches, before the merge instead of after.

## Checklist (in order, every item)

1. **Op tag map.** [Repair-tip note: post-repair G1 renumbered 104-107 (e9960c6a); the 83-86 collision below is pre-repair history; verify NO collision across 83-107 + evaluator-slot guard.] Build the full table: for each branch, tag → operation name, from
   `libraries/protocol/include/graphene/protocol/operations.hpp`. H1 claims 78–82; H2
   claims 78–103; G1 claims 78–82 (inherited) then 83–86 (`operations.hpp:141-144`).
   Produce the union and list EVERY tag assigned to DIFFERENT operations in different
   branches. **Known collision to confirm and expand: G1 `stealth_transfer`(83),
   `freeze_blind`(84), `turnstile_reproof`(85), `parity_review`(86) vs H2
   `vault_register`(83), `vault_bond_lock`(84), `oracle_bond_lock`(85),
   `oracle_bond_release`(86).** A collision is a merge blocker: a serialized op is decoded
   by number, so the merged chain cannot host both meanings. Severity Critical for
   activation; state whether H1's 78–82 are byte-identical across all three branches
   (they must be, being inherited).
2. **Object id map.** Same treatment for object ids. H1: `1.23 oracle`, `1.24 reference_asset`
   (`libraries/protocol/include/graphene/protocol/types.hpp:386-387` in H1/G1;
   `libraries/protocol/.../types.hpp` H2 extends to `1.37 reserve`, `1.38 offset_pool`).
   G1 instead uses implementation space `2.19 stealth_nullifier`, `2.20 stealth_status`
   (`libraries/chain/include/graphene/chain/types.hpp:51-52`). Confirm no collision and
   state whether implementation-space ids are the correct choice for chain-level objects.
3. **Extension-field placement.** For each new optional on a PRE-EXISTING struct
   (`bitasset_options::ext`, `common_options`, `asset_options`, `blinded_balance_object`,
   `stealth_status_object`), confirm it is APPENDED LAST on every branch and that no branch
   inserts a field into a pre-existing sequence. Cite file:line. Insertion on any branch =
   Critical.
4. **Hardfork constant collisions.** Enumerate every `.hf` constant per branch
   (`libraries/chain/hardfork.d/*.hf`): `ORACLE.hf` = `1893456000` (2030-01-01),
   `H2_BONDS.hf` = `1888185600` (2029-11-01), `H2_GATEWAY.hf` = `1890777600` (2029-12-01),
   `H2_PRICE.hf` = `1893456000` (2030-01-01), `STEALTH.hf` = `1893456000` (2030-01-01).
   **Confirm that ORACLE, H2_PRICE and STEALTH all carry the identical timestamp.** Then
   answer the question the program must answer: if these are dummies (AGENTS.md §7 says
   committee dates are open), what is the real intended phasing between BSIP-1 and BSIP-2,
   and does the in-tree schedule make BSIP-2's "independent schedule" claim false as
   currently encoded? Cite the HF files and the BSIP claim text.
5. **Theoretical-vs-actual merge.** Compute the merge conflict surface: which files are
   modified by MORE THAN ONE branch relative to `fe7000ca`? For each, state whether the
   changes are in the same region (real conflict) or disjoint (auto-mergeable). Expect
   `libraries/chain/db_maint.cpp` (H2 and G1 both insert sweeps), `db_notify.cpp`,
   `db_init.cpp`, `asset_evaluator.cpp`, `hardfork_visitor.hpp`, `proposal_evaluator.cpp`,
   `operations.hpp`, `types.hpp`, `libraries/wallet/*`, `libraries/app/*`,
   `tests/common/database_fixture.cpp`. Give the merge-base and the per-file branch counts.
6. **Maintenance hook order on the merged chain.** H2 appends four sweeps to
   `perform_chain_maintenance` after the legacy block (`db_maint.cpp:1595-1635`:
   `process_expired_h2_challenges` → `process_h2_vault_maintenance` →
   `process_h2_oracle_economics` → `process_h2_seasoning`, `process_budget` still last).
   G1 appends three (`db_maint.cpp:1593-1609`: `stealth_maybe_record_activation`,
   `stealth_reconcile_supplies`, `stealth_parity_maintenance`). What is the ORDER of all
   seven on the merged chain, and is any order-dependent safety claim unpinned? Name the
   specific interaction that breaks: e.g. `stealth_reconcile_supplies` setting `blind_halted`
   while H2's tripwire ladder is mid-sweep. Cite both.
7. **Mutual exclusion that the merge must preserve.** H2 excludes H1-priced pairs from
   bonded custody (`h2_activation.cpp:82-89`, `vault_evaluator.cpp:61-62`,
   `vault_lifecycle_evaluator.cpp:433` — the `:354-355` cited previously is stale).
   **REFRAMED 2026-10-07: the premise "does the parity fee-share apply to H2 vault pairs, H1
   reference pairs, and legacy pairs alike?" is no longer the right question.** The path is
   inside the SHARED `database::pay_market_fees` (**definition `db_market.cpp:2593`**), but it
   is OPT-IN: `stealth_parity_route_fee` returns 0 before any state access when
   `parity_fee_share_opted_in` is false (`stealth_evaluator.cpp:1266-1270`), and the flag is an
   `fc::optional<bool>` ABSENT by default, so legacy assets credit 100% "by construction"
   (`db_market.cpp:2691-2693`). Legacy coexistence therefore no longer depends on merge-time
   work — it is a per-asset property. What merge must still preserve:
   (a) that the opt-in is a per-asset extension that no merge can make default-on;
   (b) that H1 reference pairs and H2 vault pairs are handled consistently if an operator opts
   them in; (c) the joint cap. The ≤70% joint cap is
   `GRAPHENE_STEALTH_JOINT_FEE_CAP_BPS = 7000` (`stealth.hpp:226`; the `:151` cited previously
   is stale), and it IS computed on ONE shared denominator: `stealth_joint_fee_within_cap`
   (`stealth_evaluator.cpp:1309-1313`) sums `parity_share_bps + h2_diversion_bps` and compares
   against the single 7000 cap, boundary-tested at `g1_stealth_tests.cpp:4838-4842,4885`.
   **MG-04 STAYS OPEN and must not be closed on the strength of that predicate:** at its only
   consensus call site (`:1420`) it sets `o.alarms.joint_cap_breach`, an ALARM, not an assert —
   a breach signals, it does not refuse. The merge-time close is to make the breach
   fail-closed on the merged chain. (`db_maint.cpp:1613-1617` at the H2 tip is the MG-03
   maintenance-ORDER todo, NOT evidence about MG-04; a lane-1 review mis-attributed it.)
8. **Same-file same-symbol.** Grep for functions/types defined on more than one branch
   (e.g. `_fxr_burn` in `vault_lifecycle_evaluator.hpp:94`, `h2_issue_*` helpers,
   `stealth_*` helpers, `h2_observe_pool_exchange`). Any same-named symbol with different
   bodies = a silent wrong-merge.
9. **Bundle/reproducibility.** `/workspace/build/bundles/*.bundle` were regenerated at the
   round-18 tips and reconstruction-verified (README in `build/bundles/`). Confirm whether
   they match the current `45a4666e`/`efcfd0ce`/`8de355a3` tips; a bundle that
   does not reproduce the audited tip breaks the whole evidence chain. Cite the bundle
   README and the commit ids in it. (Round-17 bundles pinned `a9c5dfa6`/`0e6ea947`/`1550ca41`;
   pre-repair bundles pinned `32285c5d`/`91e866a3`/`bb6a216f`; both valid as history only.)
10. **What the merge costs.** State, concretely: the minimum change set to make the three
    branches mergeable (tag renumbering vs hardfork-boundary segregation vs rebase), which
    option preserves "BSIP-2 ships on an independent schedule", and what each option costs
    in audit re-runs. This is the deliverable the repair agent needs most from this skill.

## Output (JSON, required)

```json
{"skill":"cross-branch-merge-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"h1+h2+g1 vs develop","tip":"45a4666e+efcfd0ce+8de355a3",
"findings":[{"id":"MG-01","severity":"Critical|Important|Minor|Note","check":"tag-collision|object-ids|hf-constants|maintenance-order","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

The program has never merged these branches. If this skill reports that they cannot be
merged without wire changes, that is the most important finding in the whole program and
belongs at the top of the repair document.