---
name: hardfork-upgrade-safety-audit
description: >
  Use when auditing consensus upgrade safety for hardfork activations,
  branch merges, or new consensus mechanisms (slash-to-seizure, new
  evaluators). Triggers: preparing merge boundary tests, reviewing HF
  visitor logic, auditing state migration at HF boundaries, or closing
  MG-03/MG-04/SCALE gates in ACTIVATION-CHECKLIST.md.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Hardfork-Upgrade Safety Audit

## Overview

A hardfork is not a date — it is a **state migration** with irreversible
consensus consequences. This skill audits: branch merges (H1/H2/G1),
staged HF activations (ORACLE→H2_BONDS→H2_GATEWAY→H2_PRICE→STEALTH),
and net-new consensus mechanisms (slash-to-seizure). "The date passes"
is not safety; "state hash matches reference" is.

## When to Use

- Auditing the H1/H2/G1 three-branch merge (MG-03, MG-04)
- Reviewing BSIP-1 staged HF activation (4 HFs) and BSIP-2 (STEALTH)
- Auditing slash-to-seizure as a consensus upgrade (SCALE gate)
- Any new evaluator, object, or HF constant that changes consensus rules

Do NOT use for: implementation audit of individual branches
(`h1-oracle-implementation-audit`, `h2-bond-vault-implementation-audit`,
`g1-stealth-implementation-audit`), or cryptographic primitive audit
(`clsag-bp-plus-auditor`).

## Rules (no exceptions)

1. **Merge = composition, not union.** Three branches with disjoint op
   ranges still share: maintenance interval order, hardfork visitor,
   database indexes, fee schedule enum, genesis initialization. The
   merge audit must prove **no cross-branch invariant violation** at
   every shared touchpoint.

2. **Staged HF = state dependency chain.** ORACLE→H2_BONDS→H2_GATEWAY→H2_PRICE
   is a **sequence**, not a set. Each HF's state must be valid input
   for the next. A bond created at H2_BONDS must have a valid oracle
   feed from ORACLE; a gateway registered at H2_GATEWAY must have a
   valid bond from H2_BONDS. The audit must trace the chain.

3. **New mechanism = atomicity + determinism + irreversibility.**
   Slash-to-seizure (`vault_seize_operation`) must be: atomic
   (single `with_transaction`), deterministic (authority walk produces
   same recipient set on all nodes), irreversible (`seized` flag
   monotonic). No evaluator reverts it.

4. **Evidence is state-hash equivalence + property tests, not logs.**
   A `chain_test` log does not prove upgrade safety. The gate closes
   on: merge boundary test (H1→H2→G1 ops in one block = sequential
   application), HF visitor fuzz (random ops at every HF height),
   state hash at HF boundaries, seizure atomicity proof.

5. **Legacy isolation is a proof obligation, not a claim.**
   At every HF boundary, legacy assets (`bit*`, `HONEST.*`, gateway
   UIAs) must remain evaluatable by legacy code paths. No evaluator
   may `FC_ASSERT` on new object existence for legacy assets. This must
   be proven by test, not asserted in a comment.

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (every item, in order)

### Three-Branch Merge (MG-03, MG-04)

1. **Maintenance interval order.** Enumerate all 7 sweeps on merged
   chain (H2: `process_expired_h2_challenges` → `process_h2_vault_maintenance`
   → `process_h2_oracle_economics` → `process_h2_seasoning`; G1:
   `stealth_maybe_record_activation` → `stealth_reconcile_supplies`
   → `stealth_parity_maintenance`; legacy: `process_budget`). Prove:
   (a) no sweep reads state written by a later sweep in the same
   interval; (b) G1 `interval_routed` (burn denominator) is updated by
   G2 router **before** `stealth_parity_maintenance` reads it.
   Cite `db_maint.cpp` line ranges per sweep.

2. **Hardfork visitor collision.** `proposal_operation_hardfork_visitor`
   dispatches by HF version. Verify: (a) every op tag 78–109 maps to
   exactly one evaluator at every HF height; (b) no op falls through
   to wrong evaluator at boundary heights (pre-HF reject, post-HF accept);
   (c) `stealth_ops` typelist (104–109) gated by single `STEALTH.hf`.

3. **Database index coexistence.** H1 adds `oracle_object` + `price_feed_object`
   (space 1.23, 1.24); H2 adds `vault_object` (1.25) + `bond_object` (1.26)
   + `challenge_object` (1.27) + `dispute_object` (1.28) + `reserve_object`
   (1.37) + `offset_pool_object` (1.38); G1 adds `stealth_confidential_object`
   (2.19) + `ring_object` (2.20). Verify: (a) no `object_id_type` collision;
   (b) composite secondary index keys are disjoint (different `by_*`
   accessors); (c) implementation-space (2.x) is correct for chain-level
   stealth objects.

4. **Fee schedule parameter collision.** Each branch appends to
   `fee_parameters_type` enum. Verify: (a) legacy op fee IDs unchanged;
   (b) new fee parameters appended at end; (c) no branch inserts into
   pre-existing sequence. Cite `protocol/fee_schedule.hpp` enum.

5. **Genesis/initialization order.** `database::init_genesis` constructs
   objects in HF-gated blocks. Verify H1/H2/G1 genesis lambdas don't
   assume each other's objects exist (e.g., H2 vault genesis doesn't
   read `oracle_object` before ORACLE HF).

### Staged HF Activation (BSIP-1: 4 HFs, BSIP-2: 1 HF)

6. **State dependency chain.** For each HF in sequence, prove:
   - H2_BONDS: `vault_object` + `bond_object` require valid `oracle_object` feeds (from ORACLE HF)
   - H2_GATEWAY: `gateway_object` requires valid `bond_object` (from H2_BONDS)
   - H2_PRICE: `price_oracle_binding` requires valid `gateway_object` (from H2_GATEWAY)
   - STEALTH: stealth evaluators work with zero vaults/oracles/bonds in DB
   Cite the evaluator `FC_ASSERT` or read that enforces each dependency.

7. **Voting-side fork logic.** `hardfork_version_vote` supermajority activates
   HF. Audit: (a) dummy dates (2029-11-01 etc.) don't create "zombie HF"
   where vote passes but `head_block_time()` < date; (b) witness vote
   counting is deterministic; (c) HF activation is monotonic (no rollback).

8. **Legacy asset isolation at each HF.** At every HF boundary, run test
   suite: legacy asset (`bitUSD`, `HONEST.USD`, gateway UIA) operations
   (transfer, limit order, pool swap, HTLC, force-settle) execute with
   legacy evaluators, no new object reads. Any new `FC_ASSERT` on new
   object for legacy asset = FAIL.

### Slash-to-Seizure Consensus Upgrade (SCALE gate)

9. **Authority walk termination + determinism.** `vault_seize_operation`
   walks `authority` tree (multisig, threshold, key-weight). Prove:
   (a) walk terminates (no cycles) — max depth test with 100 nested
   multisig completes; (b) deterministic — same input authority produces
   same recipient set across all nodes; (c) respects `weight_threshold`
   exactly (no rounding drift).

10. **Slash accounting atomicity.** Seizure transfers
    `vault_object.collateral` → `seizure_pool_object` → `beneficiary`
    in one `apply_operation`. Prove: (a) wrapped in
    `database::with_transaction`; (b) no intermediate state visible to
    other ops (no reentrancy); (c) maintenance interrupt cannot observe
    partial state.

11. **Bond slashing precedence.** `bond_object.slashable_amount` slashed
    **before** `vault_object.collateral` (seniority). Prove ordering
    invariant under concurrent `bond_dispute_operation` — citation of
    the `FC_ASSERT` ordering in `vault_seize_evaluator::do_apply`.

12. **Irreversibility.** `vault_object.seized = true` is monotonic.
    Prove: (a) no evaluator has `vault_update` path that clears `seized`;
    (b) `seized` vault excluded from all mint/redeem/exit paths; (c)
    `seized` flag persisted in DB with no rollback path.

### Cross-Cutting

13. **Merge boundary test.** Single test case: apply H1 op + H2 op + G1 op
    in one block at each HF height. Assert post-state equals sequential
    single-branch application. Output: `merge-boundary-test.log`.

14. **HF visitor property test.** Fuzz test generating random op
    sequences across all HF versions (pre-ORACLE, ORACLE, H2_BONDS,
    H2_GATEWAY, H2_PRICE, STEALTH). Verify `visitor.visit(op)` returns
    correct evaluator for every op tag at every HF. Output:
    `hf-visitor-fuzz.log`.

15. **State hash equivalence.** `database::calculate_state_hash()` at
    each HF boundary matches pre-merge baseline for legacy assets; new
    assets hash to deterministic values. Output: `state-hash-at-hf.json`.

16. **Fee schedule regression.** `chain_test --run_test=fee_schedule*`
    passes at all four HF heights with legacy + new ops mixed.

## Output

```json
{"skill":"hardfork-upgrade-safety-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","phase":"merge|staged-hf|seizure|all",
 "merge":{"maintenance_order":"file:line ranges per sweep",
  "hf_visitor":"PASS|FAIL","db_indexes":"PASS|FAIL",
  "fee_schedule":"PASS|FAIL","genesis_order":"PASS|FAIL",
  "boundary_test":"build/evidence/merge-boundary-test.log"},
 "staged_hf":{"state_chain":["ORACLE→H2_BONDS","H2_BONDS→H2_GATEWAY",
  "H2_GATEWAY→H2_PRICE","STEALTH"],"legacy_isolation":["bitUSD","HONEST.USD","gateway"],
  "voting_logic":"PASS|FAIL"},
 "seizure":{"authority_walk":"PASS|FAIL (max_depth_test)",
  "atomicity":"file:line of with_transaction",
  "slash_precedence":"file:line of bond-before-collateral",
  "irreversibility":"PASS|FAIL (no seized-clear path)"},
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],
 "not_covered":["HFs not exercised in tests","state paths not traced"]}
```

Severity: `Critical` = merge boundary test missing, state dependency chain broken, seizure not atomic/irreversible, legacy isolation FAIL at any HF. `Important` = maintenance order unpinned, HF visitor collision possible, fee schedule shift risk. `Minor`/`Note` = doc deltas.

## Rationalizations (from RED baseline 2026-10-12)

| Excuse | Reality |
|---|---|
| "Merge is just git merge; tags don't collide" | Three branches share maintenance order, HF visitor, DB indexes, fee enum, genesis. Tag collision is the LEAST of it. |
| "Staged HFs are independent; dates are dummies" | They are a STATE DEPENDENCY CHAIN. H2_BONDS needs oracle feeds; H2_GATEWAY needs bonds; H2_PRICE needs gateways. Date is the trigger, state is the requirement. |
| "Slash-to-seizure is just a new evaluator" | It's a CONSENSUS UPGRADE: authority walk + atomic transfer + irreversible flag. One bug = stolen collateral or frozen vaults. |
| "Prior rounds covered replay; merge adds no new logic" | Prior rounds replayed BRANCHES IN ISOLATION. Merge boundary (H1+H2+G1 ops in one block) has NEVER been tested. That is the audit. |
| "Legacy assets are untouched by construction" | Construction is a CLAIM. Test it at every HF boundary or it's a finding. |

## Red Flags — STOP and prove

- Merge audit reports "no tag collision" and stops.
- Staged HF audit checks dates but not state dependencies.
- Slash-to-seizure audit checks evaluator exists but not `with_transaction` + authority walk + irreversibility.
- No merge boundary test (H1+H2+G1 ops in one block).
- No HF visitor fuzz test across all HF versions.
- No state hash at HF boundaries.
- Legacy asset test suite doesn't run at every HF boundary.
- Gate closed on "code compiles" or "binary runs" without the evidence artifacts.

**All of these mean: the upgrade is NOT audited. Produce the evidence or NO_GO.**