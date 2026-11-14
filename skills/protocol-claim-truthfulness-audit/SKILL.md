# Skill: protocol-claim-truthfulness-audit

Use when auditing every SAFETY CLAIM the program makes in prose against the code that
claims to implement it. Vantage: not "is the spec well-written" (that is
`bsip-readiness-review`) but **"does the code do what the document says it does, sentence
by sentence"**.

This skill exists because round-6 recon already found concrete prose/code divergences on
the frozen tips, which means the pattern is systemic, not incidental:

- ~~`issuance_evaluator.hpp:57-58` claims mint "mints FXR to the reserve coin holder
  burn-first"; the mint path only credits, never burns.~~ **RESOLVED — this finding is now
  STALE and must not be re-filed.** The header was repaired and now states the opposite
  ("FXR burns nowhere on this path... the burn-first rule lives on the wind-down junior claim
  path"), and burn-first IS enforced there: `winddown_claim_evaluator::do_apply` burns FXR at
  `vault_lifecycle_evaluator.cpp:563,576-588`. Kept as the worked example of a **third-order**
  defect: a finding that was correct once, recorded in a skill as a live divergence, went stale
  when the code was repaired, and would have been re-filed against a header that now says the
  opposite. Always re-verify a recorded finding against the CURRENT code before repeating it.
- `h2_issue_band_edge_bps` (`issuance_ops.hpp:143-156`) supplies the price-edge grid the
  band mechanism is described in terms of; no consensus path calls it (`issuance.cpp:66` is
  a `using` declaration; only `tests/tests/h2_issuance2_tests.cpp:540-547` does).
- AGENTS.md objective 5 claims legacy assets are untouched; G1's
  `pay_market_fees` routing (`db_market.cpp:2687-2696`) diverts 5% of the fee increment for
  every asset including `bit*`/`HONEST.*`/gateway UIAs. **RESOLVED — this claim was itself
  stale.** Routing requires `parity_fee_share_opted_in` (`stealth_evaluator.cpp:94-98`), the
  flag is an `fc::optional<bool>` ABSENT by default (`asset_ops.hpp:55`), and
  `db_market.cpp:2691-2693` states legacy assets credit 100% "by construction" (fixed by
  `f4de376e`). Kept here as the worked example of a SECOND-order defect: a finding written
  into a skill's own motivation became false when the code was repaired, and survived three
  re-pins because the overlay told auditors to distrust the very `file:line` that was right.
- Test names understate coverage: `h2_op_tags_pinned_83_to_100` and
  `h2_object_ids_pinned_25_to_37` (`tests/tests/h2_activation_tests.cpp:507,536`) now assert
  tags 99–103 and ids 1.37/1.38.
- Round-5 artifacts recorded 2 endpoints per H1 object; the tree exposes six, and two of the
  names are singular (`get_oracle_by_name`, `get_reference_asset_by_symbol`).

Documents to audit, in this order: `/workspace/AGENTS.md`, `/workspace/README.md`,
`/workspace/RELEASE-CHECKLIST.md`, `docs/superpowers/specs/2026-09-29-bsip1-hybrid-economics.md`,
`docs/superpowers/specs/2026-09-29-bsip2-stealth-transfers.md`,
`/workspace/build/ACTIVATION-CHECKLIST.md`, then the per-op header comments added by the
comment-only doc commits. Frozen tips as of round 18: H1 `45a4666e`, H2 `efcfd0ce`,
G1 `8de355a3` — re-verify tip before re-file. History only (do not cite as current):
round-17 tips H1 `a9c5dfa6`, H2 `0d1cb873`, G1 `5a580663`; pre-repair H1 `32285c5d`,
H2 `91e866a3`, G1 `bb6a216f`; repair lineage H2 `0e6ea947` (the R9-VERIFY
`9e73ad8b`/`6448347f` line continues to `0d1cb873`), G1 `1550ca41` (the R9-VERIFY
`a2dd340d`/`9d4b5d85` line continues to `5a580663`).
Durable fix (round-16 TR-31): express pins as greppable constructs and re-derive
counts with the tool (`tooling/r13k1_overlay_pin_census.py`), never trust a bare
count — the overlay count itself moved 174 → 180 → 188 (round-16/17/18 rows).

## Checklist (in order, every item)

1. **Claim extraction.** Pull every sentence in the documents above that asserts a
   mechanism, a constant, a count, a safety property, or a NOT-BUILT fence. Build a table:
   `claim_id | document:line | quote (short) | code symbol it names | verified?`.
   Aim for completeness over elegance; a claim that names no code symbol is itself a finding.
2. **Count claims.** Every "N ops", "M objects", "K endpoints", "tags A–B", "test count",
   "suite count" claim: recompute from the tree and report MATCH or MISMATCH with both
   numbers. Include: op tags per branch (`operations.hpp`), object ids per branch
   (`protocol/types.hpp`, `chain/types.hpp`), `database_api` endpoint names, `wallet_api`
   additions, suite and case counts cited in the BSIPs versus the logs in
   `build/logs/` — TIP-HONESTY RULE: history-only logs (filename + count
   verified) are never current-tip evidence. Current full-binary evidence at the
   round-18 frozen tips: H1 `track-b-h1-45a4666e-r18-full.log` 657 cases,
   H2 `track-b-h2-efcfd0ce-r18-full.log` 826 entered (2 PILOT-NARROWED disabled
   cases excluded by the gate) exit 0 at `efcfd0ce`, G1
   `track-g1-8de355a3-r18-full.log` 851 exit 0 at `8de355a3`. History only:
   round-17 logs H1 `track-b-h1-a9c5dfa6-final-full.log` 653, H2
   `round17-h2-full.log` 813 at `0d1cb873`, G1 `round17-g1-full.log` 849 at
   `5a580663`; `r9-verify-h2-full.log` 802/802 at
   `9e73ad8b`; `r9-verify-g1-full.log` 845/845 at `a2dd340d`;
   `track-b-h2-0e6ea947-final-full.log` 785 + 2 PILOT-NARROWED skips;
   `track-g1-r15c-final-full.log` 749; the specialist CLSAG/BP+ pin at
   `e7d978c3` 704 cases (`track-g1-e7d978c3-final-full.log`) — a document
   claiming the specialist sign-off covers `612b354e` is a finding; pre-repair
   `track-b-h1-final-full2.log` 651/651, `track-b-h2-final-full.log` 749/749,
   `track-g1-final-full.log` 667/668 + 1 WARN. Re-verify before re-file
   (`git -C <worktree> rev-parse HEAD`, re-grep the tip, fresh full-binary log
   filename required — never cite a bare count without its log).
3. **Constant claims.** Every number in a document that also exists in code: recompute. List
   mismatches with both values and both file:line. Minimum set: rmin 13000 bps, surplus
   20000 bps, door cap, conversion fee 30 bps, redeem vest 86400 s, offset vest 604800 s,
   pending-vest cap 64, offset claimant cap 64, exit fee 200 bps, K1/K2 bps, band count 7,
   tick 250 bps, challenger floor $837.50, hard-cap 3, 48h window, 80/15 burn/pool split,
   $83.75 finder cap, secure_ratio 20000, maintenance 150%, 36h top-up, 15% haircut, 512 /
   interval, recovery 15000 bps, joint cap ≤70%, parity 500 bps, per-block subsidy cap 50,
   burn warn/critical 50/80, ring shape 11–32, decoy 24h/40%/100-block, blind per-block cap
   100, freeze votes 3, expiry 86400 s (`GRAPHENE_STEALTH_FREEZE_VOTES_REQUIRED` /
   `_VOTE_EXPIRY_SEC`, `libraries/protocol/include/graphene/protocol/stealth.hpp:210-211`
   at 8de355a3 — the pre-Task-6 "2/48h" pin is stale and must not be re-filed).
4. **NOT-BUILT fences.** Each BSIP fence that says something is deliberately absent
   (spec-only items: decaying reputation, 7d timelock, 10% ESM, FXR bar, cap k1/k2 in an
   earlier framing, 90-day seasoning, disclosure enforcement; H1's "history/reputation/
   delayed-price absent by design"). Verify each is STILL absent at the tip. A fence that has
   quietly been built is as dangerous as a claim that was never built — both mean the
   document misleads. Note that `seasoning_verify_operation` (tag 98) and the 90-day
   seasoning clock (`805ba21f`) were built after some fences were written: confirm which
   fences were updated and which now contradict the tree.
5. **Safety-property sentences.** Every "fails closed", "never", "always", "cannot",
   "atomic", "permanent", "unchanged", "no cliff", "burn-first", "bounded". For each,
   name the enforcing code line, or record it as an UNENFORCED-CLAIM. Unenforced safety
   claims are the highest-severity output of this skill: they are the sentences that would
   appear in a BSIP filed publicly.
6. **Test-evidence claims.** Any statement of the form "N/N green", "verified", "tested",
   "replayed", "soaked". Verify against `build/logs/*` and `build/PROGRESS.md`, and note
   where the log, the ledger, and the document disagree. Flag any claim of full-binary
   evidence that rests on a scoped-suite run.
7. **Status/date claims.** Any date, commit id, tip, or count that pins the state of the
   program. Verify `git log` per branch and confirm the tip in each document. Note the
   deliberate distinction the program makes between "code-complete" and "activation
   authorized" and confirm no document blurs it.
8. **Cross-document consistency.** The same mechanism must be described the same way in
   AGENTS.md, README.md, the BSIPs, the code headers, and the activation checklist. List
   every mechanism described differently in two places, with both file:line.
9. **Repair-size estimate.** For each divergence, classify the repair: DOC-ONLY (change the
   prose), CODE (change behavior), or BOTH (narrow the claim AND add the missing code).
   Estimate effort in S/M/L with reasoning. The repair agent needs this to sequence doc
   deltas after code work, never before.

## Output (JSON, required)

```json
{"skill":"protocol-claim-truthfulness-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"all trees + repo docs","tip":"45a4666e+efcfd0ce+8de355a3 (round-18 frozen tips; re-verify HEAD before filing; a9c5dfa6/0d1cb873/5a580663 round-17 tips and 9e73ad8b/a2dd340d/e7d978c3/1550ca41/0e6ea947 history only)",
"findings":[{"id":"TR-01","severity":"Critical|Important|Minor|Note","check":"claim|constant|fence|count|safety-sentence","verdict":"PASS|FAIL|QUALIFIED","evidence":"doc:line + code:line","fix":"smallest concrete fix, or the exact words to change"}],
"gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```

Critical = an UNENFORCED safety claim or a NOT-BUILT fence that is now built. Important =
a wrong constant, count, or name in a document that will be filed. Note/Minor = wording.
For every finding, give the replacement sentence verbatim so the repair agent never has to
guess what the document should say.