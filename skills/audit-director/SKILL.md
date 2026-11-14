# Skill: audit-director

Use when running the full Track B + stealth implementation audit round across H1, H2, G1 branches. Vantage: orchestration — parallel read-only auditors, JSON compilation, repair TODO synthesis. NEVER parallel implementers; auditors are readers.

## Dispatch contract (batch-dispatch-parallelism shapes)

1. Dispatch the h1/h2/g1 implementation-auditor triple FIRST (disjoint scopes, read-only) —
   that was the original 3-auditor shape — then scale to the full disjoint cross-cutting skill
   set for a complete round (round-8 ran 19 auditors, round-11 ran 41, round-22 ran 39;
   disjoint scopes, never two implementers in parallel, and never two full `chain_test`
   binaries concurrently). Round 22 dispatched 39 of 40 internal audit seats (all except
   clsag-bp-plus-auditor, which is the external specialist gate covered by
   external-audit-engagement). Each seat wrote one JSON to `build/audit-round-22/<skill>.json`.
   - H1 auditor ← `h1-oracle-implementation-audit`, worktree `/workspace/worktrees/track-b-h1`, BSIP `2026-09-29-bsip-h1-rev2-oracle-base.md`
   - H2 auditor ← `h2-bond-vault-implementation-audit`, worktree `/workspace/worktrees/track-b-h2`, BSIP `2026-09-29-bsip-h2-bonds-vaults-issuance.md`
   - G1 auditor ← `g1-stealth-implementation-audit`, worktree `/workspace/worktrees/track-g1-stealth` (tip vs dirty tree distinguished), BSIP `2026-09-29-bsip-g1-stealth-transfers.md`
2. Each auditor returns ONLY the skill's JSON shape (`skill/auditor_model/branch/tip/findings[]/gate/conditions`)
   — `auditor_model` is mandatory (AGENTS.md §11, `tooling/audit_director_schema.json`) and carries the
   EXACT LLM model string that performed the seat; seats without it are rejected at aggregation.
   No prose, no fixes applied. Failed/empty/unparseable result = NEEDS_CONTEXT with evidence,
   re-dispatch narrowed — never identical retry.
3. Verify on disk after EVERY round: `git log`, `git status`, test tails, artifact bytes. Never trust self-reports alone.

### 3a. Subagents DO NOT COMMIT (hard gate, added after two violations)

Record the baseline before dispatch and diff it after. The feature trees are worktrees
of `/workspace/bitshares-core`, NOT of `/workspace` — baseline BOTH repositories and every
worktree HEAD:

```bash
git -C /workspace rev-parse HEAD                 # workspace baseline
git -C /workspace/bitshares-core rev-parse HEAD  # core baseline (worktree owner)
git -C /workspace/bitshares-core worktree list   # every feature worktree + HEAD
# ... dispatch, collect results ...
git -C /workspace rev-parse HEAD                 # MUST equal the baseline
git -C /workspace/bitshares-core rev-parse HEAD  # MUST equal the baseline
for w in /workspace/worktrees/*; do git -C "$w" rev-parse HEAD; done  # unchanged
git -C /workspace status --short                 # expected: only the paths the brief allows
```

An audit subagent that commits is a **failed dispatch**, not a convenience. Two
subagents did this during R11/R12 (`0cb632f`, `32fe74c`) and one invented a
severity vocabulary (`Critical/High/Medium/Low/Info`) that this program does not
use, so its commit message and ledger line both understated the findings (2
reported vs 11 filed). The damage was invisible to anyone trusting the ledger —
which is exactly the failure mode this program exists to prevent.

Therefore, every audit brief MUST carry this canonical clause word-for-word (re-wrapping
allowed; the bolded rule number and the "not just the evidence files" branch are
load-bearing — a brief that ships only "**NEVER COMMIT.**" diverges from the contract the
seats audit against and self-contradicts):

> **NEVER COMMIT.** Not in any worktree, not in the workspace repo, not "just
> the evidence files". The dispatcher commits. If you believe a commit is
> needed, say so in your return and stop.

When a commit is found after the fact: do NOT amend it. Audit what landed
(no secrets, authoritative sets intact), then record the discrepancy forward in
`build/PROGRESS.md` with the correct numbers, and treat the subagent's prose
claims as unverified until re-derived.

4. Compile: concatenate findings, sort Critical→Important→Minor→Note, dedupe shared claims (H1-exclusion, joint ≤70% cap, blind-position rule) with cross-references, and emit the repair TODO list.

## Repair TODO synthesis (required output)

- One TODO per Critical/Important finding: `{todo_id, severity, skill_id, branch, file:line, problem (1 sentence), fix (smallest concrete change), gate_blocked (H1/H2/G1/none), verify (command + expected output)}`.
- Minor findings batch into per-branch cleanup TODOs. Notes become BSIP-text deltas, not code TODOs.
- Order: consensus-fork risks first, theft/collusion second, inflation-containment third, honesty/doc deltas last. One writer at a time for fixes (AGENTS.md §9); bundle repairs with adjacent forward work on the same branch where the brief allows.

## Red flags

- Auditor edited code; two writers on one tree; per-dispatch commits; "confirmed wrote file" without on-disk check; H2 work pulled while H1 review open on the same branch; single-question solo round instead of the 3-auditor sweep.
- **A subagent commit after dispatch** — check `git rev-parse HEAD` against the baseline you recorded. Also red-flag any finding count in prose that disagrees with the machine-readable JSON it shipped beside it.

## Round-16 additions: concurrency, transport loss, and seat hygiene

These four procedures exist because round-16 (2026-10-16) hit all four. None of
them is optional on a round where seats are slow or the box is shared.

### A. Another dispatcher may be auditing the same tips at the same time

Round-16 ran with **five** concurrent dispatcher sessions auditing the same two
protocols at the same three frozen tips. They overlapped heavily. Do not clobber a
sibling's artifacts and do not re-derive a seat that already exists:

```bash
python3 tooling/adopt_round16_seats.py --round build/audit-round-N \
    --source build/audit-round-X build/audit-round-Y --report   # dry run
```

The tool **validates before adopting**: parses, checks the schema-required keys,
checks `gate` against the schema enum, checks every finding's `severity` and
`verdict` against their enums, requires a non-empty `auditor_model` (AGENTS.md §11),
requires the tip string to name a frozen tip, and rejects a seat with no findings,
conditions or `not_covered`. Adopting stamps a `provenance` condition saying the
seat was **not** independently re-audited by this dispatcher, and the round
synthesis must carry a seat→model table (AGENTS.md §11). Never silently merge.

### B. Subagents die to transport resets; recover before re-dispatching

A large fraction of seats died mid-run (connection resets, rate limits) with 60-400
tool calls of real analysis and no output JSON. **Recover first** — the work is
usually already done:

```bash
python3 tooling/recover_opencode_session.py <session-ids...> \
    --out build/audit-round-N/recovered
```

Then hand the recovered trail to a reconstruction seat whose job is to *finish* the
audit (verify what it keeps, write the JSON) rather than start it. Record both model
strings in the seat's `auditor_model` and a `conditions` line saying the seat is
RECONSTRUCTED, what was re-verified, and what was taken on trust. Round-16 recovered
15 trails this way and lost no analysis.

Dispatch in **small batches (3-5)**. A batch of 10 was cancelled wholesale; batches
of 3-5 lost 20-40% to resets, and the survivors were unaffected by the losses.

### C. A seat that files findings into the wrong field is a failed seat

Round-16's pilot-coverage seat put eight substantive findings — including the
chain-halt Critical — into `conditions[]` instead of `findings[]`, because `gate`
is an enum and it wanted prose. The JSON validates, so nothing catches it: the
aggregate tool silently counts **0 findings**. When a seat returns suspiciously few
findings, read its `conditions` before believing the count. The dispatcher may
re-shape the content into schema-valid findings **without changing substance**,
provided it preserves the seat's provenance in a `dispatcher_note` and says so in
the synthesis.

### D. Re-verify the headline Criticals yourself before the verdict

Do not launder a seat's Critical into the program verdict on seat authority alone.
Round-16 read the two code sites of the top finding directly and reproduced the
second one numerically (`tooling/r16_op109_sampler_multiplicity_attack.py`), which
is what turned "41 seats report a leak" into "the leak is real and the evidence for
its fix was measuring the wrong thing". Record in the adjudication which Criticals
were personally verified, which were re-read, and which rest on seat authority — a
synthesis that blurs those three is how a future round inherits a false claim.

## Round-17/18 additions: adjudication, repair waves, re-pins

Round-17 ran 41 seats / 540 raw findings and collapsed them to 3 code Critical families +
gate rows; round-18 verified the repair wave as a 12-seat lane re-audit (8 FAIL rows, all
closed in-round). Four procedures carried out of those two rounds:

### E. Adjudicate before counting

Raw rows compress ~10-14x (r16: 241 rows into 4 Critical + 14 Important families + 16 gates;
r17: 208 rows into 3 code families + gate rows). Emit `ADJUDICATION.md` beside `REPAIRS.md`.
Parallel sweeps over the same frozen tips re-detect the same vertices by design — that is
convergence evidence, not a rising finding rate. Count families, not seat rows.

### F. Repair wave, then lane re-audit, nothing in between

After a repair batch lands, dispatch fresh seats over the changed lanes only and close every
FAIL in-round; never a fresh full sweep while repairs are still landing (AGENTS.md §11). The
rounds 13-17 repair wave executed in one session (H1 +390/-11, H2 +1518/-67, G1 +416/-77,
35 files) — expect that shape, brief for it, and budget review time for it.

### G. Owed evidence stays owed

A gate no seat has executed (e.g. the PRV k-sweep) is recorded OWED and never closed, and no
verdict may cite the current k=16-only rig in its place. Missing RED evidence is reproduced
at a byte-exact revert before it is cited (round-18 reproduced the t8b2 RED on the revert).

### H. Re-pin before dispatch

Tips move between rounds. Re-run the pin censuses (`tooling/r13k1_overlay_pin_census.py`,
entropy citations, cost anchor) and re-pin every tip label in skills/specs/overlay before
seats load them; round-17 filed nine seats against a stale overlay count (PCT-R17-14). A
seat citing a stale pin is a filing error to correct at aggregation, not a finding.

## Round-25 additions: rate-limit deaths, the no-file seat, and dispatch shape under load

Round-25 (2026-10-18) ran 18 seats against the round-18 frozen tips plus two
committed design specs, in a box where other agents were concurrently active.
Three failure modes it hit are NOT covered by the round-16/17 procedures and
are recorded here.

### I. A rate-limit death leaves NO opencode.db record — nothing to recover

Section B says "recover before re-dispatching" — that is correct for TRANSPORT
RESETS, which die mid-run and leave 60-400 tool calls of real analysis in the
session store. A RATE-LIMIT death is different and worse: the dispatch is
rejected before the session is ever created, so
`tooling/recover_opencode_session.py` returns `!! session not found` and a
direct `opencode.db` query for the session id returns `None`. Round-25 lost
3 of 4 wave-1 seats this way and recovered zero, because there was no session
to recover. **Prevention is the only mitigation**: stagger dispatches to
PAIRS (2 at a time) whenever the box is shared or other agents are known to be
active; a batch of 4 failed 3-of-4 under load while every subsequent pair
succeeded. The round-16 "batches of 3-5" guidance holds for a QUIET box; use
pairs on a loaded one. Before re-dispatching a failed seat, check the DB — if
the session id is absent, do not waste a call on the recovery tool.

### J. A seat can complete its analysis and still never write its JSON

The round-16 section-C failure was findings-in-the-wrong-field. Round-25 saw
a new shape: the G1 seat ran to completion, returned a full prose summary
(gate, severity counts, six findings with evidence), but never wrote its JSON
file at all — the output contract failed at the last step. Its session also
left no DB record (see I), so its `auditor_model` could not be disclosed. Two
standing countermeasures, both adopted in-round and both now mandatory:

1. **Every seat brief MUST carry this clause** (same standing as the NEVER
   COMMIT clause in §3a — a brief without it is a failed brief):

   > MANDATORY FIRST AND LAST ACTION: write your output JSON to <path> and
   > re-read/parse it before returning. A return without the file on disk is
   > a failed seat.

2. **Dispatcher materialization is the recovery of last resort.** If a seat
   returns a complete, evidence-bearing summary but wrote no file, the
   dispatcher may write the JSON verbatim from the return, stamping
   `auditor_model` with a reconstruction note (the exact model string is
   unknown when the session left no DB record — say so; do not invent one)
   and a `conditions` line saying the seat is MATERIALIZED-BY-DISPATCHER.
   This is degraded provenance, strictly worse than a self-written file; it
   is what round-25 filed for `g1-stealth-implementation-audit.json`. Verify
   every materialized finding's file:line pins before trusting the gate.

### K. Spec-vs-code audit is a first-class lane shape

Round-25 audited two committed-but-unimplemented design specs
(`spec-audit-expired-exit-drained`, `spec-audit-winddown-fair-settlement`)
as dedicated seats: every spec pin resolved against the frozen tip, the
design's controls were checked against the closed finding families it must
not reopen, and its acceptance criteria were checked for testability against
the tree. This lane caught defects the implementation seats could not (a
false "resolve never read a price" claim, a mis-cited call site, an
unquantified seating-drift strand). When a round includes design docs as
subjects, give each its own seat keyed `spec-audit-<name>` rather than
folding it into an implementation lane; the spec seat's job is pins,
non-reopening, and testability, not implementation review.

## Round-23 additions: error returns lie, adoption at scale, the dispatch ladder

Round-23 (2026-10-18) ran 47 seats (11 native + 38 adopted) at the round-18
frozen tips while sibling dispatchers ran rounds 20/21/24/25 on the same tips.
Four lessons not covered above.

### L. A spawn error does NOT predict work death — check disk before re-dispatching

Round-25's section I says a rate-limit death leaves no record. Round-23 saw the
OPPOSITE outcome under the same error string: three seats (`h1-oracle`,
`h2-bond-vault`, `clsag-bp-plus`) returned "Rate limit exceeded" from spawn,
yet their complete, well-formed JSONs landed on disk afterwards — the work had
already run to completion. The error return is transport truth, not work
truth. **Standing order: after ANY spawn error, `ls` the target JSON path
before deciding anything.** Artifact present = seat done, do not re-dispatch
(overwriting a landed artifact destroys a completed audit). Artifact absent =
dead per section I. Record which outcome each error took in the dispatcher
notes, because the same string has both.

### M. Sibling adoption at scale — and the stale-tool warning

Round-23 adopted 38 of 47 seats from three concurrent sibling rounds via a
per-round tool (`tooling/adopt_r23_seats.py`, procedure A validation: schema
enums, non-empty `auditor_model`, tip must name a frozen code tip, one seat
per skill, never overwrite a native seat). Warning: `tooling/adopt_round16_seats.py`
hardcodes round-16-era frozen tips — using it unmodified on a later round
rejects every valid sibling seat for wrong-tip. Copy the newest `adopt_r23_*`
tool and re-pin `FROZEN` per round; the tips live in `LANES.md`. One sibling
seat was rejected for pinning its tip to the WORKSPACE HEAD (a docs commit, not
a code tip) — a seat that never audited the code cannot be adopted, however
complete its JSON looks. Adoption never re-audits; the synthesis must carry the
seat->model table with native/adopted provenance (AGENTS.md §11), and headline
Criticals from adopted seats still get a procedure-D personal read.

### N. The dispatch ladder: 3-5 -> pairs -> single, and retries ARE legal for transport

Sections B and I give batch sizes for a quiet box (3-5) and a loaded one
(pairs). Round-23 found pairs still failing 1-of-2 under sustained limits and
finished the round in STRICTLY SEQUENTIAL mode (one subagent per wave), which
never failed. The ladder is therefore: waves of 3-5 -> pairs on load -> single
under sustained pressure; rung down at the first casualty, not after losing a
wave. Also: the "never identical retry" rule in the dispatch-shape skill binds
ANALYTICAL failures (wrong scope, NEEDS_CONTEXT, per
`skills/batch-dispatch-parallelism/SKILL.md`). A pure transport/rate-limit
failure carries no analytical signal — retrying the identical brief is correct
and expected (round-23 retried the same seat brief after a spawn error and got
a clean, distinct analysis).

### O. Two more failed-seat shapes for section C's family

1. A seat may file findings with `title`/`status` keys instead of
   `check`/`verdict` (round-23's `hybrid-issuance` seat, 13 findings). The
   JSON still parses; the aggregate tool counts them as schema-valid but any
   strict validator flags every row. Dispatcher re-shape per section C,
   substance unchanged, provenance noted.
2. A DYING seat may write a stub JSON first (`findings: []`, gate holding an
   invalid enum like `"PENDING"`) and never return to fill it. A stub is not a
   zero-finding seat: it is an incomplete dispatch. Overwrite it on
   re-dispatch; never count it in gate tallies; a schema check at aggregation
   catches the invalid enum but NOT the empty-findings semantic — check both.

## Output files

- Raw auditor JSON: `build/audit-round-N/<skill-name>.json` — one file PER AUDITOR, named for
  the skill that produced it. (The `{h1,h2,g1}.json` convention was round-1..3; round-8 shipped
  19 skill-named JSONs plus nested subdirectories for late auditors. Do not reintroduce the
  three-name shape for a multi-auditor round.)
- Compiled repairs: `build/audit-round-N/REPAIRS.md` (the TODO list) + machine-readable `build/audit-round-N/repairs.json` per `tooling/audit_director_schema.json`; aggregation via `tooling/aggregate_audit_json.py`.
