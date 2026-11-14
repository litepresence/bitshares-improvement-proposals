# Skill: h2-bond-vault-implementation-audit

Use when auditing the H2 bonds/vaults branch (`feature/track-b-h2` @ `efcfd0ce`, round-18 repair tip; round-17 tip `0e6ea947` and pre-repair `91e866a3` valid as history) against its BSIP. Vantage: economic-exploit resistance — theft, collusion, griefing, and coverage-honesty.

> **ROUND-6 DELTA — items 2 and 7 below are superseded by the code.** Tags 83–103 now
> exist (`operations.hpp:146-166`), objects run 1.25.x–1.38.x, and the ISSUANCE leg
> (99–103: `hybrid_mint`, `hybrid_redeem`, `offset_deposit`, `offset_claim`,
> `issuance_exit_request`; `reserve_object` 1.37, `offset_pool_object` 1.38; LLAMMA bands,
> offset pool, batch settle, REVERT/ABSORB, cap fn) landed in `c40ff6d3`+`fbb025d2`.
> Item 7's "NO mint/convert op exists" is FALSE at this tip. The dedicated issuance
> seat `hybrid-issuance-economics-audit` is RETIRED (2026-10-31, owner direction)
> and no longer seat-audits the door; bands remain with `llamma-soft-liquidation-audit`;
> this skill keeps the bond/dispute/vault/custody scope and covers only the fence/
> absence side of the issuance leg. Item 3's challenger floor,
> item 4's fail-closed arbitration, item 5's vault custody and item 6's coverage honesty
> are unchanged and still in scope.

## Checklist (in order, every item)

1. **Gate partition.** Primitives (challenge/dispute/escalate/vote) vs gateway (`vault_register/bond_lock/bond_release/rotate`) vs price (`oracle_bond_lock/release`, `vault_resolve`, `pair_freeze`, `winddown_claim`, `oracle_fallback_set`) in `chain/h2_activation.*`; gateway<price ordering-first asserts; H1-exclusion consensus-reject in `vault_register`. Inversion acceptance = Critical.
2. **Op/object pins.** Tags 83–96 (`protocol/operations.hpp:144-157`); IDs 1.25.x–1.35.x (`protocol/types.hpp:388-398`); 7 activation cases green. Any shift = Critical.
3. **Bond economics.** Challenger floor $837.50 USD-terms-at-lock, doubling/round, hard-cap 3, 48h window, loser-pays only-slash-path, no auto-slash; 80/15 burn/pool + $83.75 finder cap; 24h cooldown, max-3 concurrent, fast-reject with forfeit; challenges never halt consumption unless coverage/tripwire fires. Verify constants at `protocol/challenge_ops.hpp:32-60`.
4. **Fail-closed arbitration.** 2/3 supermajority in-window with published quorum or auto-freeze+wind-down; same-authority ban via authority-graph check (prose-only = Important finding with exact-definition demand); missed deadline freezes.
5. **Vault custody.** Threshold authority, vesting bonds, 7d unbond + advancing exposure; `secure_ratio_bps` 20000 genesis with `mint_within_secure_ratio`; mint needs signature AND coverage AND live oracle; maintenance <150% → fixed 36h top-up (`vault_object.hpp:65`) → auto wind-down + seizure; segregated bonds; `vault_resolve` executes arbitration. "24–48h" prose = stale (36h built).
6. **Coverage honesty.** New debt blocked below 1.0; 50/30/20 routing + ≤70% joint cap; 512/interval; tripwire ladder door→debt→redemptions + 2-window reopen; Recovery 15000 bps live-only. Crash-recompute: coverage inverts as specced, halts fire in order.
7. **Issuance fence — RESOLVED, code now exists.** Tags end at 103, not 96
   (`operations.hpp:162-166`), and the mint/redeem/offset/exit door is fully built
   (ISSUANCE-2 `c40ff6d3`, `fbb025d2`). Two sub-scopes: (a) the dedicated issuer-door
   seat is retired (2026-10-31, owner direction) — do NOT audit the built door's full
   economics here; bands stay with `llamma-soft-liquidation-audit`;
   (b) confirm each still-spec-only item is STILL absent — decaying reputation
   (write-only `bootstrap_weight`), 7d timelock, 10% ESM, FXR bar, 90-day seasoning
   (this one landed: tag 98 + `805ba21f` — check whether the fence was updated), disclosure
   enforcement. Any doc claiming issuance behavior that is not in `issuance.cpp` = Critical
   BSIP-scope defect.
8. **Payout fix.** Price-invariant conversion + uint128 haircut leg (`2444b50c`); int64 overflow closed.

## Output (JSON, required)

```json
{"skill":"h2-bond-vault-implementation-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"feature/track-b-h2","tip":"efcfd0ce",
"findings":[{"id":"H2-01","severity":"Critical|Important|Minor|Note","check":"gate-partition","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

Attacker's question throughout: "what do I steal, what does it cost me, what stops me?" Every answer needs `file:line`.
