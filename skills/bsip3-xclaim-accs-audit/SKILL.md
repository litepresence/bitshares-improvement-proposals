---
name: bsip3-xclaim-accs-audit
description: >
  Use when auditing the BSIP-3 / F-track foreign-chain settlement work:
  foreign transaction oracle (equality consumption, confirmation minimums,
  bond coverage) and ACCS orderbook settlement (gateway, offer lifecycle,
  griefing bond, escrow, replay guard, bounded state). Vantage: attestation
  ceiling made explicit — no in-consensus SPV, bonds are the backstop.
version: 1.0.0
license: MIT
allowed-tools:
  - read
  - grep
  - glob
  - bash
---

# Skill: bsip3-xclaim-accs-audit

Audit BSIP-3 against its BSIP and the settlement/foreign-tx-oracle design specs.
Vantage: attestation honesty — the chain never verifies a foreign payment;
every safety claim must reduce to a bond, a delay, or a fail-closed rule.

## Checklist (in order, every item)

1. **Op/object inventory.** Confirm tags 110–117 and the three new object types
   (gateway, orderbook, offer, plus the consumed-transaction record) in
   `protocol/operations.hpp` and `protocol/types.hpp`. Any renumber or missing
   tag is Critical.
2. **Wire append-only.** New fields append last; no existing field moves or
   reinterprets. Wire-format pinning holds across builds with and without the
   hardfork flag.
3. **Dual hardfork gating.** Settlement evaluators assert (a) settlement gate
   passed, (b) oracle gate passed, and (c) fail-closed ordering refusing a
   schedule that places settlement before oracle. Proposal-smuggled activation
   is Critical.
4. **Foreign-tx oracle equality consumption.** Consumption matches on tx id,
   amount, foreign asset, event kind = send, and confirmation count at or above
   the chain minimum. Any mismatch stops the consumer; no ratio path. Confirm
   confirmation minimums are constants (6 BTC / 12 ETH / 32 SOL), not governance
   parameters.
5. **Bond coverage.** One bond covers both price and transaction series on the
   same oracle object; a transaction lie forfeits the same bond as a price lie.
   No separate bond pool.
6. **Gateway scope.** Committee declares corridors once per (chain, foreign
   asset); no per-base-asset approval; deactivation blocks new offers and leaves
   existing ones alone. Replay guard must be instance-independent and append-only
   across gateway redeclaration.
7. **Offer lifecycle.** States open → matched → fulfilled/expired/cancelled with
   the exact transitions named in the BSIP. Terminal states immutable; expired
   open offers release escrow in the maintenance sweep; service-level timeouts
   are never auto-claimed.
8. **Escrow conservation.** Create deducts base into escrow; cancel releases;
   fulfill releases to buyer; claim returns seller's escrow plus griefing bond.
   No double release on any path; conservation identity holds at every transition.
9. **Griefing bond.** Minimum enforced at match; denominated in base asset so
   its value does not depend on swap completion. Claim refused before deadline;
   fulfill refused after deadline.
10. **Bounded state.** Offers per account and per orderbook capped; consumed-tx
    record bounded; terminal offers removed from orderbook lists after the cleanup
    age while the object remains for audit. Unbounded growth is Critical.
11. **Clock ordering.** Oracle guard delay must not exceed the service-level
    window; otherwise every matched offer expires and sellers collect bonds from
    honest buyers. Verify the ordering is checked or documented.
12. **Recipient-field ceiling.** The foreign tx record has no recipient field;
    consensus cannot verify BTC reached the seller's address. Confirm this is
    documented as the permanent attestor trust ceiling, not implied away.
13. **Reorg handling.** Confirmation minimums price reorg risk but do not
    eliminate it; dispute machinery is the backstop. Confirm no false finality
    claim.
14. **API surface.** Query methods for gateway/orderbook/offer state appended to
    database API; names and counts verified.
15. **Test vectors.** Wrong amount, replayed payment, early claim, maintenance
    sweep, deadline expiry, gateway redeclaration replay, and reorg all have
    consensus-path tests. A defense with no test is a finding.

## Output

Numbered PASS/FAIL/QUALIFIED list with `file:line` evidence. Every FAIL gets
the smallest concrete fix. Ambiguity is a finding.

## Machine-Readable Output (required)

```json
{"skill":"bsip3-xclaim-accs-audit","auditor_model":"<exact model string — mandatory>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"F3-01","severity":"Critical|Important|Minor|Note","check":"op-inventory|wire|gating|equality-consumption|bond-coverage|gateway|lifecycle|escrow|griefing-bond|bounded-state|clock-ordering|recipient-ceiling|reorg|api|test-vectors","verdict":"PASS|FAIL|QUALIFIED","evidence":"file:line","fix":"smallest concrete fix"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."]}
```