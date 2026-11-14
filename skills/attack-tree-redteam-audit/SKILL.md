# Skill: attack-tree-redteam-audit

Use when red-teaming the built branches with concrete numbers instead of adjectives. Vantage: priced attacker — every attack carries prerequisites, cost, profit, and the exact rule that halts it.

## Checklist (one tree per item, all six)

1. **Vault theft in correlated crash.** BTS −70%: recompute vault bonds vs liabilities against built constants (`secure_ratio` 20000, maintenance 150%, 36h top-up, 15% haircut). Who gets paid, who takes the haircut, what seizes automatically? Show the block-by-block fire order with `file:line`.
2. **Cartel feed majority.** Votes needed, cost to acquire/bribe at pilot ($20k book) vs $5M scale, profit from moving settlement, which fail-closed rule fires first and what it costs the cartel to eat the freeze. **Owner-side runtime reconfiguration is a named capture surface** (round-17 ATR-R17-01 was found exactly there): price who can flip the fallback pointer, freeze flags, or oracle options at runtime, and what gate (if any) stops each flip. If capture is profitable at scale, say so with numbers — break-even at pilot is survival by smallness, not by design.
3. **Justice-pipeline grief.** Challenger floor × 24h cooldown × 3-concurrent cap × fast-reject forfeit: cost to clog all dispute slots for N windows; can spam alone freeze legitimate challenges past the 48h vote window into auto-freeze?
4. **Decoy poisoning vs ring floor.** Cost to surround a target given 24h minimum decoy age + 40%/100-block skew rejection + 11-output pool halt. Price the Sybil flood at pilot blind-output counts.
5. **Parity-pool drain.** Per-block subsidy cap × burn-rate alarms (50%/80%): cost to empty the pool, what blind fees float to, whether the float rule or an attacker sets the post-drain price of privacy.
6. **Junior front-run.** Batch-settle pro-rata + exit fee to reserve + mint suspension over queue threshold: residual margin an arber extracts around a wind-down announcement, in numbers.

## Output (JSON, required)

```json
{"skill":"attack-tree-redteam-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"feature/track-b-h1+feature/track-b-h2+feature/track-g1-stealth","tip":"45a4666e+efcfd0ce+8de355a3",
"findings":[{"id":"R-01","severity":"Critical|Important|Minor|Note","check":"vault-theft-crash","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

Rules: no attack without a price tag; no defense without a `file:line`; residuals (drought apathy, profitable-at-scale capture, over-15% haircuts) are findings, never footnotes.
