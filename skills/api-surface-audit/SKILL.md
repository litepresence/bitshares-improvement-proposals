# Skill: api-surface-audit

Use when auditing whether each ACTIVATION GATE and SAFETY CLAIM can be verified by a third
party using chain data alone. Vantage: a gate nobody can query is a decoration; a claim
only the committee can see is a trust assumption wearing a metric's clothes.

Activation gates come from `/workspace/build/ACTIVATION-CHECKLIST.md` (regenerated 2026-10-06, Task 11 round-8) and
the pilot requirement in `/workspace/AGENTS.md` ("≤$20k pair proving coverage 90 days").
Surfaces: `libraries/app/database_api.{hpp,cpp}`, `libraries/wallet/wallet*.{hpp,cpp}`,
`libraries/app/include/graphene/app/api.hpp` (FC_API registration lines),
`programs/cli_wallet/main.cpp`.

## Checklist (in order, every item)

1. **Gate → observable map.** For EVERY gate in `ACTIVATION-CHECKLIST.md` and every
   acceptance claim in the BSIPs, name the query that returns pass/fail. Minimum set:
   REPAIR-01 CLSAG/BP+ audit clean; ring-11/ring-16 block-time headroom; replay; multi-hour
   soak; tripwire formal algorithm; authority-walk ratification; ESM/timelock/FXR-bar;
   basket timing; joint-cap ≤70% merge assert; field-level op inventory; merged maintenance
   order; slash-to-seizure phase; T-10/T-12/T-13 admission; R-04/V-08 final values;
   halt-holder/key-holder drills; pilot coverage over 90 days; H2 legacy
   `price_oracle_id` latch/sunset. For each: verdict = QUERYABLE / NOT-QUERYABLE /
   VOTE-ONLY, with the endpoint or the word "none".
2. **Endpoint inventory.** Exact count and exact names per tree. Known: H1 exposes six
   `database_api` endpoints — `get_oracles` (`database_api.cpp:2085`),
   `get_oracle_by_name` (`:2103`), `list_oracles` (`:2112`), `get_oracles_by_owner`
   (`:2123`), `get_reference_assets` (`:2136`), `get_reference_asset_by_symbol` (`:2152`).
   H2 adds `get_h2_qualified_depth` (`:2163-2171`, declared
   `database_api.hpp:1024`, FC_API `:1643`). G1 adds NONE. Verify counts and names —
   note the singular/plural spelling of the two `*_by_*` endpoints, which a document
   claiming `get_oracles_by_name`/`get_reference_assets_by_symbol` gets wrong.
3. **The coverage gap that matters most.** From the object inventory: `vault_object` (1.25),
   `vault_bond_object` (1.26), `producer_bond_object` (1.27), `vault_challenge_object`
   (1.28), `oracle_dispute_object` (1.29), `winddown_vault_object` (1.30),
   `winddown_claim_object` (1.31), `reward_pool_object` (1.32), `guard_state_object` (1.33),
   `tripwire_state_object` (1.34), `recovery_state_object` (1.35),
   `seasoning_record_object` (1.36), `reserve_object` (1.37), `offset_pool_object` (1.38),
   `stealth_status_object` (2.20), `stealth_nullifier_object` (2.19) — confirm which have a
   dedicated endpoint and which have NO discoverable id (no op field references the
   singleton/lazily-created ones). Produce the table. Then state the consequence in one
   sentence: **what specifically cannot a pilot observer see?** Coverage, tripwire values,
   door-cap usage, offset balances, reserve/FXR balances, parity-pool balance, alarms.
4. **Coverage computability.** The pilot's success criterion is "coverage over 90 days".
   Name the exact inputs a third party needs (`vault_object.outstanding`,
   `reserve_object.bts_balance`, bonded price) and state which are reachable today by id
   lookup alone through the pre-existing `database_api::get_objects`
   (`database_api.hpp:94`, impl `:136`) given `MAP_OBJECT_ID_TO_TYPE` registration. If the
   answer is "yes but you must already know the id", say so and rate the operational cost.
5. **Alias/derived API correctness.** `get_h2_qualified_depth` (`database_api.cpp:2163-2171`)
   must agree with the consensus function `h2_issue_qualified_depth`
   (`issuance.cpp:431-454`) — same tripwire lookups, same `< 2 pools ⇒ 0` rule, same
   `GRAPHENE_H2E_TRIPWIRE_MIN_DEPTH_RAW` floor, same saturating sum. A derived API that
   disagrees with consensus is a lie to third parties = Important. Cite both sides.
6. **Wallet reachability.** `create_reference_asset` is a `wallet_api` method
   (`wallet.hpp:1267`, FC_API `:1979`, impl `wallet.cpp:837-842` /
   `wallet_asset.cpp:221-243`) **CORRECTION: it exists on ALL THREE branches** — verified at H1 `45a4666e`, H2 `efcfd0ce`,
   G1 `8de355a3`, 6 files each, identical pin lines. The older "H1 only / H2 and G1 do NOT
   have it" claim is REFUTED; do not repeat it. Enumerate
   every H2 and G1 operation that has NO wallet_api constructor at all (vault register, bond
   lock, challenge, dispute, mint, redeem, offset deposit/claim, stealth transfer). State
   the operator consequence: can a bonded oracle producer or a vault operator actually
   perform their job with the shipped wallet?
7. **cli_wallet reality.** `programs/cli_wallet/main.cpp` has no `add_command`
   registration in this codebase (it is a thin RPC console) — verify, and state that all
   exposure is via `FC_API` registration lines. Any document describing a "CLI command"
   must cite the FC_API line, not a hypothetical command.
8. **Rate limits and paging.** New endpoints vs `api_limit_get_oracles = 101`
   (`application.hpp:84`, CLI flag `application.cpp:1342`, read `:474`). Is
   `get_h2_qualified_depth` rate-limited or exempt? Does paging exist for the new
   collections? An endpoint that can page the node to death is a DoS finding.
9. **Blind-side observability.** G1's `stealth_status_object` holds `blind_halted`,
   `blinded_supply_shadow`, `parity_pool`, `alarms`, `missed_parity_reviews` — and has no
   endpoint. The program's stated rule is that protocol totals stay public. Determine
   whether the blind-side halts and alarms are observable to operators at all, and what an
   operator is supposed to watch instead.
10. **Minimum viable API patch.** Produce the smallest concrete list of endpoints that
    would make every activation gate third-party-verifiable, with the object each one
    exposes and whether it needs to be derived (computed) or raw (get_objects is enough).
    The repair agent needs this list to size the work.

## Output (JSON, required)

```json
{"skill":"api-surface-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"all three worktrees","tip":"45a4666e+efcfd0ce+8de355a3",
"findings":[{"id":"API-01","severity":"Critical|Important|Minor|Note","check":"gate-queryable|object-exposed|derived-correct","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

One sentence decides this audit: **can an outside auditor, using only documented public APIs,
verify every claim the program wants them to verify?** If not, name the three most important
unverifiable claims.