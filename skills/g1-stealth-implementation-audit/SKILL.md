# Skill: g1-stealth-implementation-audit

Use when auditing the G1 stealth branch (`feature/track-g1-stealth` @ round-15 frozen tip `5a580663`, tree CLEAN; earlier tips valid as history) against its BSIP. Vantage: cryptographic soundness + inflation containment.

> **ROUND-6 DELTA — item 7 is superseded.** The parity pool, per-block cap, float-to-cost,
> fee-share routing, review timers, op 86, default-blind wallet + batching/delay and the
> contract doc are now COMMITTED (329bfcb4, +1915/-26), not "uncommitted only". Object ids
> are `2.19 stealth_nullifier` / `2.20 stealth_status`
> (`libraries/chain/include/graphene/chain/types.hpp:51-52`). Re-audit them as tip-attributed
> code, not as a parity fence. New in round 6: the parity routing sits inside the SHARED
> `database::pay_market_fees` (**definition `db_market.cpp:2593`**; the parity block
> `:2688-2696` cited earlier is inside it and still resolves — route call `:2696`) reached
> from liquidity-pool exchange, trade, and
> force-settlement. Routing is **OPT-IN**: `parity_fee_share_opted_in`
> (`stealth_evaluator.cpp:98`, called `:1613`) over an `fc::optional<bool>` that is ABSENT by
> default, so legacy assets credit 100% by construction (the `:2696` call no-ops for
> non-opted-in assets).
> Check with `cross-branch-merge-audit` and `legacy-coexistence-invariance-audit`.

## Checklist (in order, every item)

1. **Scope fence.** Crypto touches user transfers only; market/margin/pool/settlement/vault/oracle paths transparent-aggregate permanently. Any blind-market path = Critical.
2. **Ring soundness.** `stealth_verify_ring` wired with `FC_ASSERT` — main leg `chain/stealth_evaluator.cpp:763`, alt-version leg `:784` (op109 path `:2399`/`:2413`; was mis-pinned at `:640`/`:661`, and at `:439` 201 lines off before that); verifier def `stealth_proof.cpp:555`; shape ring [11,32] asserted `:699-700`, 32+96·n wire `:708`/`:712`, dup reject `:715-716`, membership/decoy enforcement `:715-743` (decoy policy call `:743`); decoy policy 24h min age + 40%/100-block skew cap; floor-11 pool halt (`stealth.hpp:146`); cap 100. Vendor `stealth/vendor/` is reference-only, never compiled — "Monero integration" prose claiming otherwise = doc defect. Adaptation-not-verbatim (3-response, Pedersen witness) must carry `AUDIT:` markers.
3. **Range + conservation.** `stealth_verify_bp_plus` on V2 outputs + turnstile V2 legs (`:670`, `:1112`; op108 `:1982`, op109 `:2430` @ 8de355a3), V1 legacy paths preserved; `stealth_verify_excess` on spends (`:803`) + turnstile legs (`:1091`; op108 `:2061`, op109 `:2438`); verifier defs `stealth_proof.cpp:982` (BP+) / `:814` (excess); pseudo-output bound-ring `M_i`. Malformed proofs reject without state change. BP+ transcription + H-authority = specialist-audit gates.
4. **Double-spend.** `stealth_nullifier_index` registered, checked/applied (`:751`/`:753` checks, `:827`/`:832` writes; op108 `:1025`/`:1027`); `nullifier_key` is `sha256` over the 33 raw image bytes (`stealth_evaluator.cpp:109-115`) — no version byte, no spend-key; the version domain-separation lives in the image itself (`I = b*J1_v + v*J2_v`, prover `stealth_prover.cpp:225-232`, recovery verifier map `stealth_proof.cpp:803-806`); dual-verify sets; sunset spend-only-via-turnstile. Mixed rings valid only inside window.
5. **Inflation containment.** `stealth_reconcile_supplies` exact shadow-vs-counter or `blind_halted` + fire block; maintenance hook; `freeze_blind` from day one (auto tripwire + committee backup). **CORRECTED 2026-10-07 — this item previously said "2-quorum/48h-freshness", which was WRONG ON BOTH NUMBERS.** The enforced values are `GRAPHENE_STEALTH_FREEZE_VOTES_REQUIRED = 3` and `GRAPHENE_STEALTH_FREEZE_VOTE_EXPIRY_SEC = 86400` (24h) — `stealth.hpp:210-211`, documented `:446`, enforced `stealth_evaluator.cpp:928`/`:930` (voter eligibility incl. the elected-set gate in the do_evaluate block above). Quorum is SYMMETRIC (same count to freeze and to resume) and freeze wins ties (fail-closed). Also note the halt is PER-ASSET, not global: resume is refused while any asset's supply-divergence set is non-empty, so scope heals asset-by-asset. An auditor checking "2 votes in 48h" would be checking a rule the chain does not implement; G2 turnstile dated at activation. No halt path = NO_GO.
6. **Budget honesty.** The committed `WARN 2712816µs` figure is **host-dependent and has 0 hits in the tree** — it was a modeled measurement on one container, not a code constant. Treat it as historical context, NOT a claim to re-verify against code; the enforce-able facts are the constants (`GRAPHENE_STEALTH_MAX_BLIND_WORK_US_PER_BLOCK = 750000`, work units 1210/6322/2048us, `stealth.hpp:92`/`:98`/`:107`/`:112`/`:118`) and a fresh benchmark on the auditing host. R9B/R10A replaced the wall-clock question with a deterministic integer work-budget gate (`stealth_evaluator.cpp:352-437`, cap assert `:437`), so fit is now a code property, not a timing observation; ring-11 AND ring-16 full-block benchmarks on 3s blocks with headroom = pre-G1 gate. Silent budget pass = FAIL the audit, not the code.
7. **Parity/defaults fence.** Parity pool, float-to-cost, per-block cap, fee-share routing, review timers, op 86, default-blind wallet + batching/delay + contract doc are committed pre-rebase at bb6a216f (8 commits past 3116c9ef), carried through rebase (e9960c6a renumber et seq.), present at e7d978c3 and still present at 8de355a3 (R9/R10/R13/R15/R17 repairs touched `tests/` and `libraries/`, incl. the F-08 scale-escalation paragraph added to the wallet contract at 1550ca41 and the R15 representation-proof recovery; confirm per-patch at the tip under audit). Pre-329bfcb4 they existed ONLY in the uncommitted tree (+1762/-26, 1 untracked); that uncommitted-only premise and its Critical-defect clause are historical (pre-329bfcb4) scope only. Setup provenance (reproducible BP+ generators + test vector) = gate.
8. **Agility.** Version byte + sunset + dual-verify on every blind op; legacy-verify-forever never ships without deprecation vehicle; PQC paragraph honest (DL falls together; blind outputs/bonds are durable exposure).

## G2 / op109 appendix (required whenever the tree carries `stealth_pool_exchange`)

When the audited tip includes op109 (G2 blind→blind pool swaps), work items 9–15
**after** items 1–8. Per-component green on the value-pin or the ring implies nothing
about the joint — load `stealth-composition-audit` for joints and
`novel-construct-audit` for the pin/transcripts; do not re-report their verdicts.

9. **1-in/1-out soundness.** Op struct has singular `input`/`output` (no vectors).
   A multi-input form would let a prover offset a value lie across inputs. Refuse
   any future generalization without a new composite-kernel proof.
10. **Value-pin before use.** `value_pin` size == 65; `stealth_verify_value_pin`
    runs on the pseudo-output against `amount_to_sell` BEFORE ring/kernel
    acceptance and BEFORE nullifier insert. Substituted P, ±amount, substituted
    pool (varint-width change) all refuse.
11. **Link-4 fee equality.** Composite `F = f_maker + f_taker + f_tx` and
    `a − F == pool_pays − f_taker` (or the coded equivalent) enforced as
    `FC_ASSERT` (never `GRAPHENE_ASSERT`) on EVERY path — reject path included.
    An understated fee must not be accepted.
12. **Replica fidelity.** Shared pool-math helper byte-equivalent to the
    transparent `liquidity_pool_exchange` path (diff, not prose); reserve
    mutation identical including asserts; no real user balance touched
    (`GRAPHENE_TEMP_ACCOUNT` only); blind-supply `conf[A] -= …`, `conf[B] += …`
    present so neither leg mints.
13. **Ring order / decoy leak.** Final op109 ring order is shuffled (not window
    round-robin with real spliced unshuffled). PRV-01 class leak: if decoys are
    emitted deterministically, chain observers recover the real member (~16→1).
14. **Fee routing.** `g2_route_market_fee_to_parity` traces: 100% market fees of
    BOTH legs to `parity_pool`, issuers of both legs receive zero (F-03 routing —
    owner disposition gate, not silently accepted). 0%-fee pairs have no
    increment to divert (external-funder gate). Joint-cap denominator: op109
    routing is OUTSIDE the parity+h2 cap — third uncounted contributor (MG-04).
15. **Hardfork + proposal fences.** op109 in `stealth_ops` typelist behind
    STEALTH; evaluator registered; proposal fee-parameter pre-check refuses
    `stealth_pool_exchange_operation`; proposal hardfork visitor overload
    present.
15a. **Asset-policy parity (G1-FINAL-01).** op109 must enforce on both legs the
    same `allow_confidential` / `transfer_restricted` / `white_list` / market
    whitelist-blacklist / `is_authorized_asset` checks as the transparent
    equivalent — or an owner-signed exemption must be recorded in the G2 spec
    (§5/§7) and the activation checklist before pilot. Missing checks on either
    leg = Important.
15b. **Memo bound (G1-FINAL-03).** Every consensus-opaque memo field
    (`recognition.encrypted_memo`, legacy `stealth_memo`) must be size-bounded
    (<=1024) in an evaluator output loop AND priced (a data-fee leg) or
    recorded as accepted risk. Every blind op's `calculate_fee` omitting the
    data leg is the defect this item exists to catch; unbounded + unpriced =
    Important.
15c. **Sampler per-window exhaustion (G1-FINAL-02).** The op109 sampler must
    draw decoys only when at least ring_size-1 DISTINCT non-empty decoy
    creation windows exist, or throw; at k <= 7 distinct windows a passive
    observer names the real ring member with probability 1. Admission floor
    `R17B_MIN_DISTINCT_WINDOWS` (`stealth.hpp:66`, 12 at this tip; prover-side)
    is the current guard — verify it binds and that no window-singleton tell
    can be emitted.
15d. **Conservation lives in evaluators (G1-FINAL-11).** Every conservation
    kernel (`verify_sum` / `verify_excess`) must be enforced in `do_evaluate`,
    never only in `validate()` — for ALL blind ops including 39/40/41. op41 is
    the single-layer case at this tip (kernel in `confidential.cpp:81-89`
    inside `validate()`; no `verify_sum` in `confidential_evaluator.cpp` at
    `8de355a3` — round-17 CLS-FINAL-04 traced it as unreachable via the
    transaction/proposal validate() layers, tracked Minor, fix the mirror
    anyway). Any blind op whose kernel lives only in `validate()` is
    Important, because proposal embedding bypasses the op-level layer.

## Rule R appendix (required whenever the tree carries the entropy subsidy targeting)

16. **Classifier honesty.** `net_rule` / live-set delta table matches the spec
    (§1): deposits +n growth; 1-in/1-out +0; m-in/m-out +(m−1) — OR the code
    deliberately uses a different row and BSIP/spec says so. Shipping `n_out`
    while the document claims `+(m−1)` is EE3-G03, not a note.
17. **Targeting variable honesty.** `unspent_blind_count` is NOT "unspent"
    (SUP-R-04: the spent write was deleted). Do not cite it as the targeting
    variable without stating what it actually counts. E0 measurability requires
    E_dep + cumulative grant/route counters, not this field alone.
18. **Fee-floor anchor.** `tooling/verify_stealth_cost_anchor_unchanged.py`
    BYTE-IDENTICAL across the Rule R range. Subsidy cap moves the DISCOUNT only,
    never a user's fee floor. A rule that would make fees cheaper is a Critical.
19. **Pot arithmetic.** At production fee parameters the subsidy gap for
    deposits/exits is 0 (schedule ≥ anchor). Rule R reallocates ~nothing
    (F-02 OPEN). **Do not report Rule R as delivering anonymity-set subsidy**
    unless a fee-schedule change is explicitly in the activation package.
20. **Per-block cap counts OPS not VALUE.** An m→m consolidation draws the full
    m-output gap each time; named EE3-G03 consequence — confirm the cap is
    intentional or fixed.

## R15 representation-proof recovery appendix (required whenever the tree carries `turnstile_recovery_v2_extension`)

When the audited tip includes the R15 recovery extension (the F-08 re-do: proof-bound
turnstile recovery of decoyed outputs), work items 21–29 **after** items 1–20. Spec:
`docs/superpowers/specs/2026-10-08-f08-representation-proof-recovery-design.md`. The
refuted option B (`a8eabc36`, reverted `d9e22830`) is history — audit the live gate,
not the revert. Round-15 lane seats (crypto/gate/breakit) covered this at `ad3dc8f8`;
re-verify their findings at the audited tip and cover the tip delta.

21. **Two-branch gate.** Extension absent → `!spent` refusal unchanged (normal
    migrations keep full privacy, no behavior change). Extension present → owner
    match → proof verify → nullifier check on the NOW-BOUND I1/I2 (either present =
    refuse, fail-closed; mixed states must not pass) → allow despite `spent`.
22. **Check ordering.** Cheap index/uniqueness lookups MUST precede the expensive
    3-leg proof verify; an invalid tx must not burn kernel-class CPU before a
    microsecond index check refuses it.
23. **Sigma soundness.** All three equations enforced: `s_b*G+s_v*H == A_C+c*C`,
    `s_b*J1_1+s_v*J2_1 == A_1+c*I1`, `s_b*J1_2+s_v*J2_2 == A_2+c*I2`. A single-image
    substitution (true I1 + foreign I2, or vice versa) under the true proof must
    fail — a one-leg verifier otherwise passes the suite.
24. **Fiat-Shamir domain separation.** Challenge binds C, I1, I2, A_C, A_1, A_2,
    asset context, and old-commitment bytes under domain tag
    `BTS-STEALTH-RECOVERY-v1`, distinct from ring/range domains. Cross-protocol
    replay (ring proof or range proof presented as a recovery proof) must fail.
    Record whether asset binding is the full asset_id or instance.value, and
    whether a compensating evaluator assert exists.
25. **Strict parsing + edge openings.** Non-canonical scalars, identity points,
    non-curve points, and unknown version bytes refuse. `c==0` rejected explicitly
    (collapses each leg to `s_b*P+s_v*Q==A`, dropping image binding). All-zero
    responses (`s_b=s_v=0`) are representable — they must still satisfy all three
    equations and `c=0` needs a 2^256 grind; record acceptance or refusal.
26. **Replay/context binding.** Proof replayed against a different commitment
    fails (context binding). An identical (C,I1,I2,proof) tuple re-verifies BY
    DESIGN (the proof binds no nonce) — cross-op replay safety rests entirely on
    old-commitment consumption plus nullifier uniqueness; verify both are
    load-bearing and tested.
27. **Supply conservation.** Recovery moves no supply: `confidential_supply`
    delta exactly zero, fee-exact, no mint/burn on any path (decoyed success,
    really-spent refusal, malformed refusal).
28. **Wire format + tags.** Extension appended to the per-op
    `static_variant<void_t, ...>` set (never inserted); tag order pinned
    (void_t=0, recovery=1); binary round-trip stable;
    `proposal_operation_hardfork_visitor` covers the extension payload;
    size==1 enforced (oversize/wrong-type refuse).
29. **Fee/work adequacy + test sufficiency.** ~160B proof + kernel-class verify
    CPU: fee-priced or governance-accepted flat pricing recorded; work budget
    counts the recovery verify. All 9 mandatory `r15_*` cases of spec §7 present,
    incl. `r15_recovery_really_spent_refuses` (valid proof + present nullifier —
    the F08B-04 companion whose absence let a Critical through) and
    `r15_no_extension_unchanged`. Any missing mandatory case = Important.

## Output (JSON, required)

```json
{"skill":"g1-stealth-implementation-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"feature/track-g1-stealth","tip":"8de355a3",
"findings":[{"id":"G1-01","severity":"Critical|Important|Minor|Note","check":"ring-soundness","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

Inflation question throughout: "how do I print invisibly, and what halts me?" Every printing path needs a named halt with `file:line`.

Division of labor with `clsag-bp-plus-auditor`: that skill owns stateless verifier math (load it for proof-code method); this skill owns consensus integration. Run once, cite twice.
