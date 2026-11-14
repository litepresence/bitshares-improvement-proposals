<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-5 — Open order rewards on the native markets: maker rebates and time, size, and proximity weighted fee distribution

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
8. Max order and set sizes
9. Oracle windows
10. Glossary
11. Provenance
12. Test cases

---

## 1. Narrative

Resting an order on BitShares earns nothing. The maker, the party whose order sits in the book waiting, gives the market time, size, and a price the next trader can reach, and is paid nothing for it, so books sit thin and price discovery drifts to the swap pools. Each taker's fee funds a maker rebate, paid when his order fills, and a provider pool, distributed to open orders weighted by three measured qualities: how long the order has rested, capped at an hour; how much quantity remains; and how close its price sits to the market's own recent price. Gating, capping, and self-trade refusal keep the rewards unforfeitable and unpumpable. Participation is opt-in per asset, everything routed or paid stays on the ledger, and nothing changes for markets that never opt in. The pitch: waiting in the book becomes a paid service, priced in public by formula.

Books starve when waiting is free for everyone but valuable to everyone: each maker supplies the product the next trader consumes and collects nothing for it. Numbers, not arguments: a 2026-09-26 snapshot from a public API node by `tooling/bts_chain_snapshot.py` (source: `AGENTS.md`, section 6, chain facts) found order books holding about 15 dollars of depth per book, so price discovery left the books, and the swap pools that absorbed it disagree by roughly 86 percent about the value of the chain's own reference assets. Meanwhile the taker, the trader whose order fills into a resting one, already pays a full fee, the issuer's market fee on what he receives, and none of it routes back to the book.

Competitive exchanges pay for resting liquidity, and the structure of what a resting order supplies is well known in the reference: an open limit order gives the market three measurable qualities, the time it has been available, the size of what it commits, and the proximity of its price to the market price, and an order old and large but far from it supplies nothing real, because takers cannot reach it. Rewarding all three together is the design's whole point, because a reward that pays for one quality alone pays just as well for faking the other two (source: `contracts/prism/docs/whitepaper/fees.md`, section 1.1).

Two failure modes dominate any design that pays for resting orders, and both were met in the reference by gates rather than by trust. A fresh order sniping a payout wave must be priced out, and so must a ghost order parked once at a far price and never touched; the fixes are a minimum age before payout share, a maximum age credit that stops growing at an hour, and a reset of that credit on every amendment, so the reward tracks actively managed liquidity (source: `contracts/prism/docs/whitepaper/fees.md`, sections 4 and 5.4). The target is plain: rewards small relative to the fees that fund them, flowing to orders the next taker would use anyway, and computable by every node during the maintenance interval every node already runs.

Status, stated plainly. The reward formula and its gates are distilled from the owner's order book project, which implements price-time matching and this fee distribution across more than a dozen chain ports with identical fill behavior (source: `contracts/prism/README.md`; `contracts/prism/AGENTS.md`, Identity). Nothing in this document is implemented in the BitShares core program: the three proposals BSIP-1 through BSIP-3 own the current build program, and this proposal stands beside them, uncontracted and unactivated. Before any vote there must be a design spec, wire pinning, test vectors, and the binary evidence the house's other proposals file, and no designed date or audit verdict exists to promise.

## 2. Specification

Consensus law throughout: no floats, head-block time only, append-only objects and fields, hardfork gate in evaluator and proposal visitor, bounded state, and every payout leaving an audit trail on the ledger, not in process memory.

### 2.1 Opt-in funding, one leg at a time

An asset opts in through a single new field in its options, an open order fee in basis points, hundredths of a percent, proposed to range from 0, the default, to 500, charged inside the fill accounting on the asset the taker receives, in the same leg the existing market fee is charged. A market exists between two assets, and either leg may carry its own fee, so a market accumulates at most two pools, one per participating asset, each denominated in that asset (source: `contracts/prism/docs/whitepaper/spec.md`, section 4.1, where the whole taker fee splits per fill; the per-asset leg rate is the adaptation, following how market fees on this chain already ride the received asset). No escrow, no exterior contract, no new permission: issuers already hold exactly this authority over their own market fees.

Parameters change only through the empty-book rule: a rate or a split may change only while no rewards-enabled open orders rest on the affected market, so no resting order can be repriced after the fact (source: `contracts/prism/docs/whitepaper/spec.md`, section 8.2). Beyond the per-fill split inside section 2.3, the two named role fees of the reference, a founder share and a promoter share, are dropped for this chain rather than mapped, because the chain has no exchange operator to pay and creating two new fee recipients at the head of every book would be a new governance surface the program's settled design avoids; the full fee minus the rebate goes to the pool (source of the two roles: `contracts/prism/docs/whitepaper/fees.md`, section 10.1; the removal is this proposal's adaptation, argued in Rationale).

### 2.2 Book state: one field per open order

Each open order that qualifies carries exactly one consensus field: the block height at which it entered the book or was last amended (source: `contracts/prism/docs/whitepaper/spec.md`, section 1.5, where the fee state is that single field by design). Nothing is computed or stored per event; the reward weight is a snapshot at distribution time. Full fills and cancels delete the field, amendments regenerate it, and the order's remaining quantity, which every resting order already has, supplies the size leg (source: `contracts/prism/docs/whitepaper/spec.md`, section 3.4). Every amendment resets the entry: a same-price quantity change and a price change both restart the clock, which is what stops an active manager from being beaten on time by a parked order (source: `contracts/prism/docs/whitepaper/fees.md`, section 5.4 and the amend-hike analysis in `contracts/prism/docs/whitepaper/exploits.md`, section 2).

### 2.3 The per-fill split

On each fill of a participating asset, the open order fee charges on the taker's received amount, then splits:

    maker_rebate = remaining × rebate_pct / 100
    provider_pool_share = remaining - maker_rebate

with the rebate percentage proposed at 70 percent of the remaining fee and the pool taking the other 30 percent, the reference's default beyond its extra two named cuts (source: `contracts/prism/docs/whitepaper/spec.md`, section 4.1). The rebate pays to the maker whose order was consumed, immediately, into a vesting balance, the chain's existing system for balances that must age before claim, and it remains claimable after his order has fully filled or been cancelled, since it is settled at fill rather than at distribution (source: `contracts/prism/docs/whitepaper/fees.md`, section 9). Pool shares accumulate in the market's per-asset pool, and conservation holds exactly: what is charged equals what is rebated plus what pools, with integer truncation dust staying in the pool until the next distribution rounds it out (source: `contracts/prism/docs/whitepaper/fees.md`, sections 8.2 and 12.3).

### 2.4 The reward weight: time, size, proximity

At each maintenance sweep, every qualifying open order on a participating market computes one weight:

    T = min(payout_block - placement_block, MAX_T_BLOCKS), zero credit below MIN_T_BLOCKS
    S = remaining quantity of the order
    P = min(order price, reference price) / max(order price, reference price), carried as a fixed-point fraction with the denominator BASIS, a scale constant the design spec fixes at build time
    skip entirely if P < PROXIMITY_THRESHOLD, proposed at 0.9, an order ten percent or more from price
    Pk = 10 × P - 9 × BASIS, with both P and Pk carried in BASIS units
    TSP = S × T × Pk / BASIS, the single fixed-point correction, taken once at the end

(source: `contracts/prism/docs/whitepaper/spec.md`, section 5.1; `contracts/prism/docs/whitepaper/fees.md`, section 2). The weight the reference names TSP, for time, size, and proximity, has three legs that multiply because each is necessary: an order stale in time, small in size, or far in price is not the liquidity this design wants to buy, and a zero in any leg must zero the weight (source: `contracts/prism/docs/whitepaper/fees.md`, section 1.2). The time leg is capped at 300 blocks on the 12-second reference chain and its bottom gate sits at 25; these constants convert to block counts at compile time per chain, so the effective interval stays 5 minutes to 60 minutes everywhere, and at this chain's three-second block schedule that is MIN 100 blocks and MAX 1200 (source: `contracts/prism/docs/whitepaper/fees.md`, sections 13.2 and 13.4). Proximity uses the linear taper: Pk maps the interval from the threshold to exact equality linearly onto zero to one times the denominator, because a step cutoff creates a cliff to cluster against just inside the threshold (source: `contracts/prism/docs/whitepaper/fees.md`, sections 3.1 and 3.2).

The reference prices proximity against its own time-and-volume weighted oracle reading and falls back to the last traded price when the oracle window holds no data yet (source: `contracts/prism/docs/whitepaper/fees.md`, section 2.1; `contracts/prism/agents/ref.md`, invariant 17, and `contracts/prism/docs/whitepaper/exploits.md`, section 6.1). This proposal adapts that to data the chain already holds: each participating market keeps a rolling accumulator over its own fills, time-weighted across a one-hour window, and reads that accumulator's average as its reference price, with no external feed in the path. Unlike the reference, there is no last-traded-price fallback: a market with no fill data in the window pays nothing that sweep, because a reference price left over from an old fill is a number any small trade could have planted long before the payouts.

### 2.5 Distribution and its bounds

The sweep first filters each market's book by remaining quantity, keeping the largest standing orders up to a per-sweep cap proposed at 256 against the reference's 1,000, because consensus pays the loop cost here and the sweep must stay bounded whatever the book size (source of the filter and its ordering: `contracts/prism/docs/whitepaper/spec.md`, section 5.3, and `contracts/prism/docs/whitepaper/fees.md`, section 6). It then distributes the market's accrued pool for that asset among those survivors, each receiving pool × its TSP weight over the sum of all eligible weights, and the pool's integer remainder after distribution stays in the pool for the next sweep (source: `contracts/prism/docs/whitepaper/fees.md`, sections 2.6 and 8.2). The pool can only empty through distribution, since every fill only adds to it, and a sweep with no eligible orders costs nothing and pays nothing (source: `contracts/prism/docs/whitepaper/fees.md`, sections 12.3 and 8.3).

An empty book does not leak. Fees accrue while the book is empty, the minimum-age gate keeps the first post-empty order from collecting everything, and the pool stays monotone until enough qualifying orders exist to split it (source: `contracts/prism/docs/whitepaper/exploits.md`, section 1, the empty-book drain, which the reference's minimum-time gate was built to close).

### 2.6 Anti-abuse, quoted and adapted

- Fresh-order sniping is priced out by the minimum age, since a payout at any block can only be claimed by orders at least 100 blocks old at this chain's schedule (source: `contracts/prism/docs/whitepaper/fees.md`, section 4.2).
- Ghost order dominance is capped by the maximum time credit, so the oldest order earns no more weight than one an hour old, at most 12 times the youngest qualifying order (source: `contracts/prism/docs/whitepaper/fees.md`, sections 5.2 and 5.3).
- Pool draining through an emptied book is covered by the same gate, stated in section 2.5 (source: `contracts/prism/docs/whitepaper/exploits.md`, section 1).
- Payout floods of tiny spam orders are bounded twice: the sweep keeps only the top orders by standing size before any weight arithmetic, and inside that reduced set the proximity screen zeroes far-priced orders (source: `contracts/prism/docs/whitepaper/spec.md`, section 5.3; `contracts/prism/docs/whitepaper/exploits.md`, section 3; `contracts/prism/docs/whitepaper/fees.md`, section 6.3).
- Cross-account wash trading is unprofitable in the reference by an explicit proof, and the shape carries over: the maker's order exits the pool on fill and earns nothing more from it, the same owner pays the full fee on the taker leg, and an operator owning the fraction f of the book at distribution nets the rebate plus f × pool minus the fee. Under the 70-to-30 split of section 2.3 the rebate plus the pool is the whole fee, so the loss is 30 percent of the fee times the book share the operator does not own, strictly negative for every f below 1 and exactly zero at f = 1, with the chain's transaction fees for the two orders still due on top (source: `contracts/prism/docs/whitepaper/fees.md`, section 12.2, adapted here to the two-way split of section 2.3, where the formula's remaining input is the whole fee because no named cuts exist).
- Self-trading within one account is refused outright on participating markets: the fill is rejected when the aggressive order and the resting order belong to the same owner, the stronger owner-level check the reference's on-chain ports use over id comparison (source: `contracts/prism/docs/whitepaper/spec.md`, section 3.3).

### 2.7 Scope it refuses

Matching stays exactly as it is: price-time priority, same fill accounting, no new order types, and nothing here reaches margin calls, force settlement, or the settlement offer book of BSIP-3, whose offers fix price at match and have no resting rewards to distribute (BSIP-3, Specification, section 6, out of scope by design). Per-fill provider rewards are refused on purpose: paying a filled order from the pool would double-pay the same order's rebate, and the reference deletes reward eligibility at fill for exactly that reason, so the pool pays future liquidity while the rebate pays past execution, never both to once (source: `contracts/prism/docs/whitepaper/exploits.md`, section 6.2). The reward is public by rule, pools, weights, gates, and every payout being on-ledger records, because anything resting in a book is a standing public pseudonym, and this program's settled design rejects order-book blindness permanently, not as a future option (BSIP-2, Specification, permanent non-goals; settled decision 4 in `AGENTS.md`, section 4). Nothing here attests anything about a foreign chain, cross-chain or otherwise, so the attestor ceiling of the program never applies.

## 3. Theory

The invariants the implementation must preserve:

- No floats anywhere in consensus; all math is integer or 128-bit fixed point.
- Every new operation and field is appended, never inserted, and hardfork-gated in its evaluator and in the proposal visitor.
- State is bounded: the per-sweep top-N cap bounds the distribution loop, and the reward field is one field per open order.
- Every payout leaves an audit trail on the ledger, not in process memory.
- Conservation holds exactly: on each fill, the fee charged equals the rebate plus the pool share, with truncation dust kept in the pool.
- Amendment resets the time credit, so a parked order cannot outwait an active manager.
- Legacy assets and non-participating markets are untouched by construction.

Overflow discipline: 128-bit comparison for fee and pool math, saturation where a wrap would otherwise become a silent value.

## 4. Exploit analysis

- Empty-book drain: fees accrue while the book is empty, the pool never loses them, the first post-empty order cannot claim the whole lot, and a sweep with no eligible orders pays nothing (source: `contracts/prism/docs/whitepaper/exploits.md`, section 1).
- Amend-hike amplification: amendment resets the time credit, so a same-price quantity change and a price change both restart the clock (source: `contracts/prism/docs/whitepaper/exploits.md`, section 2).
- Payout flooding: the top-N filter and the proximity screen bound tiny spam orders (source: `contracts/prism/docs/whitepaper/exploits.md`, section 3).
- Sniping: the minimum age gate prices out fresh orders (source: `contracts/prism/docs/whitepaper/exploits.md`, section 4).
- Cross-account wash trading: unprofitable in aggregate; the loss is 30 percent of the fee times the book share the operator does not own, strictly negative below full ownership (source: `contracts/prism/docs/whitepaper/fees.md`, section 12.2).
- Oracle dependency: the reference marks oracle dependency as its residual external risk; this proposal removes it by using the market's own fill accumulator, at the cost of paying nothing on cold-start markets (source: `contracts/prism/docs/whitepaper/exploits.md`, section 6.1).
- Per-fill reward doubling: paying a filled order from the pool would double-pay; the reference deletes reward eligibility at fill, and this proposal follows that rule (source: `contracts/prism/docs/whitepaper/exploits.md`, section 6.2).

## 5. Gas and cost analysis

Operations are consensus objects, not smart contract calls, so cost is measured in transaction fees and block weight. The per-sweep top-N cap, proposed at 256 against the reference's 1,000, bounds the distribution loop so the sweep stays simulatable in bounded consensus work; the reference prices that same work in gas, a per-computation meter this chain does not have, so the cost here is witness time during maintenance. The reward field is one field per open order, and weights are snapshots at distribution time, so no per-event write is needed.

## 6. Error code registry

New BSIP-5 errors occupy a reserved range appended after the existing registry. Each error maps to one refusal condition: asset not opted in, parameter change while qualifying orders rest, self-trade on a participating market, fill below conservation, sweep with no eligible orders, proximity reference absent, top-N overflow, vesting balance creation failure. Legacy error codes keep their numbers.

## 7. Admin architecture

Admin is split between consensus and the issuer. Consensus owns the fill accounting, the split, the pool accrual, the sweep, and the payout. The issuer of each asset owns the opt-in rate and the split parameters, subject to the empty-book rule and the 0 to 500 basis point cap. No admin key can move pool balances by procedure alone; every payout leaves an on-ledger record.

## 8. Max order and set sizes

- Open order fee range: 0 to 500 basis points.
- Rebate percentage: 70 percent (proposed).
- Pool share: 30 percent (proposed).
- Proximity threshold: 0.9 (proposed).
- Minimum time gate: 100 blocks at this chain's 3-second schedule (5 minutes).
- Maximum time credit: 1200 blocks (1 hour).
- Per-sweep top-N cap: 256 (proposed).
- Reference price window: 1 hour.
- Time-weighted accumulator: rolling over the market's own fills.

## 9. Oracle windows

The reward's proximity reference uses a one-hour rolling accumulator over the market's own fills, measured in head-block time. There is no external feed and no last-traded-price fallback: a market with no fill data in the window pays nothing that sweep.

## 10. Glossary

- Maker: the party whose resting order is filled.
- Taker: the trader whose order fills into a resting one.
- Maker rebate: the immediate payment to the filled maker.
- Provider pool: the accrued share distributed to open orders.
- TSP: time, size, and proximity weight.
- T: time credit, capped at one hour, floored at five minutes.
- S: remaining quantity of the order.
- P: price proximity as a fixed-point fraction.
- Pk: linear taper from threshold to equality.
- Empty-book rule: parameter changes only while no qualifying orders rest.
- Top-N filter: the per-sweep cap on orders by standing size.

## 11. Provenance

This whitepaper adapts the structure of the Prism and Reciprocity whitepapers in the salvaged reference tree. The concepts are original to the BitShares Track B program, drawn from the Prism protocol. The design detail lives in `docs/superpowers/specs/2026-10-25-bsip5-open-order-rewards-design.md`. The reference implementations live under `docs/reference/contracts-prism`, including the Python float reference and the per-chain ports. The canonical fill behavior is defined by the Python reference.

## 12. Test cases

- Conservation: on each fill, the fee charged in the received asset equals exactly the rebate plus the pool share, with truncation dust kept in the pool and nothing lost.
- Rebate timing: the rebate settles into the maker's vesting balance at fill and stays claimable after the order fully fills or cancels.
- Minimum age: an order younger than the bottom gate earns zero weight at that sweep, whatever its size and price.
- Amendment reset: a same-price quantity change and a price change both restart the time credit, so a parked ghost order cannot outwait an active manager.
- Proximity screen: an order at or beyond the threshold earns zero weight, and inside the passing band the linear taper pays more the closer the order.
- Top-N bounds: the sweep keeps the largest standing orders first up to the per-sweep cap, before any weight arithmetic, so the loop cost never scales with book size.
- Empty book: fees accrue while the book is empty, the pool never loses them, the first post-empty order cannot claim the whole lot, and a sweep with no eligible orders pays nothing.
- Parameter gate: a rate or split change refuses while qualifying orders rest on the affected market.
- Self-trade refusal: a fill whose aggressive and resting orders share an owner rejects outright.
- Wash-trade bound: an operator owning a fraction of the book nets strictly negative below full ownership and breaks even only at full ownership, and only before the chain's own transaction fees, which are a real cost on every cycle.
- Cold start: an opted-in market with no fill data in the reference window skips that sweep's payouts without skipping the pool's accrual.
- Wire pinning: the asset-option field, the market reward object, and their object ids append and hold stable across builds with and without the hardfork flag.

# Copyright

This document is placed in the public domain.
path: /workspace/docs/whitepapers/bsip-5-open-order-rewards.md