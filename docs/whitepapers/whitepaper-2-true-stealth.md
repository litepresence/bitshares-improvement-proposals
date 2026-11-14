<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-2 — True stealth transfers, blind pool swaps, and fee rules aimed at the anonymity set

> **Whitepaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Table of contents

1. Narrative
2. Specification
3. Theory
4. Exploit analysis
5. Gas and cost analysis
6. Error code registry
7. Admin architecture
8. Max ring and set sizes
9. Oracle windows
10. Glossary
11. Provenance
12. Test cases

---

## 1. Narrative

BitShares can already move value privately, but today's blind transfers hide only the amount. The sender stays visible and a blind transfer costs about twelve times a fair one, so almost nobody uses one, and a crowd of almost nobody hides nobody. This proposal adds true stealth transfers: every private send hides among at least sixteen users, every hidden amount is checked by the whole network instead of by a wallet, and a fee rule aims the available subsidy at the operations that grow that crowd. It also swaps blinded assets through the same public pools without touching your account, and it makes private sending the wallet default with one tap to send publicly. Nothing existing is removed. Legacy transfers, markets, and every protocol total stay as they are. Private transfers an ordinary person can afford, beside a chain that stays public where it matters.

BitShares has offered blind transfers for years. They hide the amount inside a cryptographic envelope, and they hide nothing else: the asset type stays visible, the sending account's blinded balance is linkable, and there is no ring, so the sender is simply the owner of that balance. The privacy a blind transfer buys is thin, and it is priced as a luxury. The fee schedule puts a transparent transfer at 86869 units against 1013479 plus 337826 per output for a blind transfer at scale 10000, a penalty of about 11.7 times.

That penalty is the market test, and it failed. Nobody pays twelve times the fee to hide an amount from an audience that already sees the sender, so the crowd a blind transfer hides in stays too small to hide in. Privacy loves company, and at these prices blindness buys none.

Two deeper problems sit underneath. First, the chain does not verify hidden amounts: when this design was written no stealth evaluator checked range proofs or commitments, so wallets checked the math and the network accepted envelopes it could not open. A chain that cannot look inside its own envelopes cannot notice inflation inside them. Second, existing privacy is opt-in and stigmatized: optional shielding produced small anonymity sets and marked out the people who used it.

Stealth transfers are an added capability that removes none of the existing ones. Legacy transparent transfers keep working untouched. Nothing here mandates blindness: markets, settlement, and every protocol total stay public because transparency is the product there, and a user who wants a normal public transfer still gets exactly that.

The capability described here is implemented and has been independently audited, and activation is not authorized. Four categories of gate remain open: specialist review of the ring-signature and range-proof cryptography, verification benchmarks on ordinary hardware, activation votes, and committee hardfork dates.

## 2. Specification

This section is for developers and reviewers. Every claim cites the design document or the algorithm it comes from.

### 2.1 What a stealth transfer does

A Pedersen commitment C(v,b) = v·H + b·G hides a value v behind a random blind b on the standard secp256k1 curve, where G and H are public generators and nobody is assumed to know the number relating them, which is the discrete-log assumption. A ring signature proves that one member of a stated set of commitments authorized a spend without naming which member, and the other members are decoys the wallet picks to resemble the real input. The set of unspent blinded outputs a ring draws from is the anonymity set, and the older and larger that set is, the less a spend tells an observer. A Bulletproofs+ range proof is a short proof that a hidden amount lies between zero and 2^64, so no transfer can carry a negative or overflowing amount. A nullifier is a one-time marker derived from a spent output, which lets consensus reject a second spend of that output without learning which ring member was real.

A send therefore carries one ring signature, one range proof per blinded output, and a conservation equation in commitments showing that inputs equal outputs plus the fee while every amount stays hidden. The sender's account never appears in the transaction. Moving value back out to a visible balance uses the explicit-spend operation.

### 2.2 How the chain checks it

Every node verifies the ring, the range proofs, the conservation equation, and nullifier absence inside consensus, so a malformed proof is rejected before any state changes and a wallet's own checks are convenience rather than trust. The ring shape is bounded on the wire: the signature is 32 + 96·n bytes for a ring of n members, the range-proof blob is exactly 592 bytes, and consensus accepts rings of 16 to 32 members. The verifier's shape bound reaches down to 11, a floor reserved for a future governance decision about thin pools and never accepted today. Variable-length fields are bounded in validate() as well as in the evaluators, because an operation embedded in a proposal skips validate() and must still fail fast at relay, and an unknown ring scheme fails closed in both layers so a future ring construction ships as a new version with its own verifier instead of a reinterpretation of old rings.

### 2.3 Decoy rules and thin sets

Consensus enforces decoy quality, not only decoy presence. A decoy must be at least 24 hours old, which makes flooding an asset with fresh outputs to surround a target cost a day of parked capital per output; a ring drawing more than 40 percent of its members from one 100-block window is rejected, relaxing to 25 percent when the asset has fewer than 64 live outputs; and an asset with fewer than 11 live blinded outputs halts new ring spends while deposits stay open, capped at sixteen outputs at a time, so the set can refill but nobody spends into a crowd too thin to hide in.

### 2.4 The verification budget

Every blind operation carries a fixed estimate of its verification cost, and the chain refuses to admit more blind work per block than 750000 microseconds, about a quarter of a three-second block, leaving the rest for transparent traffic. Worst-case loops at both ends of the ring range fit that envelope with room in the current benchmarks, and repeating the measurement on ordinary target hardware or recalibrating the cap is an open gate.

### 2.5 The six operations

The BSIP-2 appends six operations and inserts none: 104 stealth_transfer, 105 freeze_blind, 106 turnstile_reproof, 107 parity_review, 108 stealth_explicit_spend, and 109 stealth_pool_exchange. All six sit behind one hardfork flag, HARDFORK_STEALTH_PASSED, so the committee sets one date and the proposal turns on together or stays off together. New state is appended rather than inserted as well: a nullifier index recording spent markers, and a status record holding the parity pool, the live-output counts, and the reporting clocks.

### 2.6 Inflation containment

A per-asset blinded-supply counter must reconcile against transparent supply plus net blind flow every maintenance interval (the chain's scheduled housekeeping block), and a divergence halts new blind activity for that asset automatically until the books re-prove. The halt ships inside the same upgrade as the proofs, because a verification bug has no rollback and the only remedy is to halt and forward-fix, so the stop path must exist before the proofs do. A manual committee freeze exists from day one as backup, callable by three distinct committee votes with 24-hour freshness, and a freeze wins any tie.

### 2.7 Versions, the turnstile, and the old blind operations

Every blind output carries a version byte. Current-version outputs verify continuously; the previous version verifies during a dual-verify window that opens at activation and closes one year after it, with both nullifier sets enforced throughout so a single opening can never spend twice. After the window, old-version funds move only through the turnstile re-proof, operation 106, which re-proves them under the current version in one step.

Legacy operations 39 through 41 keep their wire formats. After activation they also require sound range proofs and commitments on new outputs, so malformed proofs that wallets used to accept are refused by consensus. The legacy explicit legs then accept only grandfathered balances, and on operation 108 each input's representation proof is verified before the nullifier lookup, because checking nullifiers against an unverified proof would itself open a double-spend path.

### 2.8 Blind pool swaps

Operation 109 swaps one blinded asset for another against the same constant-product pool the transparent exchange uses. One blinded input, one blinded output, and no real account balance is touched; the pool's reserves, its LP shares, which are the tokens issued to liquidity providers, and its price curve stay public. Amounts stay public on purpose: the pool's balance changes reveal them exactly, and the slippage floor requires consensus to know the output amount, so shielded reserves would break pro-rata withdrawal for liquidity providers and price discovery along with them, which is a different feature rather than a stealth swap.

The soundness chain has six links and each one is necessary. A 65-byte Schnorr proof pins the pseudo-output, an intermediate commitment the prover builds, to the claimed input amount. The ring signature then binds the real input's value to that pseudo-output. The conservation equation forces the output to equal the input minus fees. An integer check forces the transaction fee to equal the slippage cover exactly. The range proof rejects negative or overflowing outputs, and the nullifier check rejects double spends. The Schnorr pin is the load-bearing addition, because without it the conservation equation constrains only the difference between input and output and a prover could understate the input to drain liquidity providers or overstate it to plant phantom reserve.

Pool math is copied from the transparent path into one shared helper, because two copies of a rounding rule would diverge and that divergence would be consensus. Market fees on a stealth swap route to the fee-parity pool in place of the issuers, which leaves both legs' issuers with nothing from stealth volume, and that routing is an open decision held behind an explicit committee vote before activation.

### 2.9 Fee rules and Rule R

A blind transfer costs about twelve times a transparent one under the production fee schedule, and this proposal treats that as the defect it is. A fee-parity pool funds blind fees down to transparent levels, capped per block, and floating to cost when the pool empties by rule rather than by surprise. Funding comes from an opt-in per-asset fee share that is absent by default plus the market-fee routing above, and whether that funding is adequate is an open question. Funding is also conditional: set statistics are reported on a 90-day cycle, two missed reports halt the subsidy until reporting resumes, and a missed six-month review defaults blind fees to cost.

Rule R aims the subsidy at what the subsidy is for. The product of this system is the live anonymity set, so eligibility and magnitude follow what an operation does to that set instead of the fact that it is blind. Operations that add outputs or value to the blind set keep the full subsidy. Circulation operations are subsidized on their base fee only, because circulating value adds no outputs. Exits that remove value are capped at the base fee and never receive a per-output subsidy, and the pool swap enters on the circulation rule. Every input the rule reads already exists on chain: operation tag, input and output counts, and pool depth. The rule therefore adds no new chain parameter, no new constant, and no new governance surface; its cap is constructed so it can only shrink a grant and never grow one; and no operation's unsubsidized floor changed. What the rule cannot do is spend money that does not exist.

### 2.10 Wallet behavior

New wallets default to blind sends with one tap to send publicly, which is the difference between a crowd that forms and a line of volunteers. Wallets must also meet concrete requirements: decoy sampling follows the consensus age and skew rules, broadcast delay and decoy batching sit behind one tested choke point so propagation timing does not link a sender, and remote-node users are warned about the timing they leak. A watch-only view key, a scan-only key handed to an accountant, is marked tally-only so the wallet refuses to build a spend from it. That guard lives in the wallet today; the stronger version belongs on the consensus side and is an open design item.

### 2.11 Permanent non-goals

Protocol totals stay permanently public: supply, reserves, queue depth, and fee inputs remain visible to anyone who asks. Order-book blindness is a permanent non-goal rather than a deferred one, because a blinded limit order would be a persistent public pseudonym whose every fill stays linkable to the same handle, buying no anonymity that the pool path does not already give. Blind pool swaps blind the participant, never the pool's reserves, LP shares, or price curve. Blind liquidity-pool deposits and withdrawals are impossible by construction, since an LP share is a user-issued asset that cannot be held blindly. Mandatory blindness for any class is rejected, as are blind asset types, blind supply totals, custodial tumbling as a substitute, and blind-funded disputes.

## 3. Theory

The cryptographic assumptions and invariants:

- Pedersen commitments hide values under the discrete-log assumption on secp256k1.
- Ring signatures provide sender ambiguity among the anonymity set.
- Bulletproofs+ range proofs bound hidden amounts to 0..2^64.
- Nullifiers prevent double spends without revealing which ring member was real.
- Conservation in commitments ensures inputs equal outputs plus fee with all amounts hidden.
- The Schnorr pin binds the pseudo-output to the claimed input amount.
- Decoy age and skew rules prevent flooding and window concentration.
- The thin-set floor prevents spending into a crowd too small to hide in.
- The per-block verification budget bounds blind work.
- Inflation containment reconciles blinded supply every maintenance interval.
- Version bytes and the dual-verify window allow safe upgrades.
- Protocol totals stay public.

## 4. Exploit analysis

- Inflation inside envelopes: the chain now verifies range proofs and commitments in consensus; a divergence halts blind activity.
- Double spend: nullifiers reject a second spend, and representation proofs are verified before nullifier lookup.
- Understated input to drain liquidity providers: the Schnorr pin closes both directions.
- Phantom reserve planting: the same pin closes the overstatement direction.
- Decoy flooding: the 24-hour age rule makes it cost a day of parked capital per output.
- Window concentration: the 40 percent skew rule rejects concentrated rings.
- Thin-set drain: the 11-output floor halts ring spends while deposits stay open.
- Verification denial of service: the 750000 microsecond per-block budget bounds blind work.
- View-key spending: the wallet guard and the specified consensus two-key proof close it; the consensus half is an open item.
- Quantum exposure: all cryptography rests on discrete-log assumptions; the version byte and turnstile are the vehicle for a future scheme.

## 5. Gas and cost analysis

Blind work is bounded per block by the verification budget. Ring size is bounded at 16 to 32 members. Range proofs are fixed at 592 bytes. Signature size is 32 + 96·n bytes. Transparent traffic keeps the remainder of each block. The fee subsidy is capped per block and floats to cost when the parity pool empties.

## 6. Error code registry

New BSIP-2 errors occupy a reserved range appended after the existing registry. Each error maps to one refusal condition: ring too small or too large, decoy too young, window too concentrated, thin set, budget exceeded, nullifier present, range proof invalid, conservation failure, fee mismatch, unknown scheme, inflation divergence. Legacy error codes keep their numbers.

## 7. Admin architecture

Admin is minimal. One hardfork flag turns the whole proposal on or off. A manual committee freeze exists as backup, callable by three distinct committee votes with 24-hour freshness. Fee-parity pool funding is opt-in per asset. Market-fee routing to the parity pool is behind an explicit committee vote. Reporting cadences are enforced by halt rules.

## 8. Max ring and set sizes

Rings carry 16 to 32 members, with a verifier floor of 11 reserved for future governance. Range proofs are fixed at 592 bytes. Signatures are 32 + 96·n bytes. An asset needs at least 11 live blinded outputs to allow ring spends. Decoy flooding costs a day of parked capital per output.

## 9. Oracle windows

The BSIP-2 does not introduce new oracle windows. It reuses the chain's maintenance interval for inflation containment and the 90-day and six-month reporting cadences for the fee subsidy. Pool swaps use the transparent pool's existing state.

## 10. Glossary

- Pedersen commitment: a value hidden behind a random blind.
- Ring signature: proves one member of a set authorized a spend without naming which.
- Decoy: a non-spending member of a ring.
- Anonymity set: the set of unspent blinded outputs a ring draws from.
- Bulletproofs+: a short range proof for hidden amounts.
- Nullifier: a one-time marker that prevents double spends.
- Conservation: inputs equal outputs plus fee in commitments.
- Schnorr pin: binds the pseudo-output to the claimed input amount.
- Turnstile: re-proves old-version funds under the current version.
- Parity pool: funds blind fees down to transparent levels.
- Rule R: subsidy aimed at anonymity-set growth.
- Thin set: fewer than 11 live blinded outputs.

## 11. Provenance

This whitepaper adapts the structure of the Prism and Reciprocity whitepapers in the salvaged reference tree. The concepts are original to the BitShares Track B program. The reference implementations live under docs/reference/contracts-prism and docs/reference/contracts-reciprocity.

## 12. Test cases

- Ring shape accepted at 16 to 32 members and refused outside that range.
- Signature blob exactly 32 + 96·n bytes and range-proof blob exactly 592 bytes.
- Unknown scheme fails closed in validate() and the evaluator.
- Decoy younger than 24 hours refused.
- Ring drawing more than 40 percent from one 100-block window refused, 25 percent when fewer than 64 live outputs.
- Asset with fewer than 11 live blinded outputs halts ring spends while deposits stay open, capped at sixteen outputs at a time.
- No block admits more than 750000 microseconds of blind work.
- Second spend of the same output refused.
- Representation proof verified before nullifier lookup.
- Conservation in commitments with range proof rejecting negative or overflowing amounts.
- Per-asset blinded supply reconciles every maintenance interval, divergence halts blind activity.
- Version window and turnstile re-proof.
- Legacy operations 39 to 41 keep wire formats but require sound proofs on new outputs.
- Six-link pool swap soundness chain.
- Pool math copied into one shared helper.
- Pool swap amounts stay public.
- Rule R subsidy allocation by operation effect.
- Subsidy inertness at production fee settings.
- Permanent non-goals hold.
- Operation tags 104 to 109 and state append across builds.

# Copyright

This document is placed in the public domain.
