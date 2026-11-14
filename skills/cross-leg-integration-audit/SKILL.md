# Skill: cross-leg-integration-audit

Use when auditing the seams where H1, H2, and G1 touch each other on their feature branches. Vantage: interface betrayal — each leg keeps its own promise while the combination breaks one.

## Checklist (in order, every item)

1. **H1-exclusion completeness.** Every H2 consumer of oracle data must refuse H1-bound inputs: tripwire medians, bond-valuation liveness, cap-function depth inputs, `vault_register` binding checks. One consumer that accepts unbonded data and can move bonded medians or gates = Critical. Cite each consumer's check or its absence.
2. **Seasoning-clock isolation.** An H1 failure must neither pause nor extend any H2 seasoning/activation clock. Find the clock(s); show the H1-failure path cannot touch them.
3. **Joint fee cap.** Total diversion across H2 routing (50/30/20) and the G1 parity pool must stay ≤70% of conversion fees with one shared denominator. Locate enforcement in code (or its absence); two independent caps that sum past 70% = Important.
4. **Blind-boundary enforcement.** H2 disputes/challenges must reject blind-funded bonds (self-dealing check needs attributable keys); G1 blinds must never appear in market/vault/oracle accounting. Find the reject rule or file its absence.
5. **Coincident-breach atomicity.** One atomic `pair_freeze` path must halt door + debt together; no sequence where one leg drains while the other halts. Trace the call path; partial-halt orderings = Critical.
6. **Maintenance ordering.** H1 median refresh, H2 top-up/seizure/tripwire/recovery sweeps, G1 supply reconciliation must have a defined, tested per-block order. List the hook order in `db_maint`/`db_update`; order-dependent safety claims without pinned order = Important.
7. **Shared-primitive drift.** Bond/dispute/challenge code built once for both legs: show gateway and price legs actually share it (no forked copies with divergent constants). **For every state object mutated by evaluated paths on more than one leg, enumerate every mutation path and show each preserves the other leg's observation hooks/gates — not just identical constants** (round-17 X-R17-01: op109 missed the H2 pool-observe hook).

## Output (JSON, required)

```json
{"skill":"cross-leg-integration-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"feature/track-b-h1+feature/track-b-h2+feature/track-g1-stealth","tip":"45a4666e+efcfd0ce+8de355a3",
"findings":[{"id":"X-01","severity":"Critical|Important|Minor|Note","check":"h1-exclusion","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

Think like the crash, not the components: the shared-oracle-input block where both legs breach at once is the exam every seam must pass.
