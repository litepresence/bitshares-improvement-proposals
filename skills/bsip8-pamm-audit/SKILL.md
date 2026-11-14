---
name: bsip8-pamm-audit
description: >
  Use when auditing the BSIP-8 PAMM account work: tank/tap custody, trading,
  performance-fee, withdrawal, and emergency taps, share/NAV accounting,
  high-water mark, custom active authorities, bounded NAV history, mark-to-
  market. Vantage: custody and share accounting honesty.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Skill: bsip8-pamm-audit

Audit BSIP-8 against its BSIP and design. Vantage: investor funds stay bounded
by taps, shares price honestly, and no manager can revoke withdrawal rights.

## Checklist (in order, every item)

1. **Tank/tap custody.** One tank per pool denominated in pool asset; trading tap
   with periodic_flow_limit and minimum_tank_level; performance-fee tap opens
   only above HWM; withdrawal tap gated by delay and per-investor cumulative
   limit; emergency tap supermajority drains to renegotiation tank and is
   required on every tank.
2. **Share accounting.** Shares are a UIA with zero initial supply; deposits mint
   at current NAV; withdrawals burn and pay at current NAV; NAV = tank_balance /
   share_supply with virtual offsets to avoid first-depositor attack.
3. **High-water mark.** Stored on tank; performance fee = fee_rate ×
   (new_equity - HWM); resets on new high; no fee below HWM.
4. **Custom active authorities.** Per-investor withdrawal authority scoped to
   their own deposit up to cumulative limit; manager trading authority
   revocable by emergency supermajority; reporting requirement on every tap.
5. **NAV history.** Rolling 365 (day, NAV) buffer; one entry per day; older
   entries fall off. `pamm_snapshot` appends today's entry if missing, gated by
   platform/auditor authority, not the manager.
6. **Mark-to-market.** NAV from bonded oracle when available, else manager
   self-report with challenge window before lock. Document the trust ceiling.
7. **Run risk.** Withdrawal delay mitigates but does not eliminate runs; verify
   the delay and queue are enforced.
8. **Bounded state.** 365-entry buffer; tap state bounded; no unbounded
   investor lists beyond caps.
9. **Hardfork gating.** `HARDFORK_PAMM_PASSED` gates tank creation, share
   issuance, and all tap operations; fail-closed at apply.
10. **Wire pinning.** New fields and ids append and hold stable across builds
    with and without the flag.
11. **Open decisions.** Foreign-asset denomination, global vs per-investor HWM,
    manager co-investment, mark-to-market source documented with owner and gate.
12. **Test vectors.** First-depositor virtual offset, HWM fee gating, emergency
    tap supermajority, withdrawal delay, NAV history bound, wire pinning.

## Output

Numbered PASS/FAIL/QUALIFIED list with `file:line` evidence. Every FAIL gets
the smallest concrete fix. Ambiguity is a finding.

## Machine-Readable Output (required)

```json
{"skill":"bsip8-pamm-audit","auditor_model":"<exact model string — mandatory>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"P8-01","severity":"Critical|Important|Minor|Note","check":"tank-tap|share-accounting|hwm|authorities|nav-history|mark-to-market|run-risk|bounded-state|gating|wire-pinning|open-decisions|test-vectors","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```