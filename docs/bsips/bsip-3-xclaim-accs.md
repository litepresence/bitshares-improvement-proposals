    BSIP: (number assigned at filing; house label BSIP-3)
    Title: Bonded cross-chain settlement: an atomic cross-chain swap (ACCS) orderbook and a foreign-chain transaction oracle
    Authors: litepresence (HONEST)
    Status: Draft
    Type: Protocol
    Created: 2026-10-25

# Abstract

BitShares users who want real BTC must trust a third-party issuer: the token is an IOU and the coins sit in the issuer's wallet. This proposal replaces that promise with escrow and bonds. An atomic cross-chain swap (ACCS) completes both legs together or neither. A holder posts an on-chain offer selling any BitShares asset for BTC, a BTC holder matches it and sends BTC from their own wallet, and a bonded oracle that watches Bitcoin reports that payment to consensus; only then does the chain release the seller's escrowed assets to the buyer (escrow: the chain holds the coins until conditions are met). No vault custodies foreign coins, no committee approves individual swaps, and a false published record forfeits a bond. The BSIP-3 may be approved alone or combined with the other two proposals.

Pitch: real BTC on BitShares, settled by escrow and bonds instead of a custodian's word.

# Motivation

A gateway is an issuer that promises to hold the real asset elsewhere and redeem its token on demand, so a gateway token is an IOU, and today that promise is enforced by nothing but the issuer's reputation and revocable permissions (the settlement layer design spec, `docs/superpowers/specs/2026-10-20-f2-accs-orderbook-settlement.md`, problem statement). When the promise fails, holders discover that the coins were never theirs. BitShares has run on custodial gateway IOUs for years, and its feed producers once posted prices with no bond behind them, so this program prices every trust relationship in bonds instead of asking the community to trust again (AGENTS.md, section 4, settled decisions).

There is a hard ceiling behind that problem, and this proposal is shaped by it rather than denying it. BitShares has no general smart contracts, so no proof of a Bitcoin payment can ever be verified inside consensus: no in-consensus light client, no SPV relay (simplified payment verification against a foreign chain's headers), no programmatic slashing of a foreign-chain liar (AGENTS.md, section 4, settled decision 6). Anything that moves foreign value across this chain therefore rests on one of two foundations: custody, where somebody holds the coins for you, or attestation, where somebody posts a bond and swears under penalty that a payment happened. The BSIP-3 chooses attestation and prices it in bonds, disputes, and fail-closed rules.

A price oracle reports a ratio. A foreign-chain transaction oracle reports that a specific transaction happened on another chain: which transaction, of what size, confirmed how many blocks deep (the transaction oracle design spec, `docs/superpowers/specs/2026-10-20-f1-foreign-chain-tx-oracle.md`, section 0). The settlement proposed here needs the second kind. The two kinds share bond machinery but differ in what they report and how a consumer uses them, so this BSIP names the transaction oracle the transaction oracle and the settlement layer built on it the settlement layer.

The program first attempted this bridge as vaults: BTC locked on Bitcoin with a BitShares IOU minted against the locked coins. That design was fully built and audited, and was then retired in favor of peer-to-peer settlement. Every vault surface was removed: the foreign mint, redeem, challenge and unlock operations, the foreign backing fields, and the coverage machinery. Two things survived the retirement. The transaction oracle from the first step of that design became the transaction oracle used here unchanged, and the swap idea was recast from lock-and-mint into orderbook settlement as the settlement layer (split design document, `docs/superpowers/specs/2026-10-24-three-track-split-design.md`, sections 1 and D6). This BSIP is a simpler design that came out of that retirement, not a rebrand of the vault approach.

The replacement works peer to peer. Alice holds a BitShares asset and wants real BTC. Bob holds real BTC and wants Alice's asset. Alice posts an offer on an on-chain orderbook, locking her asset in escrow. Bob matches the offer and posts a griefing bond, which is his own money held by the chain: Alice keeps it if he never pays and he gets it back when he delivers. Bob then sends BTC from his own wallet to the address Alice wrote in the offer. Bonded attestors (off-chain operators who watch foreign chains and publish transaction records behind a bond) report the payment. Consensus compares the report to the offer, and on an exact match Alice's escrowed asset releases to Bob and his bond returns. If Bob never pays, Alice claims his bond and gets her escrow back. No vault, no custodian, no per-swap committee vote (the settlement layer design spec, sections 0 and 3). The only trust assumption in the flow is the attestor bond, stated openly rather than papered over.

BSIP-3 is one of three independently votable proposals in this program. BSIP-2 is stealth transfers, BSIP-1 is the bonded price oracle and its smartcoins, and BSIP-3 is this proposal, each a separate upgrade. The community may approve any one, any two, or all three, and the wire format is pinned so that a signed transaction means the same thing in every configuration (BSIP split design, sections 1, 3 and D2). Nothing here needs the other proposals to be useful, and nothing here is switched on by their approval.

Status, stated plainly. The BSIP-3 is implemented and has been independently audited by two separate review efforts, whose reports live at `build/accs-audit-2026-10-24.md` and `build/f1f2-multiskill-audit-2026-10-24.md`. Those audits found open defects that are being repaired, and the Risks section names each one in plain words rather than summarizing the code as finished. Activation is not authorized. Nothing changes for anyone until every gate in `build/ACTIVATION-CHECKLIST.md` closes, which includes the human gates this document cannot close. The first trading pair is BTC against BTS, and it must prove coverage in a pilot capped at $20,000 face value for 90 days of live observation before any USD issuance is allowed to depend on it (`docs/superpowers/specs/2026-10-17-pilot-decision.md`, sections 1 through 3; AGENTS.md, section 4, settled decision 2).

# What changes for you

- If you hold legacy assets, nothing about your balances, markets, or fees changes, and after activation you may also sell anything you hold directly for BTC through an escrowed offer.
- If you trade, you gain a new on-chain orderbook per asset against each declared foreign chain, where the price is fixed the moment a counterparty matches and settlement happens within a published deadline.
- If you run a witness node, your software gains eight appended operations and three new object types behind their own hardfork gate, and if the proposal is not approved, its operations are declared but refused when a node tries to apply them.
- If you operate a gateway, your existing IOUs are untouched, and you may run attestors or compete with the swap book on the same chain.
- If you develop on BitShares, you get operations at fixed wire tags, fields that append and never insert, and an oracle subsystem that is byte identical whichever combination of proposals a build contains.
- If you hold BTC, you send coins from your own wallet to the address written on an offer and receive the buyer's BitShares assets directly, with your payment protected by the seller's on-chain escrow and the attestors' bonds rather than by the other party's promises.

# Specifications

## 1. Track structure and activation gates

The registry is pinned across all BSIP configurations: operation tags 0 through 77 are the existing chain operations, then the price oracle and smartcoin proposals, then stealth transfers, and the BSIP-3 owns tags 110 through 117 (BSIP split design, D2). Every branch declares every operation, object id, and hardfork name, so ids never shift with the approved set; an operation whose proposal was not approved has no evaluator registered and fails closed at apply (BSIP split design, D4).

the transaction oracle keeps the oracle subsystem's gate, the oracle hardfork, because the oracle subsystem is shared machinery that ships with whichever proposal needs it. the settlement layer receives its own gate, and its evaluators assert three things: that the settlement gate has passed, that the oracle gate has passed, and a fail-closed ordering check that refuses a chain schedule placing the settlement gate before the oracle gate (BSIP split design, D5). A schedule in which BSIP-3 activation precedes the oracle activation it depends on is refused rather than clamped.

## 2. the transaction oracle, the foreign-chain transaction oracle

the transaction oracle adds one oracle series kind for foreign transactions, distinct from price series, published by a new operation (the transaction oracle design spec, section 2.1). Each published record carries the foreign chain identifier, a fixed-size foreign transaction id, an event kind of lock, unlock, or send, the amount in the foreign chain's smallest denomination, an index identifying the foreign asset (zero meaning the chain's native asset), the foreign block height at which the transaction confirmed, and the number of confirmations observed.

Each foreign chain has a minimum confirmation count, proposed at 6 for Bitcoin, 12 for Ethereum, and 32 for Solana. These are protocol constants, not governance parameters, so changing one requires a hardfork (the transaction oracle design spec, section 2.2).

Producers post one bond against the oracle object, and that bond covers both price series and transaction series production, so a producer who lies about a transaction forfeits the same bond as a producer who lies about a price (the transaction oracle design spec, section 2.3). The guard machinery is shared with the price oracle: a delay queue before promotion, freezes, rate limits, promotion proofs, and a challenge path where a sustained challenge slashes the bond and winds the producer down. A missed publication deadline freezes and reimburses rather than slashing, because liveness failure and dishonesty are different faults (the transaction oracle design spec, sections 4 and 6; the settlement layer design spec, section 7.3).

Consumption is by equality, not by ratio (the transaction oracle design spec, section 3). A consumer matches a promoted record against a specific expected transaction: same transaction id, same amount, same foreign asset, event kind of send, and a confirmation count at or above the chain minimum. A match lets the consumer proceed; any mismatch stops it, and the deadline consequences fall where the consuming protocol puts them.

## 3. the settlement layer, the gateway, the orderbook, and the offer

A gateway in this design is not a custodian. It is a consensus record, created by a committee member, binding a foreign chain and optionally a foreign subtoken to an the transaction oracle oracle (the settlement layer design spec, section 2.1). The committee declares a Bitcoin gateway, or a USDT-on-Ethereum gateway, once per chain and foreign asset pair, and after that any BitShares base asset may open an orderbook against it. There is no per-base-asset committee approval, so the committee's power is to open and close corridors, not to approve trades. Deactivation blocks new offers while leaving existing ones alone.

An orderbook is keyed by base asset, foreign chain, and foreign asset, and is created lazily by the first offer against that triple (the settlement layer design spec, section 2.2). An offer records the seller, the base amount locked in escrow, the minimum foreign amount the seller wants, the seller's foreign-chain recipient address, an optional expiry block, and a state of open, matched, fulfilled, expired, or cancelled (the settlement layer design spec, sections 2.3 and 2.4). The transitions are exactly: open to matched when a buyer matches, open to cancelled when the seller cancels, open to expired when the maintenance sweep passes the expiry, matched to fulfilled when a valid foreign transaction record is consumed, and matched to expired when the service-level deadline passes and the seller claims. Fulfilled, expired, and cancelled are terminal.

Seven operations implement the flow (the settlement layer design spec, sections 3 and 6). Creating an offer deducts the base amount from the seller into escrow and validates that the chain has an active gateway and that the recipient address is well formed. Cancelling releases the escrow, and only the seller may cancel, and only while the offer is open. Matching requires the buyer to post a griefing bond at or above the minimum, sets the offer's service-level deadline to the current block plus a fixed window, and moves the offer to matched, at which point the seller can no longer cancel. Fulfilling releases the escrow to the buyer and returns his bond; it requires the promoted foreign record to match the offer on transaction id, amount, foreign asset, event kind, and confirmation count, and it is refused once the deadline has passed. Claiming the griefing bond transfers it to the seller and returns her escrow, and it is refused before the deadline. Declaring and deactivating gateways are the two committee operations.

Maintenance integration is small and explicit (the settlement layer design spec, section 4). Expired open offers release their escrow in the maintenance sweep. Service-level timeouts are never claimed automatically, because automated fee farming would follow, so the seller must claim explicitly. Terminal offers are removed from the orderbook's lists after an age, while the object itself remains for audit.

The constants, as proposed: a minimum offer lifetime of 100 blocks, a service-level window of 86,400 blocks, roughly three days at the chain's three-second block time, a minimum griefing bond of 8,375 USD cents' worth of the base asset, and a terminal cleanup age of 100,800 blocks (the settlement layer design spec, section 5).

## 4. One swap, end to end

Alice calls create offer, naming BTS as the base asset, Bitcoin as the chain, her own Bitcoin address as the recipient, and a minimum BTC amount as her price; her BTS leaves her balance and sits in escrow on the offer. Bob matches the offer and posts his griefing bond. Bob sends the agreed BTC from his wallet. Attestors running Bitcoin listeners observe the transaction, wait out the confirmation minimum, and publish a foreign transaction record. Once that record clears the oracle's delay queue and is promoted, anyone may call fulfill. Consensus compares the promoted record to the offer, finds every term equal, releases the BTS to Bob, returns his bond, and marks the offer fulfilled. If instead Bob never sends, or sends the wrong amount, the record will not match, the deadline passes, Alice calls claim griefing, and she leaves with her BTS plus Bob's bond.

One term of that comparison cannot be enforced by consensus, and the design says so rather than implying otherwise. The foreign transaction record has no recipient field, so consensus cannot verify that the BTC reached Alice's stated address; that leg is attested by the bond of whoever published the record, not verified by the chain (audit report, `build/accs-audit-2026-10-24.md`, the note on recipient checking; audit report, `build/f1f2-multiskill-audit-2026-10-24.md`, the same finding on the foreign transaction value). This is the attestor trust ceiling of the whole proposal, and it is permanent under the current object format.

## 5. Client-side attestor infrastructure

Attestors are client-side software, not consensus (the settlement layer design spec, section 7). For each supported foreign chain, an attestor runs a listener that watches foreign blocks for transactions to addresses named in open offers, extracts the transaction id, amount, recipient, and confirmation count, and publishes the record. Subtoken support adds event-log decoding on top, since a token transfer on Ethereum is a contract event rather than a native transfer. Consensus verifies none of this; the bond is what stands behind a published record.

Attestors are paid from production fees on the operations that consume their records, from bond yield shared with price oracle producers, and from a share of slashed bonds distributed to successful challengers. The fee split and yield rate are committee-governable parameters, not hardcoded constants (the settlement layer design spec, section 7.2). The attestor allowlist per foreign chain is a committee operation, and an attestor must post a bond, run the listener for the chains they declare, and publish within the deadline (the settlement layer design spec, section 7.3).

## 6. Out of scope by design

Matching is first in line by creation block, not price-time priority, and one offer carries one base asset (the settlement layer design spec, section 8). There are no partial claims, no multi-leg offers, no two-sided swap inside a single transaction: the reverse direction, foreign coins for a BitShares asset, is simply another offer from a different seller (the settlement layer design spec, section 1, decision 6). Foreign chain fees are the buyer's problem, not the protocol's. Confirmation rules beyond the three named chains are not proposed here.

# Rationale

Settlement by orderbook replaced settlement by vault for reasons that were earned, not chosen lightly. The vault model concentrated foreign custody in one contract, which then needed coverage ratios, challenge windows, wind-downs, and seizure mechanics to stay honest, and each is another surface where a captured vote or a bad oracle reading can hurt a holder. The swap model has none of those surfaces because nobody custodies the other side: each participant self-custodies until the moment they choose to trade, and the chain holds only its own native assets in escrow (the settlement layer design spec, section 1, decision 7; BSIP split design, section 1). A design with fewer ways to hurt its users is worth more than a design with more features.

The attestor model over a verifier model is a ceiling, not a preference. Consensus cannot verify a Bitcoin payment, so this BSIP names its trust assumption and bonds it, consistently with the program-wide decision that no in-consensus SPV or programmatic slash is claimed (AGENTS.md, section 4, settled decision 6). Bonds, transparency, and vote arbitration with fail-closed rules are the entire backstop. A reader who prices attestor collusion above the bond can decline to use the proposal without the chain lying to them about why.

An on-chain orderbook beat an off-chain bulletin board because consensus-enforced state transitions are the product (the settlement layer design spec, section 1, decision 1). An off-chain book would put matching, cancellation, and dispute timing back in the hands of a website, which is the exact trust problem this proposal exists to remove. The price is fixed at match time so no participant carries price movement risk after committing (the settlement layer design spec, section 1, decision 4). The griefing bond is denominated in the base asset so that its value does not depend on the swap completing (the settlement layer design spec, section 1, decision 5). The flow is simplex, one direction per offer, because a single-transaction two-sided swap would need both chains' state in one consensus step, which this chain cannot have.

Committee power is deliberately narrow. The committee declares corridors, once per chain and foreign asset, and never touches individual trades, prices, or users (the settlement layer design spec, section 1, decision 3). That narrowing is why the capture analysis reduces to a bond-size question rather than a per-trade censorship question, and why the governance ceiling in the Risks section is a real one.

Uncapped minting against foreign collateral was rejected outright for this program, and the swap design inherits that rejection by having no minting at all (AGENTS.md, section 4, settled decision 3). Nothing here creates a BitShares asset against a promise of foreign backing. Assets change hands, on chain, between two parties who both signed.

The operation tag numbers were assigned to F rather than taking the first free slots, because stealth transfer tags were already fixed in the union build, the wire format tests, and the sibling stealth BSIP, while F's tags lived in one branch and two recent specs. Renumbering the newer component is the cheaper break (BSIP split design, D2).

# Risks and failure modes

The replay guard can be reset by a gateway redeclaration, and that is the most serious open finding. The record of which foreign transaction ids have already been consumed lives on the gateway object, so a committee that deactivates a gateway and declares a fresh one for the same chain and foreign asset starts from an empty memory, and one Bitcoin payment could then fulfill a second offer, releasing a second seller's escrow against a single coin. The repair under way moves that memory to an instance-independent, append-only record that redeclaration cannot clear. The buyer bears this risk once the proposal activates: he pays and then waits for escrow that a replay could redirect. Source: `build/accs-audit-2026-10-24.md` and `build/f1f2-multiskill-audit-2026-10-24.md`.

The offer lists and the consumed-transaction record have no caps, so consensus state can grow without bound. A determined party could open offers, or publish records, until every honest node pays for the bloat. The repair under way caps offers per account and per orderbook and bounds the consumed set. Source: both audit reports.

The settlement suite proves the happy paths, not the attacks. No test yet pushes a wrong amount, a replayed payment, an early claim, or a maintenance sweep through the settlement code, so the defenses are asserted by the code and not yet demonstrated by evidence, and a defense that no test can break is a defense nobody has tried to break. Source: both audit reports.

Governance capture is a priced ceiling, not a solved problem. The committee declares and deactivates gateways, binds each gateway to an oracle, and maintains the attestor allowlist, and there is no minority override on any of those powers. A captured committee can bind a colluding attestor set, stall a challenge, and release escrow to a friend. The mitigation is the bond and the sustained-challenge path, and the design assumes those are large enough relative to the corridor, which is why the first corridor is small and the pilot is capped. This ceiling follows directly from the attestor model decision (AGENTS.md, section 4, settled decision 6) and is documented rather than mitigated away. Source: governance section of the multi-skill audit report.

The recipient term of the equality match cannot be checked by consensus, as specified in section 4. A dishonest attestor set can attest that a payment reached the seller's address when it did not, and consensus has no field to compare. The bond is the only backstop. Source: both audit reports.

If the oracle's guard delay is longer than the swap's service-level window, no promoted record can arrive before the deadline, every matched offer expires, and sellers collect griefing bonds from buyers who paid honestly. The two clocks must be ordered relative to each other before activation, and that ordering is an open item in the audit record. Source: mechanism interaction section of the multi-skill audit report.

Foreign chain reorganizations can invalidate a payment that was already promoted. The confirmation minimums, six blocks for Bitcoin and so on, price that risk but do not eliminate it. The dispute machinery, not a proof, is the backstop.

Nothing activates on this BSIP. The gate register at `build/ACTIVATION-CHECKLIST.md` carries every remaining condition with an owner and a pass predicate, and the kill criteria are explicit: no closure of the open findings above with consensus-path tests, no replay and soak evidence, no pilot, means no activation. The first corridor is BTC against BTS, and it must survive 90 days of live observation at a $20,000 face cap before any USD issuance depends on it (`docs/superpowers/specs/2026-10-17-pilot-decision.md`; AGENTS.md, section 4, settled decision 2).

# Backwards compatibility

The BSIP-3 appends eight operations at tags 110 through 117 and three new object types. No existing operation, object id, or serialized field moves, and every new field appends last (BSIP split design, D2 and D4). Legacy assets, meaning the `bit*` smartcoins, the `HONEST.*` family, and gateway UIAs, keep their balances, markets, wire formats, and fee accounting, and this proposal migrates, re-issues, or caps nothing. There is no migration path to describe, because the proposal adds a parallel venue rather than modifying an existing one: an account that never opens a swap offer sees no change to any transaction it signs.

Because the proposals are independently votable, a build containing F alone declares the stealth and smartcoin operations but refuses them at apply, and the same holds for any subset of the three. A pin suite holds operation tags and object ids constant across all configurations, so an integration built against one approved set keeps working after the community approves a different one (BSIP split design, sections 3 and 10).

The retired vault program is not part of the compatibility surface: its operations, fields, and tests were removed rather than left dormant, so no legacy client depends on them.

# Test cases

Each row names behavior to pin before any build, derived from what this document's specification already states and cites, and the last rows are the properties the audit rounds and the pilot gate depend on. The suite does not exist, and this section claims no recorded result; it claims what any implementation must refuse or match. Every row belongs to the design spec's assertion suite and to the wire-format pinning the Reference implementation section files.

- Tag stability: the BSIP-3 owns tags 110 through 117 and object ids append after `1.38.x`, holding their numbers across all BSIP configurations, so a transaction built against one approved set deserializes identically against any other (source: Specification, section 1; Backwards compatibility).
- Gate ordering: a chain schedule placing the settlement gate before the oracle gate is refused rather than clamped, and the settlement layer evaluators assert the settlement gate, the oracle gate, and the ordering check (source: Specification, section 1).
- Foreign publish: a published record carries chain, transaction id, event kind, amount, foreign asset index, foreign block height, and confirmation count; a record with zero amount or zero transaction id is refused at validate (source: Specification, section 2).
- Confirmation minimums: each foreign chain has a minimum confirmation count (6 Bitcoin, 12 Ethereum, 32 Solana) that is a protocol constant, not a governance parameter (source: Specification, section 2).
- Consumption equality: a consumer matches on transaction id, amount, foreign asset, event kind of send, and confirmation count at or above the minimum; any mismatch stops it (source: Specification, section 2).
- Gateway declaration: a committee member declares one gateway per (chain, foreign asset) pair, deactivation blocks new offers while leaving existing ones alone, and redeclaration reuses the inactive record in place (source: Specification, section 3).
- Offer lifecycle: transitions are exactly open to matched, open to cancelled, open to expired, matched to fulfilled, matched to expired, and fulfilled, expired, and cancelled are terminal (source: Specification, section 3).
- Fulfill match: fulfill requires the promoted record to match on transaction id, amount, foreign asset, event kind, and confirmation count, and is refused after the deadline (source: Specification, section 3).
- Claim timing: claiming the griefing bond is refused before the deadline, and service-level timeouts are never claimed automatically (source: Specification, section 3).
- Recipient non-verification: consensus cannot verify that the foreign payment reached the seller's stated address, and the bond is the only backstop (source: Specification, section 4; Risks).
- Maintenance sweep: expired open offers release escrow, terminal offers are removed from orderbook lists after an age while the object remains for audit (source: Specification, section 3).
- Constants: minimum offer lifetime 100 blocks, service-level window 86400 blocks, minimum griefing bond 8375 cents' worth, terminal cleanup age 100800 blocks (source: Specification, section 3).
- Registry completeness: every branch declares every operation, object id, and hardfork name, and an operation whose proposal was not approved has no evaluator and fails closed at apply (source: Specification, section 1; Backwards compatibility).
- Wire pinning: operation tags 110 through 117 and the new object ids append and hold stable across builds with and without the hardfork flag (source: Backwards compatibility).

# Reference implementation

This BSIP's design detail lives in `docs/superpowers/specs/2026-10-20-f1-foreign-chain-tx-oracle.md` and `docs/superpowers/specs/2026-10-20-f2-accs-orderbook-settlement.md`, with the split and the earlier vault design retirement record in `docs/superpowers/specs/2026-10-24-three-track-split-design.md`. Implementation status, open findings, and the activation gates are tracked in `build/ACTIVATION-CHECKLIST.md`.

# Discussion

Contested threads, rejected alternatives, and the audit must-close list for this draft:

- Rejected alternatives and why: see the Motivation and Specification sections; the design deliberately rejects unbonded voted feeds, uncapped mint/burn (Terra rule), mandating blindness, and legacy migration/caps.
- Contested threads: hardfork dates, pilot scale, committee acknowledgment of any legacy-capability removal (e.g. sha1 HTLC sunset), and the specialist CLSAG/BP+ gate where applicable.
- Audit must-close list: `build/audit-round-bsip-readiness/bsip-readiness-review.json` (BSIPR-01/03/12) and the per-BSIP rounds `build/audit-round-bsip{3..8}/`. Activation remains unauthorized until every row of `build/ACTIVATION-CHECKLIST.md` clears.

# Summary for Shareholders

- Cost: nothing to existing holders by construction. Legacy assets (`bit*`, `HONEST.*`, gateway UIAs) keep their semantics, fees, and settlement rules; no migration, no caps, no repricing.
- Who pays: new work is funded by opt-in fees, forfeits, pilot-capped issuance, and bonded producer stakes on the new asset class only, not by a standing BTS subsidy.
- What changes for existing holders: nothing, unless they opt into the new class. Every new operation, object, and field is appended and hardfork-gated, so pre-fork nodes keep their current behavior.
- Review date: any standing policy (parity share, fee schedule, subsidy) is re-reviewed against pilot evidence before activation; see `build/ACTIVATION-CHECKLIST.md`.

# See Also

- BSIP-1: `docs/bsips/bsip-1-gamified-smartcoins.md`
- BSIP-2: `docs/bsips/bsip-2-true-stealth.md`
- BSIP-3: `docs/bsips/bsip-3-xclaim-accs.md`
- BSIP-4: `docs/bsips/bsip-4-n-asset-pools.md`
- BSIP-5: `docs/bsips/bsip-5-open-order-rewards.md`
- BSIP-6: `docs/bsips/bsip-6-stableswap.md`
- BSIP-7: `docs/bsips/bsip-7-concentrated-liquidity.md`
- BSIP-8: `docs/bsips/bsip-8-pamm.md`
- Track map: `docs/TRACKS.md`
- Activation gates: `build/ACTIVATION-CHECKLIST.md`

# Copyright

This document is placed in the public domain.
