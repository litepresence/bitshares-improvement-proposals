---
name: external-audit-engagement
description: >
  Use when commissioning, receiving, or closing an external/specialist
  cryptographic audit. Triggers: hiring a specialist firm, writing an
  audit engagement scope, packaging an evidence packet, re-importing a
  returned report, or marking a specialist sign-off gate closed.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# External-Audit Engagement

## Overview

An external report is a **claim**, exactly like a companion proof document
(`clsag-bp-plus-auditor` Operating Rule 12). Re-derive every finding before
it enters the ledger. Authority is not evidence — this program has run
seventeen internal rounds and never once engaged an external party, so the
gap is not hypothetical.

## When to Use

- Commissioning an external/specialist cryptographic review.
- Deciding what goes in the evidence packet, and what is withheld.
- A returned report needs re-importing into `repairs.json` / the ledger.
- A checklist gate reads "specialist audit" and someone proposes closing it.

Do NOT use for: performing the audit yourself (use the domain skill —
`clsag-bp-plus-auditor`, `novel-construct-audit`, `stealth-composition-audit`),
or for internal round dispatch (`audit-director`).

## Rules (no exceptions)

1. **Scope first, in writing.** No engagement without a named claim
   inventory. An unscoped review returns a report that cannot be mapped to
   a gate.
2. **Send the claim map.** Operation → claimed property → enforcing
   `file:line` → the test/vector that falsifies it. This is the artifact
   that lets them check *coverage* instead of spot-checking.
3. **Send the prior rounds.** Withholding findings to avoid anchoring
   destroys the audit trail; independence is about who gets paid, not about
   what they can read. Send claims AND dispositions, clearly labelled.
4. **Re-derive every returned finding.** Record as received is not record
   as verified. A disagreement triggers re-derivation, never argument.
5. **Silence is not coverage.** An unscoped report's silence on a claim is
   indistinguishable from never having read the file. Anything unaddressed
   is `NOT_COVERED`.
6. **A gate closes on a named claim, not a verdict.** "No vulnerabilities
   found" closes a different claim than "the anonymity properties hold at
   these parameters". Read the gate line back and map each half.

**Violating the letter of these rules is violating the spirit of the rules.**

## Workflow

1. **Scope.** Name the exact claim surface the external party adjudicates
   and what we retain. Enumerate claims as independently falsifiable
   sentences, each with a named refuter (e.g. "M-leg 3-response soundness
   over (rb,rv,rd) with the M binding leg" — refuted by a prover that
   inflates `P` and leaves an H-multiple whose `log_G(H)` it lacks).
2. **Packet.** Pinned tip + commit range + build/test commands + full-binary
   log citation form (`No errors detected` and case count grepped from the
   SAME named log; never an assert total) + adversarial vectors + prior
   rounds with dispositions + known open gates. Omitting the open gates
   makes them rediscover known items as novel.
3. **Receive.** Re-derive each finding against the tree. A returned finding
   that contradicts ours triggers re-derivation, not defense.
4. **Log scope drift.** Findings outside the agreed scope get filed, not
   argued — they may expose a gap in the scope itself.
5. **Map to gates.** Each finding maps to a named activation gate, or
   becomes a new one. An engagement closing no gate is a learning
   exercise; say so rather than closing one on a vibe.

## Output

```json
{"skill":"external-audit-engagement","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","engagement":"<firm/specialist>","tip":"<commit>",
 "claims":[{"claim":"<falsifiable sentence>","refuter":"<what would disprove it>","verdict":"CONFIRMED|REFUTED|OPEN|NOT_COVERED","evidence":"file:line"}],
 "docket":["<packet artifact: tip pins, binary sha256, log citations, census outputs, vectors, NOT_COVERED annex, vendor hash transcript>"],
 "findings":[{"id":"EXT-01","severity":"Critical|Important|Minor|Note","check":"engagement-scope|packet|re-import|gate-mapping","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["claims the engagement never addressed"]}
```

Severity: `Critical` = a gate closed on unverified evidence, or an
external finding recorded as verified without re-derivation. `Important` =
scope/packet gap that manufactures coverage (silence read as confirmation).
`Minor`/`Note` = process hygiene.

## Rationalizations (RED baseline 2026-10-12 — all five shortcuts taken)

| Excuse | Reality |
|---|---|
| "The claim map is a day of work I don't have" | It is the artifact that makes coverage checkable. Send it scoped-and-partial rather than not at all; `NOT_COVERED` beats an unscoped report. |
| "They're the experts; re-verifying doubles the cost" | An unconfirmed audit is a purchased opinion. Their Important becomes yours unverified, and you cannot tell a real bug from a misread of your spec. |
| "Silence on a claim means they checked it" | Silence is indistinguishable from never-opened. An unscoped report's silence is a scope conclusion, not a content one. |
| "The gate says 'specialist audit'; a clean report closes it" | Read the gate line back — "audit AND benchmarks" is two items. Map each half; a soundness opinion closes the audit half only. |
| "Withhold our verdicts so they aren't anchored" | Withholding destroys the audit trail. Independence is who gets paid, not what they can read. Send dispositions labelled as dispositions. |

Also load: seventeen internal rounds with few Criticals is WEAK evidence of a
clean field when the rounds were scoped by the person who wrote the design —
the shared blind spot is the reason the external party exists.

## Red Flags — STOP and re-derive

- Closing a gate because a report arrived.
- A finding filed as verified with no `file:line` of your own.
- An engagement with no written claim inventory.
- Treating a partial or scoped report as full coverage.
- "No Critical from the specialists" as a ledger line.

**All of these mean: re-derive, map to a named claim, and record what was never addressed.**