# Skill: track-independence-audit

Use when verifying a track branch carries no evaluator, hardfork gate, or include
from a track it should not contain, and that the shared substrate is byte-identical
across branches that carry it. Run over `feature/track-f`, `feature/track-g1-stealth`,
`feature/track-h`.

## Checks per branch

1. **Evaluator registration.** Read `db_init.cpp` registration block. A branch
   registers only the subsystems it owns:
   - F: oracle lane (substrate) + accs. No vault, issuance, price, reward, stealth.
   - G: stealth only. No oracle, accs, vault, issuance. (Oracle declarations
     present, no registrations.)
   - H: oracle lane (substrate) + price + smartcoins. No accs, stealth.
2. **Hardfork gates.** A branch references only its own gates plus the substrate's
   ORACLE gate. F must not reference H2_BONDS/GATEWAY/PRICE; G must not reference
   ORACLE/H2_*; H must not reference F_ACCS/STEALTH.
3. **Includes.** No `#include` of a track-owned header from a branch that does not
   own it (e.g. G including `accs_ops.hpp` is allowed for declarations; including
   `accs_evaluator.hpp` for registration is not).
4. **Substrate byte-identity.** The shared substrate files (oracle core, guard
   machinery, bond helpers) must be byte-identical across every branch that
   carries them. Diff them; any difference is a merge-conflict seed.
5. **Test suites.** A branch's test list contains no suite from an absent track
   (no h2_* on F/G, no stealth_* on F/H, no accs_* on G/H) unless compiled
   conditionally and documented.

## Evidence emitted

One JSON per branch at `build/audit-round-split/track-independence-<branch>.json`
with `auditor_model`, `branch`, `tip`, `findings[]` (file:line per violation),
`gate` (GO/NO_GO), and `conditions[]`. Aggregate with
`tooling/aggregate_audit_json.py`.

## Red flags

- A branch that "passes" because the check only grepped for one track's symbols.
- Declaring substrate identical without actually diffing the files.
- Treating declaration-only (allowed) as registration (not allowed), or vice versa.
- Skipping the test-suite check (suites leak tracks silently).
