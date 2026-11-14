    BSIP: (number assigned at filing; house label BSIP-2)
    Title: True stealth transfers, blind pool swaps, and fee rules aimed at the anonymity set
    Authors: litepresence (HONEST)
    Status: Draft
    Type: Protocol
    Created: 2026-10-25

# Abstract

BitShares can already move value privately, but today's blind transfers hide only the amount. The sender stays visible and a blind transfer costs about twelve times a fair one, so almost nobody uses one, and a crowd of almost nobody hides nobody. This proposal adds true stealth transfers: every private send hides among at least sixteen users, every hidden amount is checked by the whole network instead of by a wallet, and a fee rule aims the available subsidy at the operations that grow that crowd. It also swaps blinded assets through the same public pools without touching your account, and it makes private sending the wallet default with one tap to send publicly. Nothing existing is removed. Legacy transfers, markets, and every protocol total stay as they are. Private transfers an ordinary person can afford, beside a chain that stays public where it matters.

# Motivation

BitShares has offered blind transfers for years. They hide the amount inside a cryptographic envelope, and they hide nothing else: the asset type stays visible, the sending account's blinded balance is linkable, and there is no ring, so the sender is simply the owner of that balance. The privacy a blind transfer buys is thin, and it is priced as a luxury. The fee schedule read through `get_global_properties` on 2026-09-26 at head block ~114706236 puts a transparent transfer at 86869 units against 1013479 plus 337826 per output for a blind transfer at scale 10000, a penalty of about 11.7 times (source: `docs/superpowers/specs/2026-09-29-bsip-g1-stealth-transfers.md`, abstract).

That penalty is the market test, and it failed. Nobody pays twelve times the fee to hide an amount from an audience that already sees the sender, so the crowd a blind transfer hides in stays too small to hide in. Privacy loves company, and at these prices blindness buys none.

Two deeper problems sit underneath. First, the chain does not verify hidden amounts: when this design was written no stealth evaluator checked range proofs or commitments, so wallets checked the math and the network accepted envelopes it could not open (source: `docs/superpowers/specs/2026-09-29-bsip-g1-stealth-transfers.md`, abstract). A chain that cannot look inside its own envelopes cannot notice inflation inside them, which is why this design copies Monero's post-2018 supply check (see Specification). Second, existing privacy is opt-in and stigmatized: optional shielding on Zcash produced small anonymity sets, meaning small crowds to hide in, and marked out the people who used it, while Monero's fee premium shows users will pay for sender ambiguity when they can get it.

HONEST's work on BitShares began as a protest against a witness cartel that fed false prices to shield marginers while breaking forced conversion for holders. A community that learned to check instead of trust has no interest in a privacy feature that asks to be believed, so every mechanism below is verified in consensus rather than promised by a wallet.

Stealth transfers are an added capability that removes none of the existing ones. Legacy transparent transfers keep working untouched. Nothing here mandates blindness: markets, settlement, and every protocol total stay public because transparency is the product there, and a user who wants a normal public transfer still gets exactly that.

The capability described here is implemented and has been independently audited, and activation is not authorized. Four categories of gate remain open: specialist review of the ring-signature and range-proof cryptography, verification benchmarks on ordinary hardware, activation votes, and committee hardfork dates. Each open gate has an owner and a pass condition in `build/ACTIVATION-CHECKLIST.md`, which is the register this document defers to.

This proposal is one of three independently votable proposals. BSIP-3 carries the foreign-chain transaction oracle and its settlement, BSIP-1 carries the price oracle and the smartcoins built on it, and BSIP-2 carries stealth transfers, blind pool swaps, and the fee rules below. Any one, any two, or all three may be approved; each subset builds from the same source tree, and a signed transaction means the same thing in every subset (source: `docs/superpowers/specs/2026-10-24-three-track-split-design.md`, §3 and §6). Nothing is activated by this document.

# Scope

The six operations (104 through 109) and Rule R ship behind one hardfork flag, `HARDFORK_STEALTH_PASSED`, because they share the same ring-signature and commitment crypto and the same nullifier machinery; splitting the flag would double the consensus surface and the audit cost for no independent gain, and the fee rule reads state the chain already keeps, so it adds no new gate of its own. A separate fee-rule gate would only be warranted if Rule R needed to slip on its own, which the design does not anticipate; if it does, the acceptance criterion is a pilot measurement showing the subsidy moves the anonymity set.

# What changes for you

- If you hold assets on BitShares today, nothing changes about your balances, your history, your fees, or your trading.
- If you trade, markets, pools, prices, margin, and settlement stay public and unchanged, and you gain a private way to swap one asset for another through the same pools.
- If you run a witness node, the upgrade adds six operations and two new kinds of state behind one hardfork date, with no change to legacy assets, markets, or fee accounting.
- If you operate a gateway, deposits and withdrawals stay transparent by necessity, and your users keep documented paths for sending publicly whenever a counterparty requires a record.
- If you develop against the chain, six operations append at tags 104 through 109 behind a single hardfork flag, no existing operation, object, or field moves, and the old blind operations gain a proof check on new outputs after activation.

No date is set, and nothing changes until witnesses adopt the upgrade and the committee sets the hardfork dates.

# Specifications

This section is for developers and reviewers. Every claim cites the design document or the algorithm it comes from, and the design documents live in `docs/superpowers/specs/`.

## What a stealth transfer does

A Pedersen commitment `C(v,b) = v·H + b·G` hides a value `v` behind a random blind `b` on the standard secp256k1 curve, where `G` and `H` are public generators and nobody is assumed to know the number relating them, which is the discrete-log assumption. A ring signature proves that one member of a stated set of commitments authorized a spend without naming which member, and the other members are decoys the wallet picks to resemble the real input. The set of unspent blinded outputs a ring draws from is the anonymity set, and the older and larger that set is, the less a spend tells an observer. A Bulletproofs+ range proof is a short proof that a hidden amount lies between zero and 2^64, so no transfer can carry a negative or overflowing amount. A nullifier is a one-time marker derived from a spent output, which lets consensus reject a second spend of that output without learning which ring member was real.

A send therefore carries one ring signature, one range proof per blinded output, and a conservation equation in commitments showing that inputs equal outputs plus the fee while every amount stays hidden. The sender's account never appears in the transaction. Moving value back out to a visible balance uses the explicit-spend operation.

## How the chain checks it

Every node verifies the ring, the range proofs, the conservation equation, and nullifier absence inside consensus, so a malformed proof is rejected before any state changes and a wallet's own checks are convenience rather than trust (source: `docs/superpowers/specs/2026-09-29-bsip2-stealth-transfers.md`, §1). The ring shape is bounded on the wire: the signature is 32 + 96·n bytes for a ring of n members, the range-proof blob is exactly 592 bytes, and consensus accepts rings of 16 to 32 members. The verifier's shape bound reaches down to 11, a floor reserved for a future governance decision about thin pools and never accepted today. Variable-length fields are bounded in `validate()` as well as in the evaluators, because an operation embedded in a proposal skips `validate()` and must still fail fast at relay, and an unknown ring scheme fails closed in both layers so a future ring construction ships as a new version with its own verifier instead of a reinterpretation of old rings (same source, §5).

## Decoy rules and thin sets

Consensus enforces decoy quality, not only decoy presence. A decoy must be at least 24 hours old, which makes flooding an asset with fresh outputs to surround a target cost a day of parked capital per output; a ring drawing more than 40 percent of its members from one 100-block window is rejected, relaxing to 25 percent when the asset has fewer than 64 live outputs; and an asset with fewer than 11 live blinded outputs halts new ring spends while deposits stay open, capped at sixteen outputs at a time, so the set can refill but nobody spends into a crowd too thin to hide in (source: `docs/superpowers/specs/2026-09-29-bsip2-stealth-transfers.md`, §4; floor rule also in `docs/superpowers/specs/2026-09-29-bsip-g1-stealth-transfers.md`, §1).

## The verification budget

Every blind operation carries a fixed estimate of its verification cost, and the chain refuses to admit more blind work per block than 750000 microseconds, about a quarter of a three-second block, leaving the rest for transparent traffic. Worst-case loops at both ends of the ring range fit that envelope with room in the current benchmarks, and repeating the measurement on ordinary target hardware or recalibrating the cap is an open gate (see Risks).

## The six operations

The BSIP-2 appends six operations and inserts none: 104 `stealth_transfer`, 105 `freeze_blind`, 106 `turnstile_reproof`, 107 `parity_review`, 108 `stealth_explicit_spend`, and 109 `stealth_pool_exchange`. All six sit behind one hardfork flag, `HARDFORK_STEALTH_PASSED`, so the committee sets one date and the proposal turns on together or stays off together (source: `docs/superpowers/specs/2026-10-10-stealth-pool-swaps-design.md`, §2). New state is appended rather than inserted as well: a nullifier index recording spent markers, and a status record holding the parity pool, the live-output counts, and the reporting clocks.

## Inflation containment

A per-asset blinded-supply counter must reconcile against transparent supply plus net blind flow every maintenance interval (the chain's scheduled housekeeping block), and a divergence halts new blind activity for that asset automatically until the books re-prove (source: `docs/superpowers/specs/2026-09-29-bsip2-stealth-transfers.md`, §1). The halt ships inside the same upgrade as the proofs, because a verification bug has no rollback and the only remedy is to halt and forward-fix, so the stop path must exist before the proofs do. A manual committee freeze exists from day one as backup, callable by three distinct committee votes with 24-hour freshness, and a freeze wins any tie.

## Versions, the turnstile, and the old blind operations

Every blind output carries a version byte. Current-version outputs verify continuously; the previous version verifies during a dual-verify window that opens at activation and closes one year after it, with both nullifier sets enforced throughout so a single opening can never spend twice. After the window, old-version funds move only through the turnstile re-proof, operation 106, which re-proves them under the current version in one step (source: `docs/superpowers/specs/2026-09-29-bsip2-stealth-transfers.md`, §5).

Legacy operations 39 through 41 keep their wire formats. After activation they also require sound range proofs and commitments on new outputs, so malformed proofs that wallets used to accept are refused by consensus. The legacy explicit legs then accept only grandfathered balances, and on operation 108 each input's representation proof is verified before the nullifier lookup, because checking nullifiers against an unverified proof would itself open a double-spend path (sources: `docs/superpowers/specs/2026-10-09-nullifier-double-spend-design.md`; `docs/superpowers/specs/2026-10-10-stealth-pool-swaps-design.md`, §3 link 6).

## Blind pool swaps

Operation 109 swaps one blinded asset for another against the same constant-product pool the transparent exchange uses. One blinded input, one blinded output, and no real account balance is touched; the pool's reserves, its LP shares, which are the tokens issued to liquidity providers, and its price curve stay public (source: `docs/superpowers/specs/2026-10-10-stealth-pool-swaps-design.md`, §1). Amounts stay public on purpose: the pool's balance changes reveal them exactly, and the slippage floor requires consensus to know the output amount, so shielded reserves would break pro-rata withdrawal for liquidity providers and price discovery along with them, which is a different feature rather than a stealth swap (same source, §4).

The soundness chain has six links and each one is necessary (same source, §3; proved in `docs/superpowers/specs/2026-10-10-stealth-pool-swaps-kernel-proof.md`). A 65-byte Schnorr proof pins the pseudo-output, an intermediate commitment the prover builds, to the claimed input amount. The ring signature then binds the real input's value to that pseudo-output. The conservation equation forces the output to equal the input minus fees. An integer check forces the transaction fee to equal the slippage cover exactly. The range proof rejects negative or overflowing outputs, and the nullifier check rejects double spends. The Schnorr pin is the load-bearing addition, because without it the conservation equation constrains only the difference between input and output and a prover could understate the input to drain liquidity providers or overstate it to plant phantom reserve; the proof shows the pin closes both directions (kernel proof, Lemma 1).

Pool math is copied from the transparent path into one shared helper, because two copies of a rounding rule would diverge and that divergence would be consensus (same source, §4). Market fees on a stealth swap route to the fee-parity pool in place of the issuers, which leaves both legs' issuers with nothing from stealth volume, and that routing is an open decision held behind an explicit committee vote before activation (same source, §5).

## Fee rules and Rule R

A blind transfer costs about twelve times a transparent one under the production fee schedule, and this proposal treats that as the defect it is. A fee-parity pool funds blind fees down to transparent levels, capped per block, and floating to cost when the pool empties by rule rather than by surprise. Funding comes from an opt-in per-asset fee share that is absent by default plus the market-fee routing above, and whether that funding is adequate is an open question (source: `docs/superpowers/specs/2026-10-10-stealth-entropy-economics.md`, §7). Funding is also conditional: set statistics are reported on a 90-day cycle, two missed reports halt the subsidy until reporting resumes, and a missed six-month review defaults blind fees to cost (source: `docs/superpowers/specs/2026-09-29-bsip2-stealth-transfers.md`, §2).

Rule R aims the subsidy at what the subsidy is for. The product of this system is the live anonymity set, so eligibility and magnitude follow what an operation does to that set instead of the fact that it is blind. Operations that add outputs or value to the blind set keep the full subsidy. Circulation operations are subsidized on their base fee only, because circulating value adds no outputs. Exits that remove value are capped at the base fee and never receive a per-output subsidy, and the pool swap enters on the circulation rule. Every input the rule reads already exists on chain: operation tag, input and output counts, and pool depth. The rule therefore adds no new chain parameter, no new constant, and no new governance surface; its cap is constructed so it can only shrink a grant and never grow one; and no operation's unsubsidized floor changed (same source, §4). What the rule cannot do is spend money that does not exist, which the Risks section takes up directly.

## Wallet behavior

New wallets default to blind sends with one tap to send publicly, which is the difference between a crowd that forms and a line of volunteers. Wallets must also meet concrete requirements: decoy sampling follows the consensus age and skew rules, broadcast delay and decoy batching sit behind one tested choke point so propagation timing does not link a sender, and remote-node users are warned about the timing they leak. A watch-only view key, a scan-only key handed to an accountant, is marked tally-only so the wallet refuses to build a spend from it. That guard lives in the wallet today; the stronger version belongs on the consensus side and is an open design item, described under Risks.

## Permanent non-goals

Protocol totals stay permanently public: supply, reserves, queue depth, and fee inputs remain visible to anyone who asks. Order-book blindness is a permanent non-goal rather than a deferred one, because a blinded limit order would be a persistent public pseudonym whose every fill stays linkable to the same handle, buying no anonymity that the pool path does not already give (source: `docs/superpowers/specs/2026-10-10-stealth-pool-swaps-design.md`, §1). Blind pool swaps blind the participant, never the pool's reserves, LP shares, or price curve. Blind liquidity-pool deposits and withdrawals are impossible by construction, since an LP share is a user-issued asset that cannot be held blindly. Mandatory blindness for any class is rejected, as are blind asset types, blind supply totals, custodial tumbling as a substitute, and blind-funded disputes (source: `docs/superpowers/specs/2026-09-29-bsip2-stealth-transfers.md`, §3).

# Rationale

Privacy works by company. Amount privacy and sender ambiguity are separate properties, and either one alone leaves a linking path, so this design ships both together or not at all.

- Mandatory blindness for all transfers would destroy per-asset supply auditability, start a delisting fight on every listing, and either exempt the exchange, leaving traders linkable at the moment that matters, or rebuild the order book on commitments, trading the chain's clearest product for privacy nobody asked the book to have.
- Opt-in shielding without a default produces the Zcash result, thin crowds and stigma on the people who opt in, while a default captures most of what mandating would gain at none of its cost.
- Blind order books are rejected permanently, for the entropy reason above: a maker who stays in the book is a standing pseudonym.
- Hiding pool reserves is rejected because pro-rata withdrawal, price discovery, and on-screen backing all require public reserves.
- Multi-input pool swaps are rejected because a prover could offset a false input value against a true one, and the conservation equation sees only the total.
- The first draft of the pool-swap proof leaned on a conservation equation that pinned only the difference between input and output, and it was wrong; the Schnorr pin described above is the fix, and it uses standard discrete-log techniques rather than novel cryptography (sources: `docs/superpowers/specs/2026-10-10-stealth-pool-swaps-design.md`, §0; `docs/superpowers/specs/2026-10-10-stealth-pool-swaps-kernel-proof.md`, §1).
- Steering knobs for the fee rules, such as a tunable premium on transparent transfers, were set aside because every tunable number is a lever a future vote could move, and this program is built on the assumption that votes can be captured. Rule R reads state the chain already keeps.
- A separate halt feature shipped after launch would arrive too late, because a verification bug cannot be rolled back; the stop path ships with the proofs.
- One hardfork flag covers the whole proposal for the smallest blast radius, and this proposal's schedule stays independent of the other proposals, so this proposal can pass, fail, or slip without touching them.

# Risks and failure modes

- A watch-only key must not be able to spend. Today a private view key, the scan-only half of a stealth address, is enough to recover an output's opening, and a ring spend authorizes by knowledge of that opening, so a view-key holder could construct a spend instead of only watching. The fix is designed in two layers. The wallet layer makes new outputs derive their opening from the spend key and strips the usable opening from the message that lets a wallet recognize its own outputs. The consensus layer adds a two-key proof over the ring's owner keys, so a spend must show the spend key instead of only knowledge of the opening. The wallet half is implemented; the consensus half is specified but not yet contracted to a specialist reviewer, so this item is open and nothing activates until it lands and is audited (source: `docs/superpowers/specs/2026-10-19-viewkey-ring-auth-design.md`, §1 and §2).
- The fee subsidy is inert as configured, and that is the known open economics item. The subsidy is the gap between a hardcoded cost floor and the committee's scheduled fee, and at production fee settings that gap computes to nearly nothing, so Rule R aims an almost empty pot at the only operations that grow the crowd. The mechanism stays inert until governance changes the fee schedule, which is committee-controlled by design. Nobody should read this document as promising fee parity or funded anonymity-set growth under the current schedule (source: `docs/superpowers/specs/2026-10-10-stealth-entropy-economics.md`, §4.1).
- No clean specialist review of the ring-signature and range-proof cryptography means no activation. The implementation adapts the published CLSAG and Bulletproofs+ constructions rather than copying them, and the pool-swap proof names that review as the owner of its load-bearing assumptions, so an unfinished or adverse review stops activation.
- Verification cost on ordinary hardware is not yet measured outside development machines; worst-case loops fit the per-block envelope in current benchmarks, and the gate is a repeat run on target hardware or a recalibration of the cap.
- A verification bug could create coins out of nowhere. The mitigation is an automatic halt of blind activity when per-asset totals stop reconciling, the halt ships before any blind transfer can occur, and the kill criterion is that a halted asset stays halted until its books re-prove. There is no rollback.
- Anonymity sets are load dependent. Early sets are thin, wallets warn below recommended minimums, and the floor rule trades availability for privacy on purpose. Set size follows adoption, which is why the inert fee rule above matters more than its mechanics.
- Privacy covers one leg of a journey, not the whole journey. A private transfer that meets an exchange deposit, a market trade, or any other public endpoint can be linked back at that point, and this document claims no more than leg privacy.
- Stealth swap fees currently bypass issuers. Market fees on a stealth swap route to the parity pool, leaving both legs' issuers with nothing, and the routing stays behind an explicit committee vote until someone decides it; pairs with a zero market fee divert nothing in the first place.
- All of the cryptography here rests on discrete-log assumptions and falls together to a cryptographically relevant quantum computer. The version byte, the dual-verify window, and the turnstile are the vehicle for a future post-quantum scheme; no such scheme ships here, and long-lived blinded outputs are the durable exposure.
- The remaining gates are open, and gates close only with evidence: specialist review, target-hardware benchmarks, replay and soak runs, the pilot report at or below twenty thousand dollars of face value together with the anonymity-set measurement it feeds, activation votes, and committee hardfork dates. Every one has an owner and a pass condition in `build/ACTIVATION-CHECKLIST.md`, and no clean review, no activation.

# Backwards compatibility

Nothing migrates. The BSIP-2 appends operations at tags 104 through 109 and inserts none, no existing operation, object id, or field moves, and the operation registry is pinned to the same numbers in every BSIP configuration, so a transaction built against any approved subset serializes to the same bytes (source: `docs/superpowers/specs/2026-10-24-three-track-split-design.md`, §6). This proposal carries no dependency on the oracle or settlement proposals: it registers no evaluator from them, so an unapproved operation fails closed at apply.

Legacy assets (`bit*`, `HONEST.*`, gateway UIAs) keep their balances, markets, wire formats, and fee accounting, and no migration re-issues, re-prices, or caps anything. The migration path for this feature is explicitly none, because it is a parallel capability rather than a replacement. Transparent transfers are untouched by construction, since the stealth hardfork gate covers only the new operations and the new checks on blind outputs. Balances created by today's blind operations keep spending: they verify under the current rules for one year from activation and then move through the turnstile re-proof, and after activation new outputs on the legacy blind operations must carry sound proofs, which changes what wallets attach rather than the wire format. The parity fee share is opt-in per asset and absent by default, so an issuer that never opts in sees no fee change.

# Test cases

Each row names behavior to pin before any build, derived from what this document's specification already states and cites, and the last rows are the properties the audit rounds and the specialist review depend on. The suite does not exist, and this section claims no recorded result; it claims what any implementation must refuse or match. Every row belongs to the design spec's assertion suite and to the wire-format pinning the Reference implementation section files.

- Ring shape: a ring of 16 to 32 members is accepted, a ring below 16 or above 32 is refused, and the signature blob is exactly 32 + 96·n bytes with a range-proof blob of exactly 592 bytes (source: Specification, "How the chain checks it").
- Unknown scheme: a ring whose scheme byte is not the current scheme fails closed in both `validate()` and the evaluator, so a future ring construction ships as a new version rather than a reinterpretation of old rings (source: Specification, "How the chain checks it").
- Decoy age: a decoy younger than 24 hours is refused, and a ring drawing more than 40 percent of its members from one 100-block window is refused, relaxing to 25 percent when the asset has fewer than 64 live outputs (source: Specification, "Decoy rules and thin sets").
- Thin set: an asset with fewer than 11 live blinded outputs halts new ring spends while deposits stay open, capped at sixteen outputs at a time (source: Specification, "Decoy rules and thin sets").
- Verification budget: no block admits more blind work than 750000 microseconds, and worst-case loops at both ends of the ring range fit that envelope (source: Specification, "The verification budget").
- Nullifier absence: a second spend of the same output is refused, and each input's representation proof is verified before the nullifier lookup so an unverified proof cannot open a double-spend path (source: Specification, "Versions, the turnstile, and the old blind operations").
- Conservation: inputs equal outputs plus fee in commitments, every amount stays hidden, and the range proof rejects negative or overflowing amounts (source: Specification, "What a stealth transfer does"; "How the chain checks it").
- Inflation containment: per-asset blinded supply reconciles against transparent supply plus net blind flow every maintenance interval, and a divergence halts new blind activity for that asset until the books re-prove (source: Specification, "Inflation containment").
- Version window: current-version outputs verify continuously, the previous version verifies during the dual-verify window, both nullifier sets are enforced throughout, and after the window old-version funds move only through the turnstile re-proof (source: Specification, "Versions, the turnstile, and the old blind operations").
- Legacy soundness: operations 39 through 41 keep their wire formats but require sound range proofs and commitments on new outputs after activation, so malformed proofs are refused by consensus (source: Specification, "Versions, the turnstile, and the old blind operations").
- Pool swap soundness: the six-link chain (Schnorr pin, ring binding, conservation, integer fee check, range proof, nullifier) holds, and without the Schnorr pin the conservation equation constrains only the difference (source: Specification, "Blind pool swaps"; kernel proof, Lemma 1).
- Pool math parity: pool math is copied into one shared helper so two copies of a rounding rule cannot diverge into a consensus split (source: Specification, "Blind pool swaps").
- Amount publicity: pool swap amounts stay public because the pool's balance changes reveal them and the slippage floor requires consensus to know the output amount (source: Specification, "Blind pool swaps").
- Rule R: operations that add outputs or value keep the full subsidy, circulation operations are subsidized on base fee only, exits are capped at base fee with no per-output subsidy, and the pool swap enters on the circulation rule, all from state the chain already keeps (source: Specification, "Fee rules and Rule R").
- Subsidy inertness: the subsidy is the gap between a hardcoded cost floor and the committee's scheduled fee, and at production fee settings that gap computes to nearly nothing (source: Specification, "Fee rules and Rule R"; Risks).
- Permanent non-goals: protocol totals stay public, order-book blindness and blind pool deposits and withdrawals are impossible by construction, and mandatory blindness is rejected (source: Specification, "Permanent non-goals").
- Wire pinning: operation tags 104 through 109 and the nullifier index and status record append and hold stable across builds with and without the hardfork flag (source: Backwards compatibility).

# Reference implementation

Branch `the implementation branch` carries the BSIP-2: transfers, pool swaps, the fee rules, and the wallet side. Evidence of what has run and of what still must run lives in the repository's own records, `build/PROGRESS.md` for executed work and `build/ACTIVATION-CHECKLIST.md` for the open gates.

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
