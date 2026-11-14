    BSIP: (number assigned at filing; house label BSIP-5)
    Title: Open order rewards on the native markets: maker rebates and time, size, and proximity weighted fee distribution
    Authors: litepresence (HONEST)
    Status: Draft
    Type: Protocol
    Created: 2026-10-25

# Abstract

Resting an order on BitShares earns nothing. The maker, the party whose order sits in the book waiting, gives the market time, size, and a price the next trader can reach, and is paid nothing for it, so books sit thin and price discovery drifts to the swap pools. Each taker's fee funds a maker rebate, paid when his order fills, and a provider pool, distributed to open orders weighted by three measured qualities: how long the order has rested, capped at an hour; how much quantity remains; and how close its price sits to the market's own recent price. Gating, capping, and self-trade refusal keep the rewards unforfeitable and unpumpable. Participation is opt-in per asset, everything routed or paid stays on the ledger, and nothing changes for markets that never opt in. The pitch: waiting in the book becomes a paid service, priced in public by formula.

# Motivation

Books starve when waiting is free for everyone but valuable to everyone: each maker supplies the product the next trader consumes and collects nothing for it. Numbers, not arguments: a 2026-09-26 snapshot from a public API node by `tooling/bts_chain_snapshot.py` at head block ~114706236 (source: AGENTS.md, section 6, chain facts) found order books holding about 15 dollars of depth per book, so price discovery left the books, and the swap pools that absorbed it disagree by roughly 86 percent about the value of the chain's own reference assets. Meanwhile the taker, the trader whose order fills into a resting one, already pays a full fee, the issuer's market fee on what he receives, and none of it routes back to the book.

Competitive exchanges pay for resting liquidity, and the structure of what a resting order supplies is well known in the reference: an open limit order gives the market three measurable qualities, the time it has been available, the size of what it commits, and the proximity of its price to the market price, and an order old and large but far from it supplies nothing real, because takers cannot reach it. Rewarding all three together is the design's whole point, because a reward that pays for one quality alone pays just as well for faking the other two (source: `contracts/prism/docs/whitepaper/fees.md`, §1.1).

Two failure modes dominate any design that pays for resting orders, and both were met in the reference by gates rather than by trust. A fresh order sniping a payout wave must be priced out, and so must a ghost order parked once at a far price and never touched; the fixes are a minimum age before payout share, a maximum age credit that stops growing at an hour, and a reset of that credit on every amendment, so the reward tracks actively managed liquidity (source: `contracts/prism/docs/whitepaper/fees.md`, §4, §5.4). The target is plain: rewards small relative to the fees that fund them, flowing to orders the next taker would use anyway, and computable by every node during the maintenance interval every node already runs.

Status, stated plainly. The reward formula and its gates are distilled from the owner's order book project, which implements price-time matching and this fee distribution across more than a dozen chain ports with identical fill behavior (source: `contracts/prism/README.md`; `contracts/prism/AGENTS.md`, Identity). Nothing in this document is implemented in the BitShares core program: the three proposals BSIP-1 through BSIP-3 own the current build program, and this proposal stands beside them, uncontracted and unactivated. Before any vote there must be a design spec, wire pinning, test vectors, and the binary evidence the house's other proposals file, and no designed date or audit verdict exists to promise.

# What changes for you

- If you hold `bit*`, `HONEST.*`, or gateway assets, nothing about your balances, markets, or fees changes, and markets your assets never opt into behave exactly as they do now.
- If you trade as a maker, a fill of your resting order pays the rebate at once, and surviving orders share a daily pool by measured time, size, and proximity instead of earning nothing.
- If you trade as a taker, your cost structure grows only where the assets you trade have opted in, at a rate the issuer sets once, in public, and your fills still match in the same price-time order as today.
- If you witness, you gain no new duty: the pools accumulate and distribute inside the already scheduled maintenance sweep, bounded by a per-sweep order count.
- If you operate a gateway, your IOUs are ordinary assets here too, eligible only if you opt in, and deposits and withdrawals stay what they are.
- If you develop on BitShares, a market reward object appends, new per-asset fields append to asset options, and the sweep gains one bounded loop per participating market, with tags and fields pinned by tests before any hardfork.

# Specifications

This section is for developers and reviewers. The reward math is quoted from the reference fee document, and each adaptation is named as such. Readers outside development can stop here and lose only the numbers.

Consensus law throughout: no floats, head-block time only, append-only objects and fields, hardfork gate in evaluator and proposal visitor, bounded state, and every payout leaving an audit trail on the ledger, not in process memory.

## 1. Opt-in funding, one leg at a time

An asset opts in through a single new field in its options, an open order fee in basis points, hundredths of a percent, proposed to range from 0, the default, to 500, charged inside the fill accounting on the asset the taker receives, in the same leg the existing market fee is charged. A market exists between two assets, and either leg may carry its own fee, so a market accumulates at most two pools, one per participating asset, each denominated in that asset (source: `contracts/prism/docs/whitepaper/spec.md`, §4.1, where the whole taker fee splits per fill; the per-asset leg rate is the adaptation, following how market fees on this chain already ride the received asset). No escrow, no exterior contract, no new permission: issuers already hold exactly this authority over their own market fees.

Parameters change only through the empty-book rule: a rate or a split may change only while no rewards-enabled open orders rest on the affected market, so no resting order can be repriced after the fact (source: `contracts/prism/docs/whitepaper/spec.md`, §8.2). Beyond the per-fill split inside section 3, the two named role fees of the reference, a founder share and a promoter share, are dropped for this chain rather than mapped, because the chain has no exchange operator to pay and creating two new fee recipients at the head of every book would be a new governance surface the program's settled design avoids; the full fee minus the rebate goes to the pool (source of the two roles: `contracts/prism/docs/whitepaper/fees.md`, §10.1; the removal is this proposal's adaptation, argued in Rationale). Review cadence: the open-order fee range (0..500 bps) and the rebate/pool split are standing parameters re-reviewed by the committee at each hardfork that touches this proposal and at least annually otherwise; the empty-book rule is the guard between reviews, not a substitute for them.

## 2. Book state: one field per open order

Each open order that qualifies carries exactly one consensus field: the block height at which it entered the book or was last amended (source: `contracts/prism/docs/whitepaper/spec.md`, §1.5, where the fee state is that single field by design). Nothing is computed or stored per event; the reward weight is a snapshot at distribution time. Full fills and cancels delete the field, amendments regenerate it, and the order's remaining quantity, which every resting order already has, supplies the size leg (source: `contracts/prism/docs/whitepaper/spec.md`, §3.4). Every amendment resets the entry: a same-price quantity change and a price change both restart the clock, which is what stops an active manager from being beaten on time by a parked order (source: `contracts/prism/docs/whitepaper/fees.md`, §5.4 and the amend-hike analysis in `contracts/prism/docs/whitepaper/exploits.md`, §2).

## 3. The per-fill split

On each fill of a participating asset, the open order fee charges on the taker's received amount, then splits:

    maker_rebate = remaining × rebate_pct / 100
    provider_pool_share = remaining - maker_rebate

with the rebate percentage proposed at 70 percent of the remaining fee and the pool taking the other 30 percent, the reference's default beyond its extra two named cuts (source: `contracts/prism/docs/whitepaper/spec.md`, §4.1). The rebate pays to the maker whose order was consumed, immediately, into a vesting balance, the chain's existing system for balances that must age before claim, and it remains claimable after his order has fully filled or been cancelled, since it is settled at fill rather than at distribution (source: `contracts/prism/docs/whitepaper/fees.md`, §9). Pool shares accumulate in the market's per-asset pool, and conservation holds exactly: what is charged equals what is rebated plus what pools, with integer truncation dust staying in the pool until the next distribution rounds it out (source: `contracts/prism/docs/whitepaper/fees.md`, §8.2 and §12.3).

## 4. The reward weight: time, size, proximity

At each maintenance sweep, every qualifying open order on a participating market computes one weight:

    T = min(payout_block - placement_block, MAX_T_BLOCKS), zero credit below MIN_T_BLOCKS
    S = remaining quantity of the order
    P = min(order price, reference price) / max(order price, reference price), carried as a fixed-point fraction with the denominator BASIS, a scale constant the design spec fixes at build time
    skip entirely if P < PROXIMITY_THRESHOLD, proposed at 0.9, an order ten percent or more from price
    Pk = 10 × P - 9 × BASIS, with both P and Pk carried in BASIS units
    TSP = S × T × Pk / BASIS, the single fixed-point correction, taken once at the end

(source: `contracts/prism/docs/whitepaper/spec.md`, §5.1; `contracts/prism/docs/whitepaper/fees.md`, §2). The weight the reference names TSP, for time, size, and proximity, has three legs that multiply because each is necessary: an order stale in time, small in size, or far in price is not the liquidity this design wants to buy, and a zero in any leg must zero the weight (source: `contracts/prism/docs/whitepaper/fees.md`, §1.2). The time leg is capped at 300 blocks on the 12-second reference chain and its bottom gate sits at 25; these constants convert to block counts at compile time per chain, so the effective interval stays 5 minutes to 60 minutes everywhere, and at this chain's three-second block schedule that is MIN 100 blocks and MAX 1200 (source: `contracts/prism/docs/whitepaper/fees.md`, §13.2 and §13.4). Proximity uses the linear taper: Pk maps the interval from the threshold to exact equality linearly onto zero to one times the denominator, because a step cutoff creates a cliff to cluster against just inside the threshold (source: `contracts/prism/docs/whitepaper/fees.md`, §3.1 and §3.2).

The reference prices proximity against its own time-and-volume weighted oracle reading and falls back to the last traded price when the oracle window holds no data yet (source: `contracts/prism/docs/whitepaper/fees.md`, §2.1; `contracts/prism/agents/ref.md`, invariant 17, and `contracts/prism/docs/whitepaper/exploits.md`, §6.1). This proposal adapts that to data the chain already holds: each participating market keeps a rolling accumulator over its own fills, time-weighted across a one-hour window, and reads that accumulator's average as its reference price, with no external feed in the path. Unlike the reference, there is no last-traded-price fallback: a market with no fill data in the window pays nothing that sweep, because a reference price left over from an old fill is a number any small trade could have planted long before the payouts.

## 5. Distribution and its bounds

The sweep first filters each market's book by remaining quantity, keeping the largest standing orders up to a per-sweep cap proposed at 256 against the reference's 1,000, because consensus pays the loop cost here and the sweep must stay bounded whatever the book size (source of the filter and its ordering: `contracts/prism/docs/whitepaper/spec.md`, §5.3, and `contracts/prism/docs/whitepaper/fees.md`, §6). It then distributes the market's accrued pool for that asset among those survivors, each receiving pool × its TSP weight over the sum of all eligible weights, and the pool's integer remainder after distribution stays in the pool for the next sweep (source: `contracts/prism/docs/whitepaper/fees.md`, §2.6 and §8.2). The pool can only empty through distribution, since every fill only adds to it, and a sweep with no eligible orders costs nothing and pays nothing (source: `contracts/prism/docs/whitepaper/fees.md`, §12.3 and §8.3).

An empty book does not leak. Fees accrue while the book is empty, the minimum-age gate keeps the first post-empty order from collecting everything, and the pool stays monotone until enough qualifying orders exist to split it (source: `contracts/prism/docs/whitepaper/exploits.md`, §1, the empty-book drain, which the reference's minimum-time gate was built to close).

## 6. Anti-abuse, quoted and adapted

- Fresh-order sniping is priced out by the minimum age, since a payout at any block can only be claimed by orders at least 100 blocks old at this chain's schedule (source: `contracts/prism/docs/whitepaper/fees.md`, §4.2).
- Ghost order dominance is capped by the maximum time credit, so the oldest order earns no more weight than one an hour old, at most 12 times the youngest qualifying order (source: `contracts/prism/docs/whitepaper/fees.md`, §5.2 and §5.3).
- Pool draining through an emptied book is covered by the same gate, stated in section 5 (source: `contracts/prism/docs/whitepaper/exploits.md`, §1).
- Payout floods of tiny spam orders are bounded twice: the sweep keeps only the top orders by standing size before any weight arithmetic, and inside that reduced set the proximity screen zeroes far-priced orders (source: `contracts/prism/docs/whitepaper/spec.md`, §5.3; `contracts/prism/docs/whitepaper/exploits.md`, §3; `contracts/prism/docs/whitepaper/fees.md`, §6.3).
- Cross-account wash trading is unprofitable in the reference by an explicit proof, and the shape carries over: the maker's order exits the pool on fill and earns nothing more from it, the same owner pays the full fee on the taker leg, and an operator owning the fraction f of the book at distribution nets the rebate plus f × pool minus the fee. Under the 70-to-30 split of section 3 the rebate plus the pool is the whole fee, so the loss is 30 percent of the fee times the book share the operator does not own, strictly negative for every f below 1 and exactly zero at f = 1, with the chain's transaction fees for the two orders still due on top (source: `contracts/prism/docs/whitepaper/fees.md`, §12.2, adapted here to the two-way split of section 3, where the formula's remaining input is the whole fee because no named cuts exist).
- Self-trading within one account is refused outright on participating markets: the fill is rejected when the aggressive order and the resting order belong to the same owner, the stronger owner-level check the reference's on-chain ports use over id comparison (source: `contracts/prism/docs/whitepaper/spec.md`, §3.3).

## 7. Scope it refuses

Matching stays exactly as it is: price-time priority, same fill accounting, no new order types, and nothing here reaches margin calls, force settlement, or the settlement offer book of BSIP-3, whose offers fix price at match and have no resting rewards to distribute (BSIP-3, Specification, section 6, out of scope by design). Per-fill provider rewards are refused on purpose: paying a filled order from the pool would double-pay the same order's rebate, and the reference deletes reward eligibility at fill for exactly that reason, so the pool pays future liquidity while the rebate pays past execution, never both to once (source: `contracts/prism/docs/whitepaper/exploits.md`, §6.2). The reward is public by rule, pools, weights, gates, and every payout being on-ledger records, because anything resting in a book is a standing public pseudonym, and this program's settled design rejects order-book blindness permanently, not as a future option (BSIP-2, Specification, permanent non-goals; AGENTS.md, section 4, settled decision 4). Nothing here attests anything about a foreign chain, cross-chain or otherwise, so the attestor ceiling of the program never applies.

# Rationale

Open order rewards buy the chain's scarcest good: standing liquidity on books whose alternative is disassembled order flow. The three-legged weight is the honest shape of that good, since time, size, and proximity are each cheap to fake alone, and an old, large order ten percent from price is standing liquidity a market does not want to pay for. The multiplicative form bakes that judgment into arithmetic, not discretion: every leg gates, no leg overweights alone (source: `contracts/prism/docs/whitepaper/fees.md`, §1.1 through §1.3).

Snapshot weights over per-event accounting is the load-bearing state-shaped decision. Each open order carries one block field, weights are estimated at distribution from data the book already holds, and no per-event write is needed, so the feature's cost lands inside the sweep rather than on every fill's accounting path (source: `contracts/prism/docs/whitepaper/spec.md`, §1.5 and §5.1). The per-sweep top-N cap and the block-based constants keep the sweep simulatable in bounded consensus work; the reference prices that same work in gas, a per-computation meter this chain does not have, so the cost here is witness time during maintenance.

The rebate-pool bifurcation separates past and future liquidity, paying the consumed order now and the standing orders later, refusing both rewards to any one order at any time (source: `contracts/prism/docs/whitepaper/exploits.md`, §6.2). Filled makers exit pool eligibility by rule, not by penalty, so fill-and-earn loops against a collaborator have nothing to loop into. The empty-book rule on parameter changes keeps issuers from repricing a live book, as the reference keeps its own promoter from doing it (source: `contracts/prism/docs/whitepaper/spec.md`, §8.2).

Dropping the founder and promoter cuts is the biggest divergence from the source, and it cuts the wash-trade margin from 5 percent to zero at full ownership, which the Risks section hands to the arithmetic: the reference's proof ends with losses bounded below by those cuts plus gas, and this adaptation can only promise that rebate plus pool equals the fee exactly, so a wash operator who owns the whole book breaks even before the chain's own transaction fees, which are a real cost on every cycle, not a profit for anyone. Named fee recipients would be a real capture surface, and the empty default of section 1 keeps every market on this chain a zero-new-take market until its own issuers act.

Rejected alternatives, each with its reason. Funding from the legacy market fee itself was rejected: touching every existing market fee flow without opt-in is exactly the legacy migration the program refuses. Paying rewards per fill per taker was refused as unbounded consensus work and double-paying. Rewarding takers was refused because takers already pay, and paying them back is a fee rebate in costume. Proximity against an external feed was refused because every unbonded price is the cartel lesson priced; the reference marks oracle dependency as its residual external risk (source: `contracts/prism/docs/whitepaper/exploits.md`, §6.1). Reputation-earned weights were refused because a patient attacker farms them, as with every reputation channel the program has already rejected (BSIP-1, Rationale).

# Risks and failure modes

The time leg saturates. Between the one-hour cap on time credit and the daily maintenance interval the rewards ride, most mature orders sit at the cap, and the working gradient that remains is size and proximity; more frequent distribution would add sweep work and buy more time differentiation, a cadence decision the design spec must weigh and file rather than this document assume. The daily cadence is the chain's default, committee-settable as the maintenance_interval parameter (source: `bitshares-core/libraries/protocol/include/graphene/protocol/config.hpp`, the one-day interval define, and `bitshares-core/libraries/protocol/include/graphene/protocol/chain_parameters.hpp`, the parameter field). Rewards therefore start life as a size-and-proximity market, which is still the majority of the value the formula measures.

A hostile issuer controls the tap on their own asset. Raising the open order fee taxes that asset's takers, which today's market fee already lets them do, the empty-book rule keeps them from repricing live orders, but the rebate percentage choice can route most of it back to whoever they prefer. The 500 basis point cap, the zero default, the visible split, and the top-N cap bound the damage, and a market whose books dislike the terms can trade wherever the other leg lives.

Wash trading is not impossible, only unprofitable in aggregate; the residual levers matter: the operator pays the fee cycle on every round trip, and the pool share he recovers rises with his fraction of the book, so a monopolist of a thin market can run near break-even before the chain's own fees, and no rule here turns that into a profit. The stronger owner-level self-trade refusal of section 6 closes the account-level version (source: `contracts/prism/docs/whitepaper/fees.md`, §12.2; `contracts/prism/docs/whitepaper/spec.md`, §3.3).

State growth lives in the reward field per open order and in the pool accumulator per market, plus whatever caps the design spec puts on participating markets per asset. Unbounded book growth itself predates this proposal, and rewards make resting orders slightly more attractive to place and abandon; the top-N filter at sweep time already prices abandoned far orders out of the arithmetic, but the memory cost of abandoned orders is a graph bloat question the design spec must cap explicitly, as the settlement proposal's review demanded (BSIP-3, Risks).

Cold start is real: an opted-in market with no recent fills has no proximity reference, and pays nothing that sweep. That is the intended fail-closed shape: the feature rewards markets that trade, not markets that promise to trade. It also means the first pools accrue while the book is empty, gated as section 5 states, so early qualifying orders split a small pool rather than a windfall.

No test vector, no build, no activation. The gates this proposal files for itself are ordinary: design spec with wire pinning and bounded-state caps, assertion suite including the wash-trade check and the adversarial vectors the reference already names (source: `contracts/prism/docs/whitepaper/exploits.md`, sections 1 through 5), then replay and soak evidence before any committee date or activation vote. Nothing here promises a date, a test count, or a verdict.

# Backwards compatibility

Nothing migrates. Markets that never opt in keep every behavior they have, since the funding leg, the split, the field, and the sweep all live behind one hardfork flag and per-asset participation. New fields append to asset options and new object types append to the registry, so no existing wire format changes, orders and books price identically, and a wallet that never learns the field keeps working on non-participating markets. Gate dates are the committee's to set or not set, and where the flag has not passed, the opt-in and sweep refuse at apply, the program's established fail-closed pattern. The rebate payment target is the vesting balance system, which already exists and needs no new type.

# Test cases

Each row names behavior to pin before any build, derived from what this document's specification already states and cites, and the last rows are the properties the reference's own adversarial analyses name. The suite does not exist, and this section claims no recorded result; it claims what any implementation must refuse or match. Every row belongs to the design spec's assertion suite and to the wire-format pinning the Reference implementation section files.

- Conservation: on each fill, the fee charged in the received asset equals exactly the rebate plus the pool share, with truncation dust kept in the pool and nothing lost (source: Specification, section 3).
- Rebate timing: the rebate settles into the maker's vesting balance at fill and stays claimable after the order fully fills or cancels (source: Specification, section 3).
- Minimum age: an order younger than the bottom gate earns zero weight at that sweep, whatever its size and price (source: Specification, section 4).
- Amendment reset: a same-price quantity change and a price change both restart the time credit, so a parked ghost order cannot outwait an active manager (source: Specification, section 2).
- Proximity screen: an order at or beyond the threshold earns zero weight, and inside the passing band the linear taper pays more the closer the order (source: Specification, section 4).
- Top-N bounds: the sweep keeps the largest standing orders first up to the per-sweep cap, before any weight arithmetic, so the loop cost never scales with book size (source: Specification, section 5).
- Empty book: fees accrue while the book is empty, the pool never loses them, the first post-empty order cannot claim the whole lot, and a sweep with no eligible orders pays nothing (source: Specification, section 5).
- Parameter gate: a rate or split change refuses while qualifying orders rest on the affected market (source: Specification, section 1).
- Self-trade refusal: a fill whose aggressive and resting orders share an owner rejects outright (source: Specification, section 6).
- Wash-trade bound: an operator owning a fraction of the book nets strictly negative below full ownership and breaks even only at full ownership, and only before the chain's own transaction fees, which are a real cost on every cycle (source: Specification, section 6).
- Cold start: an opted-in market with no fill data in the reference window skips that sweep's payouts without skipping the pool's accrual (source: Specification, section 4).
- Wire pinning: the asset-option field, the market reward object, and their object ids append and hold stable across builds with and without the hardfork flag (source: Backwards compatibility).

# Reference implementation

The proposal is a distillation of protocol sources read for this document, not a port that has landed in this program's core branches. In the reference tree:

- `contracts/prism/docs/whitepaper/fees.md` carries the reward mechanism quoted here: value dimensions §1, the TSP formula §2, the linear proximity taper §3, the minimum-time gate §4, the maximum-time cap and amend reset §5, the top-N filter §6, gas economics the bounded sweep replaces §7, distribution and conservation §8, the maker rebate §9, the fee flow and worked example §10 through §11, the wash-trade proof §12.2, and chain block conversions §13.
- `contracts/prism/docs/whitepaper/spec.md` carries the matching and lifecycle surfaces this proposal adapts: order types §1 through §2, price-time matching and amend semantics §3, the per-fill split §4, the payout formula §5, pool operations §6, claims and parameter change rules §7 through §8, and the invariants list §13.
- `contracts/prism/docs/whitepaper/exploits.md` carries the adversarial analyses cited throughout: empty-book drain §1, amend-hike amplification §2, payout flooding §3, sniping gated by minimum age §4, cross-account wash trades §5, the oracle-dependency residual §6.1, and the no-per-fill-reward rule §6.2.
- `contracts/prism/agents/ref.md` condenses the invariant set, including the last-price fill hook the proximity fallback rests on and the amend reset rule.
- Demonstrations live in `contracts/prism/demonstrations/python/` for fill behavior and `contracts/prism/contracts/solidity/` among the per-chain ports in `contracts/prism/contracts/`, with `contracts/prism/ui/` wrapping the demonstration.

What still needs writing before any build: the design spec in `docs/superpowers/specs/` fixing the asset-option field, the market reward object, sweep bounds, the payout cadence this document defers to it, block constant filing, and the self-trade refusal wire shape; the assertion suite with the adversarial vectors above; wire-format pinning across builds with and without this feature; and full-binary evidence gates before any replay or pilot discussion. None exists today, and this document claims none of it.

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
