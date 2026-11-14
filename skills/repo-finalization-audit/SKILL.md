# Skill: repo-finalization-audit

Use when finishing a repo program: sweep for construction debris, retired-feature
traces, stale tips in prose, dead links, untracked build artifacts, and docs that
contradict each other. Emits a checklist, not a verdict. Run before declaring a
program done.

## Checks

1. **Retired-feature traces.** `grep -rn "H3\|vault_foreign\|h3_attestor" docs/ build/*.md AGENTS.md` should
   only hit lines that explain the retirement. Fix or reword any live claim.
2. **Stale tips in prose.** Every tip hash cited in AGENTS.md, docs/TRACKS.md, README.md,
   RELEASE-CHECKLIST.md, build/PROGRESS.md must resolve to a real commit
   (`git cat-file -t <hash>`). Stale tips (pre-repair, retired worktrees) get
   marked as history, not live.
3. **Dead links.** Every cited path in docs/ and *.md must exist on disk. Every file:line
   citation must resolve. Use `tooling/` citation verifiers where they exist.
4. **Untracked debris.** `git status --short` at repo root and in every worktree.
   Only known artifacts (logs, bundles, recovered trails) may be untracked.
   Commit or remove the rest.
5. **Doc contradictions.** AGENTS.md, docs/TRACKS.md, README.md, RELEASE-CHECKLIST.md,
   build/PROGRESS.md, build/TRACK-SPLIT-CHECKLIST.md must agree on tips, task
   state, open findings, and branch names.
6. **Docs lint.** `tooling/docs_masterclass_lint.py --docs` must report FAIL=0.
7. **Ignored artifacts.** build/track-*, build/logs/, worktree binaries must be ignored
   or committed per the program's ignore rules. No stray binaries in the tree.

## Evidence emitted

A markdown checklist at `build/repo-finalization-audit-<date>.md` with one row per
check: PASS / FAIL, evidence (command + output snippet), and the fixing commit if
FAIL was repaired in-round. This skill never edits code; it reports.

## Red flags

- A check that "passes" because the search term was misspelled.
- Treating a retired tip as live because it still exists in git history.
- Skipping the worktree status check (debris hides in worktrees).
- Counting docs lint WARN as FAIL or FAIL as WARN.
