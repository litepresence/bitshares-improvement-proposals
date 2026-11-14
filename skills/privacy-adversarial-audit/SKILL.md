# Skill: privacy-adversarial-audit

Attack the anonymity, not the cryptography. Assume primitives hold; break the system around
them the way chain-analysis firms do.

## Checklist (every item, in order)

1. **Set measurement.** What is the actual anonymity set at launch, at steady state, under
   load? Count it from chain data or state the assumption. A set asserted without a number
   is a finding.
2. **Decoy quality.** Who selects decoys, from what distribution, with what age profile?
   Temporal bias, spent-output reuse, and known-spend poisoning each get a dedicated pass.
   If selection is wallet-side with no consensus rule, say whose wallets and what enforces it.
3. **Endpoint leakage.** Exchanges, gateways, and the transparent opt-out path: where must a
   user de-blind, and what links the blind history to the identity at that point? Map every
   blind↔transparent boundary as an attack surface with the data visible there.
4. **Amount correlation.** With asset types visible and amounts hidden, what can value-flow
   analysis still infer (timing, fee fingerprinting, pool interaction sizes, challenge bonds
   with fixed denominations)? Fixed-denomination protocol flows (bonds, fees, caps) are
   fingerprints — list them.
5. **Metadata and network layer.** Transaction propagation, timing, API usage patterns: does
   the design need network-layer protections (dandelion-style propagation, RPC hygiene) or
   does it silently assume them? Silent assumptions are findings.
6. **Degradation modes.** Under spam, under low participation, under halt: what does the set
   shrink to, and does any mode collapse to effectively-transparent? Name the floor and the
   alarm that fires before reaching it.
7. **Auditability tension.** Which privacy properties conflict with which safety properties
   (supply reconciliation, fee verification, challenge evidence)? Each conflict needs its
   resolution stated in one sentence, with the loser named.

## Output

Numbered attacks with set-size impact, endpoint maps, fingerprint inventory, floor + alarm
definitions, and tension resolutions. Assume the math holds; break everything else.

## Machine-Readable Output (required)

```json
{"skill":"privacy-adversarial-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"PRV-01","severity":"Critical|Important|Minor|Note","check":"attack-inventory|fingerprint|endpoint|floor-alarm|tension","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["attack classes not attempted"]}
```

Severity: `Critical` = anonymity claim the code does not deliver (linkable
blind path, depleted set, defeating leak). `Important` = a fingerprint or
endpoint that narrows the set materially. `Minor`/`Note` = hardening and
doc deltas. An attack the pass did not attempt is `NOT_COVERED`, never
"no issue found".
