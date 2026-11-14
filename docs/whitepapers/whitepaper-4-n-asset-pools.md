<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-4 — Parallel N-asset liquidity pools: geometric-mean invariant, imbalance-priced fee, and pool-computed price oracle

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
8. Max pool and set sizes
9. Oracle windows
10. Glossary
11. Provenance
12. Test cases

---

## 1. Narrative

BitShares swap pools each hold two assets and price by constant product, so liquidity for bigger baskets sits scattered across chains of pairwise pools, and every pool charges a rate fixed by its creator. This proposal adds a parallel pool class. One pool holds up to eight assets, the pool's share token is a proportional claim on the whole basket, and pricing follows a geometric-mean invariant, a single number computed from every token ratio at once. Each swap's fee is derived from how imbalanced the trade leaves the pool, and two self-steering regulators move that fee toward volatility targets with no scheduled parameter changes. Every pool publishes its own time-and-volume weighted average price, computed in consensus from the pool's own trades. Legacy two-asset pools stay untouched. The pitch: deposit a basket once, and every asset inside it trades, pays, and publishes from one pool.

The chain's trade machinery compares well with what other chains run, and the weak part is not the machinery but the shape of the liquidity sitting in it. Order books hold about fifteen dollars of depth per pair, so trading routed to the swap pools instead, and the pools disagree with each other by roughly 86 percent about the same reference pair, since the BTS/USD pool cleared 59.7 BTS per bitUSD while the BTS/CNY pool cleared 15.6 BTS per bitCNY, an implied cross-rate of about 3.8 CNY per USD, from dividing the two pool prices (source: a 2026-09-26 snapshot from a public API node by `tooling/bts_chain_snapshot.py`, recorded in `AGENTS.md`, section 6, chain facts). Each pool holds exactly two assets, because the existing liquidity pool design is a two-asset constant-product exchange (source: `contracts/reciprocity/docs/ports/bitshares/bsip.md`, Motivation). A trader wanting one honest price between two assets gets one from each pool separately, and the gaps between them are where the disagreement lives.

The structural fix is a pool that holds more than two assets. An N-asset pool, where N is the number of assets in the pool, lets a liquidity provider, someone who deposits assets into a pool in exchange for a share of it, deposit a diversified basket in one transaction, lets the pool price any pair of its own assets from one shared state, and reduces divergence loss for pools of three or more assets relative to the same exposure run as separate two-asset pools (source: `contracts/reciprocity/docs/ports/bitshares/bsip.md`, Motivation). Inside such a pool, arbitrage pushes each asset's share of pool value toward its declared weight, so an 80 percent BTC and 20 percent CNY basket rebalances itself as prices move (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 14.1).

The second gap is the fee. A constant-product pool charges the rate its creator set, whether the trade is balanced or violently lopsided. The design below prices each swap by imbalance: a fairness curve measures how far the trade leaves the pool from proportion, and the fee follows it quadratically, so balanced trades cost little and lopsided ones carry their own cost (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 2). Two regulators, one on the average fee level and one on its volatility, steer the fee continuously, which puts fee calibration in consensus arithmetic instead of in a scheduled governance vote (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 6.1).

The third gap is publishing. A pool's own trades already contain price information, and this design turns that into a public record: each pool accumulates a time-and-volume weighted average price, or TVWAP, over three fixed windows, computed by every node from the pool's own state (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 7). No external feed is trusted, no committee approves a price, and the oracle costs nothing to run. The record stays a public aggregate for readers and tools; it does not gate smartcoin debt, because market-pegged debt is authorized by bonded price oracles, not by pool telemetry (settled decisions 1 and 3 in `AGENTS.md`, section 4).

The formulas above are not inventions of this document. The reference implements them exactly, with identical integer outputs, across 15 blockchain platforms (source: `contracts/reciprocity/README.md`; `contracts/reciprocity/AGENTS.md`, Identity), and its tree contains a worked BitShares port carrying these pools as objects, operations, and evaluators (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, sections 1, 4, and 12). What follows distills that reference for this chain.

Status, stated plainly. The protocol named Reciprocity exists and runs in the reference tree; this program's core branches do not carry it, they carry the three proposals BSIP-1 through BSIP-3. Nothing in this document switches on, and nothing here touches the `bit*`, `HONEST.*`, or gateway assets (settled decision 1 in `AGENTS.md`, section 4). Before any build or vote, the work begins with its own design spec, test vectors, and the wire-format pinning discipline the other proposals use.

The project name is the settlement rule: whenever value enters the pool, the pool can settle part of it in its own share tokens, and the net share change across a swap is the number the reference calls reciprocity, refused if it would go negative (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 4.2).

## 2. Specification

Consensus law throughout, matching the program's rules: no floats, no `unordered_*` containers, head-block time only; operations and extension fields appended, never inserted; every new operation hardfork-gated in its evaluator and in the proposal hardfork visitor; bounded state; 128-bit comparison with declared rounding.

### 2.1 What ships

Two new object types ship, the N-asset pool itself and the factory provenance record that attests how each pool was created. Six operations ship behind one hardfork flag: pool creation, deposit, withdraw, exchange, and the two fee claims of the royalty model in section 2.5. New object types take fresh ids appended after the existing registry, and new operations take fresh wire tags appended after the operations the approved proposals already own; the exact numbers belong to the design spec that precedes any build (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, sections 1.3 and 12). The pool asset bounds are 2 to 8 tokens, eight fitting the protocol-level operation size limit that constrains the port (source: `contracts/reciprocity/docs/ports/bitshares/bsip.md`, Rationale, the eight-token maximum; the BitShares per-port limit is committee-set at hardfork rather than runtime-changeable, source: `contracts/reciprocity/docs/whitepaper/max_tokens.md`, table row "BitShares").

The pool's share token, the object representing a proportional claim on the whole basket, is a user-issued asset created with zero initial supply, and market-issued assets are refused as pool shares (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, section 1.4). That choice settles the privacy consequence the program has already fixed: pool shares are ordinary visible assets, so a blind liquidity position is impossible by construction, as market and vault paths stay transparent-aggregate permanently (BSIP-2, Specification, permanent non-goals).

### 2.2 The invariant

The pool prices with one number, called the konstant, K. For a pool of n assets with balances b and genesis balances g, the balances recorded at first capitalization, the equal-weight form is:

    K = s × ( (b1/g1) × (b2/g2) × ... × (bn/gn) )^(1/n)

where s is a scale factor, 10 raised to the digit count of the largest genesis balance plus a fixed buffer of 4, so the integer arithmetic has headroom (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 1.1). Genesis balances are set at first capitalization and updated only on re-initialization after a full exit, because re-anchoring them on every operation would make K path-dependent (source: section 1.3). A weighted form lets the creator bias composition, each token a weight over the sum of weights in the exponent; the normalization keeps K homogeneous of degree 1, doubling all balances doubles K, which share accounting requires (source: section 1.2).

Consensus computes K without floats through a Padé approximation, a rational-function approximation of the logarithm and exponential accurate over the relevant ranges, evaluated on ratios r = b × BASIS / g with a fixed-point denominator of 10,000, then combined by the log-sum-exp path or an integer nth root in the equal-weight case (source: section 1.4). Accuracy, from the reference's own analysis: exact for uniform ratios, within 0.1 percent for ratios between one half and two, within 1 percent up to a two-times single-token deviation, about 13 percent error at a ten-times single-token deviation in the worst case (source: section 1.4; `contracts/reciprocity/docs/whitepaper/exploits.md`, section 1.2). The matching risk is priced in Risks: what a trade must pay to reach that deviation dwarfs the error.

### 2.3 The fairness curve: aequitas

Aequitas, from the Latin for fairness, measures how imbalanced a swap leaves the pool. Consensus compares pre-trade and post-trade balances, computes per-token ratios between the smaller and larger state, averages the absolute pairwise distance among those ratios, and applies a quadratic curve:

    dev = max(0, BASIS - avg_err)
    spread = 1 + BASIS - dev² / BASIS

giving a spread from 1 basis point at the floor to roughly the full basis when the trade is maximally imbalanced (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 2.1). A spread near 1 means every token moved proportionally; a spread near the full basis means one token was dumped and another bought. The floor keeps a balanced trade from producing a zero that would stall the regulators (source: section 2.1). The curve's monotonicity and convergence are proved in the reference's theory document (source: `contracts/reciprocity/docs/whitepaper/theory.md`, section 5, cross-referenced from spec section 2.3).

### 2.4 The fee

The fee rate is the fairness spread passed through the regulators of section 2.7, and the fee is charged on each output asset leg as

    fee_i = ceil(amount_out_i × spread / BASIS)

with two bounds. The hard bound is the trade itself: the fee can never exceed the received amount. The floor is drip protection: an output of 3 units or fewer is taken entirely as the fee, and otherwise the fee is floored at 3 units, so dust farming cannot strip a pool three units at a time while the royalty split of section 2.5 keeps at least a unit for each recipient (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 3.1). There is no fee cap constant, on purpose; its absence is defended in Rationale.

### 2.5 Fee split: reinvestment and the two named takes

Fees split three ways inside consensus arithmetic, with guaranteed units first and percentages second, so tiny fees still pay every recipient at least one unit and nothing can underflow (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 3.2). The reference hard-codes a founder royalty and a promoter royalty at 250 basis points each, 2.5 percent, with no setter operation (source: section 3.2 and section 16.2). The LP share, the remainder, is not paid out: it is reinjected into the pool, so fee income compounds into every share token, and the accounting identity is that the share supply equals the konstant of the reinvested balances after the split (source: section 4.1).

This proposal adopts the reference's fee circuit but maps the two named takes onto BitShares governance, as the reference's own BitShares port already does: the founder-style share routes to the committee account and the promoter-style share to each deposited asset's issuer, with global royalty parameters that are committee-controlled and bounded between 0 and 500 basis points each (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, section 5.1). Payment to the two claimants runs through the chain's existing vesting balance system, balances that must age under a vesting policy before they can be claimed; on whether claims create vesting balance objects or transfer directly from factory accumulators the reference's own port notes diverge, and this proposal fixes the vesting route for its on-ledger audit trail (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, section 7 note, and section 8.2 note). The default at activation is 0 for both, so pools open with LPs keeping 100 percent of swap fees, and any take requires a visible governance action rather than a hidden default. Because asset issuers already earn market fees on their own assets under the existing schedule, the issuer share duplicates an existing lane, and a committee that sets it above zero twice pays the same traders once. That redundancy is part of why the default is 0 and the cap is small.

### 2.6 The swap and the reciprocity rule

An exchange order carries vectors of assets and amounts to sell and to receive, an optional amount of share tokens to stake, a minimum share floor, and an optional deadline (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 8.2). The pool computes post-trade balances, runs the fairness curve, charges the fee, then settles what remains in shares: reciprocity equals the shares staked plus the reinvested konstant minus the pre-trade konstant, and a negative result rejects the whole trade rather than burning the trader's shares (source: section 4.2). The rejection is a griefing defense, since letting a trader destroy share value at zero cost invites pool attacks; the same net position is reachable legitimately by unstaking instead (source: section 4.2). The share floor carries slippage protection, slippage being the gap between the shares quoted at submission and the shares settled at execution, and rejects the trade rather than delivering less than promised.

### 2.7 The governor

Two channels steer the fee. An exponential moving average, EMA, tracks the average spread over 500 observations, and an average true range, ATR, tracks spread volatility over 100. When the average spread sits above its target of 50 basis points the EMA channel lowers a fixed additive by 1, and when below, raises it by 1. When volatility sits above its target of 10 basis points the ATR channel shrinks a multiplicative rate by 1 percent per swap, and when below, grows it by 1 percent. Both channels carry their fractional remainders in dedicated accumulators, so integer division never deadlocks. The first two observations are warmup: the pool prices at the EMA target while the channels seed, and full convergence takes hundreds of swaps, the intended smoothness (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 6.1; `contracts/reciprocity/agents/ref.md`, Spread Governor).

The consequence is a fee policy with no scheduled changes: pools deep in sandwich traffic watch the channels drive fees down to attract stabilizing arbitrage, and calm pools slowly restore revenue, all in deterministic arithmetic every node computes identically (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, section 1.1). Targets are set at creation, bounded at 100 basis points, and reconfigurable only by re-initialization after a full exit (source: section 8.3).

### 2.8 Pool operations

Creation sets identity once: the token set, optional weights, the governor targets, and the royalty recipients; it transfers nothing (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 8.1). Deposits mint shares equal to the konstant growth and set genesis balances on first capitalization; a re-deposit into a fully exited pool re-initializes, updating genesis and optionally weights and targets, minting shares so the first depositor after a full exit is not diluted to nothing (source: section 8.3). Withdrawals burn shares and pay pro-rata balances.

Full exit has one rule worth stating: an account holding every remaining share sweeps the entire pool with zero dust, the LP royalty accumulator and all oracle state reset, the royalty accumulators of the two named roles persist across the exit because they were segregated at swap time, and the timestamps are restored only after the reset, because restoring before resetting would poison the next window with an astronomically large time delta (source: section 8.4, section 7.5, and section 8.5's full-exit note). Nobody can drain a pool silently: a full exit is visible, and the identity, the token set and weights, survives it (source: section 9).

Two invariants hold across every operation (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 9): share supply tracks the konstant of actual balances approximately, with bounded drift from incremental updates instead of full recomputation, since recomputing from K drifts the supply away from the share token's real supply by about 0.03 percent per operation (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, section 9); and the once-initialized flag never resets, so creation cannot be replayed into a live pool.

### 2.9 The pool oracle

Every state-mutating pool operation updates three volume accumulators, one each for 15-minute, 1-hour, and 24-hour windows, measured with head-block time only. The volume metric is the trade's gross inflow: the konstant of balances after the inflow is added but before the outflow is removed, which measures volume by the inbound leg and cannot be understated by choosing a small output (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 7.2). Within one block the time delta is zero, so nothing moves the accumulators inside a block, and a flash loan cannot fabricate a price; pushing the average requires holding an imbalanced position across blocks while paying fees (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, section 1.3). Queries are by exact window only, with a revert-on-empty rule for windows that hold no data, and the zero-window query returns the pool's spot reading, its share supply standing in for the konstant of balances (source: section 7.4). The BitShares port exposes this through read-only API calls returning pool state and window readings (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, section 10).

### 2.10 Conformance and limits

Everything above is computed in consensus, deterministically, under the standing rules. Arithmetically, every multiplication path carries explicit overflow checks in the reference and gets 128-bit math here (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, section 3.2). The weights cap at a sum of 100 after normalization by the greatest common divisor of the weight set, which bounds the pairwise arithmetic of the fairness curve (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 14.1 for the cap; `contracts/reciprocity/docs/ports/bitshares/engineering.md`, section 9 on why the normalization bound exists).

State is bounded by construction: a pool holds at most 8 asset ids, 8 balance and 8 genesis entries, per-token fee accumulators, and 13 fixed oracle fields; the open growth surface is the pool count, which the creation fee and a committee-set ceiling bound in the design spec, because unbounded object creation is the problem the settlement proposal's own review surfaced first (BSIP-3, Risks, the state-bound finding).

What does not ship matters as much as what does. This proposal creates no blind form of anything: the blind-to-blind pool exchange in BSIP-2 is proved for the constant-product kernel by an explicit value pin, and extending that proof to the geometric-mean invariant is new cryptography this design does not attempt, so the N-asset pools remain a transparent-aggregate path permanently (BSIP-2, Specification, blind pool swaps and permanent non-goals). Pools do not mint smartcoins and do not gate debt; the pool-issued index money class parked among the settled decisions stays parked (settled decision 8 in `AGENTS.md`, section 4). No foreign-chain anything lives in a pool: a pool holds what BitShares holds, and cross-chain settlement stays with the offer book of BSIP-3, designed honestly about what consensus cannot know.

## 3. Theory

The invariants the implementation must preserve:

- No floats anywhere in consensus; all math is integer or 128-bit fixed point.
- Every new operation and field is appended, never inserted, and hardfork-gated in its evaluator and in the proposal visitor.
- State is bounded: producer sets, history rings, subscriber lists, and claimant queues have fixed caps; a pool holds at most 8 assets.
- Aggregation is select-only where aggregation occurs; no weighted averages that hide a single bad input.
- Bonds and coverage, where referenced, denominate in USD terms at lock.
- The konstant is homogeneous of degree 1: doubling all balances doubles K.
- Genesis balances anchor K and are not re-anchored per operation.
- The once-initialized flag never resets.
- Legacy assets are untouched by construction.

Overflow discipline: 128-bit comparison for bond and coverage math, 256-bit conversion where a product of two 128-bit values is needed, and saturation on price conversion so a wrap becomes a refusal rather than a silent value.

## 4. Exploit analysis

- Sandwich loops: the reference's red-team analysis finds the attack unprofitable because the governor cuts fees exactly when volatility rises, and the attacker pays on both legs of the sandwich (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, section 1.1).
- Approximation error: at a ten-times single-token deviation the share accounting could err about 13 percent of the konstant, and the reference's bound on reaching that state is at least 30 percent of pool value paid in fees on both access and exit (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, section 1.2). The residual risk is a market where the deviation is cheap to hold, a deeply illiquid asset nobody arbitrages; the defense is creation-time validation refusing duplicates and refusing weights computed to game the sum cap (source: `contracts/reciprocity/docs/whitepaper/spec.md`, section 14.1).
- Flash loans: inside one block the time delta is zero, so no oracle accumulator moves and no price fabricates; pushing the average requires holding an imbalanced position across blocks while paying fees (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, section 1.3).
- Dust farming: the drip floor and the amount-received bound keep fees from being stripped three units at a time (source: section 2.4).
- Governance capture on royalties: committee and issuer rates are controlled, small, and default to zero; LPs can migrate to a new pool created by anyone, since creation is permissionless, and a royalty rise that exceeds the competitiveness of the asset's obvious pairs splits the pool rather than enriching the takers.
- Unbounded pool creation: bounded by the design spec, which must file a creation ceiling and per-account pool caps, because an object type whose instances grow with every spam transaction taxes every node forever, the same unbounded-state lesson the settlement proposal's own review surfaced first (BSIP-3, Risks).

## 5. Gas and cost analysis

Operations are consensus objects, not smart contract calls, so cost is measured in transaction fees and block weight. The per-operation math is bounded by the 8-asset cap and the 100-sum weight cap, so a swap's cost scales with the pool's asset count rather than with book depth. The governor's EMA and ATR updates are O(1) per swap. The TVWAP accumulators update once per state-mutating operation. State growth is bounded by the pool cap filed in the design spec.

## 6. Error code registry

New BSIP-4 errors occupy a reserved range appended after the existing registry. Each error maps to one refusal condition: pool not found, asset count out of bounds, market-issued share asset refused, genesis mismatch, weight sum out of bounds, swap negative reciprocity, share floor breached, full-exit not sole holder, re-initialization on live pool, window empty on query, overflow in safe multiplication. Legacy error codes keep their numbers.

## 7. Admin architecture

Admin is split between consensus and committee. Consensus owns pool creation, deposit, withdraw, swap, fee split, and oracle accumulation. The committee owns the hardfork date, the creation ceiling, per-account pool caps, and the global royalty parameters. No admin key can move pool balances by procedure alone. LPs and traders interact with pools through the six operations; the committee cannot reprice a live pool or re-anchor genesis.

## 8. Max pool and set sizes

- Pool assets: 2 to 8.
- Weight sum after normalization: 100.
- Producers per oracle: 32 (shared substrate).
- History entries: 64 (shared substrate).
- Subscribers: 64 (shared substrate).
- Pending guard slots: 16 (shared substrate).
- Offset-style claimants: up to 64 (shared substrate, where used).
- TVWAP windows: 3 (15 minutes, 1 hour, 24 hours).
- Governor EMA period: 500 observations.
- Governor ATR period: 100 observations.

## 9. Oracle windows

The pool oracle carries three windows: 15 minutes, 1 hour, and 24 hours, all measured in head-block time. Within one block the time delta is zero, so no accumulator moves inside a block. Queries are by exact window only, with a revert-on-empty rule for windows holding no data, and the zero-window query returns the pool's spot reading.

## 10. Glossary

- Konstant (K): the single number pricing the pool.
- Genesis balances: balances recorded at first capitalization, anchoring K.
- Aequitas: the fairness curve measuring swap imbalance.
- Spread: the fee rate derived from aequitas and the governor.
- Governor: the dual EMA and ATR regulator of the fee.
- Reciprocity: the net share change across a swap, refused if negative.
- Share token: the UIA representing a proportional claim on the basket.
- TVWAP: time-and-volume weighted average price over three windows.
- Drip protection: the fee floor of 3 units with full-take below it.
- Full exit: one account holding every share sweeps the pool.

## 11. Provenance

This whitepaper adapts the structure of the Prism and Reciprocity whitepapers in the salvaged reference tree. The concepts are original to the BitShares Track B program, drawn from the Reciprocity protocol. The design detail lives in `docs/superpowers/specs/2026-10-25-bsip4-n-asset-pools-design.md`. The reference implementations live under `docs/reference/contracts-reciprocity`, including the BitShares port under `docs/reference/contracts-reciprocity/docs/ports/bitshares/`. The canonical integer reference is `contracts/reciprocity/demonstrations/python/integer/`.

## 12. Test cases

- Invariant homogeneity: a deposit that doubles every balance against its genesis balance doubles K.
- Fee bounds: an output of three units or fewer is taken entirely as the fee; any larger fee stays under the amount received; the drip floor never strips a pool three units at a time.
- Split underflow: a fee too small for exact proportions still pays each named recipient at least one unit and reinvests the LP remainder.
- Reciprocity refusal: a swap whose settlement computes negative rejects outright and burns none of the trader's shares.
- Slippage floor: a trade that would settle below the share floor quoted at submission rejects instead of delivering less.
- Full exit: one account holding every share sweeps the pool to zero dust, the LP and oracle accumulators reset before their timestamps restore, and the once-initialized flag survives.
- Weight gate: a weight set normalizes by its greatest common divisor to a sum of 100, and a set built to game the cap refuses at creation.
- Oracle windows: inside one block the accumulators move zero, a query on a window holding no data reverts, and the zero-window query returns the pool's spot reading.
- Sandwich property: a sandwich loop nets a loss once the governor prices the volatility it creates.
- Approximation-error property: the share accounting error stays inside the bound quoted in Risks, and reaching that deviation costs more in fees than the error is worth.
- Flash property: a sequence inside one block moves no oracle accumulator and no price.
- Wire pinning: operation tags and object ids append and hold stable across builds with and without the hardfork flag.

# Copyright

This document is placed in the public domain.
path: /workspace/docs/whitepapers/bsip-4-n-asset-pools.md