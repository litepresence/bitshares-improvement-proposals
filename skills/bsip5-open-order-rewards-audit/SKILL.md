---
name: bsip5-open-order-rewards-audit
description: >
  Use when auditing the BSIP-5 open-order rewards work: opt-in per-asset
  funding, per-fill maker rebate / provider pool split, TSP weight (time,
  size, proximity), anti-abuse gates, bounded sweep, self-trade refusal.
  Vantage: rewards cannot be farmed, double-paid, or pumped.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Skill: bsip5-open-order-rewards-audit

Audit BSIP-5 against its BSIP and design. Vantage: resting liquidity is paid
only for time, size, and proximity, with no double pay and no farmable payout.

## Checklist (in order, every item)

1. **Opt-in funding.** Single new field in asset options, 0–500 bps, charged on
   the received asset in the same leg as the market fee. Per-asset, per-market
   accumulation; empty default. Parameter changes only through the empty-book
   rule.
2. **Per-fill split conservation.** `maker_rebate = remaining × rebate_pct / 100`;
   `provider_pool_share = remaining - maker_rebate`; dust stays in pool. Rebate
   pays into vesting balance at fill and stays claimable after fill/cancel.
   Conservation identity holds exactly.
3. **Book state.** One block-height field per qualifying open order; amendments
   regenerate it; full fills and cancels delete it. Remaining quantity supplies
   the size leg.
4. **TSP weight.** T = min(payout_block - placement_block, MAX) with zero below
   MIN; S = remaining quantity; P = min/max price as fixed-point; skip if P <
   threshold; Pk = 10×P - 9×BASIS; TSP = S×T×Pk/BASIS. Constants convert to
   3-second blocks (MIN 100, MAX 1200). No external feed fallback for reference
   price; no fills in window means no payouts that sweep.
5. **Bounded sweep.** Per-sweep cap (top orders by standing size) before weight
   arithmetic; loop cost independent of book size. Pool remainder stays in pool.
6. **Empty-book handling.** Fees accrue while empty; pool monotone; first
   post-empty order cannot claim the whole pool; sweep with no eligible orders
   pays nothing.
7. **Amendment reset.** Same-price quantity change and price change both restart
   time credit.
8. **Proximity screen.** At or beyond threshold earns zero; linear taper inside
   the band.
9. **Self-trade refusal.** Fill rejected when aggressive and resting orders share
   an owner on participating markets.
10. **Wash-trade bound.** Operator owning fraction f of book nets strictly
    negative below full ownership; breaks even only at f = 1 before chain fees.
11. **Wire pinning.** Asset-option field, market reward object, and ids append
    and hold stable across builds with and without the flag.
12. **Hardfork gating.** Flag-gated; refuse at apply when not passed.
13. **Test vectors.** Conservation, rebate timing, minimum age, amendment reset,
    proximity screen, top-N bounds, empty book, parameter gate, self-trade
    refusal, wash-trade bound, cold start, wire pinning.

## Output

Numbered PASS/FAIL/QUALIFIED list with `file:line` evidence. Every FAIL gets
the smallest concrete fix. Ambiguity is a finding.

## Machine-Readable Output (required)

```json
{"skill":"bsip5-open-order-rewards-audit","auditor_model":"<exact model string — mandatory>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"OOR-01","severity":"Critical|Important|Minor|Note","check":"opt-in|split-conservation|book-state|tsp-weight|bounded-sweep|empty-book|amendment-reset|proximity|self-trade|wash-trade|wire-pinning|gating|test-vectors","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```