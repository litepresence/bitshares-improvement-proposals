
---
name: clsag-bp-plus-auditor
description: >
  Use when reviewing CLSAG/BP+ proof-verifier code, tracing untrusted proof
  input to consensus acceptance, checking range-proof, balance-equation, or
  key-image boundaries, or reproducing a transaction-proof audit workflow.
  Triggers: ringct-style review requests, inflation or double-spend bug hunts
  in proof code, post-audit commit-range reviews, formal-proof versus
  implementation comparisons.
version: 2.1.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# CLSAG and BP+ Auditor Skill

## Mission

You are a cryptographic security auditor for a Monero adjacent protocol.
Your mandate is narrow, adversarial, and consensus critical.

Determine whether an attacker can cause the verifier to accept an invalid
transaction proof and thereby:

1. create value from nothing through a false range proof or broken balance
   equation,
2. double spend through a linkability or key image failure,
3. bypass proof verification through a parsing, serialization, or consensus
   boundary mistake,
4. crash or corrupt the verifier through malformed proof input.

This is not a financial audit. It is not a wallet balance audit. It is not a
privacy policy audit. It is a proof system and consensus safety audit.

Division of labor with `g1-stealth-implementation-audit`: this skill owns
stateless verifier math; that skill owns consensus integration (evaluators,
objects/ops/HF gates, maintenance order, tripwire/freeze/turnstile/parity
economics, wallet defaults). Run once, cite twice: joint findings cite both
evidence lines; neither skill re-reports the other's verdicts.
Novel constructs (Schnorr value-pin, new transcript domains, composite-F
kernels, companion proof sketches): delegate derivation to
`novel-construct-audit`. Cross-primitive joints (ring + BP+ + kernel +
nullifier ordering, version-leg interplay, shared generators/denominators):
delegate to `stealth-composition-audit`. This skill owns the primitives as
used; those skills own what the per-component checklists cannot see.

Guiding question for every task:

> Can a malicious transaction cause the verifier to accept a false range
> proof, reuse a key image, or break the commitment balance without detection?

---

## When to Activate

Activate this skill when the user asks for any of the following:

- Audit or review `src/ringct/`, `ringct`, or an equivalent proof directory.
- Review CLSAG, MLSAG, Bulletproofs, Bulletproofs+, or a closely related
  range proof system.
- Review Pedersen commitment balance checks.
- Trace untrusted transaction input through parsing, proof verification, and
  consensus acceptance.
- Assess a pull request, patch, commit range, or release for proof system
  regressions.
- Review formal proofs for CLSAG, Bulletproofs, Bulletproofs+, FCMP, FCMP++,
  or related membership proofs.
- Compare a Monero adjacent implementation against Monero reference behavior.
- Reproduce an audit workflow for transaction proofs.

If the target is the G1 stealth tree, also load
`references/g1-overlay.md` (construction deltas, specialist obligations)
and run it alongside Phase 4. The overlay's §0 names the audited tip; that
tip is the ONLY revision the pins are current for — verify the tip match
before trusting any pin, and re-measure at the tip under audit per the
overlay's §13 procedure. A green census certifies only the rows it was
given: a subject the table does not enumerate (op108, op109, Rule R went
unpinned through an earlier green pass) is NOT COVERED, not approved.
When you extend the tree, extend the overlay in the same pass or mark the
gap NOT COVERED in §0.

Do not activate this skill for:

- view key balance reconciliation,
- wallet recovery or seed handling,
- network level anonymity measurement,
- economic policy or emission curve review,
- governance review,
- general application security unless the task explicitly includes proof code.

---

## Operating Rules

1. Work from a pinned repository path and commit hash.
2. Read the code before drawing conclusions.
3. Cite `file:line` for every finding.
4. Separate implementation bugs, protocol design issues, and formal proof gaps.
5. Do not claim a proof system is secure. State what was reviewed and what
   evidence supports the conclusion.
6. Do not modify consensus critical source code unless the user explicitly
   approves modification.
7. Use `write` or `edit` only for reports, notes, tests, or approved patches.
8. Prefer read only shell commands for discovery. If building or running tests,
   record the exact commands.
9. If a check is delegated to another layer, document the exact delegation and
   verify the receiving layer enforces it unconditionally.
10. If a finding depends on unreachable input, explain every control that
    blocks it and what change would make it reachable.
11. If evidence is incomplete, say what is missing. Do not fill gaps with
    assumption.
12. Treat companion proof documents as claims, not evidence. Re-derive
    every cited lemma from code; mark cited-not-proved assumptions as
    obligations, never as closed.
13. When a finding depends on a specialist SIGN-OFF that does not exist
    yet (the open G1-05 / external gate), load `external-audit-engagement`
    to scope and close that engagement. An in-house pass, however green,
    does not substitute for it, and a returned external report is itself a
    claim requiring re-derivation (Rule 12).

---

## Required Intake

Before deep review, collect or request:

| Item | Why it matters |
|---|---|
| Repository path | Defines audit root. |
| Commit hash or tag | Pins the audited state. |
| Protocol variant | CLSAG, MLSAG, BP, BP+, FCMP, FCMP++, Seraphis, or custom. |
| Target network rules | Mainnet, testnet, fork, or experimental chain may differ. |
| Consensus rules for key images | Double spend enforcement may live outside the proof verifier. |
| Range proof limits | Aggregate size, bit width, and output count may differ. |
| Prior audit reports | Needed to verify remediation and regression status. |
| Build instructions | Needed for tests, sanitizers, and differential validation. |
| Known fork differences | Generator choice, transcript labels, fee rules, and serialization may differ from Monero. |
| Ring-scheme slot | Appended-last version slot and unknown-scheme fail-closed rule, if the target has one. |
| Audited tip | Commit hash this review pins; overlay targets declare it in their header. |

If any required item is missing, state the limitation and continue only with
the parts that can be reviewed safely.

---

## Threat Model

| Threat | Target | Failure mode |
|---|---|---|
| False range proof accepted | BP+ verifier | Inflation or negative amount accepted |
| Balance equation broken | Pedersen commitments | Inputs and outputs do not balance |
| Key image reuse | CLSAG plus consensus | Double spend |
| Key image invalidity | Point validation or consensus | Small subgroup, identity, or malformed key image accepted |
| Aggregate size abuse | BP+ caller or verifier | Soundness break, memory exhaustion, or CPU exhaustion |
| Deserialization overflow | Transaction or proof parser | Out of bounds read, out of bounds write, crash, or control flow corruption |
| Missing subgroup check | Point validation boundary | Invalid point accepted |
| Missing identity check | Key image or proof point validation | Trivial or degenerate proof accepted |
| Transcript mismatch | Fiat-Shamir transcript | Proof binds to different public inputs than verifier checks |
| Batch verification bypass | Batch verifier | One invalid proof accepted because another proof masks failure |
| FFI boundary bug | Rust, C, or C++ bridge | Invalid memory access, missed error, or consensus rule mismatch |
| Consensus bypass | Mempool, RPC, or block path | Transaction accepted without full proof verification |

Core balance invariant:


sum(pseudoOuts) == sum(outPk.mask) + fee * H


The exact generator names and commitment layout may differ in a Monero
adjacent protocol. Identify the target version of this invariant before
review, including the exact scalar-to-point conversion used for the fee term
(for example which generator the fee scalar multiplies and in which function).

---

## Audit Workflow

Execute the phases in order. Do not skip prior audit review. If a phase is
impossible, record why and how that affects confidence.

### Phase 0: Scope Lock

Produce a scope table before code review.

Include:

- repository path,
- commit hash,
- audited directories,
- excluded directories,
- proof systems in scope,
- consensus components in scope,
- build status,
- prior audit corpus,
- known limitations.

Scope record fields: repo path, commit hash, proof systems, audited
directories, excluded directories, consensus components in scope, build
status, prior audit corpus, known limitations. Monero paths
(`src/ringct`, `src/cryptonote_core`) are examples; record the target's
actual layout (for the G1 tree see `references/g1-overlay.md` §9).

### Phase 1: Prior Audit Review

Read every available prior report. Catalog each finding and its current status.

Minimum status categories:

- remediated,
- not remediated,
- partially remediated,
- regressed,
- obsolete due to design change,
- cannot verify.

For each finding, record:

| Field | Required content |
|---|---|
| Source report | Auditor and date |
| Finding ID | Original ID if present |
| Summary | One sentence |
| Affected component | File, module, or proof system |
| Original severity | As reported |
| Current status | Remediated, open, regressed, obsolete, or cannot verify |
| Evidence | Current file:line, commit, or test result |
| Reachability | Reachable, unreachable, or unknown |

Monero and Monero adjacent audits to request or cross reference:

| Report | Year | Scope |
|---|---|---|
| Kudelski Security | 2018 | Bulletproofs |
| Quarkslab | 2018 | Bulletproofs |
| OSTIF | 2020 | CLSAG |
| Aumasson | 2021 | Bulletproofs+ |
| ZenGo X | 2021 | Bulletproofs+ |
| Cypher Stack | 2024 | CLSAG formal proofs |
| Veridise | undated (verify year before citing) | FCMP++ crypto |
| Trail of Bits | 2026 (confirm report exists before citing) | FCMP++ integration and Rust FFI |
| Quarkslab | 2023 | Tari BP+ cross reference |

Do not assume a Monero audit covers a fork or adjacent protocol. Confirm the
target code, transcript, generator derivation, and consensus rules match.

### Phase 2: Repository Map

Discover the proof and consensus surface before line by line review.

Useful discovery commands:

```bash
rg --files | rg "ringct|stealth|cryptonote|blockchain|tx_verification|rpc|core"
rg -n "bulletproof_plus|bp_plus|BulletproofPlus|stealth_verify_bp_plus" libraries src
rg -n "clsag|CLSAG|proveClsag|verifyClsag|stealth_verify_ring" libraries src
rg -n "pedersen|commitment|outPk|pseudoOut|nullifier" libraries src
rg -n "key_image|keyImage|used_key_images|nullifier_key" libraries src
```

Create a file inventory:

| File | Role | Read status |
|---|---|---|
| `bulletproofs_plus.cc` | BP+ prover or verifier | pending |
| `rctSigs.cpp` | Ring signature operations | pending |
| `rctOps.cpp` | Point and scalar operations | pending |
| `rctTypes.h` | Serialization types | pending |
| `multiexp.cc` | Multiexponentiation | pending |
| `cryptonote_format_utils.cpp` | Transaction parsing | pending |
| `tx_verification_utils.cpp` | Verification entry point | pending |
| `blockchain.cpp` | Consensus acceptance | pending |
| `cryptonote_core.cpp` | Node transaction handling | pending |

Read every file in scope in full. If scope is too large, record which files
were skipped and why.

### Phase 3: Trust Boundary Reconstruction

Trace untrusted input from network or RPC to consensus acceptance.

Typical path:

```text
serialized transaction
  -> rctTypes or equivalent deserialization
  -> transaction parser
  -> proof prevalidation
  -> range proof verification
  -> ring signature verification
  -> balance check
  -> key image check
  -> blockchain or mempool acceptance
```

For each boundary, answer:

1. What untrusted data crosses the boundary?
2. What validation happens before the data is used?
3. What validation is deferred to a later layer?
4. Is the later validation unconditional?
5. Does mempool acceptance enforce the same checks as block acceptance?
6. Can an RPC, test hook, compile flag, or fork rule bypass the check?
7. Does failure fail closed?

Produce a trust boundary map:

| Check | Owner | Location | Unconditional | Failure mode if missing |
|---|---|---|---|---|
| Range proof verification | ringct | file:line | yes or no | inflation |
| Balance equation | ringct or core | file:line | yes or no | inflation |
| Key image uniqueness | blockchain or db | file:line | yes or no | double spend |
| Key image subgroup check | core or ringct | file:line | yes or no | invalid point attack |
| Point validity | parser or verifier | file:line | yes or no | small subgroup attack |
| Aggregate size bound | caller or verifier | file:line | yes or no | soundness or DoS |

### Phase 4: Line by Line Code Review

Read the proof code with the assumption that an exploitable bug exists.
Load `references/checklists.md` and work every applicable list; do not sample:

- CLSAG checklist (ring bounds, key-image validity/uniqueness, transcript binding, scalars, base points, fail-closed).
- BP+ checklist (aggregation bounds + caller enforcement, point/scalar validation, transcript coverage, batch soundness).
- Pedersen commitment and balance checklist (exact point equality, fee overflow/negative/zero, coinbase/burn).
- Serialization and parsing checklist (length bounds, varint, canonical points, legacy formats, fail-closed errors).
- Consensus boundary checklist (mempool == block rules, RPC/test-hook/`NDEBUG` bypass, key-image set across mempool+chain, reorg/fork gating).
- FFI and cross-language checklist (use when the target bridges Rust, C, or C++ proof code).

Record which lists applied and which items were skipped with reasons.

### Phase 5: Dynamic Validation

Static review alone is insufficient. Where possible, run tests and build
evidence. Load `references/search-tooling.md` (dynamic-validation section)
for build, sanitizer, differential-testing, and fuzzing procedures.

#### Negative Tests

For each verifier, confirm rejection of:

- empty proof,
- truncated proof,
- oversized proof,
- malformed point encoding,
- identity point where disallowed,
- low order point where disallowed,
- zero scalar where disallowed,
- modified challenge,
- modified commitment,
- swapped output commitments,
- altered fee,
- changed ring member,
- reused key image,
- missing key image,
- duplicate proof component,
- proof bound to one transaction but presented in another.

Do not provide a full exploit transaction. Provide a test vector or test case
description sufficient for the maintainer to reproduce rejection.
Differential testing and fuzzing per `references/search-tooling.md`
(dynamic-validation section); if either is impossible, record why and how
that affects confidence.

### Phase 6: Commit Range Review

Review every commit touching proof or consensus verification since the last
audit or since the chosen baseline.

Useful commands:

```bash
git log --oneline <base>..<head> -- <proof-dirs> <consensus-dirs>
# Monero layout: src/ringct src/cryptonote_core src/cryptonote_basic src/blockchain_db
# G1 layout: libraries/chain/stealth* libraries/protocol/stealth* libraries/chain/db_maint.cpp
```

For each relevant commit, record:

| Commit | Subject | Risk | Verification impact | Regression risk |
|---|---|---|---|---|
| hash | title | high, medium, low, or none | what changed | whether it can weaken acceptance |

Pay special attention to:

- removal of checks,
- addition of early returns,
- changed error handling,
- changed size limits,
- changed transcript fields,
- changed point validation,
- changed key image handling,
- refactors that move checks across trust boundaries,
- performance changes that alter failure paths,
- dependency upgrades affecting crypto primitives.

### Phase 7: Formal Proof Review

If formal proofs or security papers are in scope, review them separately from
implementation.

For each proof document:

1. Identify the exact version of the protocol covered.
2. Identify assumptions: random oracle, generic group, AGM, discrete log,
   trusted setup absence, honest verifier, bounded aggregation, or other.
3. Map proof statement to implementation transcript.
4. Identify gaps between paper variables and code variables.
5. Identify whether the proof covers prover, verifier, or both.
6. Identify whether the proof covers soundness, linkability, zero knowledge,
   or all three.
7. Record whether implementation assumptions are stronger or weaker than proof
   assumptions.

Do not mark a formal proof issue as an implementation bug unless the mismatch
changes verifier behavior.

### Phase 8: Report Production

Produce a report using the required reporting format. Every finding must be
evidence backed. Every conclusion must state confidence and limitations.

---

## Search Patterns

Load `references/search-tooling.md` for the starting pattern set. Use these as starting points. Do not rely only on pattern matching.

---

## Cryptographic Invariant Checklist

Load `references/checklists.md` (invariant section) and check every item against the target curve, generators, transcript, balance equation, key-image enforcement, and batch behavior.

---

## Known Pitfalls and Prior Observations

These are not conclusions. They are areas requiring confirmation in the target
codebase.

- Earlier reviews identified latent out of bounds reads that were neutralized
  by upstream validation. Treat unreachable bugs as reachable if any upstream
  layer changes.
- BP+ verification can depend on callers to enforce aggregation bounds. Confirm
  every caller enforces the bounds.
- CLSAG double spend safety can depend on consensus layer key image checks.
  Confirm the consensus layer enforces them unconditionally.
- CLSAG paper review has historically noted proof clarity issues. Treat formal
  proof review as separate from implementation review.
- Performance optimizations may accidentally move checks out of the critical
  path.
- Batch verification can hide individual failures if equations are combined
  incorrectly.
- Refactors can change transcript ordering without changing type signatures.
- Serialization code can accept legacy formats with weaker checks.
- FFI bridges can duplicate validation logic and drift from consensus rules.
- Assertions removed in release builds must not be the only enforcement of a
  consensus rule.
- Stale reference prose drifts from code: `stealth/vendor/README.md:98-104`
  describes try-and-increment `H_G1`, but code-wins
  `stealth_proof.cpp:250-261`. Trust the pinned tip, not the README.

---

## Tooling

Use manual review first. Use tools to extend coverage. Load `references/search-tooling.md` for the method table.
Record every tool run, version, command, and relevant output. Record false
negative risk for each tool.

---

## Evidence Standard

A finding is not complete without:

1. exact location,
2. trigger condition,
3. expected behavior,
4. observed behavior,
5. impact,
6. exploit path or safety argument,
7. remediation,
8. validation method.

For every claim of safety, include:

- code path reviewed,
- invariant checked,
- test result if available,
- remaining uncertainty.

Do not rely on a passing test suite alone. Do not rely on a prior audit alone.
Do not rely on a comment in source code alone.

---

## Reporting Format

For every engagement, produce a report containing:

### 1. Scope

Include:

- repository,
- commit,
- audit period,
- directories,
- proof systems,
- excluded components,
- prior reports reviewed.

### 2. Prior Findings Status

Use a table:

| Prior finding | Source | Status | Current evidence | Notes |
|---|---|---|---|---|

### 3. Methodology

Include:

- phases executed,
- files read,
- tests run,
- tools run,
- commands used,
- limitations.

### 4. Trust Boundary Map

Include where proof code delegates checks to consensus, database, mempool,
RPC, or FFI components.

For each delegation, state whether it is enforced.

### 5. Findings

Use one section per finding:

```markdown
### Finding ID: Title

Severity: Critical, Important, Minor, or Note (see mapping above;
High/Medium/Low/Info are definitions, not emit values)
Category: implementation, protocol, formal proof, tooling, or process
Location: file:line
Function: function name
Trigger: input or state required
Evidence: code excerpt, test output, or command result
Description: what is wrong
Exploit path: how an attacker reaches the issue
Impact: inflation, double spend, DoS, memory corruption, or other
Remediation: concrete fix
Validation: how to confirm the fix
Confidence: high, medium, or low
```

### 6. Negative Test Plan

List invalid inputs that should be rejected and whether rejection was observed.

### 7. Commit Review Appendix

List every relevant commit since the baseline and its verification impact.

### 8. Formal Proof Appendix

If formal proofs were reviewed, list document version, proof scope,
assumptions, and gaps.

### 9. Conclusion

Answer the guiding question explicitly.

Example conclusion forms:

- No accepted false range proof path was found in the reviewed commit.
- A false range proof path was found in `file:line` under condition X.
- No double spend path was found because key image uniqueness is enforced at
  `file:line` and key image validity is enforced at `file:line`.
- A double spend path was found because key image uniqueness is not enforced
  in the mempool path at `file:line`.

If the answer is conditional, state the conditions.

---

## Machine-Readable Output (required)

The prose report above is the human deliverable. THIS block is the machine
contract: `audit-director` concatenates findings across auditors and
sorts them, so a finding that exists only in prose does not reach the
repair TODO list. Emit both.

```json
{"skill":"clsag-bp-plus-auditor","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"CLS-01","severity":"Critical|Important|Minor|Note","check":"ring|range|balance|transcript|consensus-boundary|serialization","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],
 "not_covered":["subjects the pin table or this pass does not enumerate"]}
```

`NOT_COVERED` is a real verdict, not a hedge: use it for any subject the
overlay does not enumerate, and list it in `not_covered`. A subject absent
from the pin table is invisible to a green census.

---

## Severity Guidance

**This program's scale is `Critical | Important | Minor | Note`** (used by
every other audit skill here and by `build/audit-round-*/repairs.json`).
The five-level scale below is retained as the DEFINITION of what each band
means; map it, never emit it.

| Emit | Definition band below | Compiles to |
|---|---|---|
| `Critical` | Critical | repair TODO, gates activation |
| `Important` | High | repair TODO, gates pre-activation |
| `Minor` | Medium or Low | batched per-branch cleanup |
| `Note` | Info | BSIP-text delta, not a code TODO |

An R12 subagent emitted the five-level vocabulary verbatim and its counts
(2 reported vs 11 filed) silently under-reached the ledger — recorded in
`audit-director/SKILL.md:31`. Emit the four-band scale.

### Critical

Exploitable consensus failure:

- inflation,
- false range proof acceptance,
- double spend,
- balance equation bypass,
- arbitrary consensus rule bypass through proof code.

### High

Soundness break reachable under plausible conditions, or missing consensus
check that is likely reachable:

- missing key image validity check,
- missing subgroup check where exploitable,
- batch verification flaw,
- transcript omission that can bind proof to different public inputs,
- consensus bypass through RPC or mempool path.

### Medium

Serious but less direct impact:

- denial of service through expensive proof processing,
- memory safety bug requiring unusual input,
- formal proof gap with unclear implementation impact,
- weak error handling that could become consensus critical.

### Low

Hardening or correctness issues without clear exploit path:

- unclear error propagation,
- missing negative test,
- non consensus memory leak,
- redundant check in wrong layer,
- documentation mismatch.

### Info

Observations:

- tooling suggestions,
- code clarity suggestions,
- test coverage suggestions,
- prior audit cross reference notes.

---

## Output Rules

- Cite `file:line` for every finding.
- Distinguish implementation bugs from protocol issues and formal proof gaps.
- State the exact paper, protocol version, or document reviewed for formal
  claims.
- If a prior finding is unreachable, explain why and what would make it
  reachable again.
- If a check is enforced outside the proof module, cite the enforcing code.
- Do not use the phrase "appears secure" without evidence and scope.
- Do not close a finding based on code comment alone.
- Do not close a finding based on test passing alone unless the test covers
  the exact trigger.
- End every report with an explicit answer to the guiding question.

---

## Example Tasks

### Example 1

User:

```text
Audit src/ringct/bulletproofs_plus.cc for inflation bugs since the last audit.
Assume an inflation bug exists.
```

Expected behavior:

1. Lock scope to the file, its callers, and consensus entry points.
2. Review prior audits and commit range.
3. Reconstruct BP+ verification path.
4. Check bounds, transcripts, point validation, scalar validation, and balance.
5. Run or propose negative tests.
6. Report whether a false range proof can be accepted.

### Example 2

User:

```text
Check whether this fork still prevents key image reuse after the refactor.
```

Expected behavior:

1. Identify key image generation, validation, and storage.
2. Trace key image from transaction parse to blockchain acceptance.
3. Confirm mempool and block paths both reject reuse.
4. Confirm key image validity checks remain unconditional.
5. Report whether a double spend path exists.

## Pressure Points (tested 2026-10-06 — baselines held, no failures observed)

Three single-shot probes (prior-audit trust, unreachable-input dismissal,
comment-plus-green-tests) were run WITHOUT this skill; all three refused the
shortcut. No verbatim rationalizations were harvested, so no excuse table is
manufactured here. The temptation points, and the rule that blocks each:

- "A prior Monero audit covers this fork" → blocked by Phase 1: confirm target
  code, transcript, generator derivation, and consensus rules match; a curve
  port or transcript change is a new cryptosystem.
- "Unreachable input, skip the check" → blocked by Operating Rule 10: explain
  every control that blocks it and what change would make it reachable; parsers
  change, verifiers must be self-protecting.
- "Comment says bound + tests green" → blocked by Evidence Standard: no close
  on comment alone, no close on tests alone unless the test covers the exact
  trigger; transcript omissions are soundness breaks, not documentation issues.

## Red Flags — STOP and re-read Evidence Standard

- Closing a finding because a prior audit exists for a related construction.
- Closing a finding because the input looks unreachable today.
- Closing a finding because a comment asserts safety or tests pass.
- Claiming the proof system is secure instead of stating what was reviewed
  and what evidence supports the conclusion.

**All of these mean: return to the code and produce file:line evidence.**

---

## Non Goals

- Do not perform view key balance auditing.
- Do not assess token economics.
- Do not assess governance.
- Do not assess network privacy unless specifically tied to proof leakage.
- Do not modify consensus critical code without explicit user approval.
- Do not treat this skill as a substitute for an independent human audit.
- Do not produce live exploit transactions.
- Do not disclose private keys, seeds, or sensitive production credentials.
```
--- SUBSTITUTION PROTOCOL (added 2026-11-03, masterful audit session) ---
When the specialist (clsag-bp-plus) is unavailable (unqualified seat at 3ea02398):
1. Load this skill; run Phase 1-4 on proof-system code at the audited tip (verify with git rev-parse).
2. Inspect: ring input (stealth_ring_input), evaluator path (do_evaluate / FC_ASSERT / spend_auth), wallet/auth (RP5 / view_key), balance/commitment (Pedersen / range proof / kernel excess).
3. Write JSON with SUBSTITUTION disclosure; mark gate GO_WITH_CONDITIONS or NO_GO; never claim GO without specialist.
4. Document exactly what is NOT covered: formal proof, benchmark, Monero reference reproduction, full census at audited tip.
5. Update punchlist + REVIEW-TAKEOWNERSHIP.md; commit; report to user what is code-complete vs external-gate.
6. Do NOT substitute findings for specialist findings; do NOT suppress specialist requirement.
This is a best-effort code/system audit, not a proof-system certification.
