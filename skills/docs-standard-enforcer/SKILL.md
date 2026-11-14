# Skill: docs-standard-enforcer

Use when enforcing the masterclass documentation standard in
`docs/superpowers/specs/2026-10-19-docs-masterclass-standard.md`. Wraps
`tooling/docs_masterclass_lint.py` so the standard has an owning skill.

## Procedure

1. Run `tooling/docs_masterclass_lint.py --docs` (or `--file <path>` for a single
   file). Capture FAIL and WARN counts.
2. For every FAIL: read the file, fix the violation per the standard (no
   placeholders/TBD/TODO in committed docs; open items go in the spec's "does not
   close" section or the activation checklist with an owner and a gate).
3. For WARN: triage. Fix those that are cheap and clearly wrong; record the rest
   as accepted with a one-line rationale.
4. Re-run until FAIL=0. WARN=0 is the target for community-facing docs; WARN>0
   is acceptable for internal ledgers with a recorded rationale.
5. Commit the fixes with a message citing this skill and the lint counts.

## Evidence emitted

`build/docs-standard-enforcer-<date>.md` with the lint command, before/after
counts, the list of fixed files, and the commit hash.

## Red flags

- Editing code to satisfy a docs lint (the standard is prose-only).
- Treating WARN as FAIL (blocks progress) or FAIL as WARN (hides defects).
- Leaving a TODO/TBD in a committed doc because "it's internal".
- Running the lint on a subset of docs and calling the whole tree clean.
