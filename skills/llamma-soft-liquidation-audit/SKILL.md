# Skill: llamma-soft-liquidation-audit

Use when auditing the BAND / soft-liquidation machinery of `feature/track-b-h2`
(`libraries/chain/issuance.cpp`, band state on `reserve_object`), against the program's
standing claim: **"overcollateralized base with LLAMMA soft bands and soft (never cliff)
liquidations"** (AGENTS.md §4.3) and the BSIP's description of them.

This is an HONESTY audit first and an arithmetic audit second. Vantage: read the code's
own words and ask whether the claim they support is the claim being made.

## Checklist (in order, every item)

1. **Locate the claim.** Quote, with file:line, every BSIP/spec/AGENTS sentence that
   describes the band mechanism as soft liquidation, LLAMMA, bands as price edges, or
   "never a cliff": `docs/superpowers/specs/2026-09-29-bsip1-hybrid-economics.md`,
   `/workspace/AGENTS.md`, `/workspace/README.md`. The repair agent needs exact text to
   narrow, not a paraphrase.
2. **Edge grid liveness.** `h2_issue_band_edge_bps` (`issuance_ops.hpp:143-156`) returns
   9000/9500/9750/10000/10250/10500/11000/11500. Grep every call site in `libraries/`.
   If the only consumers are `issuance.cpp:66` (a `using` declaration) and
   `tests/tests/h2_issuance2_tests.cpp:540-547` (pure-function asserts), then the price-edge
   grid is dead code in consensus: no price level is ever compared to a band edge. Report as
   Important (dead consensus-relevant code + a mechanism the BSIP names that no code path
   implements), with the exact grep output as evidence.
3. **What actually moves bands.** `h2_issue_maybe_rebalance` (`issuance.cpp:276-313`):
   anchor-on-first (`:282-288`), half-tick dead zone (`:289-290`), direction from
   `h2_issue_price_up` (`:201-208`), source = fullest band lowest-index (`:292-296`),
   quantum = `band[src] / 7` INTEGER DIVISION (`:297-298`), destination ±1 index with
   clamp at 6/0 (`:299-303`), skip both-zero (`:304`), anchor always advances (`:311-312`).
   Verify: the mechanism is an accounting rebalance across 7 buckets, one index step per
   conversion — NOT a price-triggered liquidation. State plainly what it is.
4. **Is any position ever soft-liquidated?** Determine whether per-position borrowing,
   collateral ratios, position-level redemption-at-band, or individual liquidations exist
   anywhere. `reserve_object` band fields are pair-level (`issuance_object.hpp:86-89`:
   "partitioned … as accounting, never custody"). If no position-level mechanism exists,
   then "LLAMMA soft liquidation" describes a system that does not exist in this code, and
   the only price enforcement is the pool absorb (`h2_offset_should_absorb`,
   `issuance.cpp:233-246`) plus the rmin floor. That is a Critical honesty finding IF the
   BSIP tells a reader holders are protected by soft liquidation — otherwise Important.
5. **Conservation.** `h2_issue_bands_spend` (`issuance.cpp:183-197`), order 3,4,2,5,1,6,0,
   asserts at `:195-196`. `h2_issue_ensure_bands` (`:163-175`) asserts size 0 or 7. Prove
   Σ`band_bts` and Σ`band_stable` are preserved across mint (`:1002-1005`), redeem
   (`:1186-1187`), absorb (`:392`), revert (`:688-691`), exit settle (`:613`), exit request
   (`:1519`). Any path that moves value without a matching band write is Critical.
6. **Rounding dust.** Shift quantum `band[src]/7` floors: quantify the value stranded per
   rebalance at pilot scale and at $1M. Ask whether stranded dust accumulates into a band
   that can never be spent (band 0/6 clamp, `:300-303`), i.e. value that becomes
   permanently unreachable. Give the number.
7. **Fairness.** Batch-exit pro-rata (`issuance.cpp:599-605`), queue order = push order
   (`:581-582`; `req.block` recorded at `:1528` and UNUSED), dust-forfeit consumes the entry
   (`:618-621`). Test whether ordering by `req.block` (available but unused) would change
   outcomes — if yes, that is an unused ordering field and a fairness Minor/Important.
8. **Hysteresis boundary.** `h2_issue_price_moved` (`:212-231`): bound `BAND_TICK_BPS/2`
   (`:229`), `dist*10000 > bound` so EQUALITY IS NOT A MOVE (`:230`), degenerate inputs
   never move (`:215-217`). State the exact dead zone in bps and what an attacker gains by
   parking the price inside it.
9. **Halt coverage.** Enumerate every path that can halt/freeze/wind-down a pair and prove
   each reaches `h2_issue_on_pair_halt` (`issuance.cpp:634-705`): `h2_create_winddown`
   (`vault_lifecycle_evaluator.cpp:55`), `pair_freeze` (`:381`), tripwire ladder
   (`h2_economics.cpp:280`), and the maintenance absorb (`issuance.cpp:555`). A halt path
   that reaches NONE leaves band state stale and redemptions open = Critical.
10. **Test honesty.** Which band behaviours have attack-input tests and which do not
    (index clamp, both-zero skip, conservation failure, dust stranded at clamp edges,
    pro-rata leg binding). Report each gap with the exact test to write.

## Output (JSON, required)

```json
{"skill":"llamma-soft-liquidation-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"feature/track-b-h2","tip":"efcfd0ce",
"findings":[{"id":"LL-01","severity":"Critical|Important|Minor|Note","check":"claim-vs-code","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

The one question that decides this audit: **does a holder reading the BSIP believe they are
protected by soft liquidation, and does the code deliver that?** Answer with the BSIP
sentence and the code line, side by side.