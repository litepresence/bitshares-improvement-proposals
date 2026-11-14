# Skill: crypto-agility-audit

Cryptography expires. Audit what happens the day an algorithm in the design weakens — and
whether the system can migrate without breaking itself.

## Checklist (every item, in order)

1. **Algorithm inventory.** Every cryptographic assumption in the design: signatures (ring,
   threshold, multisig), proofs (range, conservation), hashes (HTLC preimages, deposit IDs,
   nullifiers), randomness (decoy selection, producer selection, challenge ordering). Each
   gets: assumption name (e.g. discrete-log, hash collision-resistance), strength margin
   today, and what breaks first if it weakens.
2. **Migration vehicles.** For each algorithm: is there a shipped migration path (turnstile,
   re-proof, key rotation, op versioning)? A primitive with no migration vehicle is a
   finding — the turnstile precedent should generalize, not stay bespoke.
3. **Agility vs stability tension.** Consensus pins algorithms (legacy-verify-forever);
   security demands replacing them. Map exactly which pins exist and what breaks them:
   versioned ops, sunset heights, dual-verify windows. Confirm wire/tag stability pins
   exist and pass at the frozen tips (serialization/tag-stability is a first-class
   checklist row — round-17 AGI-R17-11 covered it ad hoc). An unpinned migration is a
   fork risk; an unmigratable pin is a time bomb. Both directions get checked.
4. **Trusted-setup and parameter provenance.** Any setup ceremony, initial parameters, or
   hash-to-curve choices: who generated them, is the toxic waste destroyed or multi-party,
   can a verifier reproduce the parameters from source? Halo-style setup-free designs set
   the bar — anything below it needs justification, not silence.
5. **Quantum horizon note.** Not a demand for PQC today — a demand for honesty: which
   components fall first to a cryptographically-relevant quantum computer (all discrete-log:
   rings, Bulletproofs, threshold ECDSA, HTLC hashes only second-preimage-wise), what the
   exposure window is (long-lived blind outputs vs 3s ephemeral), and whether migration
   vehicles from (2) could carry PQC replacements. One paragraph, no hype, no dismissal.
6. **Longevity without heroics.** Year-3 steady state: decaying reputations decayed, bootstrap
   subsidies lapsed, auditors rotated, original authors gone. Which security properties hold
   on autopilot (bonds, caps, halts, timelocks) and which need ongoing human excellence
   (drills, reviews, fee politics)? Every hero-dependent property gets a mechanization
   proposal or an accepted-risk label.

## Output

Algorithm table with margins and break-first analysis, migration vehicle per primitive (or
finding), pin-vs-migration map, setup provenance verdicts, quantum paragraph, and the
autopilot-vs-hero split with mechanization proposals. Design for 2027 facing means surviving
2030 cryptography.

## Machine-Readable Output (required)

```json
{"skill":"crypto-agility-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"AGI-01","severity":"Critical|Important|Minor|Note","check":"break-first|migration-vehicle|pin-vs-migration|setup-provenance|quantum","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["primitives with no break-first analysis"]}
```

Severity: `Critical` = a primitive with no migration vehicle (the version
byte exists but nothing rides it). `Important` = pinned algorithm with no
deprecation path, or opaque setup parameters without reproducible
provenance. `Minor`/`Note` = quantum and doc deltas.
