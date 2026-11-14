# G1 Stealth Overlay (reference)

## 0. Provenance — read this before trusting any pin

- **ROUND-18 UPDATE (2026-10-17): audited tip is now `8de355a3`** (SI-R18-05
  comment scope over the `24cde5fd` round-18
  cleanup of the fixed lanes: R17I-01 wallet-log guard made non-vacuous,
  R18-PRV-01 sampler assertion made real, R18-PRV-02 comment dedupe in
  `stealth_pool_builder.cpp` — comment-only, 7 lines deleted). Pin
  consequence: all ten §15 rows sit below the deleted block and shifted -7
  (`OV-R-a` guard region `:221-226` -> `:214-219`, `OV-R-b :227` -> `:220`,
  `OV-R-b2 :228` -> `:221`, `OV-R-c :264` -> `:257`, `OV-R-d :283` -> `:276`,
  `OV-R-e :307` -> `:300`, `OV-R-g2 :234-239` -> `:227-232`,
  `OV-R-h2 :241-256` -> `:234-249`, `OV-R-i2 :257-258` -> `:250-251`,
  `OV-R-j2 :279-295` -> `:272-288`), re-pinned by hand in
  `tooling/r13k1_overlay_pin_census.py` (its header note records the wave).
  Census re-run at `24cde5fd`: `{'OK': 188, 'MISS': 0, 'OOR': 0, 'NOFILE':
  0}` rc=0, anchor BYTE-IDENTICAL (sha256 `015eee29…`, 1932 B), entropy
  citations 39/39 PASS (transcript:
  `build/audit-g1-clsag-bp-specialist-packet/RERUN-24cde5fd.txt`).
  Follow-on, same day: `8de355a3` (comment-only SI-R18-05 scope in
  `g1_stealth_tests.cpp`; no pin moves, no code) — census 188/188, anchor
  BYTE-IDENTICAL, entropy 39/39
  (`build/audit-g1-clsag-bp-specialist-packet/RERUN-8de355a3.txt`).
  Full record: §14g.

- **ROUND-17 UPDATE (2026-10-17): audited tip is now `b547e650`** (round-17
  repair batch: sampler blend floor 12 + re-derived instruments, op109 re-spend
  vector, wallet log hygiene, ratio-order total). Two pin consequences:
  (a) 19 rows drift-renumbered via
  `python3 /workspace/tooling/apply_pin_rebase.py 5a580663 b547e650 --apply`
  (10 rows absorbed the `stealth.hpp` +3 line shift, 9 the builder comment
  growth); (b) OV-R-a/b were **deliberately re-pinned by hand** — R17-B landed
  the exclusion as CONDITIONAL (`distinct_windows >= R17B_MIN_DISTINCT_WINDOWS`,
  floor 12); below the floor the real's window is now ADMITTED as a decoy
  bucket, so the round-16 "real-window exclusion" reading is superseded below
  12. Rows: a = the guard (`stealth_pool_builder.cpp:226`), b = the exclusion
  branch (`:227`), b2 = the shared admission path (`:228`, new row). Census
  after the re-pin: `{'OK': 188, 'MISS': 0, 'OOR': 0, 'NOFILE': 0}` rc=0
  (checked 188; the +1 over `5a580663` is OV-R-b2). Docs-only follow-up on the
  same day: `3aefa642` (vendor manifest correction, EXT-R17-05 — no code or
  binary change). The correction did shift one pin: the vendor README
  stale-prose range moved 98-104 -> 110-112, so OV-2e and the §2 prose were
  re-pinned and the whole trio re-run at `3aefa642`: census `{'OK': 188,
  'MISS': 0, 'OOR': 0, 'NOFILE': 0}` rc=0, anchor byte-identical, entropy
  39/39 PASS (transcript:
  `build/audit-g1-clsag-bp-specialist-packet/RERUN-3aefa642.txt`).
  Full record: §14f.

- **ROUND-16 UPDATE (2026-10-16), and it changes how you read this file.**
  Three corrections, each found by a round-16 seat and each verified at the tip:
  1. **audited tip is now `5a580663`**, not `bd3504ff`. The single commit
     `bd3504ff` → `5a580663` **is** the PRV-R11-01 op109 sampler repair
     (`stealth_pool_builder.cpp`). A census pinned one commit *behind* the repair
     while the checklist cited that census as the closed gap.
  2. **The census had ZERO rows for `stealth_pool_builder.cpp`.** "174/174 OK"
     therefore certified nothing about the repaired file (VAC-R16-02,
     CLS-FINAL-03). Five rows (OV-R-a..e) were added in §15 pinning the sampler,
     *including the two constructs that make the real input identifiable* — the
     real-window exclusion `:178-186` and the post-hoc splice `:221`. A census
     that skips the file a repair lives in is the single most expensive silence
     in this overlay; a per-file **coverage floor** now makes it loud
     (`REQUIRED_COVERAGE` in the census tool, exit 1 on a gap).
   3. **The m-guard claim below is WRONG and is superseded.** Round-17 seat
      CLS-FINAL-02 re-tested it: at `5a580663`, **BOTH** guards — the
      `m != 1 && m != 2` range guard at `:990` and the `m != 1` wiring guard at
      `:995` — precede the first `m`-sized allocation at `:1011`; **either alone
      suffices** against `m = 0` and `m >= 2`; **only removing both is a
      memory-safety break** (the empty blob then reaches `:998`). The overlay
      text below ("memory-access violation", "crash primitive") is retained as
      history and must not be cited. Treat "the guard is memory-safety
      load-bearing by itself" as REFUTED unless re-proven.

- **audited tip (superseded, history): `9d4b5d85`** (`feature/track-g1-stealth`,
  worktree `/workspace/worktrees/track-g1-stealth`, tree clean).
- **Delta since the overlay tip (`9d4b5d85` → worktree HEAD `bd3504ff` at
  2026-10-15 final-pilot audit):** ONE commit, **wallet-library only**
  (`bd3504ff` WPC-06 wallet log secret hygiene + regression test). No
  consensus/verifier file touched. Overlay pin rows should still resolve;
  **re-run the census at the HEAD under audit** before trusting any row
  (`python3 /workspace/tooling/r13k1_overlay_pin_census.py`). If a row MISSes
  at HEAD, re-pin — do not assume wallet-only means zero line shifts in
  shared headers.
- **Delta since the previous pass (`a2dd340d` → `9d4b5d85`):** two commits,
  **no consensus behaviour change to any verifier** (one code-minor, one
  comment-only). `27a65111` folds four inline V1↔V2 ternaries into the
  `stealth_alt_proof_version` helper (AGI-04), charges the second kernel in
  the G2 work function (NC-14), refuses zero scalars (NC-16), and adds
  hardfork-gate refusal tests (CI-05) — full binary 848 cases exit 0, only
  pre-existing RP4/RP5 environmental. `9d4b5d85` is comment-only across six
  headers/objects. 132 pin windows re-measured via git-diff line map (130
  mechanical + OV-1g/OV-5b hand-mapped across the helper fold); census
  174/174 green, entropy 39/39 PASS, anchor byte-identical. See §14d.
- **Delta since the previous pass (`d0a9c2c5` → `3a398bb6`):** two commits,
  **no consensus behaviour change and no verifier change**.
  `a68ede10` corrects seven stale in-tree citations plus three the §14c table
  omitted, and narrows the false `stealth_evaluator.hpp` invariant reported as
  EE3-M1 ("no user's fee is computed against differently" is false for op109,
  which *gained* a float-rule floor under Rule R). `3a398bb6` adds seven
  test-only vectors (four Note acceptance gaps + three V1 refusals). The
  `stealth_evaluator.hpp` edit shifted that header by 20 lines and moved one
  census row (OV-R-g), which was re-pinned and re-verified by reading the
  printed line.
- **HISTORY — the "memory-access violation" claim below was REFUTED by
  CLS-FINAL-02 and must not be cited.** Corrected statement (verified at
  `5a580663`, census rows OV-4g0/OV-4g1/OV-4g2/OV-4g3): BOTH guards (`:990`
  and `:995`) precede the first `m`-sized allocation at `:1011`; either alone
  suffices against `m = 0` and `m >= 2`; only removing both is a memory-safety
  break (empty blob then reaches `:998`). The original claim follows, retained
  only so it cannot be quietly reintroduced:
  ~~Deleting the BP+ aggregation guard at `stealth_proof.cpp:984`
  (`m != 1 && m != 2`) does not produce a clean rejection — it produces a
  **memory-access violation**. `stealth_bp_proof_size(0)` returns 0, so the
  blob-size guard at `:991` does not catch `m = 0`, and `V[m]` then indexes an
  empty vector. The `m` guard is therefore **memory-safety load-bearing**, not
  merely policy, and any future relaxation of it is a crash primitive, not a
  rule relaxation.~~
- **Every `file:line` below resolves at `5a580663` (round-17 re-run,
  2026-10-16).** The census is machine-checkable and has been run clean at the
  current tip: `python3 /workspace/tooling/r13k1_overlay_pin_census.py` —
  **187 rows, `{OK: 187, MISS: 0, OOR: 0, NOFILE: 0}`** (rc=0), plus 97 prose
  pins in §1-§13 all in-bounds, **plus a per-file coverage report** (6 required
  files, all at or above floor; `stealth_pool_builder.cpp` floor raised to 7,
  `stealth_proof.cpp` m-guard chain now pinned OV-4g0/4g1/4g2/4g3). Each row
  carries an `expect` regex taken from the claim it
  supports and prints the *matching line*, so a green census cannot be
  satisfied by an incidental keyword. Every printed matched line was read
  before the tally was believed (see §13 step 5).
- **Pinned root: `ac159a3e`.** §14b of the previous pass pinned against a
  DIRTY worktree (`4f56b170` + uncommitted nullifier-migration WIP). That WIP was
  later committed as `ac159a3e`, and replaying the §14b pin table at
  `ac159a3e` scores **121 OK / 0 MISS** — i.e. `ac159a3e` IS the §14b basis, now
  reproducible as a commit instead of a dirty tree. That is what makes this pass
  mechanically derivable at all. **How the root was proven, precisely:** the
  §14b table was replayed against `ac159a3e` with
  `OLDREV=ac159a3e python3 tooling/r13k1_overlay_pin_census.py --old` and scored
  **121 OK / 0 MISS** — i.e. the pins §14b *had* are correct at `ac159a3e`. The
  current (re-pinned) table does NOT replay against `ac159a3e` and is not
  expected to (32 OK / 115 MISS / 21 OOR / 6 NOFILE) — `--old` mode is a
  "was this pin set ever correct at that tip" diagnostic, not a standing
  assertion. Do not read its failure as a broken re-pin; do read it as the reason
  to export and prove a root BEFORE trusting anything mapped from it.
- Line-shift arithmetic was **mechanically derived, not hand-shifted**:
  `OLDREV=ac159a3e python3 /workspace/tooling/r13k1_repin_map.py range <relpath> <old…>`
  diffs the pinned revision against `d0a9c2c5` and reports content identity
  per line (`CHANGED` flags a line whose text moved, not just its number).
- History only, never cite these as current: `1550ca41` (prior overlay tip),
  `ac159a3e` (this pass's root), `e7d978c3` (prior specialist sign-off),
  `612b354e` (first delta), `3ea02398` (F-03 fix). `bb6a216f` is the pre-repair
  tip. The R9A–R10A range (`3ccddcd6`, `be4872f3`, `43dbe8c9`, `612b354e`)
  replaced the BP+ Fiat–Shamir transcript; §4 is superseded by the R9C chained
  transcript and says so inline.
- base: bitshares-core `develop @ fe7000ca` (pristine). `libraries/chain/stealth/vendor/`
  = reference-only, never compiled. Load alongside SKILL.md Phase 4.
  Construction is adaptation, NOT verbatim CLSAG/BP+.
- Re-pin procedure: §13. Change log: §14 (2026-10-07 pass), §14b (2026-10-10,
  G2-0), §14c (**2026-10-10, this pass, `ac159a3e` → `d0a9c2c5`**).
- SKILL.md:70-76 points here and asserts the overlay's pins are current as of the
  audited tip in §0. **That assertion is only true for the tip named in §0.** At
  `d0a9c2c5` it is true; at the previous tip `1550ca41` it was false (74 of 75
  pins failed to resolve there). Do not read SKILL.md as a standing guarantee.

**THE REVIEW GAP THIS PASS CLOSED — read it as a warning, not a formality.**
Before this pass the overlay enumerated ops 104–107 only and had **zero pins on
op108, op109, or Rule R**. The census was *green* at that gap: 121 OK / 0 MISS
at the §14b basis, because the census only checks the rows it was given. **A green
census is not coverage.** The tree carried op108 (`397c2525`-lineage R15 nullifier
migration) and op109 plus Rule R with no specialist pin on any of it, and the
"absence of review" would have read to the next auditor as approval. §10/§11/§12
now carry 50 rows over those three subjects; if you extend the tree, extend the
overlay in the same pass or mark the gap `NOT COVERED` in §0.

## 1. Dual-nullifier linkage

Delta vs Monero: no single `I = x·Hp(P)`. The spent opening `(b, v)` for member `Q = b·G + v·H` produces `I = b·J1_v + v·J2_v` with per-version bases; both the op-version and alternate-version images must verify, not merely parse.

- Doc: `libraries/protocol/include/graphene/protocol/stealth.hpp:289-290` (the `I = b*J1 + v*J2 … domain-separated per proof version (J1/J2 differ per version)` contract on `stealth_ring_input`); seeds `GRAPHENE_STEALTH_J1_SEED` / `J2_SEED` at `stealth.hpp:204-205`.
- Spec: `libraries/chain/include/graphene/chain/stealth_proof.hpp:60-63` (`AUDIT:` J1/J2 must differ per version), equations `:182-185` (`L_i`/`R_i`/`M_i`/`c_{i+1}`), linkage/soundness note `:194-199` (pseudo-outputs are never range-proven; their value bound is inherited transitively through the M leg).
- Evaluator (`stealth_evaluator.cpp`): version-domain comment `:617` (`I = b*J1_v + v*J2_v, J bases differ per version`), dual-verify window comment + `alt_version` mapping `:615-624`, uniqueness checks on both images `:736-739`, nullifier stores `:814` (op version) / `:819` (alt version), `nullifier_key` def `:104-110`.
- Key cache: `J_cache()` def `libraries/chain/stealth_proof.cpp:281-294` (the `[J1v1, J2v1, J1v2, J2v2]` derivation from the two seeds), H bytes `stealth_proof.cpp:268-279`, `stealth_H_point` `stealth_proof.cpp:381-384`.

Specialist must verify: per-version domain separation is real (J1/J2 differ per version with distinct seeds), and BOTH legs verify — an unverified alternate image is a lying-prover bypass (spend once per version).
Closes: `G1-05` sub-question 1, `PRG-04` nullifier scope.

## 2. H authority and generator provenance

`H` is the S2/Elements constant verbatim so legacy `fc::blind` commitments verify under it; `fc` submodule pinned `@ 0d43f15`.

- `H_bytes()` `stealth_proof.cpp:268-279`, consumed at `:352` (`c.H = decode_or_null( H_bytes() );`); `stealth_H_point()` `:381-384`.
- Spec `stealth_proof.hpp:43-56` (`AUDIT:`, S2/Elements verbatim + legacy-compat rationale).
- Upstream source: `stealth/vendor/secp256k1-zkp-modules/generator/main_impl.h:32` (`secp256k1_generator_const_h`, byte-identical x-coordinates).
- Stale-prose warning: `stealth/vendor/README.md:110-112` describes try-and-increment derivation; code wins (`stealth_proof.cpp:268-279`).
- Bulletproof generators `Gi/Hi` `stealth_proof.cpp:296-318` (`BPG_cache()` / `BPH_cache()`, 64 entries each).

Specialist must verify: H bytes equal the S2/Elements constant exactly, J1/J2 derive reproducibly from the in-code seeds, and golden test vectors pin all three (BSIP-2 §5(2) setup-provenance gate: opaque-parameter libraries rejected).
Closes: `G1-05` sub-question 2, `PRG-04` H-authority scope.

## 3. 3-response adaptation (not verbatim CLSAG)

Each ring member carries `(rb, rv, rd)`; the witness is the Pedersen opening `(b, v)`, and a pseudo-output `M` leg binds the kernel. Equations (`stealth_proof.cpp:637-670`, batched `EC_POINTs_mul` at `:650,660,670`):

- `L = rb·G + rv·H + c·Q`, `R = rb·J1 + rv·J2 + c·I`, `M = rd·G + c·D` with `D = P − Q_i`.
- Per-member scalar parse `stealth_proof.cpp:627-629` (`scalar_parse_strict`, def + AUDIT note `:110-117`); `c0` canonical check `:618-619`.
- Spec: adaptation notice `stealth_proof.hpp:201-211` (`AUDIT:`), transcript/weight note `:303-309`, principal-root note `stealth_proof.cpp:213-227`.
- Challenge domain `G1R2` (`stealth_proof.cpp:436`), message domain `G1RM` (`stealth_proof.cpp:483`); `stealth_challenge` `stealth_proof.cpp:431-449`, `stealth_ring_message` `stealth_proof.cpp:475-507`.
- Verifier: `stealth_verify_ring` def `stealth_proof.cpp:551`; calls `stealth_evaluator.cpp:748` (op version) and `:769` (alt version).

Specialist must verify: 3-response soundness over `(rb, rv, rd)` with the `M` binding leg — an inflated `P` must leave an `H`-multiple whose `log_G(H)` the prover lacks — and that `G1R2`/`G1RM` domains separate kernel from message binding.
Closes: `G1-05` sub-question 3, `PRG-04` 3-response scope.

## 4. Bulletproofs+ transcription (2020-735 §§3–4, secp256k1 port)

Fixed weight-1 single proofs, prime-order form, no cofactor clearing, G1 labels (not S1):

- **SUPERSEDED at `612b354e` by R9C (`43dbe8c9`): the transcript is now CHAINED.** It is
   `state_0 = z`, `e_j = stealth_bp_hash("G1BPr", {state_{j-1}, L_j, R_j})` with
   `state_j = e_j`, and `e = stealth_bp_hash("G1BPf", {state_last, A1, B})`. The pre-R9C
   flat form used a single `G1BPe` for both round and final challenges and is now REJECTED
   (a deliberate pre-activation protocol break). Labels are partitioned
   `G1BPy` / `G1BPz` / `G1BPr` / `G1BPf` (`stealth_proof.cpp:1092,1095,1108,1118`); hash helper
   `stealth_bp_hash` def `:451`, chaining documented `:1064-1078` and
   `stealth_proof.hpp:87-90,306-308`. The old `G1BPe` label survives only in comments
   (`:1070`, `hpp:308`) and in the test-only flat reference prover.
- Ring-challenge domain is `G1R2` (`stealth_proof.cpp:436`), def `stealth_challenge` `:431`. **There is no
  `G1R1` domain in the tree** — comments at `:421-430` and `hpp:75-81` contrast against a
  G1b-2 pre-kernel domain that no longer exists anywhere in `libraries/` (verify with
  `rg -n "pre-kernel" libraries/`, which returns comments only; a bare `rg G1R1 libraries/`
  additionally matches `egenesis/genesis.json` base58 — see the CAUTION at `:428-430`).
- Size gate `stealth_bp_proof_size` `stealth_proof.cpp:941` (`m ∈ {1,2}`, `rounds = 6 + logM`); exact-blob
  acceptance enforced at `:991` (`proof.size() != stealth_bp_proof_size(m)` → `"blob size"`).
  Wire sizes are 592 B (m=1) and 658 B (m=2); only m=1 is wired to consensus (R9A `3ccddcd6`,
  gate `stealth_proof.cpp:989` — `if( m != 1 ) { err = "only m=1 is wired"; return false; }`,
  rationale at `:985-988`).
- Strict scalar discipline: canonical nonzero helper `scalar_canonical_nz` `stealth_proof.cpp:134`;
  **ring-path** rejections `c0 not canonical` `:619` and `responses not canonical` `:631` (both
  inside `stealth_verify_ring`, NOT the BP+ path — corrected 2026-10-07, see §14); the
   BP+ path's strict canonical-AND-nonzero parse is the `load_scalar` lambda at `:1025-1031`.
- Verifier def `stealth_verify_bp_plus` `stealth_proof.cpp:976`. Calls, all m=1: transfer V2 outputs
  `stealth_evaluator.cpp:655`, turnstile V2 leg `:1088`; V1 legacy paths preserved.
- Balance kernel `stealth_verify_excess` def `stealth_proof.cpp:808`; spend wiring
  `stealth_evaluator.cpp:788`, turnstile wiring `:1067`.

Specialist must verify: transcript labels bind commitments + fee + statement with nothing varying outside the transcript; **the chain threads forward and its seed is the full statement**; no label does double duty; weight fixed 1; canonical/nonzero scalar discipline; exact-blob acceptance; V2-only enforcement with V1 paths genuinely preserved.
Closes: `G1-05` sub-question 4, `PRG-04` BP+ transcription scope.

## 4b. Ring-message preimage (separate domain, `G1RM`)

`stealth_ring_message` def `stealth_proof.cpp:475-507`; domain `"G1RM"` at `:483`. Preimage
order is `"G1RM" ‖ version ‖ LE64(n) ‖ LE64(m) ‖ asset ‖ image33 ‖ ring[33n] ‖ pseudo[33] ‖ outputs[33m] ‖ fee`,
hashed with `sha256_of` — one flat concatenation, distinct from the BP+ stage labels. The
count prefix itself is `stealth_proof.cpp:492-497`; the spec's rendering of it is
`stealth_proof.hpp:105-106` (with the full per-field encoding table at `:108-121`).

**F-03 (mimo-2.6, Minor) is CLOSED at `d0a9c2c5`.** `stealth_ring_message` prepends
fixed-width LE64 `ring.size()` and `outputs.size()` to the preimage (fix commit `3ea02398`,
re-pinned here), so the commitment run can no longer be re-partitioned. Verified by
exhaustive search K=2..8: 0 collisions in the new layout (the old layout produced 35 at
K=8), plus 83 image-length x partition combinations with 0 collisions (old: 76).

Preimage as implemented (post-fix, `stealth_proof.cpp`):
  "G1RM" || version(1) || LE64(n) || LE64(m) || asset(varint, VARIABLE width) ||
  image33(33) || ring[33n] || pseudo(33) || outputs[33m] || fee(8, fixed LE)

Do NOT report F-03 as open. Do report it as open if `ring.size()`/`outputs.size()` are
ever absent from this preimage again — a message-hash change, so only legal PRE-ACTIVATION.
**op109's `G2RM` (`:519`, §11) carries the SAME LE64 count prefix discipline**; check both
domains together, not one.

Specialist must verify for this domain: the preimage cannot be re-partitioned into a
different `(n, m, ring, pseudo, outputs, fee)` tuple yielding the same hash; `n` and `m` are
canonical-length-prefixed; version and asset are bound; `pseudo` cannot be confused with a
ring member or an output commitment.

## 5. Turnstile, dual-verify window, sunset, ring-scheme slot

- `GRAPHENE_STEALTH_V1_LIFETIME_SEC = 31536000` (`protocol/stealth.hpp:187`): V1 spends become spend-only-via-turnstile at sunset; turnstile G2 IS the sunset date, fixed at activation.
- Dual window `[activation, sunset]`: both nullifier sets enforced; every spend authorizes under both versions (`stealth_evaluator.cpp:615-624`).
- `ring_scheme` slot: consensus check `stealth_evaluator.cpp:688-691` (unknown schemes fail closed); relay `validate()` must also reject unknown schemes — verify both layers (`protocol/stealth.cpp:33-35` states the no-divergence rule; `stealth_transfer_operation::validate` at `:44`, whose relay-side `ring_scheme` assert is `stealth.cpp:74-75`).
- Ops **104–109** in the op enum (`protocol/operations.hpp:168-173`); all six evaluators registered `chain/db_init.cpp:162-167`; nullifier/status indices `:199-200`; the proposal hardfork visitor covers all six (`hardfork_visitor.hpp:98-103`) and the proposal fee-parameter bans at `proposal_evaluator.cpp:324-327`.
- **OBSERVATION, not a finding — comment asymmetry in the op enum.** Lines `operations.hpp:168-171` (ops 104–107) each carry a trailing `// … (STEALTH hardfork-gated)` comment; the op108 line (`:172`) carries **no comment at all**, and op109's (`:173`) carries one. The consensus gate does not live in that comment — it lives in `hardfork_visitor.hpp` + the per-evaluator `HARDFORK_STEALTH_PASSED` asserts (§9) — so this is cosmetic. Verify the gate at the real sites; do not infer gating from the comment, and do not infer its absence from the missing one.

Specialist must verify: sunset actually disables V1 direct spends, dual-verify is verify-not-parse on both legs, and unknown `ring_scheme` fails closed in validate() AND consensus (a one-layer gate is a consensus bypass).
Closes: BSIP-2 §5 vehicle requirement; `G1-02`-class double-spend reasoning.

## 6. Per-asset tripwire and `freeze_blind`

- `stealth_reconcile_supplies` def `stealth_evaluator.cpp:241`, per-asset halt scope comment `:246-255`, cross-version linkage note `:260-264`; maintenance call `chain/db_maint.cpp:1600` (with activation record `:1599`, parity `:1605`).
- Counter scope is counter-only (`stealth_evaluator.cpp:257-266` — read before extending).
- `freeze_blind`: struct `protocol/stealth.hpp:418`; evaluator `stealth_evaluator.cpp:870-884` (`do_evaluate :873`, `do_apply :884`); committee backup halt, works-while-halted by design; auto tripwire + 3-quorum/24h backup, freeze-wins-ties.
- Diverged assets halt individually (no global flap); resume refused while the diverged set is non-empty.

Specialist/auditor must verify: reconcile runs every maintenance interval without exception paths, halt is per-asset exact (shadow-vs-counter equality or `blind_halted` + fire block), and no zero-fee abuse re-opens the halt (`75438845` justification: caller-auth + vote-record bounded).
Closes: BSIP-2 §1 containment; G1-skill item 5.

## 7. Parity pool and joint-cap (merge-sensitive)

- `stealth_joint_fee_within_cap` (`stealth_evaluator.cpp:1577-1581`): joint `parity + h2 ≤ 7000` on one shared denominator; breach alarm wiring `:1686-1689`; per-call ceiling assert `:1549`.
- `MG-04` open: merged-denominator assert is merge-time work — per-tree checks are green, the single-denominator merge assert is not yet pinned. **Worsened by op109** (§11): `g2_route_market_fee_to_parity` is explicitly OUTSIDE the joint cap (its own doc comment, `:2058-2061`), so the merged denominator now has a third, uncounted contributor.
- Parity maintenance `db_maint.cpp:1605`; G1c lineage `329bfcb4` (parity/wallet economics commit).
- **CORRECTED 2026-10-07 — the previous "stale-pin warning" here was FALSE and caused a Critical.** It claimed `pay_market_fees` at `db_market.cpp:2687-2696` "did not resolve at `e7d978c3` (hits at `:1799,2018,2026`)". Re-verified 2026-10-07 **and again 2026-10-10** (`db_market.cpp` is byte-identical across `ac159a3e..d0a9c2c5`): `db_market.cpp:2687-2696` resolves byte-identically and is the G1c parity-routing block **inside** `pay_market_fees`. `:1799` (fill_order) and `:2026` (settle_order) are the two live call sites; `:2018` is a prose comment that names the function; the definition is `db_market.cpp:2593-2594`. Because this file told auditors to distrust that pin, a *false* claim survived in three baselines — see `legacy-coexistence-invariance-audit/SKILL.md` and `protocol-claim-truthfulness-audit/SKILL.md`, both corrected 2026-10-07. Treat a stale-pin warning in this file as a claim to verify, not as fact. The one defensible lesson: `pay_market_fees` has many references, so cite the definition (`:2593-2594`) and the routing gate, not a call site.

Auditor must verify: the joint cap is enforced on ONE denominator after merge (two per-tree checks are not a joint check), and per-asset parity opt-in holds (default false).
Closes: BSIP-2 §2; round-8 `MG-04`.

## 8. Ring shape, decoy policy, and the RETIRED spent-exclusion rule

**THIS SECTION WAS REWRITTEN 2026-10-10 (task G2-4). The previous text is
recorded in §14b because it is now FALSE, and a false prose pin is how this file
manufactured the Critical in §7.**

- Floor policy: consensus floor is TARGET 16 in all non-halted states; MIN-11 reserved for a future governance-opened emergency, never accepted today (`protocol/stealth.hpp:46-57`: `RING_MIN = 11` `:53`, `RING_TARGET = 16` `:55`, `RING_MAX = 32` `:57`).
- Verifier shape (`stealth_proof.cpp:557-563`: version/size/message/image/sig-size `32+96·n` `:563`), dup reject (`:590-592`), `c0` canonical (`:618-619`).
- Consensus (`stealth_evaluator.cpp`, re-verified at `d0a9c2c5` 2026-10-10): scheme gate `:688-691` (`GRAPHENE_STEALTH_RING_SCHEME_V1`), ring-block context `:661`, membership loop `:703-722`, decoy policy call `:728`, ring verify `:748` with alt-version verify `:769`. Ring-message construction `stealth_proof.cpp:475-507` — see §4b. Pool depth `stealth_evaluator.cpp:154`, pool-floor gate `:612`, blind-halt predicate `:118-123` and its op gate `:301`. Verification-work budget (R9B) `stealth_evaluator.cpp:338-422` with per-block counter `:404-411`.
- Decoy: `DECOY_MIN_AGE_SEC = 86400` (`protocol/stealth.hpp:114`); policy impl `stealth_proof.cpp:1299-1351`, spec `hpp:328-350`.
- Regime numbers: 24h min age, 40% skew cap (25% under 64-output thin regime, ring≥16), halt new blind-ins under 11 outputs, alarm under 22, `get_stealth_effective_set` queryable, `min_mix_blocks` enforced at input selection, sampler conformance tests.

### 8a. SPENT-EXCLUSION IS GONE — do not audit it as if it existed

The old §8 claimed **"spent-exclusion `:611`"** and **"Spent-flag write, which loops
ALL ring members: `:745-752`"**. **Both constructs were DELETED** by the Task-3
nullifier migration, and the deletion is intentional:

- `stealth_evaluator.cpp:717-719` carries the explicit notice: *"DELETED (Task 3,
  nullifier migration). The spent flag is retired; nullified outputs are
  decoy-eligible again so the anonymity set grows with use (Task 6: C-02's
  rationale evaporated; images unlinkable; §8.2)."*
- The old whole-ring `b.spent = true` write loop and its coupled
  `stealth_pool_count_add( …, -1 )` are quoted as deletions at
  `stealth_evaluator.cpp:828-830`. No `b.spent = true` write exists in the G1
  path at all now.
- Ring-path double-spend protection moved entirely to the **nullifier** checks,
  `:736-739`. op108 and op109 add a *representation-proof-before-nullifier*
  ordering on top (§10, §11), which is the live control.

**Auditor must verify the NEW invariant, not the old one:** that every path which
can spend a blinded output either (a) nullifies both version legs, or (b) is an
explicit op whose recovery proof is verified BEFORE the nullifier set is
consulted. Do **not** re-report "spent outputs are excluded from membership" as
a missing control — admitting nullified outputs as decoys is by design and
growing the anonymity set with use is the point. Do **not** report the missing
`b.spent` write as a defect either; it is the deliberate retirement named above.

- Turnstile-path `spent` use at `:930`'s neighbourhood is **also gone**: the
  `"Turnstile old commitment already spent"` assert was deleted by `5616bebd`
  ("turnstile requires the R15 extension on every path"). op106 now refuses an
  extension-less turnstile outright (`:987`) and binds the old commitment's
  opening with a representation proof (`:994-998`). Census row `OV-8t` was
  **RETIRED as GONE** for this reason, not "fixed to green" — see §14c.

Closes: BSIP-2 §4; round-8 `G1-03`.

## 9. Graphene plumbing map

- Objects `2.19 stealth_nullifier` / `2.20 stealth_status` (`chain/types.hpp:51-52`).
- Ops **83–109** union (H2 83–103 + G1 104–107 + G1-R15 108 + G2 109); `STEALTH` hardfork dummy (see `MG-05`: all five HF dates named incl `STEALTH.hf`).
- **Hardfork gating — the check that matters.** Every new op is gated in three places, not two: `hardfork_visitor.hpp:98-103` (the `stealth_ops` typelist, now six members), `proposal_evaluator.cpp:512-516` (the per-op `HARDFORK_STEALTH_PASSED` visitor overloads) and `:324-327` (fee parameters for a pre-fork op are refused), plus a `HARDFORK_STEALTH_PASSED` assert inside each evaluator's `do_evaluate`. A specialist missing any ONE of those four is looking at a bypass.
- `MG-03` open: merged 7-sweep maintenance order unpinned (H2 sweeps + G1 sweeps per-tree; merged order needs an order-assertion test at merge).
- Relay vs consensus: `protocol/stealth.cpp:33-35` (validate() must not verify crypto; divergence forks the network); consensus lives in evaluators, hardfork-gated.
- **Shared pool math (G2).** `liquidity_pool_evaluator.hpp:143-155` declares
  `pool_exchange_amounts` + `calculate_pool_exchange_amounts`;
  `liquidity_pool_evaluator.cpp:38` defines it, and the TRANSPARENT pool swap now
  calls that same helper at `:459` instead of its own inline copy. Both the
  transparent and the blind G2 swap therefore share one rounding rule, which is
  the point (`949cf304`). Auditor must verify the transparent call site really was
  refactored onto the helper and not left as a parallel copy — two copies of a
  rounding rule diverge, and that divergence is consensus.
- Wallet: default-blind + batching/delay behind one tested choke point; `min_mix_blocks` at input selection; RPC/p2p hygiene required with conformance tests (non-consensus, wallet-enforced).

Auditor must verify: every new op/field hardfork-gated in evaluators AND `proposal_operation_hardfork_visitor` AND the proposal fee-parameter ban; extension fields appended never inserted; no floats/`unordered_*`/wall-clock (head-block time only).

## 10. op108 `stealth_explicit_spend` — proof-first double-spend guard (ADDED 2026-10-10)

**This section and §11 existed in the tree with NO specialist pin before this
pass. Read it as first-review material, not as re-confirmation.**

One operation, two legs: `outputs` non-empty ⇒ blind→blind (mirrors op40);
`outputs` empty ⇒ blind→transparent exit (mirrors op41). Ops 39/40/41 keep
their wire format.

- Wire: `protocol/stealth.hpp:568` (struct), members `:581-584`
  (`proof_version` defaults to **V2**, `excess`, `extensions` appended last);
  per-input `stealth_spend_input` carries `(commitment, owner, key_image_v1,
  key_image_v2, recovery_proof)`. Enum slot `operations.hpp:172`; evaluator
  registration `db_init.cpp:166`; hardfork gating per §9.
- Evaluator: section banner `stealth_evaluator.cpp:1794`, `do_evaluate :1797`,
  `do_apply :1958`.
- **The security-relevant ordering** (this is the whole point of the op):
  `stealth_evaluator.cpp:1890-1904` verifies each input's 163 B recovery proof
  FIRST and only THEN consults the nullifier set, both version legs. A
  proof-less op therefore spends nothing and reveals nothing. Verify the order by
  reading, not by pattern-matching the two asserts.
- Recovery verifier `stealth_verify_recovery_proof` def
  `stealth_proof.cpp:704`, domain `"BTS-STEALTH-RECOVERY-v1"` at `:749`,
  declared `stealth_proof.hpp:222`. 163 B layout `A_C(33)||A_1(33)||A_2(33)||s_b(32)||s_v(32)`.
- Conservation kernel `stealth_verify_excess` at `stealth_evaluator.cpp:1951`
  (blind→blind leg; the exit leg's kernel is `verify_sum` in
  `confidential_evaluator.cpp`, not this one).
- Work accounting `stealth_work_for_explicit_spend` `stealth_evaluator.cpp:361`
  (saturating 64-bit; runs BEFORE the input/output-count asserts so a
  proposal-embedded op cannot wrap the budget).
- Live-set effect `stealth_pool_count_add( …, -spent_counted )` at `:2026` —
  this is the counter read by Rule R (§12), so it is consensus-relevant twice.
- **Narrowing (not deleting) of the legacy paths:** op40 and op41 now refuse a
  post-fork blinded output outright — `confidential_evaluator.cpp:147-148` and
  `:230-231`, both `FC_ASSERT( itr->created_block == 0, "Post-fork stealth output
  must be spent via stealth_explicit_spend" )`. Evaluator-only refusal, by
  design. Verify the grandfather predicate is `created_block == 0` on BOTH legs
  and that neither assert was quietly dropped.

Specialist must verify: per-input recovery proofs are checked before any nullifier
consultation on every input and both version legs; the two-leg discrimination
(`outputs` empty vs non-empty) is decided in consensus and not in `validate()`;
the work calculator cannot be starved or overflowed by a proposal-embedded op;
the exit leg cannot be used to smuggle a blind→blind spend.
Closes: BSIP-2 §4(6) (op108's original brief item).

## 11. op109 `stealth_pool_exchange` — blind pool swap, 1-in/1-out (ADDED 2026-10-10)

- Wire: `protocol/stealth.hpp:619` (struct), members `:627-634` (`pool`,
  `amount_to_sell`, `min_to_receive`, `excess`, singular `input`/`output` —
  **no vectors**, and `value_pin` last). Enum slot `operations.hpp:173`;
  registration `db_init.cpp:167`.
- **1-in/1-out is a SOUNDNESS requirement, not a simplification** (struct doc,
  `stealth.hpp:601-604`): with multiple inputs a prover could offset a lie about
  one input's value against another, because the kernel constrains only the total.
  Verify the singular members really are singular on the wire and that
  `validate()`/evaluator do not accept a multi-element form.
- Evaluator: banner `stealth_evaluator.cpp:2048`, `do_evaluate :2112`,
  `do_apply :2331`. Gate order is load-bearing: fork gate → blind count gate →
  work gate (`stealth_work_for_pool_exchange :374`, which charges
  `GRAPHENE_STEALTH_WORK_US_PER_POOL_MATH` = `stealth.hpp:109`) → per-asset
  asset gates on BOTH legs → …
- Six-link soundness chain (spec `§3`): Schnorr value-pin → ring M-leg →
  conservation kernel → evaluator integer check → BP+ range → nullifiers.
  Pins: value-pin **call** `:2220`, value-pin **verifier** `stealth_proof.cpp:857`
  with domain `"G2VP"` at `:890` (declared `stealth_proof.hpp:267`); ring
  message `"G2RM"` at `stealth_proof.cpp:519` with its ULEB128 `pool_varint`
  and fixed-width amounts/pool/fee fields at `:519-546`; link-4 fee-vs-slippage
  equality assert `:2255-2257` (`amount − F == pool_pays − taker_fee`,
  `fc::uint128_t`, never `GRAPHENE_ASSERT`); nullifier absence both legs
  `:2303-2309`, again proof-before-nullifier.
- Pool math is SHARED with the transparent swap — see §9's
  `calculate_pool_exchange_amounts`; the blind swap calls it at
  `stealth_evaluator.cpp:2231`.
- **F-03 IS OPEN HERE (Important, owner decision) — do not report it as
  reviewed.** op109 routes **100% of its market fees** to `parity_pool` via
  `g2_route_market_fee_to_parity` (`stealth_evaluator.cpp:2070`, called twice
  from `do_apply` at `:2364-2365`). **Issuers of BOTH legs receive zero.** This is
  deliberate and untouched by the entropy-economics build; it is recorded as an
  open item, not an endorsement.
- **0%-market-fee pairs cannot self-fund.** op109's route diverts a share of the
  market-fee *increment*; a 0%-fee pair has no increment, so such a pair needs an
  external funder for its parity pool.

Specialist must verify: the value-pin actually binds the pseudo-output to
`amount_to_sell` (a prover must not be able to under- or over-state the sold
amount); the pool instance is bound into `G2RM` so a swap cannot be replayed
against another pool with the same reserves; `fee.asset_id == input asset`;
BP+ is m=1; `validate()` (relay) and the evaluator agree on the gate ORDER, since
a proposal-embedded op bypasses `validate()` entirely.
Closes: BSIP-2 §2 pool-swaps leg; G2 design `2026-10-10-stealth-pool-swaps-design`.

- **Decoy SAMPLER (R11R4, PRV-R11-01, verified at `5a580663`):** decoys are drawn
  from `created_block/100` windows visited in per-attempt shuffled key order with
  the real's window EXCLUDED from the draw (`libraries/chain/stealth_pool_builder.cpp:178-213`,
  shuffle via `shuffle_indices`, RAND_bytes fail-closed; exhaustion throws,
  `:214-215`), then the final ring gets the R9 Fisher-Yates reorder (`:236-252`,
  `38c80dad` logic intact). Pre-R11R4 the ascending round-robin made the emitted
  window set canonical-minus/plus the real's window (rig `window_hit_rate=1.0000`);
  post-fix it is chance-level (~1/16, rig `tooling/r11r4_window_set_rig.py`).
  Residuals (documented, not defects): <16 non-empty windows double decoy windows
  across passes and rings throw when <15 outside decoys exist — availability yields
  to privacy; pilot must hold ≥16 non-empty windows for the uniformity claim.

## 12. Rule R — entropy-economics subsidy retargeting (ADDED 2026-10-10)

Rule R retargets the G1c parity subsidy so it follows the anonymity-set effect
rather than paying every blind op equally. Spec: `docs/superpowers/specs/2026-10-10-stealth-entropy-economics.md`.
Commits: `bed7c6ef` (Rule R) → `529968a1` (live-set model correction) → `42e1987c`
(adversarial/regression suite) → `d0a9c2c5` (burn-denominator fix).

- Classifier `subsidy_rule_visitor` `stealth_evaluator.cpp:1226`; its shared
  `net_rule( n_out, n_in )` helper `:1234`; entry point
  `stealth_subsidy_output_count( op )` `:1306`, contract
  `stealth_evaluator.hpp:261-265` (`subsidy_n = min( n_blind_outputs,
  stealth_subsidy_output_count(op) )`).
- Per-op classification: op39 deposit `:1243` (the only net source of blind
  value, full component), op41 exit → 0, op40/op104 ring spends consume nothing
  so `ΔD_a = +n_out`, op106 turnstile → 1, op108 explicit
  `:1286-1288` (the one non-ring op using real inputs), op109 `:1294` → 1.
  **Everything else returns 0 — the classifier fails closed.**
- The cap is applied only to the DISCOUNT, never to the FLOOR:
  `stealth_parity_gap_core` `:1359`, `const uint32_t n = ( subsidy_n <
  n_blind_outputs ) ? subsidy_n : n_blind_outputs;` at `:1365`. The call site
  (`:2270` for op109, `:2032` for op108) still passes the op's REAL
  `n_blind_outputs`, and `stealth_cost_anchor_core` `:1321` is untouched —
  verified byte-identical by `tooling/verify_stealth_cost_anchor_unchanged.py
  ee67fd75 d0a9c2c5` (exit 0). **Verify that byte-identity claim yourself if you
  re-pin; it is a load-bearing economic guarantee, not a style note.**
- Burn-rate accounting lives in `stealth_parity_maintenance`
  (`stealth_evaluator.cpp:1598`): the `interval_granted`/`interval_routed`
  upserts at `:1513-1515` / `:1567-1569` / `:2102-2104`, the burn ratio at
  `:1620-1638`, and — the reason `d0a9c2c5` exists — op109's router now ALSO
  upserts `interval_routed` (`:2102-2104`). Without it a G2-only-funded asset
  credits the pool but leaves the burn DENOMINATOR at zero, so the first grant
  reads as 100% burn, arms `burn_critical`, and halts every later subsidy.

**F-02 IS OPEN (Important, cryptoeconomic audit) and the mechanism is inert as
specified.** At production fee parameters the subsidy gap is **0** for deposits
and exits; Rule R moves ~1.89 BTS between two set-neutral op classes and
**0 toward the only net source of blind value**. Independently corroborated by
the governance-capture audit: the pot Rule R reallocates is arithmetically empty
at every fee schedule this repository contains. Making it non-inert requires a
fee-schedule change, which is committee-controlled — i.e. **politically
defeasible by design**. Report this as open. Do not report Rule R as delivering
anonymity-set subsidy.

Specialist must verify: the `min()` cap is monotonic (a larger `subsidy_n` is
never a raise); no path lets `subsidy_n` lower a user's fee FLOOR; the anchor is
untouched; the fail-closed `return 0` default covers any op not enumerated; and
the burn-ratio denominator is populated on every funding path that credits the
pool — including op109's, which is the bug `d0a9c2c5` fixed.

## 13. Re-pin procedure (each G1 tip) — revised 2026-10-10

1. `git log --oneline <old>..<new> -- libraries/chain/stealth* libraries/protocol/stealth* libraries/chain/db_maint.cpp` — review every touching commit per SKILL.md Phase 6. Record which of the pinned files the range actually touched: a file the range did **not** touch has unchanged line numbers, which is a fact worth stating (the 2026-10-07 pass recorded that `stealth_evaluator.cpp` was byte-identical across `612b354e..1550ca41`; the 2026-10-10 pass recorded `db_maint.cpp`, `db_market.cpp`, `types.hpp` and both vendor files byte-identical across `ac159a3e..d0a9c2c5`, so their pins needed no shift). **Prove byte-identity with `cmp`, not with a line count** — `stealth_object.hpp` in the 2026-10-10 range has the same line count and DIFFERENT content.
2. Sweep `AUDIT:` markers in `stealth_proof.hpp` / `stealth_proof.cpp` — every marker must still describe the code beside it.
3. Re-verify H/J1/J2 bytes against S2 source + seeds + golden vectors.
4. Re-measure, never hand-shift: for each pin run
   `OLDREV=<pinned-rev> python3 tooling/r13k1_repin_map.py range <relpath> <old_start> <old_end>`
   (`OLDREV` selects the revision the pin came from; export it first with
   `r13h_export_pin_revs.sh` if it is not already in `/tmp/opencode/pin-revs/`).
   The tool diffs the two revisions, maps old lines to new, and marks any line
   whose TEXT changed as `CHANGED` — those must be re-read, not arithmetic-trusted.
   Hand-shifting pins is how this file accumulated 74 dead ones.   **difflib's positional map is not an oracle.** In the 2026-10-10 pass it
   confidently mapped the deleted `"Turnstile old commitment already spent"`
   assert onto unrelated lines of a `replace` hunk. Every `CHANGED` line was
   re-read in the tree, and one resolved to a construct that does not exist —
   which is how `OV-8t` was correctly retired rather than repointed at the
   nearest plausible line.
5. Run the census: `python3 tooling/r13k1_overlay_pin_census.py`. A pin that MISSes
   is obvious; a pin that resolves to the WRONG construct is not, and it is the
   more dangerous failure. So (a) give every row an `expect` regex taken from the
   claim it supports rather than a loose keyword, and (b) read the printed
   matching line for every row before believing the tally. Two live examples from
   the 2026-10-07 pass: `stealth_proof.cpp:477-483` scored OK only because line
   480 contains the literal `msg32`, and `stealth_proof.cpp:1012-1064` scored OK
   only because line 1064 is the `// Decoy policy` section banner.
   **And (c): a green census only certifies the rows it was given.** A subject the
   table does not mention is INVISIBLE to it — that is precisely how §10/§11/§12
   went unpinned through an earlier green pass. When you extend the tree, extend
   the table in the same pass, or name the gap `NOT COVERED` in §0.
6. Bump this header's `audited-tip`; move the old tip to history-only; append a
   §14c change-log row per moved pin. If a construct is gone, mark the claim
   `SUPERSEDED` or `GONE` with evidence — never delete it, never leave it on a
   merely-plausible line.
7. **Treat every stale-pin warning IN THIS FILE as an unverified claim.** This is
   the lesson from the `pay_market_fees` incident in §7: a "stale pin" note here
   told auditors to distrust `db_market.cpp:2687-2696`, the warning was itself
   false, and a refuted claim survived three re-pins and three baselines because
   each re-pin trusted the warning instead of the tree. Re-derive stale-pin
   warnings from `tooling/r13k1_repin_map.py`, never from a prior pin table.
   The 2026-10-10 pass found five STALE **in-tree** citations of the same kind
   (§14c, "STALE IN-TREE CITATIONS") — the tree cites its own line numbers, and
   they drift exactly like this file's do. Read the cited construct, not the
   citation.

## 14. Re-pin change log — 2026-10-07, `e7d978c3`/`612b354e` → `1550ca41`

Machine-checked before/after: lane-1 census went from **74 of 75 overlay pins
failing** to **0 of 123 rows failing** (`tooling/r13h_lane1_pins.py` →
`tooling/r13k1_overlay_pin_census.py`). "Root" = the revision the old pin came
from. Nothing below changes a finding, a severity, or a gate status.

**Totals: 66 pins re-pinned, 3 pins added, 0 marked GONE, 0 newly marked
SUPERSEDED, 31 verified in place and left alone.** Every construct the file
describes still exists at `1550ca41` — that is the one piece of structural good
news in this pass: there was no claim to retire, only lines to move. Two pins in §13 deliberately quote OLD values as worked examples of
pins that resolved to the wrong construct; they are whitelisted in the census
tool's `ANNOTATED` set so nobody "fixes" them back.

| # | Pin (old) | Pin (new) | Root | Why the new range is the right one |
|---|---|---|---|---|
| 1 | `protocol/stealth.hpp:225-226` | `stealth.hpp:263-264` | e7 | `stealth_ring_input`'s `I = b*J1 + v*J2 … domain-separated per proof version` contract — the dual-nullifier statement this section claims. |
| 2 | `stealth.hpp:140-141` | `stealth.hpp:178-179` | e7 | The two `GRAPHENE_STEALTH_J1_SEED` / `J2_SEED` `#define`s. |
| 3 | `chain/stealth_proof.hpp:113-116` | `hpp:141-144` | 612 | The `L_i`/`R_i`/`M_i`/`c_{i+1}` equation block. Old range landed on the F-03 varint comment and scored OK only on the literal `=`. |
| 4 | `chain/stealth_proof.hpp:129-131` | `hpp:153-158` | 612 | Pseudo-outputs are never range-proven; value bound inherited through the M leg (the linkage/soundness note). |
| 5 | `stealth_evaluator.cpp:417` | `:505` | e7 | The `key image is version-domain (I = b*J1_v + v*J2_v …)` comment line itself. |
| 6 | `stealth_evaluator.cpp:416-424` | `:503-512` | e7 | Task-5 dual-verify comment block plus the `alt_version` mapping it introduces. |
| 7 | `stealth_evaluator.cpp:536-539` | `:628-631` | e7 | The two nullifier-uniqueness asserts (`Key image already spent`, `Alternate key image already spent`). |
| 8 | `stealth_evaluator.cpp:614` | `:706` | e7 | `do_apply` nullifier create for the op version. |
| 9 | `stealth_evaluator.cpp:619` | `:711` | e7 | `do_apply` nullifier create for the alt version. |
| 10 | `stealth_evaluator.cpp:96-99` | `:100-106` | e7 | `nullifier_key()` def + its AUDIT note + the `sha256(image)` body. |
| 11 | `stealth_proof.cpp:261` | `:263-276` | e7 | `J_cache()` def plus the `[J1v1, J2v1, J1v2, J2v2]` seed derivation. |
| 12 | `stealth_proof.cpp:249` | `:250-261` | e7 | S2 H-constant comment + `H_bytes()` body. |
| 13 | `stealth_proof.cpp:361-363` | `:363-366` | e7 | `stealth_H_point()` def + `return H_bytes();`. |
| 14 | `stealth_proof.cpp:332` | `:334` | e7 | `c.H = decode_or_null( H_bytes();` — where the S2 H enters the decoded-point cache. |
| 15 | `stealth_proof.cpp:249-259` | `:250-261` | e7 | Same range as #12: the code that WINS over the vendor README's try-and-increment prose. |
| 16 | `stealth_proof.cpp:276-298` | `:278-300` | e7 | `BPG_cache()` / `BPH_cache()`, the `Gi`/`Hi` generators. Old range was MIS-PINNED at *every* revision examined (its regex never matched the file) — it had drifted unnoticed since `e7d978c3`. |
| 17 | `stealth_proof.cpp:557-590` | `:581-614` | e7 | The batched L/R/M multiexp block, inclusive of all three `EC_POINTs_mul` calls. |
| 18 | `stealth_proof.cpp:570,580,590` | `:594,604,614` | e7 | The three `EC_POINTs_mul` calls, respectively: old `:570`→new `:594` (L leg), old `:580`→new `:604` (R leg), old `:590`→new `:614` (M leg). |
| 19 | `stealth_proof.cpp:547-549` | `:571-573` | e7 | Per-member `(rb,rv,rd)` strict scalar parse. |
| 20 | `stealth_proof.cpp:538` | `:562-563` | e7 | `c0` strict parse + the `"c0 not canonical"` rejection (single line → the parse+rejection pair, because the claim names both). |
| 21 | `stealth_proof.cpp:110` | `:110-117` | e7 | `scalar_parse_strict` AUDIT note + def + body; old pin sat on the AUDIT comment only. |
| 22 | `chain/stealth_proof.hpp:125-135` | `hpp:160-170` | e7 | `AUDIT: this is an adaptation, NOT verbatim CLSAG` block. |
| 23 | `chain/stealth_proof.hpp:174-180` | `hpp:221-227` | e7 | AUDIT items (1)/(2)/(3): transcript labels differ from S1, weight fixed 1, audit headline. |
| 24 | `stealth_proof.cpp:193-207` | `:195-209` | e7 | `hash_to_point` doc block incl. the principal-root AUDIT; old range started 2 lines early. |
| 25 | `stealth_proof.cpp:407` | `:418` | e7 | `std::string pre = "G1R2";` — the literal ring-challenge domain. |
| 26 | `stealth_proof.cpp:454` | `:465` | e7 | `std::string pre = "G1RM";` — the literal ring-message domain. |
| 27 | `stealth_proof.cpp:402-420` | `:413-431` | e7 | `stealth_challenge` def + body. |
| 28 | `stealth_proof.cpp:446-465` | `:457-489` | e7 | `stealth_ring_message` def + body (now including the F-03 count prefix). |
| 29 | `stealth_proof.cpp:471` | `:495` | e7 | `bool stealth_verify_ring(` definition line. |
| 30 | `stealth_evaluator.cpp:548` | `:640` | e7 | Op-version `stealth_verify_ring` call. |
| 31 | `stealth_evaluator.cpp:569` | `:661` | e7 | Alt-version `stealth_verify_ring` call. |
| 32 | `stealth_proof.cpp:426` | `:433` | 612 | `fc::sha256 stealth_bp_hash(` definition line. |
| 33 | `chain/stealth_proof.hpp:86-87` | `hpp:87-90` | 612 | "The BP+ path (R9C) honors this chaining literally…" — the 4-line chaining note. |
| 34 | `chain/stealth_proof.hpp:199-200` | `hpp:224-226` | 612 | Chaining AUDIT inside the BP+ verifier doc (`state_0 = z`, rounds under `G1BPr`, final under `G1BPf`, pre-R9C rejection). |
| 35 | `stealth_proof.cpp:818` | `:838` | 612 | The only surviving mention of the removed `G1BPe` round/final label reuse. |
| 36 | `chain/stealth_proof.hpp:200` | `hpp:226` | 612 | The only surviving `G1BPe` mention in the header ("flat-`G1BPe` proofs reject"). |
| 37 | `stealth_proof.cpp:411` | `:418` | 612 | `std::string pre = "G1R2";`. |
| 38 | `stealth_proof.cpp:406` | `:413` | 612 | `fc::sha256 stealth_challenge(` definition line. |
| 39 | `stealth_proof.cpp:794` | `:793-799` | 612 | The `load_scalar` lambda: strict canonical AND nonzero, both the check and its rationale. |
| — | *(added)* | `stealth_proof.cpp:474-479` | new | The F-03 fix in code: `LE64(n)` / `LE64(m)` count prefix. Previously §4b asserted F-03 CLOSED with no code pin for the fix itself. |
| — | *(added)* | `chain/stealth_proof.hpp:105-106` | new | The spec's rendering of the post-fix preimage (encoding table at `:108-121`). |
| — | *(added)* | `protocol/stealth.cpp:74-75` | new | The relay-side `ring_scheme` assert, so §5's "verify both layers" claim has a pin on each layer. |
| 40 | `protocol/stealth.hpp:123` | `stealth.hpp:161` | e7 | `GRAPHENE_STEALTH_V1_LIFETIME_SEC = 31536000`. |
| 41 | `stealth_evaluator.cpp:416-424` | `:503-512` | e7 | Same construct as #6, cited from §5. |
| 42 | `stealth_evaluator.cpp:484-487` | `:576-579` | e7 | The `ring_scheme` consensus assert plus its two-line Task-6 comment. |
| 43 | `protocol/stealth.cpp:39` | `stealth.cpp:44` | e7 | `void stealth_transfer_operation::validate()const`. Old line 39 is mid-comment; it scored OK only on the literal `validate` inside "REJECTS what the verifier ACCEPTS". |
| 44 | `chain/db_init.cpp:162` | `db_init.cpp:162-165` | e7 | Widened: ops 104–107 register four evaluators, not one. |
| 45 | `stealth_evaluator.cpp:227` | `:234` | e7 | `void stealth_reconcile_supplies(` definition line. |
| 46 | `stealth_evaluator.cpp:236` | `:239-248` | e7 | Widened from one mid-comment line to the whole per-asset halt-scope comment. |
| 47 | `stealth_evaluator.cpp:247` | `:253-257` | e7 | The SCOPE carve-out naming the cross-version double-spend (GC-07). |
| 48 | `stealth_evaluator.cpp:243-252` | `:250-259` | e7 | The whole SCOPE (counter-only) paragraph §6 tells auditors to read before extending. |
| 49 | `protocol/stealth.hpp:352` | `stealth.hpp:391` | e7 | `struct freeze_blind_operation : public base_operation`. |
| 50 | `stealth_evaluator.cpp:692-706` | `:799-813` | e7 | The `freeze_blind` section banner through the `do_apply` signature. |
| 51 | `stealth_evaluator.cpp:695` | `:802` | e7 | `freeze_blind_evaluator::do_evaluate`. |
| 52 | `stealth_evaluator.cpp:706` | `:813` | e7 | `freeze_blind_evaluator::do_apply`. |
| 53 | `stealth_evaluator.cpp:1189-1192` | `:1309-1313` | e7 | `stealth_joint_fee_within_cap` def + the single shared-denominator compare. |
| 54 | `stealth_evaluator.cpp:1300` | `:1418-1421` | e7 | Joint-cap breach alarm wiring (comment + `joint_cap_breach` assignment). |
| 55 | `stealth_evaluator.cpp:1161` | `:1281` | e7 | `FC_ASSERT( share <= cap_bound, "Parity fee share exceeds joint ceiling" );`. |
| 56 | `protocol/stealth.hpp:48-57` | `stealth.hpp:46-57` | e7 | Widened 2 lines up so the range includes the "It is NOT the consensus floor" rationale alongside `RING_MIN`/`TARGET`/`MAX` at `:53/:55/:57`. |
| 57 | `stealth_proof.cpp:477-483` | `:501-507` | e7 | The five verifier shape gates (version, ring size, message, image, sig size `32+96·n`). Old range scored OK at the new tip only because `:480` contains the literal `msg32` — a resolved pin pointing at the wrong code. |
| 58 | `stealth_proof.cpp:510-512` | `:534-536` | e7 | The duplicate-ring-member reject. |
| 59 | `stealth_proof.cpp:538` | `:562-563` | e7 | `c0` strict parse + rejection (cited from §8; same construct as #20). |
| 60 | `stealth_evaluator.cpp:593-611` | `:591-614` | e7 | Widened to cover the membership loop *and* its "every member must be a live blinded balance" comment; inner pins `:604`/`:611` unchanged. |
| 61 | `stealth_evaluator.cpp:578` | `:576-579` | e7 | Widened to the full two-line `ring_scheme` assert. |
| 62 | `stealth_proof.cpp:1012-1064` | `:1067-1119` | e7 | `stealth_check_decoy_policy` implementation. Old range scored OK only because `:1064` is the `// Decoy policy` section banner — a resolved pin pointing at the wrong code. |
| 63 | `chain/stealth_proof.hpp:194-216` | `hpp:246-268` | e7 | Decoy-policy spec block + declaration. |
| 64 | `protocol/stealth.hpp:70` | `stealth.hpp:108` | e7 | `GRAPHENE_STEALTH_DECOY_MIN_AGE_SEC = 86400`. |
| 65 | `stealth_evaluator.cpp:114-118` | `:114-119` | e7 | Widened 1 line to include the closing brace of `stealth_blind_halted`. |
| 66 | `stealth/vendor/…/main_impl.h:32` | `main_impl.h:32-35` | e7 | Widened to cover the full `secp256k1_generator_const_h` initializer — a specialist checking H bytes against S2 needs both coordinate lines, not just the declaration. |
| — | `stealth.cpp:33-35`; `db_maint.cpp:1599,1600,1605`; `operations.hpp:168-171`; `db_init.cpp:197-198`; `types.hpp:51-52`; `confidential_evaluator.cpp:135,208`; `db_market.cpp:1799,2018,2026,2593-2594,2687-2696`; `stealth_proof.cpp:134,403-412,563,575,647,680,709,744,759,832-846,860,863,876,886`; `chain/stealth_proof.hpp:43-56,60-63,75-81`; `stealth/vendor/README.md:98-104` | unchanged | — | 31 pins verified in place at `1550ca41`; no shift needed. |

### Label corrections (citation text, not findings)

- §4 called `:563` (`"c0 not canonical"`) and `:575` (`"responses not canonical"`) the
  **BP-path** rejections. Both live inside `stealth_verify_ring`, i.e. the RING path; the
  BP+ path's scalar gate is the `load_scalar` lambda at `:793-799` (rejections `"bad r1"` /
  `"bad s1"` / `"bad d1"` at `:800-802`). §4 now says RING-path. No severity or status
  changes; the labels were wrong, the lines were right.
- §7's `pay_market_fees` note called `:1799`/`:2018`/`:2026` "CALL SITES". `:1799` and
  `:2026` are; `:2018` is a prose comment naming the function. §7 now distinguishes them.
- No pin in this file is `SUPERSEDED` or `GONE` as of `1550ca41`: every construct the file
  describes still exists at the tip. The one pre-existing `SUPERSEDED` marker (the pre-R9C
  flat `G1BPe` transcript, §4 bullet 1) stays superseded — its subject is rejected by design,
  and its pins now point at the two surviving comments (`:838`, `hpp:226`).

### F-03 status (unchanged, restated for the record)

**F-03 remains CLOSED at `1550ca41`.** Fix `3ea02398`, re-pinned here as
`stealth_proof.cpp:474-479` + `stealth_proof.hpp:105-106`. Re-report it as open only if
`ring.size()` / `outputs.size()` ever leave the `G1RM` preimage again.

## 14b. Re-pin change log — 2026-10-10, `1550ca41` → worktree `4f56b170` + uncommitted Task-3 WIP (Task G2-0)

Machine-checked before/after: census went from **71 OK / 52 MISS** at the dirty
worktree to **121 OK / 0 MISS / 0 OOR / 0 NOFILE over 121 rows**
(`python3 /workspace/tooling/r13k1_overlay_pin_census.py`, rc=0), plus 63 prose
pins in §1–§10 all in-bounds. Basis is the WORKTREE (committed tip `4f56b170`
plus uncommitted nullifier-migration work in 4 files) — NOT a clean commit; if
the dirty files move, pins drift. Range `1550ca41..4f56b170` touched
`stealth_evaluator.cpp` (+361/-net, R15 verifier+gate+fixes, op108 evaluator),
`stealth_proof.cpp` (+105, R15 recovery verifier `c8eb1b44` + c=0/uniqueness
`9fd0fed7`), `stealth_proof.hpp` (+14/-), `protocol/stealth.hpp` (+100, op108
struct), `protocol/stealth.cpp` (+75, op108 validate), `operations.hpp` (tag
108), `db_init.cpp` (+1, op108 registration). Untouched by the range:
`db_maint.cpp`, `db_market.cpp`, `confidential_evaluator.cpp` (committed —
dirty only), all vendor files. Line-shift arithmetic mechanically derived via
`OLDREV=1550ca41 python3 /workspace/tooling/r13k1_repin_map.py range <rel>
<old_start> <old_end>` against a fresh `/tmp/opencode/pin-revs/1550ca41/`
snapshot (`1550ca41` added to the tool's `OLD_REVS`); every `CHANGED` line
re-read in the tree, never arithmetic-trusted. Nothing below changes a
finding, a severity, or a gate status — except the two retirements, which
record in-flight consensus behavior change (nullifier migration), not audit
verdict change.

Causes (see rows): (A) R15 recovery verifier inserted ~105 lines above the BP+
block (`stealth_proof.cpp` +105/+8). (B) op108 evaluator + `T_r15_nullify`
wiring inserted lines through `stealth_evaluator.cpp` (+16 above §1 block).
(C) Nullifier-migration WIP (dirty) DELETED ~43 net lines in the
membership/do_apply region (spent-flag retirement) and reworded the op40/op41
guards. (D) `stealth_proof.hpp` +8 net (recovery extension types).

| # | Pin (old, @1550ca41) | Pin (new, worktree) | Cause | Why the new range is the right one |
|---|---|---|---|---|
| 1 | `stealth_evaluator.cpp:505` | `:521` | B | Same version-domain comment line (`I = b*J1_v + v*J2_v …`), content-identical. |
| 2 | `stealth_evaluator.cpp:503-512` | `:519-528` | B | Same dual-verify block + `alt_version` mapping, content-identical. |
| 3 | `stealth_evaluator.cpp:628-631` | `:640-643` | B/C | Same two nullifier-uniqueness asserts (`Key image already spent` / `Alternate …`), content-identical. |
| 4 | `stealth_evaluator.cpp:706` | `:718` | B/C | Same op-version nullifier create (`n.key_image = nullifier_key( in.key_image )`), content-identical. |
| 5 | `stealth_evaluator.cpp:711` | `:723` | B/C | Same alt-version nullifier create, content-identical. |
| 6 | `stealth_evaluator.cpp:640` | `:652` | C | Same op-version `stealth_verify_ring` call, content-identical. |
| 7 | `stealth_evaluator.cpp:661` | `:673` | C | Same alt-version `stealth_verify_ring` call, content-identical. |
| 8 | `stealth_evaluator.cpp:543` | `:559` | B | Same transfer-V2 `stealth_verify_bp_plus` call, content-identical. |
| 9 | `stealth_evaluator.cpp:969` | `:992` | B/C | Same turnstile-V2 `stealth_verify_bp_plus` call, content-identical. |
| 10 | `stealth_evaluator.cpp:680` | `:692` | C | Same spend `stealth_verify_excess` call, content-identical. |
| 11 | `stealth_evaluator.cpp:948` | `:971` | B/C | Same turnstile `stealth_verify_excess` call, content-identical. |
| 12 | `stealth_evaluator.cpp:576-579` (§5 + §8) | `:592-595` | B/C | Same two-line `ring_scheme` consensus assert, content-identical. |
| 13 | `stealth_evaluator.cpp:234` | `:237` | B | Same `stealth_reconcile_supplies` def line, content-identical. |
| 14 | `stealth_evaluator.cpp:239-248` | `:242-251` | B | Same per-asset halt-scope comment, content-identical. |
| 15 | `stealth_evaluator.cpp:253-257` | `:256-260` | B | Same GC-07 carve-out note, content-identical. |
| 16 | `stealth_evaluator.cpp:250-259` | `:253-262` | B | Same SCOPE paragraph, content-identical. |
| 17 | `stealth_evaluator.cpp:799-813` | `:772-786` | C | Same `freeze_blind` banner → `do_apply` signature; net −27 lines above (spent-flag retirement deletions), content-identical per line. |
| 18 | `stealth_evaluator.cpp:802` | `:775` | C | Same `freeze_blind_evaluator::do_evaluate` line. |
| 19 | `stealth_evaluator.cpp:813` | `:786` | C | Same `freeze_blind_evaluator::do_apply` line. |
| 20 | `stealth_evaluator.cpp:1309-1313` | `:1343-1347` | B/C | Same `stealth_joint_fee_within_cap` def; body now takes `(parity_share_bps, h2_diversion_bps)` but the single-shared-denominator compare (`joint <= …JOINT_FEE_CAP_BPS`) is intact. |
| 21 | `stealth_evaluator.cpp:1418-1421` | `:1452-1455` | B/C | Same joint-cap breach alarm wiring, content-identical. |
| 22 | `stealth_evaluator.cpp:1281` | `:1315` | B/C | Same ceiling assert (`Parity fee share exceeds joint ceiling`), content-identical. |
| 23 | `stealth_evaluator.cpp:549` | `:565` | B/C | Same ring-block context comment, content-identical. |
| 24 | `stealth_evaluator.cpp:591-614` | `:607-626` | C | Membership loop + its comment; range re-cut (not just shifted): old range's first 16 lines now cover the duplicate-member loop — a green-but-wrong-construct pin, caught by reading the matched line. New range is comment `:607-610` + loop `:615-626`. |
| 25 | `stealth_evaluator.cpp:604` | `:620` | C | Same grandfather-exclusion assert, content-identical. |
| 26 | `stealth_evaluator.cpp:620` | `:632` | C | Same `stealth_check_decoy_policy` call, content-identical. |
| 27 | `stealth_evaluator.cpp:147` | `:150` | B | Same `stealth_pool_depth` def line, content-identical. |
| 28 | `stealth_evaluator.cpp:500` | `:516` | B | Same pool-floor gate assert, content-identical. |
| 29 | `stealth_evaluator.cpp:294` | `:297` | B | Same blind-halt op gate, content-identical. |
| 30 | `stealth_evaluator.cpp:331-372` | `:334-388` | B | Same work-budget block; widened +16 for the inserted op108 `stealth_work_for_explicit_spend` (`:357-369`, new function — NOT pinned here, G2-4/overlay-op108 gap). |
| 31 | `stealth_evaluator.cpp:354-361` | `:370-377` | B | Same `stealth_blind_work_this_block` def, content-identical. |
| 32 | `stealth_evaluator.cpp:885` | `:886` | C | Same turnstile spent-refusal assert (no-extension branch), content-identical; difflib's positional map said `:877` (CHANGED) — resolved by reading: the true content sits at `:886`. |
| 33 | `stealth_proof.cpp:860` | `:965` | A | Same `G1BPy` hash line, content-identical. |
| 34 | `stealth_proof.cpp:863` | `:968` | A | Same `G1BPz` hash line, content-identical. |
| 35 | `stealth_proof.cpp:876` | `:981` | A | Same `G1BPr` hash line, content-identical. |
| 36 | `stealth_proof.cpp:886` | `:991` | A | Same `G1BPf` hash line, content-identical. |
| 37 | `stealth_proof.cpp:832-846` | `:937-951` | A | Same CHAINED-transcript comment block, content-identical. |
| 38 | `stealth_proof.cpp:838` | `:943` | A | Same surviving-`G1BPe`-mention line, content-identical. |
| 39 | `stealth_proof.cpp:709` | `:814` | A | Same `stealth_bp_proof_size` def line, content-identical. |
| 40 | `stealth_proof.cpp:759` | `:864` | A | Same exact-blob assert, content-identical. |
| 41 | `stealth_proof.cpp:793-799` | `:898-904` | A | Same `load_scalar` lambda, content-identical. |
| 42 | `stealth_proof.cpp:744` | `:849` | A | Same `stealth_verify_bp_plus` def line, content-identical. |
| 43 | `stealth_proof.cpp:647` | `:752` | A | Same `stealth_verify_excess` def line, content-identical. |
| 44 | `stealth_proof.cpp:1067-1119` | `:1172-1224` | A | Same `stealth_check_decoy_policy` body, content-identical. |
| 45 | `stealth_proof.hpp:221-227` | `:229-235` | D | Same AUDIT (1)/(2)/(3) block, content-identical. |
| 46 | `stealth_proof.hpp:224-226` | `:232-234` | D | Same chaining AUDIT, content-identical. |
| 47 | `stealth_proof.hpp:226` | `:234` | D | Same surviving-`G1BPe` line, content-identical. |
| 48 | `stealth_proof.hpp:246-268` | `:254-276` | D | Same decoy-policy spec block + declaration, content-identical. |
| 49 | `confidential_evaluator.cpp:135` | `:137-138` | C | Guard REWORDED, same claim: op40 cross-path refusal is now `FC_ASSERT( itr->created_block == 0, "Post-fork stealth output must be spent via stealth_explicit_spend" )`. Census expect regex updated to the new refusal text (old text has zero hits tree-wide). |
| 50 | `confidential_evaluator.cpp:208` | `:214-215` | C | Same rewording for the op41 guard; same regex update. |

RETIRED (rows deleted from the census table, 123 → 121 rows — NOT renumbered, no
plausible line exists):

- OV-8i (`stealth_evaluator.cpp:611`, `FC_ASSERT( !itr->spent, "Ring member already
  spent …" )`): assert DELETED by the dirty nullifier-migration WIP; `:621-623`
  now carry an explicit `// DELETED (Task 3, nullifier migration)` notice ("The
  spent flag is retired; nullified outputs are decoy-eligible again"). Ring-path
  double-spend protection moved to the nullifier checks (OV-1h, `:640-643`).
  **RESOLVED 2026-10-10: the §8 prose that described this construct is REWRITTEN
  as §8a, which states the retirement rather than the old invariant.**
- OV-8w (`stealth_evaluator.cpp:745-752`, whole-ring `b.spent = true` write
  loop): loop DELETED; the deletion is quoted in comments at `:728-734`
  (`// DELETED: for every ring member: … b.spent = true …`). The only remaining
  `b.spent = true`-shaped text is inside that deletion quote — pinning it would
  be the wrong-construct failure §10 exists to prevent. **RESOLVED 2026-10-10:
  rewritten as §8a; the deletion notice is now pinned at `:770-772`.**

STILL STALE at the time of this pass (see §14c, which resolved them):

- §0 header claimed audited tip `1550ca41` and §§5/9 enumerated ops 104–107
  while the tree carried op108. **RESOLVED 2026-10-10 — see §14c.**
- F-03 note above re-verified in-bounds and content-correct (body scan confirms
  in-bounds; LE64 prefix lines read identical).
- AUDIT-marker sweep: all markers re-read; each still describes adjacent code. New
  domain `"BTS-STEALTH-RECOVERY-v1"` (`stealth_proof.cpp:693` at that worktree)
  does not collide with the `:403` ONLY-ring-challenge-domain claim. Recovery
  verifier reuses `J_cache()` (same seed derivation).
- H/J1/J2: seeds, `H_bytes()`, `J_cache()`, `stealth_H_point()`, S2 vendor bytes
  all content-identical since `1550ca41`.

## 14c. Re-pin change log — 2026-10-10, `ac159a3e` → `d0a9c2c5` (task G2-4 / EE-4)

Machine-checked before/after: census went from **32 OK / 89 MISS over 121 rows**
at `d0a9c2c5` (the §14b table simply no longer pointed at anything real) to
**174 OK / 0 MISS / 0 OOR / 0 NOFILE over 174 rows**
(`python3 /workspace/tooling/r13k1_overlay_pin_census.py`, rc=0), plus 95 prose
pins in §1–§13 all in-bounds.

Root `ac159a3e` — see §0: it is the commit that captured the previously-uncommitted
WIP the §14b table was pinned against, proven by that table replaying 121 OK / 0
MISS at `ac159a3e`. Range `ac159a3e..d0a9c2c5` touched
`stealth_evaluator.cpp` (+584 net), `stealth_proof.cpp` (+127),
`stealth_proof.hpp` (+74), `protocol/stealth.hpp` (+74), `protocol/stealth.cpp`
(+45), `confidential_evaluator.cpp` (+18), `db_init.cpp` (+1),
`operations.hpp` (+1), `liquidity_pool_evaluator.{cpp,hpp}` (shared exchange
math), plus new files (`stealth_prover.cpp/.hpp`,
`stealth_pool_builder.cpp/.hpp`). **Byte-identical across the range (verified
with `cmp`, so their pins needed no shift): `db_maint.cpp`, `db_market.cpp`,
`chain/types.hpp`, `protocol/types.hpp`, `stealth/vendor/README.md`,
`main_impl.h`.** `chain/include/graphene/chain/stealth_object.hpp` has the SAME
line count and DIFFERENT content — a reminder that line-count comparison is not
identity (§13 step 1).

Causes: (A) op108 evaluator + nullifier migration inserted ~584 lines above the
§7/§8 tail of `stealth_evaluator.cpp`; (B) the R15 recovery verifier and the G2
value-pin verifier inserted 127 lines into `stealth_proof.cpp`; (C) new op structs
appended to `protocol/stealth.hpp`; (D) new ops 108/109 to `operations.hpp` and
`db_init.cpp`.

**Totals: 121 rows carried forward (109 shifted, 12 verified in place),
1 row RETIRED as GONE (OV-8t), **53 rows ADDED** — 50 over three
previously-uncovered subjects (§10 op108, §11 op109, §12 Rule R), 2 pinning the
§8a retirement notices, 1 pinning the m=1 consensus gate that §4 asserted but
never row-checked — 8 prose claims rewritten, 1 section restructured. No finding, severity, or gate status changed by this pass.**

| # | Change | Detail |
|---|---|---|
| 1 | RETIRED `OV-8t` (`stealth_evaluator.cpp:886`, `"Turnstile old commitment already spent"`) — **GONE, not repointed.** | Deleted by `5616bebd` ("turnstile requires the R15 extension on every path"). difflib's positional map happily reported a new line inside a `replace` hunk; re-reading the tree showed the assert no longer exists. op106 now refuses an extension-less turnstile at `:929` and binds the old commitment with the R15 representation proof at `:936-940`. §8a records the retirement with evidence. |
| 2 | §8 prose **rewritten** into §8 + §8a. | The old "spent-exclusion `:611`" and "Spent-flag write … `:745-752`" claims were FALSE for two pins and have been replaced by the retirement record (`:659-661`, `:770-772`) plus the live control (nullifier uniqueness `:678-681`). §14b had flagged both as "STALE — G2-4 must rewrite"; that obligation is discharged here. |
| 3 | §0 rewritten. | Audited tip `1550ca41` → `d0a9c2c5`; census 121 → 174 rows; the review-gap warning added; SKILL.md's pin-currency claim re-scoped to the tip named in §0. |
| 4 | §5 op enumeration 104–107 → **104–109**; `operations.hpp:168-171` → `:168-173`; `db_init.cpp:162-165` → `:162-167`; `db_init.cpp:197-198` → `:199-200`. | The index pin previously resolved only on its SECOND line (`stealth_nullifier_index`) and would have gone green-but-wrong; re-cut to `:199-200` so both indices are covered by the expect regex. |
| 5 | §9 op union 83–107 → **83–109**; hardfork gating now pinned at FOUR sites (`hardfork_visitor.hpp:98-103`, `proposal_evaluator.cpp:512-516`, `:324-327`, per-evaluator asserts). | §9's own instruction was "gated in evaluators AND `proposal_operation_hardfork_visitor`"; a third site (the proposal fee-parameter ban) existed unpinned. |
| 6 | §7 `MG-04` note amended. | op109's `g2_route_market_fee_to_parity` is explicitly outside the joint cap (`stealth_evaluator.cpp:2000-2003`), so the merged denominator now has a third uncounted contributor. The gate's status is unchanged (still open). |
| 7 | §4 bullet 1 m=1 gate pin `:757` → `:989` (rationale `:985-988`). | Re-measured against `d0a9c2c5`; the BP+ gate moved with the R15/G2 verifier insertions. **A first guess of `:801` was caught by reading the printed line — `:801` is `return true;` inside the recovery verifier.** Recorded because it is the §13 step 5 failure mode in miniature, one pass after this file was written. |
| 8 | §4b gained the op109 `G2RM` cross-reference. | Two `*-message` domains now carry the same LE64 count-prefix discipline; auditing one and not the other is the gap. |

### STALE IN-TREE CITATIONS found at `d0a9c2c5` (recorded, NOT fixed — out of EE-4's scope)

The tree cites its own line numbers in comments, and they drift exactly like this
file's pins do. Five were found during this pass, all in code that a specialist
would follow. **Left in place with evidence** — EE-4 does not own
`worktrees/`. Correct them in a worktree pass; do not "fix" them here.

| Site | Cites | Actually at | Subject |
|---|---|---|---|
| `protocol/stealth.hpp:604-607` (op109 struct doc) | `op108 :1697-1712` | `stealth_evaluator.cpp:1832-1846` | "proof-before-nullifier order, op108 … discipline". Off by ~135 lines. |
| `stealth_evaluator.cpp:2245` (op109 nullifier step 11) | `op108 :1697-1712` | `:1832-1846` | Same stale citation, second copy. |
| `stealth_evaluator.cpp:1158` (Rule R model note) | `stealth_evaluator.cpp:1924` | `:1968` | "`-spent_counted` … (op108)". Off by 44. |
| `stealth_evaluator.cpp:1157` (same note) | `confidential_evaluator.cpp:185` | `:187` | "`-spent_counted` at … (op41)". Off by 2. |
| `stealth_evaluator.cpp:1150-1153` (same note) | `:769-771` and `op104 :791` | `:770-772` and `:792` | The DELETED-counter notice and the op104 `pool_count_add` witness. Off by 1. |
| `stealth_evaluator.cpp:376` (`stealth_work_for_pool_exchange`) | `stealth_work_for_stealth_transfer (:334-347)` | `:338-351` | Off by 4. |
| `stealth_proof.cpp:521` (`G2RM`) | `stealth_ring_message :467-479` | `:475-507` | Off by 8. |

None of these change a finding: every one is a prose pointer to a construct that
exists and is pinned correctly in §1–§12 above. They are listed because §13
step 7 says to re-derive stale citations from the tree, and because a stale
citation that survived six commits is exactly the `pay_market_fees` failure mode.

### F-03 status at `d0a9c2c5`

**F-03 (mimo-2.6 ring-message re-partitioning) remains CLOSED** — pins
`stealth_proof.cpp:492-497` + `stealth_proof.hpp:105-106`. **Separately, a
DIFFERENT and still-OPEN item also carries the label F-03**: op109 routes 100%
of its market fees to `parity_pool` and pays neither issuer anything (§11). Two
items, one label — read the section, not the tag.

## 14d. Re-pin change log — 2026-10-15, `a2dd340d` → `9d4b5d85` (round-9 minor drift)

Machine-checked before/after: census went from **70 OK / 104 MISS over 174
rows** at `9d4b5d85` (two commits of drift past the §14c-era table) to
**174 OK / 0 MISS / 0 OOR / 0 NOFILE over 174 rows**
(`python3 /workspace/tooling/r13k1_overlay_pin_census.py`, rc=0). Entropy
citations went from 11 OK / 28 MISS to **39 OK / 0 MISS, RESULT: PASS**
(`python3 /workspace/tooling/verify_entropy_economics_citations.py`).
Cost anchor **byte-identical** across the range (sha256
`015eee29c5ac7e02b5ab7de329227445b9f72c13523871c92c14aacc1b56d9df`,
1932 B, `verify_stealth_cost_anchor_unchanged.py a2dd340d` rc=0).

Range `a2dd340d..9d4b5d85` is exactly two commits: `27a65111` (round-9 code
minors: AGI-04 helper fold +15 lines at `stealth_evaluator.cpp:117-131`,
NC-14 two-kernel charge comment + charge, NC-16 zero-scalar refusal,
CI-05 op83/op99 hardfork tests in `htlc_evaluator.cpp`/`oracle_object.hpp`/
`g1_stealth_tests.cpp` +225) and `9d4b5d85` (comment-only, six files).
No verifier logic changed except the NC-16 refusal (fail-closed tightening)
and the AGI-04 fold (identical semantics, unit-tested).

**Totals: 132 rows renumbered (130 via `tooling/apply_pin_rebase.py`
git-diff line map, verified by expect-regex inside each mapped window;
2 rows OV-1g/OV-5b hand-mapped 615-624 → 632-639 — the old window sat
inside a `replace` hunk where the V1↔V2 ternary became the helper call, so
the map had no entry; the comment block is intact at the new lines and was
read, not arithmetically assumed). 0 rows retired, 0 rows added, 0 regexes /
paths / claims changed. No finding, severity, or gate status changed by
this pass.**

Out of scope (recorded, not fixed): overlay body prose pins (§1–§13) and
spec prose citations still read the `a2dd340d` numbers — the machine tables
above are the current record (same precedent as §14c-era R9-REPIN
`891d533`). MG-04 (third uncounted contributor) and the SUP-R-04
`unspent_blind_count` semantics are unchanged and still open.

## 14e. Census hardening — 2026-10-16, round-17, tip `5a580663`

Machine-checked before/after: **180 → 187 rows; `{OK: 187, MISS: 0, OOR: 0,
NOFILE: 0}`, rc=0** (`python3 /workspace/tooling/r13k1_overlay_pin_census.py`),
six coverage floors green, 97 body-prose pins in-bounds (2
annotated-historical). No existing row needed renumbering at `5a580663`.
No finding, severity, or gate status changed by this pass.

- **CLS-FINAL-02** (round-17): the §0 m-guard claim was refuted at round-16 but
  its correction was prose-only, so the next reader could still reintroduce or
  mis-cite it. Three rows added so the ordering is census-checkable:
  `OV-4g0` (`stealth_proof.cpp:949`, `stealth_bp_proof_size` returns 0 for
  `m ∉ {1,2}`), `OV-4g1` (`:990`, the `m != 1 && m != 2` range guard),
  `OV-4g3` (`:1011`, the FIRST `m`-sized allocation `std::vector<EC_POINT*> V(m)`);
  `OV-4g2` (`:995`, the `m != 1` wiring guard) already existed. §0 updated to
  the exact statement: both guards precede `:1011`, either alone suffices,
  only removing both is a memory-safety break (empty blob reaches `:998`).
- **CLS-FINAL-03** (round-17): four more `stealth_pool_builder.cpp` rows added
  — `OV-R-g2` (`:191-196`, per-attempt bucket-key shuffle + fail-closed RNG
  assert), `OV-R-h2` (`:198-213`, one-decoy-per-bucket-per-pass draw),
  `OV-R-i2` (`:214-215`, exhaustion `FC_ASSERT`), `OV-R-j2` (`:236-252`, final
  whole-ring shuffle). Coverage floor for the file **raised 1 → 7** so the
  round-16 "zero rows for the repaired file" silence cannot recur at a lower
  bar. Rule: any file a tip touches gets rows in the same pass that re-pins,
  or §0 records it `NOT_COVERED`.

## 14f. Re-pin change log — 2026-10-17, `5a580663` → `b547e650` (round-17 repair batch)

Machine-checked: census `{'OK': 188, 'MISS': 0, 'OOR': 0, 'NOFILE': 0}`,
rc=0, six coverage floors green (`stealth_pool_builder.cpp` rows 10, floor 7),
98 body-prose pins scanned, 0 out-of-range (2 annotated-historical). Anchor
**BYTE-IDENTICAL** sha256 `015eee29…`, 1932 B
(`tooling/verify_stealth_cost_anchor_unchanged.py 5a580663`, run with CWD
inside the G1 worktree on the `b547e650` tree). Entropy citations **39/39
PASS** (`tooling/verify_entropy_economics_citations.py`).

What moved the pins:

- **R17-B (`stealth_pool_builder.cpp` + `stealth.hpp`):** the sampler repair
  landed as a floored blend — exclusion only at/above
  `R17B_MIN_DISTINCT_WINDOWS` (12); below the floor the real's window is
  admitted as a decoy bucket. The R17B comment block grew the builder file;
  the new constant shifted `stealth.hpp` +3 lines.
- `stealth.hpp` comment/constant re-derivations and `g1_stealth_tests.cpp`
  re-derivations (incl. the VAC-R17-05 re-spend vector) are test/comment-side
  and added no pin rows here.
- `oracle.cpp` ratio-order total (`operator<` tie-break base then quote;
  `==` untouched) affected no row in this table; the census at `b547e650`
  (run 2026-10-17 after the commit + re-pin) is green in full.

Re-pin record:

- 19 rows drift-renumbered through the `git diff 5a580663 b547e650` line map
  (`tooling/apply_pin_rebase.py 5a580663 b547e650 --apply`): 10 ×
  `stealth.hpp` (+3 each), 9 × `stealth_pool_builder.cpp` (comment growth).
- OV-R-a/b **deliberately re-pinned by hand** — the drift tool maps line
  numbers and must not rewrite claims. The old unconditional "real-window
  exclusion" pair became: OV-R-a = the conditional guard (`:226`), OV-R-b =
  the exclusion branch (`:227`), OV-R-b2 = the shared admission path (`:228`,
  new row). Below-floor semantics change recorded in the census tool's §15
  header the same day.
- No finding, severity, or gate status changed by this pass. PRV-R17-01
  closure remains NOT claimed (verification-rig k-sweep owed; round-18).
- Docs-only follow-up the same day: `3aefa642` — vendor manifest SHA256
  correction (`EXT-R17-05`: two vendor-time typos, `generator/tests_impl.h`
  and `rangeproof/rangeproof_impl.h`; both files byte-stable since `cd97a59d`,
  measured). No code or binary change; the correction shifted the §2
  stale-prose pin (OV-2e re-pinned `:98-104` -> `:110-112` in the tool and
  this doc), after which the census re-run at `3aefa642` returns `{'OK': 188,
  'MISS': 0, 'OOR': 0, 'NOFILE': 0}` rc=0 and the BYTE-IDENTICAL anchor
  (sha256 `015eee29…`, 1932 B) and 39/39 entropy citations stand. Transcripts:
  `build/audit-g1-clsag-bp-specialist-packet/RERUN-3aefa642.txt` (re-runs) and
  `.../VENDOR-SHA256-TRANSCRIPT.txt` (the correction).

## 14g. Re-pin change log — 2026-10-17, `3aefa642` → `24cde5fd` (round-18 cleanup)

Machine-checked: census `{'OK': 188, 'MISS': 0, 'OOR': 0, 'NOFILE': 0}` rc=0,
six coverage floors green (`stealth_pool_builder.cpp` rows 10, floor 7).

What moved the pins: `24cde5fd` deduplicated a mangled comment block in
`stealth_pool_builder.cpp` (R18-PRV-02: dangling "The real input's own"
fragment plus a verbatim restart of the stratification description; 7 comment
lines deleted, no code line touched). Every §15 sampler row sits below the
deletion, so all ten shifted -7 (row-by-row map in the §0 ROUND-18 bullet);
re-pinned by hand in `tooling/r13k1_overlay_pin_census.py` (header note
records this wave). Re-run at `24cde5fd`: 188/188 green, anchor
**BYTE-IDENTICAL** sha256 `015eee29…` (1932 B), entropy citations **39/39
PASS**. Transcript:
`build/audit-g1-clsag-bp-specialist-packet/RERUN-24cde5fd.txt`.

Follow-on (same day): `8de355a3` scoped the op109 re-spend fixture note to
Leg B (SI-R18-05; comment-only, `g1_stealth_tests.cpp`). No §15 pin moves; the
full gate at `8de355a3` is 851 cases EXIT_0
(`build/logs/track-g1-8de355a3-r18-full.log`, binary `c5f96c15…`), census
188/188, anchor BYTE-IDENTICAL, entropy 39/39
(`build/audit-g1-clsag-bp-specialist-packet/RERUN-8de355a3.txt`).

- The same commit carries test/hygiene-side files only otherwise
  (`g1_stealth_tests.cpp` one deleted tautological clause plus comment,
  `test_wpc06_wallet_log_hygiene.py` nesting-aware guard rewrite); no
  consensus logic changed and no other file's pins moved.
- No finding, severity, or gate status changed by this pass. PRV-R11-01 /
  PRV-R17-01 closure remains NOT claimed (verification-rig k-sweep owed;
  round-18 delta seats hold the fixed-lane evidence: wallet-prover
  R17I-01 closed PASS; privacy delta PASS on R18-PRV-01/02 with one
  hygiene condition carried, R18-PRV-03-DELTA).
