# Skill: supply-invariant-audit

Use when auditing whether the PROTOCOL TOTALS add up — across H1, H2 and G1, at every
block, not just in the happy-path tests. Vantage: the one number a holder, an auditor, or
a regulator reads. If a total is wrong, every coverage claim built on it is wrong.

Design invariant under test (AGENTS.md §4.4): *blind positions/transfers permitted;
protocol totals permanently public*. So a private system must still have a **provably
correct public total**. This skill proves it or finds where it breaks.

## Checklist (in order, every item)

1. **Total inventory.** For each new asset/supply counter, list: the object field, every
   consensus write site, and every read site. Minimum set: `oracle_object` aggregate supply
   (H1), `reserve_object.{stable_supply,fxr_supply,bts_balance}` (H2,
   `issuance_object.hpp:94-101`), the stable asset's `current_supply`,
   `vault_object.outstanding`, `offset_pool_object.balance`, `band_bts`/`band_stable`,
   `winddown_vault_object`, `stealth_status_object.{blinded_supply_shadow,unspent_blind_count,
   parity_pool, interval_granted, interval_routed}` (G1, `stealth_object.hpp:132,142,149,173-174`).
   Cite file:line per write site. A counter with a write but no read is a finding (item 4).
2. **H2 issuance identity.** Prove `stable_supply == Σ outstanding face` after every
   mutation: mint (`issuance.cpp:999,1048-1050`), redeem (`:1184,1170-1172`), exit request
   (`:1518,1513-1515`), REVERT (`:690,682`). Note the code only asserts the mirror on the
   full-book exit path (**`issuance.cpp:1248-1249`**; the `:1127-1128` cited previously is stale) — verify what is NOT asserted elsewhere.
3. **Reserve solvency identity.** `reserve_object.bts_balance` must equal
   `Σ (offset_pool.balance) + Σ bts actually held`. Trace every debit/credit of
   `bts_balance`: mint credit (`:979,998`), redeem debit (`:1183`), absorb credit (`:391`),
   exit-settle debit (`:612`), REVERT credit (`:688`), offset deposit debit (`:1313` — note
   this debits the DEPOSITOR, not the reserve), offset claim credit (`:1414` — note this
   credits the CLAIMANT, not the reserve). Confirm the offset pool is never counted twice,
   and that `uncovered_shortfall` (`:395-397`) is a record, not a balance — but note it is NOT
   inert (see item 4: it gates the mint door).
4. **Write-only census.** Grep every new field for reads in `libraries/`. Anything written
   and never read is either dead state or an unenforced promise. Known to check:
   `uncovered_shortfall` — **CORRECTED 2026-10-07: this is NOT write-only.** It reads in
   consensus and HALTS THE MINT DOOR: `issuance.cpp:955-959` asserts it is zero whenever the
   mint does not itself create the reserve ("mints halt until the offset pool absorbs it"),
   pinned by `h2_issuance_tests.cpp:1123`. The older "writes at `:355,397`, no consensus read"
   text was REFUTED; an auditor working from it files the inverse of the code. The open question
   is only who is owed and who is repaid first, not whether it is enforced.
   `stealth_status_object.interval_routed` vs `interval_granted`, `activation_k39_fee`,
   `activation_k39_ppo`, `blind_window_counts`, `last_ratio_bps`. For each, state the
   consequence of it being write-only (an operator watching it sees a number the chain
   does not enforce).
5. **G1 shadow supply.** `stealth_reconcile_supplies` (`stealth_evaluator.cpp:194-228`):
   exact `blinded_supply_shadow` vs `confidential_supply` comparison, or
   `blind_halted` + fire block (`:220-226`). Verify: (a) the comparison is integer/exact,
   not tolerance-based; (b) every op that creates or destroys a blinded balance updates
   BOTH sides (`confidential_evaluator.cpp:51-62,77-100,125-136,161-171,198-212,226-253`);
   (c) grandfathered pre-fork `blinded_balance_object`s (`confidential_object.hpp:48-53`
   appended `created_block`/`created_time`) are handled so the shadow starts at zero;
   (d) **CORRECTED 2026-10-07: the halt is NOT global, and the old question is now answered.**
   There are TWO distinct gates. `stealth_blind_halted` (`stealth_evaluator.cpp:114-118`, gate
   `:294`) IS global, but it is raised only by the tripwire or a committee freeze. Supply
   DIVERGENCE is per-asset: `stealth_asset_halted` (`:124`, gate `:307-308`) consults
   `diverged_assets`, and the code comment at `:242` says so explicitly — the per-asset path
   exists precisely to avoid flapping the global flag. So one asset's divergence CANNOT force a
   whole-chain halt, and spamming one asset only denies that asset. Verify the two gates stay
   distinct and that the global one cannot be reached by a single asset's divergence.
6. **Cross-leg interference.** Can a G1 blinded balance, an H2 vesting balance, or an
   offset-pool claim be moved into a path that reads a liquid balance and therefore inflate
   a coverage or rmin computation? Check the reads at `issuance.cpp:791-792,1085-1086,
   1276-1277,1441-1442` (liquid only?) and the blind-venue accounting in G1.
7. **Dust.** For every division in the supply paths, name the rounding direction and who
   gains. Specifically: `floor` vs `ceil` in `pro_rata` (`:599-604`), `h2_issue_sat_add`
   (`:89-100`), `h2_issue_coverage_bps` (`:744-754`), `fxr_price` and `fxr` (`:926-930`),
   batch-settle multiply (`:591-595`), band shift quantum (`:297-298`). Any systematic
   rounding that accumulates into a one-sided gain for an actor = Important with the amount.
8. **Saturation.** Prove every saturating helper saturates rather than wrapping:
   `h2_issue_sat_add` (`:89-100`), coverage saturation (`:744-754`), 256-bit conversion
   math (`:489-494` required-coverage multiply, `:451-453` depth sum). A wrap at max supply
   inverts a guarantee = Critical. Report max-supply headroom numerically.
9. **Third-party recompute.** Pick the smallest honest path to recompute the protocol's
   public totals from chain data alone (using only registered objects and `get_objects`).
   State whether a third party CAN recompute them today, and list every quantity they
   cannot. This is the operational form of invariant #1: "protocol totals permanently
   public" is only true if the data is retrievable.
10. **Fixture honesty.** `tests/common/database_fixture.cpp:736-740` (reserve supply counting)
    and `:747-751` (offset-pool counting) are TEST-ONLY counters. Confirm no consensus path
    depends on them, and that the fixture's notion of "supply" matches the BSIP's. A test
    fixture that counts a pool the chain does not = a false green.

11. **BSIP-3..8 totals (new asset classes).** For each new supply counter, list object
    field, every consensus write site, and every read site, and state whether a third party
    can recompute it from chain data alone:
    - BSIP-3: escrowed base amounts per offer, griefing bonds held, consumed-tx record
      growth, offer count per account/orderbook.
    - BSIP-4: pool share supply, per-token balances and fee accumulators, factory royalty
      accumulators, pool count.
    - BSIP-5: per-market provider pool accrual, vesting rebate balances, reward field per
      open order.
    - BSIP-6/7: pool invariant D (stable) / k (AB), share supply, fee accumulators.
    - BSIP-8: tank balance, share supply, NAV, HWM, tap utilization, 365-entry NAV buffer.
    A counter with a write but no read, or a total invisible to `get_objects`, is a finding.

## Output (JSON, required)

```json
{"skill":"supply-invariant-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"all three worktrees","tip":"45a4666e+efcfd0ce+8de355a3",
"findings":[{"id":"SUP-01","severity":"Critical|Important|Minor|Note","check":"identity-or-write-only-or-dust","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

Answer one sentence, with evidence: **after any block, can a third party with chain data
alone state the true total supply of every new asset class?** If not, name the exact number
they cannot see.