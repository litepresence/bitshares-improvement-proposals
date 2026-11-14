---
name: afk-keep-rolling
description: Use when the user says they are going AFK, away from keyboard, or says keep rolling until they return during a repair or audit loop.
---

# AFK Keep Rolling

## Overview

AFK means the agent is the driver: no idle stops, no questions, no waiting. The todo list carries one permanent tail item so the loop survives batch boundaries.

## When to Use

- User says "going afk", "away for a bit", "keep rolling", "keep going until I get back", or equivalent during iterative work (repair triage, audit rounds, remediation batches)
- Symptoms: punch-list batches remain, agent would otherwise end its turn after one batch and wait

When NOT to use: user is present and responsive (normal todo discipline applies); single one-shot tasks with nothing queued behind them.

## Contract

1. **Tail invariant.** The last todo item is always the keep-rolling item (e.g. "Keep rolling until user returns — next punch-list batch, no idle stop"). It stays `pending` until the user sends a new message. Never mark it complete while the user is away. All new round items insert ABOVE it.
2. **No idle stop.** After every batch commit, immediately begin the next logical punch-list batch (next REVIEW §XX rows, standing carries, or next audit round). A turn never ends with a clean tree and no next batch started.
3. **No questions while AFK.** The question tool would block forever — decide per §Contract-4 rules below and log assumptions in the report/commit message instead.
4. **Decision rules (self-contained; no external skill required).** Fix-point rule: when ambiguous without user, choose the option that (1) preserves prior committed/reviewed state, (2) prefers reversible over irreversible, (3) logs the assumption. Union-first: combine non-conflicting directives rather than picking. On-disk truth: verify files with tools, never trust self-reports.
5. **Keep the established pattern.** Batch discipline is unchanged: dispatch contract, verify on disk, small focused commits, fold deltas into REVIEW/status/QC. This skill only governs CONTINUATION, not quality.
6. **Commit-policy guard.** This skill never overrides the workspace commit rule (here: commit only when asked / per brief on feature branches). Where commits are barred, "commit the batch" reads as "record to ledger + reports."
7. **Circuit breaker.** Stop and report position if: BLOCKED twice consecutively on the same item with no new information, or any task exceeds its brief with no path forward. Heartbeat: one ledger line per completed task so a returning user reconstructs the run.
8. **Stop only when:** the user sends any new message (stop, report position), or zero Blocking/Major/Minor items remain anywhere (resolve the final petty round too, then stop and report shippable).

## Red Flags — STOP and re-read this skill

- Ending a turn with "waiting for you" / "let me know" while AFK flag is set
- Asking the user anything before continuing
- Marking the keep-rolling tail item complete while the user is still away
- Starting a batch without inserting it into the todo list first
- Committing generated artifacts (`dist/`, `__pycache__/`, `tmp/`, backups)

## Quick Reference

| Situation | Action |
|---|---|
| Batch committed, items remain | Insert next batch above tail, start it same turn |
| Ambiguous fix, user unreachable | Follow `final-repair-triage` fix-point rule, log in commit |
| New user message arrives | AFK over: stop loop, mark tail complete, report |
| Punch list fully empty | Final petty round, then stop and report |
