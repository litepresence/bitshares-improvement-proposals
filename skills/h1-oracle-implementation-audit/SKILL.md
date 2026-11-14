# Skill: h1-oracle-implementation-audit

Use when auditing the H1 Rev-2 oracle branch (`feature/track-b-h1` @ `45a4666e`, round-18 repair tip; round-17 tip `a9c5dfa6` and pre-repair `32285c5d` valid as history) against its BSIP. Vantage: consensus-plumbing correctness — objects, wire format, aggregation, guards, gating.

## Checklist (in order, every item)

1. **Op/object inventory.** Confirm tags 78–82 (`oracle_create/update/delete/publish`, `reference_asset_create_operation`) in `protocol/operations.hpp`; object spaces `1.23.x` + `1.24.x` in `protocol/types.hpp`. Any renumber or missing tag is Critical.
2. **Wire append-only.** `reference_mapping` + binding fields appended LAST in `bitasset_options::ext` (`protocol/asset_ops.hpp`); `base`/`quote` are `reference_asset_id_type` (`protocol/oracle.hpp`). Insertion = FAIL.
3. **Dual hardfork gating.** Each op rejected pre-`HARDFORK_ORACLE_TIME` in its evaluator (`chain/oracle_evaluator.cpp`, `chain/asset_evaluator.cpp`, `chain/proposal_evaluator.cpp`) AND `chain/hardfork_visitor.hpp` oracle typelist. Proposal-smuggled activation = Critical.
4. **Aggregation determinism.** Select-only median, per-producer outlier exclusion, no floats/`unordered_*`/wall-clock, bounded state (32 live producers; `reputation` is pruned to the producer set at `oracle_object.cpp:226-232`; **`submissions` is NOT pruned today — H1-FINAL-01/R16-F open; claim it bounded only after that repair**; 64 history / 64 subscribers / 4 series / 16 pending). 256-bit conversion with declared rounding; bid≤ask, down≤up reject. **Grep for erase/reset on every bounded-by-X map rather than trusting the doc comment.**
5. **Drift fail-closed.** Bind-time + read-time validation; drifted mapping yields no price, holds last-good delayed, blocks new debt; one pair's drift never poisons another.
6. **Guard honesty.** Flags `frozen`/`guard_rate_limit`/`guard_delay` opt-in default OFF; delay default 0; rate-limit skipped unless opted in (`chain/oracle_object.cpp:346,387`); legacy path byte-identical (`db_update.cpp:307-310`, `t8e`). Report any "H1 pair = guarded" claim as BSIP-text defect.
7. **API truth.** SIX endpoints exist, verify names and count: `get_oracles`, `get_oracle_by_name` (SINGULAR), `list_oracles`, `get_oracles_by_owner`, `get_reference_assets`, `get_reference_asset_by_symbol` (SINGULAR) — `libraries/app/database_api.cpp:2085,2103,2112,2123,2136,2152`. A doc claiming 5, or the plural spellings, is wrong. **History/reputation/delayed-price were 'missing by design' in Rev-1 and ARE NOW BUILT** — `history` at `libraries/chain/include/graphene/chain/oracle_object.hpp:79`, `reputation` at `:155`, and last-good-delayed at read time. Verified at `a9c5dfa6`. A doc STILL claiming they are absent is wrong; flag it. Also: nothing in H2's or G1's object set has a dedicated endpoint (see `api-surface-audit`).
8. **Guard-state interaction matrix.** For each guard flag, trace the state matrix
   {never-promoted, promoted, quorum-lost, frozen} × {guard_rate_limit on/off} × {guard_delay
   on/off} and assert in each of the 16 cells that (a) consumers fail closed when there is no
   usable value and (b) the rate-limit reference is never destroyed by a fail-closed reset.
   This single matrix finds H1-FINAL-02 (round-16).
9. **Cross-branch propagation.** Diff `libraries/chain/oracle_object.cpp`, `oracle_evaluator.cpp`
   and `db_update.cpp` against the H2 and G1 worktrees; record every difference as
   additive-append or semantic, and re-file every H1 defect on each branch that inherited it
   with branch+tip.
10. **Expiry — now BUILT, audit it as code.** `consensus_expir`/expiry-height no longer gaps: binding-time + read-time gates + immutability landed in `c01d9178`+`9b25f6bc` (tests `t9a`-`t9f`), and `seasoning_verify_operation` (tag 98) + the 90-day BTC seasoning clock landed in `805ba21f`; H2-gated management op required. Two things remain open and are the real findings here: (a) the LEGACY `price_oracle_id` path still has no expiry (dodge, test-pinned at `asset_evaluator.cpp:875-893`) — this is REPAIR-02, the one Important code gate; (b) legacy pairs are exempt (`oracle_tests.cpp:3467 t9d`), so a pair can dodge expiry forever by staying on the legacy field.

## Output (JSON, required)

```json
{"skill":"h1-oracle-implementation-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"feature/track-b-h1","tip":"45a4666e",
"findings":[{"id":"H1-01","severity":"Critical|Important|Minor|Note","check":"op-inventory","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

Severities: Critical = fork/theft/collusion hole or false safety claim; Important = must fix pre-H2; Minor = pre-activation cleanup; Note = honest doc delta. Every FAIL cites `file:line`; ambiguity is a finding.
