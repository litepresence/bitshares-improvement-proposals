# Skill: legacy-coexistence-invariance-audit

Use when auditing AGENTS.md objective 5 as a CLAIM rather than an intention: **"`bit*`,
`HONEST.*`, gateway UIAs coexist unchanged, permanently"** — no migration, no caps, no
haircuts, no behavioral change. The final activation gate in
`/workspace/build/ACTIVATION-CHECKLIST.md` is "Legacy invariant re-verified at merge", and
this is the only skill that tests it directly.

Vantage: this is the program's reputational and political insurance. The stated reason the
work is a parallel class rather than a fork is that legacy holders are untouched. If a
legacy holder's economics change at activation, the entire premise — and the coalition that
would approve it — is broken. Provenance: AGENTS.md §4.1 (`debt_cap_A` growth-throttle was
designed, then RETIRED for pure coexistence), §5, and the pilot's HONEST-branded pairs.

Trees: H1 `feature/track-b-h1 @ 45a4666e`, H2 `feature/track-b-h2 @ efcfd0ce`,
G1 `feature/track-g1-stealth @ 8de355a3` (round-18 repair tips; round-17 tips `a9c5dfa6`/`0e6ea947`/`5a580663`,
specialist-audited tip `e7d978c3` and pre-repair `32285c5d`/`91e866a3`/`bb6a216f` valid as history), upstream `develop @ fe7000ca`.

## Checklist (in order, every item)

1. **Diff shape.** `git diff --stat fe7000ca..<tip>` per tree. Produce the list of PRE-EXISTING
   files modified (not added), per tree. The interesting set: `asset_evaluator.cpp`,
   `asset_object.cpp`, `db_update.cpp`, `db_notify.cpp`, `db_init.cpp`, `db_block.cpp`,
   `db_maint.cpp`, `db_market.cpp`, `confidential_evaluator.cpp`, `confidential_object.hpp`,
   `htlc_evaluator.cpp`, `liquidity_pool_evaluator.cpp`, `small_objects.cpp`,
   `proposal_evaluator.cpp`, `hardfork_visitor.hpp`, `chain_parameters.{hpp,cpp}`,
   `asset_ops.{hpp,cpp}`, `libraries/app/*`, `libraries/wallet/*`, `tests/common/*`.
2. **Behavioral-change census.** For EVERY modified pre-existing file, list each hunk with
   file:line and classify it as: (a) purely additive/gated — unreachable for legacy state;
   (b) **legacy-reachable** — changes behavior for an asset or account that exists today;
   (c) legacy-capability REMOVAL. Do not accept "gated" without reading the gate condition.
3. **The known exposures — verify or refute each.**
   - G1 `db_market.cpp:2593` (`database::pay_market_fees`; the `:2687-2696` range is inside it and
     still resolves): diverts a share of the fee increment to
     the per-asset parity pool — but **ONLY for assets that opted in**. Routing is gated by
     `parity_fee_share_opted_in` (`stealth_evaluator.cpp:94-98`, called at `:1269`), which
     reads `parity_fee_share_enabled`, an `fc::optional<bool>` that is ABSENT by default
     (`asset_ops.hpp:55`). `db_market.cpp:2691-2693` states it outright: "non-opted-in assets
     are no-ops inside (increment untouched, so legacy assets credit 100% to accumulated_fees
     by construction)". Legacy coexistence is by CONSTRUCTION, not by convention. This was
     fixed by `f4de376e` (round-6 Critical GP-01/02/03). **The pre-`f4de376e` claim — "for
     EVERY asset, no exception… 5%… fee revenue shrinks by 5%" — is REFUTED; do not repeat it,
     and treat any document still asserting it as a finding.** The residual exposure is the
     opt-in itself: quantify what a LEGACY asset would lose IF it opted in
     (`GRAPHENE_STEALTH_PARITY_FEE_SHARE_BPS = 500`, `stealth.hpp:131` = 5% of the fee
     increment, post-`HARDFORK_STEALTH`), and confirm no code path can set the flag on a
     legacy asset without an explicit opt-in operation. Call sites of `pay_market_fees`
     include `liquidity_pool_evaluator.cpp:454-456`, `db_market.cpp:1799`, `:2026`, and
     `asset_evaluator.cpp:1226,1300,1366` (force-settlement; the `:1501,1554` cited previously
     is stale — those were CALL SITES, not the
     definition, which is at `db_market.cpp:2593`.
   - G1 `confidential_evaluator.cpp` (5 evaluators, ops 39/40/41 + the blind path):
     blind transfers of ANY asset, including legacy UIAs, gain the halt gate, the
     sound-range-proof gate, the float-to-cost floor, and the subsidy path. Confirm whether
     any pre-existing non-blind transfer path is altered, and whether legacy UIA confidential
     balances are grandfathered.
   - H2 `htlc_evaluator.cpp:93-100`: new sha1-HTLC creates are REFUSED post-
     `HARDFORK_H2_BONDS`. This is a chain-wide REMOVAL of a legacy capability with no
     migration path. Enumerate: which existing HTLC types, whether in-flight sha1 HTLCs can
     still be redeemed, whether redeem paths are gated too or only create, and whether any
     exchange or wallet depends on sha1 HTLC. Classify as legacy-capability removal and rate
     it.
   - H2 `liquidity_pool_evaluator.cpp:486-491`: `h2_observe_pool_exchange` samples every
     pool exchange post-fork. Confirm it is observation-only (no state written on the legacy
     path) and cannot alter pool accounting.
   - H2/G1 `db_maint.cpp` insertions: confirm they are no-ops pre-fork and appended after the
     legacy block, with `process_budget` still last (H2 `db_maint.cpp:1595-1635`, G1
     `:1593-1609`).
4. **The legacy feed path.** H1 `db_update.cpp:281` (`update_bitasset_current_feed`;
   the `:302-390` cited previously is stale): confirm the untouched legacy feed path is reached
   byte-identically when no new binding and no `price_oracle_id` are set, and that
   `asset_object.cpp` (`update_feed_from_oracle`, a NEW method) is only invoked for bound assets.
   **CORRECTION 2026-10-07: there is NO `price_feed_producer` symbol — 0 hits in H1 and 0 hits
   in upstream `develop @ fe7000ca`. It was a PHANTOM carried in this skill; do not grep for it
   and do not cite it in a finding.** Check the tests that pin this:
   `oracle_tests.cpp:3215` (`t8e_legacy_feed_path_is_untouched`), `:3484`
   (`t9d_legacy_pairs_ignore_expiry`) — both were previously mis-pinned at `:3198`/`:3467`.
   Verify those tests would FAIL if the legacy branch were altered (i.e. that they are not
   vacuous).
5. **"New pairs only" — structural or explicit?** There is no `creation_block` /
   `is_new_pair` flag. The restriction is structural: new optionals are absent on existing
   objects and `check_bitasset_options_hf_oracle` (`asset_evaluator.cpp:133-145`) only fires
   when the field is present. Assess: does any legacy asset have a way to ACQUIRE a new
   binding (`asset_update`, `asset_update_bitasset`) and thereby pull itself into the H1/H2
   machinery? Is that a governance decision (someone must pay to bind) or an automatic
   capture? Classify: if the only path is an explicit signed op by the asset's issuer, the
   invariant holds; say so with the file:line.
6. **The expiry dodge, restated against the invariant.** H1 leaves the LEGACY
   `price_oracle_id` path without expiry (`asset_evaluator.cpp:875-893`, test-pinned). This
   means a legacy-priced pair keeps an unbonded, non-expiring feed forever — while the
   program markets bonded oracles as the improvement. Frame this for the repair agent as a
   coexistence question, not just an H1 bug: the fix (REPAIR-02, latch/sunset) touches the
   legacy path by definition, which the coexistence rule forbids. Name the smallest change
   that closes the dodge without altering behavior for pairs that never bound a Track A
   oracle.
7. **Pool/graph touchpoints.** Legacy pools whose shares include the new classes; BTS used as
   bond/collateral/fee asset by the new legs; legacy margin engine sharing BTS collateral.
   Confirm the new classes never become collateral or fee assets in a legacy market in a way
   that moves legacy asset prices or collateral requirements. Cite file:line or state ABSENT.
8. **Pilot-specific check.** The pilot pair is HONEST-scale. Confirm (a) a pilot pair can be
   created without binding any legacy pair, (b) the pilot can run entirely on new assets with
   `develop` behavior unchanged, and (c) no legacy asset is required as a bond, fee, or
   collateral asset by any new op. Any new op that REQUIRES BTS (`hybrid_mint` requires a BTS
   deposit, `vault_bond_lock`, `oracle_bond_lock`) is a shared-resource dependency on a legacy
   asset's supply, not on its behavior — state the distinction explicitly so the repair agent
   does not over- or under-report it.
9. **Verdict discipline.** Separate three claims: (i) no legacy ASSET STATE is mutated —
   the strong claim; (ii) no legacy FEE/INCENTIVE stream is diverted — the one G1's
   `pay_market_fees` routing appears to violate; (iii) no legacy CAPABILITY is removed —
   the one H2's sha1-HTLC sunset violates. Report each separately with a verdict. Do not let
   a PASS on (i) paper over a FAIL on (ii) or (iii).

## Output (JSON, required)

```json
{"skill":"legacy-coexistence-invariance-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"h1+h2+g1 vs develop","tip":"45a4666e+efcfd0ce+8de355a3",
"findings":[{"id":"LEG-01","severity":"Critical|Important|Minor|Note","check":"legacy-reachable|legacy-removal|legacy-diversion","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

Report this one with unusual bluntness. If the honest answer is "legacy assets keep working,
but their fee revenue shrinks by 5% and one legacy HTLC variant is switched off", then say
that in those words, because that is what the committee will be asked to approve.