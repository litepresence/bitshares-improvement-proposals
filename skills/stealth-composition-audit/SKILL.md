---
name: stealth-composition-audit
description: >
  Use when reviewing the combination of audited stealth primitives —
  CLSAG ring plus Bulletproofs+ plus conservation kernel plus nullifier
  set — or any wiring where per-component green is cited as composition
  safety. Triggers: joint-soundness questions, proof-before-nullifier
  order checks, version-leg interplay, shared-generator or fee-routing
  joints, pool-math replica fidelity.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Stealth-Composition Audit

## Overview

Components audited green still fail at their joints. Audit the
composition — shared state, ordering, and cross-primitive assumptions —
that no per-component checklist covers.

## When to Use

- Target combines ≥2 of: CLSAG/ring, BP/BP+, excess kernel, nullifier set.
- Wiring change (new op, new spend path, pool-swap replica, turnstile leg).
- Per-component green (prior audit, census, test suite) is cited as safety.
- Ordering (proof-before-nullifier), version-leg, generator-sharing, or
  fee-routing questions arise.

Do NOT use for: single-primitive verifier math (delegate to
`clsag-bp-plus-auditor`); novel pin/transcript derivation itself (delegate
to `novel-construct-audit`); economics or governance (delegate to
`cryptoeconomic-quant` / `governance-capture-audit`).

## Rules (no exceptions)

1. **Per-component green implies nothing about the joint.** Each joint
   needs its own enforcing line + refusal vector.
2. **Order is a security property.** Proof-before-nullifier, pool-math
   placement, gate order — verify by reading the sequence, not by
   pattern-matching two asserts.
3. **Both legs or it didn't verify.** Dual-version designs must
   verify-not-parse on every leg, on every path.
4. **Shared anything is a joint.** Shared generators (H/J/Gi/Hi), shared
   denominators (joint cap), shared helpers (pool math) — verify the
   sharing, don't assume independence.

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (every item, in order)

1. **Spend-path enumeration.** List every path that can spend a blinded
   output (ring spends, explicit spends, turnstile, pool swap, exit leg).
   For each: recovery/representation proof verified BEFORE nullifier
   lookup, both version legs absent pre-acceptance + inserted at apply.
2. **Version-leg interplay.** Both legs verify (not parse); key images
   version-domain-separated (J-bases differ per version); replay under one
   version gains nothing without forging both rings.
3. **Generator sharing.** H/J/Gi/Hi provenance + independence across
   pin + ring + kernel; single-H fee discipline (fee asset pinned to input
   asset); no per-asset generator confusion.
4. **Kernel composition.** Composite-F construction (128-bit, caps,
   `F ≤ a`, per-term pins); link-4 equality (`a − F == pool_pays −
   f_taker`) exact integer, `FC_ASSERT` never `GRAPHENE_ASSERT`; BP+ m=1
   enforced on every caller (exact-blob gate + consensus literal).
5. **Replica fidelity.** Shared pool-math helper byte-equivalent to the
   transparent path (diff, not prose); reserve mutation identical incl.
   asserts; fee routing (parity pool, committee cut, forfeited referral)
   traced per leg; blind-supply accounting (`conf −=`, `conf +=`) present
   so no leg mints.
6. **Denominator joints.** Joint-cap assert covers ALL contributors on ONE
   denominator (a third uncounted contributor = open finding, not a note
   about docs); burn-ratio denominator populated on every funding path.
   **Burn-gate feedback (round-17 SC-R17-09):** enumerate every gate that can
   zero routing/grants for an asset (report lapse, opt-in gates, fee-schedule
   changes) and prove the availability rule it is claimed not to affect is
   actually not affected; check the scope of the burn flag (per-asset vs global).

## Output

Per-joint verdict (PASS/FAIL/QUALIFIED/NOT_COVERED) with enforcing
`file:line` + the refusal vector proving it (guard-deletion or retarget
test). End with: the joint most likely to hide the next Critical and why.
Constants in parentheses (592B blob, 7000 cap) are tip instances —
re-measure at the tip under audit, never trust the number here.

`NOT_COVERED` is a real verdict, not a hedge. Every joint you did not
enumerate goes in `not_covered`; do not let it vanish because a sibling
joint passed.

```json
{"skill":"stealth-composition-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"SC-01","severity":"Critical|Important|Minor|Note","check":"spend-path|version-leg|generators|kernel|replica|denominator","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],
 "next_critical":"joint most likely to hide it, and why",
 "not_covered":["joints not enumerated"]}
```

Severity: `Critical` = a joint that permits forging acceptance (double
spend through an unproven leg, mint through an unchecked fee route, value
created/destroyed across primitives). `Important` = a real joint gap that
blocks sign-off but needs no adversary today (an uncounted denominator
contributor, an order reversal that changes error precedence only, an
undiffed replica claim). `Minor`/`Note` = hardening and doc deltas.

**Do not file a FAIL on letter alone.** If a joint deviates from the
skill's preferred shape but no state mutates before acceptance and no
attacker path follows, it is `Important` with the safe-in-practice
reasoning stated — not `Critical`. Reserve Critical for a reachable
forge, and say so in the evidence.

## Rationalizations (from RED baselines 2026-10-12 — all three probes failed)

| Excuse | Reality |
|---|---|
| "Just wires together reviewed parts, joint analysis is overkill" | The joint is the unreviewed part. Enumerate paths or state NOT COVERED. |
| "Per-component green (0 Critical / 0 Important) implies safe" | Green components compose into red systems. Each joint needs its own line. |
| "Covered by the op108 audit / trusted G2-4a / Lemma 6 citation" | A citation is a pointer, not a proof. Read the order, both legs, the equation. |
| "Replica matches BSIP-84 / pool math, assumed" | Two copies of a rounding rule diverge. Diff the helper or file OPEN. |

## Red Flags — STOP and enumerate joints

- "No new crypto, so no new risk" for a wiring change.
- One assert cited for an ordering property.
- Single-leg verification on a dual-leg design.
- A shared helper, generator, or denominator with no joint row.

**All of these mean: list every spend path and prove each joint with lines.**
