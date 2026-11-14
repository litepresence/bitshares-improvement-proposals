# Skill: batch-dispatch-parallelism

Use when planning a subagent dispatch round in THIS workspace (BitShares consensus work).
Revised for this repo: the original version targeted massively parallel translation work and
does not apply here — parallel implementers on one tree cause merge conflicts in consensus
code and are banned (AGENTS.md §9). This version states exactly what may run in parallel.

## When to Use

- Multiple independent READS: research sweeps, paper reviews, code surveys, audit rounds
  over disjoint questions. Parallelize freely; reviewers cross-check shared claims after.
- NEVER parallel implementers on the same branch. One writer per branch, always. Parallel implementers on the same branch are banned — they produce merge conflicts in consensus code (AGENTS.md §9). Different branches may be implemented in parallel only when their write sets are disjoint.
- Sequential implementation may overlap with review of the PREVIOUS task only (reviewer
  reads a frozen diff while the next implementer works ahead solely when the next task's
  brief does not depend on the review outcome — otherwise wait for the gate).

## Dispatch Shapes (this workspace)

| Shape | Contents | When |
|---|---|---|
| Research sweep | N agents, disjoint questions, read-only | Papers, codebase surveys, audit rounds |
| Implement + review chain | One implementer → one reviewer → fix loop | Every consensus task, strictly sequential |
| Repair bundle | Fix findings + tightly adjacent forward work, same branch | Review found issues; forward work doesn't depend on the fix outcome |

## Standing Rules

- Disjoint write sets or it doesn't dispatch. Two writers never touch the same tree or branch.
- Bundle repairs with forward work on the same branch where the brief allows; never a lone
  repair dispatch when forward work is queued.
- Verify on disk after EVERY dispatch (files exist, real content, no placeholders). Never
  trust self-reports alone — check `git log`, `git status`, test tails, artifact bytes.
- Commit discipline follows the workspace rule and the task brief (feature branches,
  reviewed commits only) — never per-dispatch-round commits by default.
- A failed or empty result is NEEDS_CONTEXT with evidence, then re-dispatch with changed
  context (smaller scope, more capable agent, or corrected brief) — never an identical retry.
- **Stagger dispatches to pairs (2 at a time) when the box is shared or other agents are
  known to be active.** A batch of 4 concurrent seats failed 3-of-4 to rate limits in
  round-25 under concurrent load while every subsequent pair succeeded. A RATE-LIMIT death
  is unrecoverable — unlike a transport reset, it is rejected before the session is
  created, so `tooling/recover_opencode_session.py` finds no record and there is nothing
  to recover; prevention (smaller concurrent batches) is the only mitigation. Check the
  DB for the session id before attempting recovery on any failed seat. Under
  SUSTAINED pressure where even pairs drop seats, fall back to strictly
  sequential dispatch (one subagent per wave — round-23 finished 47 seats this
  way without a loss) and rung down at the first casualty rather than after a
  lost wave. After any spawn error, check the artifact on disk before
  re-dispatching: a "Rate limit exceeded" return can still be followed by the
  seat completing and writing its JSON (round-23 saw three), and re-dispatch
  would clobber a finished seat. Identical-brief retries are legal for pure
  transport failures; the never-identical-retry rule binds analytical failures
  only.

## Red Flags

- "Dispatch 3 implementers on the three legs to go faster"
- "The subagent confirmed it wrote the file"
- "Pull in H2 work while H1 review is still open on the same branch"
- "Dispatch two implementers on the same branch in parallel"
- "Commit after every dispatch"
- "Only 1 research question ready — run it alone" (bundle independent reads instead)
