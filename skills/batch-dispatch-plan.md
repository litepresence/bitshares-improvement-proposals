# Batch-dispatch plan — audit repairs + 8 BSIPs (≤5 subagents)

Rule: ≤5 concurrent; 1 per branch; survey/plan can parallelize; actual repair commits sequential per branch (cycle law §9). Launch next sub when prior is compiling/not blocked on merge.

Assignments (each gets its branch + dirty-state + REPAIRS.md + this punchlist):

S1 — bsip3-xclaim-accs (worktree bsip3-xclaim-accs, tip 0d640214, clean). Task: confirm REPAIR-01/04 complete, verify f_gate_tests 4/4, check audit seat bsip3-xclaim-accs-audit.json for any un-adjudicated Critical (2C raw), propose any remaining repair. Report build/task-S1-report.md. No new commit unless gate requires.

S2 — bsip4-n-asset-pools (worktree bsip4-n-asset-pools, tip 3af9279b, dirty M reciprocity_evaluator.cpp + bsip48_75_tests.cpp, build/bsip4-repair07 running). Task: complete REPAIR-07 (green test), then survey REPAIR-27/29/30 needs; fix dirty files; verify build exit 0; check merge conflict potential with bsip5/bsip8. Report S2-report.md. Committable only after green.

S3 — bsip5-open-order-rewards (worktree bsip5-open-order-rewards, tip 1badfa6e, clean). Task: verify 43/0C from deepen-3; check proposal visitor reciprocity (REPAIR-29 cross-applies); check open-order reward arithmetic; confirm no new Critical. Report S3-report.md.

S4 — bsip6/bsip7/bsip8 group (worktrees bsip6-stableswap fc638af7, bsip7-ab-pool cdb16393, bsip8-pamm 871b8764; all clean). Task: survey each for implementation errors; note any cross-branch exposure (H2-24-01, G2 op109, pool math); produce one combined report (S4-report.md) with per-BSIP findings; propose fixes only after verification. No commit until S2 merge state known.

S5 — track-g1-stealth / ViewKey family (worktree track-g1-stealth 5a529837, clean) + G1 audit seats (g1-stealth-implementation-audit.json, clsag-bp-plus-auditor.json). Task: family-adjudicate REPAIR-12/20 against G1-05 gate; do NOT open new repair; verify stealth_evaluator.cpp line anchors; confirm no new ViewKey critical; document gate status only. Report S5-report.md.

Launch order (do not wait for full binary — compile/verify is the signal):
- Launch S1 + S3 + S4 + S5 immediately (independent branches, clean state).
- Launch S2 after S1/S3 confirm no shared evaluator conflict (they don't share files; can go now, but S2 is the only dirty one — start S2 right after, build may take 3-5 min at -j2).
- Cap at 5; if any sub finishes early, do not replace with new until current wave closes (prevents merge chaos).

Every subagent brief: read punchlist.md first, then branch's REPAIRS.md, then dirty-state, then report to build/task-SN-report.md; never edit contracts/; never activate; record auditor_model if writing audit JSON.
