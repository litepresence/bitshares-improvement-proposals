---
name: novel-construct-audit
description: >
  Use when reviewing a new Schnorr or representation proof (value-pin,
  nonce, malleability), a new transcript or message-hash domain
  (G2RM/G2VP, domain separation), a composite-fee conservation kernel, or
  a companion proof sketch offered as audit support. Triggers: pin review
  requests, transcript-coverage questions, kernel-soundness claims,
  cross-domain replay concerns, nonce-handling checks.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Novel-Construct Audit

## Overview

New crypto grafted onto reviewed primitives is where soundness breaks. Re-derive the construct from code; never inherit trust from its design doc, proof sketch, or a prior coverage note.

## When to Use

- New representation/Schnorr proof (value-pin, `(T,s)` over `Y = P − a·H`).
- New or changed hash domain (ring message, pin challenge, BP+ labels).
- Composite-fee or multi-term kernel (`F = f_maker + f_taker + f_tx`).
- A companion proof document is cited as support (kernel-proof, soundness sketch).
- Nonce, malleability, or cross-domain replay is in question.

Do NOT use for: CLSAG/BP+ verifier math itself (delegate to
`clsag-bp-plus-auditor`); op wiring, fee routing, gates (delegate to
`g1-stealth-implementation-audit`); cross-primitive joints (delegate to
`stealth-composition-audit`).

## Rules (no exceptions)

1. **Proof doc is a claim, not evidence.** Re-derive every lemma from code.
   Cite `file:line` for the equation you verified, not the doc section.
2. **Coverage notes expire on read.** A prior "assessed safe" note
   (e.g. G2VP omits version/asset) must be re-verified against current code
   with fresh enforcing lines, or re-filed as open.
3. **Byte-compare transcripts.** Preimage field order, widths, and labels
   must match code byte-for-byte. No paraphrase acceptance.
4. **Test the adversary, not the happy path.** Each check needs a refusal
   vector (substituted `P`/amount/pool, retargeted ring, tampered fee).

**Violating the letter of these rules is violating the spirit of the rules.**

## Checklist (every item, in order)

1. **Nonce:** `k` fresh per proof, `T = k·G` never reused; constant-time,
   zeroized; challenge-zero refused; strict scalar/point parse before use.
2. **Malleability:** verification `s·G = T + e·Y` rejects substituted `P`,
   ±1 amount both directions, substituted pool (incl. varint-width change),
   malformed 65B (short/empty T, short/non-canonical/wrong s).
3. **Replay across domains:** same ring retargeted at different
   pool/amount/counter-asset fails; cross-op `(T,s)` reuse requires reusing
   `P`, and `P` is bound by the ring message to ring/outputs/pool — prove
   it with lines, don't assert it.
4. **Domain separation:** labels (`G1R2/G1RM/G2RM/G2VP/G1BPr/G1BPf`) do no
   double duty; challenge derivation deterministic under ROM; ring challenge
   unchanged by the new domain.
5. **G2VP re-verification:** confirm what G2VP binds (`P`, amount, pool,
   `T`) AND what it omits (version, asset); for each omission cite the
   enforcing line elsewhere (ring-message version binding per leg,
   pair-gate + membership asset check) or file the gap.
6. **Composite-F kernel:** re-derive `excess·G + F·H + C_out − P = 0 ⇒
   v_out = v_P − F` from code; verify 128-bit sum, `0 ≤ F ≤ MAX_SUPPLY`,
   `F ≤ a`, per-term `≥ 0`; verify each fee term pinned to a public
   computation (no mass-shifting); mark cited-not-proved lemmas (M-leg,
   BP+, nullifier mechanism) as obligations, never as closed.
7. **Under-binding enumeration.** Enumerate every operation field that is
   absent from all hashed messages; for each, state who can rewrite it
   without invalidating any proof. No field may be rewriteable by a party
   who does not control the proof (round-17 NC context: pending-op
   recipient-metadata rewrites).
8. **Zero-challenge at EVERY Fiat-Shamir site.** Challenge-zero refusal must
   be enforced at every site, including mid-chain round challenges — not
   only the first — and each site cited; a single unguarded mid-chain
   challenge collapses that leg (round-15/17 recovery-proof item).
9. **Instrumentation consumption.** For any invariant helper or
   instrumentation, state whether it is consumed by a `do_evaluate` money
   path. Instrumentation that is not enforced is evidence only, never a
   guard.

## Output

Per-check verdict (PASS/FAIL/QUALIFIED/NOT_COVERED) with `file:line` +
refusal-vector result. End with: which lemmas were re-derived, which are
cited obligations, and what would falsify each. Sizes in parentheses (65B
pin, 592B blob) are G2-tip instances — re-measure at the tip under audit,
never trust the number here.

`NOT_COVERED` is a real verdict, not a hedge. Use it for anything you
could not derive from code, and list it in `not_covered`.

```json
{"skill":"novel-construct-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"NC-01","severity":"Critical|Important|Minor|Note","check":"nonce|malleability|replay|domain-separation|coverage|composite-f","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],
 "rederived":["lemmas re-derived from code"],"obligations":["cited-not-proved assumptions"],
 "not_covered":["subjects this pass did not enumerate"]}
```

Severity: `Critical` = forging acceptance (false range/pin/kernel
acceptance, inflation, double spend). `Important` = soundness break
reachable under plausible conditions, or a cited-not-proved assumption
that gates the claim. `Minor`/`Note` = hardening and doc deltas.

## Rationalizations (from RED baselines 2026-10-12 — all three probes failed)

| Excuse | Reality |
|---|---|
| "Proof doc is by the designer / prior auditors accepted it" | Designer prose is the claim under test. Re-derive or state OPEN. |
| "Coverage note already assessed it safe, re-check is redundant" | Notes expire. Fresh code lines or re-file. |
| "Short on time, trusted prior audit / design §8" | Time pressure is when pins lie. Byte-compare or record NOT VERIFIED. |
| "Happy-path test passes, adversarial vectors are overkill" | Soundness IS the refusal set. No refusal vectors = no verdict. |

## Red Flags — STOP and re-derive

- Citing a proof-doc section as evidence for a code property.
- Closing a domain omission on a prior note without fresh lines.
- Accepting a transcript from prose without byte-comparison.
- A verdict with zero refusal vectors.

**All of these mean: return to the code and produce file:line evidence.**
