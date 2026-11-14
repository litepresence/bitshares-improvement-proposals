    BSIP: (number assigned at filing; house label BSIP-4)
    Title: Parallel N-asset liquidity pools: a geometric-mean invariant, an imbalance-priced swap fee, and a pool-computed price oracle
    Authors: litepresence (HONEST)
    Status: Draft
    Type: Protocol
    Created: 2026-10-25

# Abstract

BitShares swap pools each hold two assets and price by constant product, so liquidity for bigger baskets sits scattered across chains of pairwise pools, and every pool charges a rate fixed by its creator. This proposal adds a parallel pool class. One pool holds up to eight assets, the pool's share token is a proportional claim on the whole basket, and pricing follows a geometric-mean invariant, a single number computed from every token ratio at once. Each swap's fee is derived from how imbalanced the trade leaves the pool, and two self-steering regulators move that fee toward volatility targets with no scheduled parameter changes. Every pool publishes its own time-and-volume weighted average price, computed in consensus from the pool's own trades. Legacy two-asset pools stay untouched. The pitch: deposit a basket once, and every asset inside it trades, pays, and publishes from one pool.

# Motivation

The chain's trade machinery compares well with what other chains run, and the weak part is not the machinery but the shape of the liquidity sitting in it. Every figure below was measured on 2026-09-26 against a public API node with `tooling/bts_chain_snapshot.py` at head block ~114706236 (source: AGENTS.md, section 6, chain facts): order books held about 15 dollars of depth per pair, so trading routed to the swap pools instead, and the pools disagree with each other by roughly 86 percent about the same reference pair, since the BTS/USD pool cleared 59.7 BTS per bitUSD while the BTS/CNY pool cleared 15.6 BTS per bitCNY, an implied cross-rate of about 3.8 CNY per USD, from dividing the two pool prices (source: same snapshot). Each pool holds exactly two assets, because the existing liquidity pool design is a two-asset constant-product exchange (source: `contracts/reciprocity/docs/ports/bitshares/bsip.md`, Motivation). A trader wanting one honest price between two assets gets one from each pool separately, and the gaps between them are where the disagreement lives.

The structural fix is a pool that holds more than two assets. An N-asset pool, where N is the number of assets in the pool, lets a liquidity provider, someone who deposits assets into a pool in exchange for a share of it, deposit a diversified basket in one transaction, lets the pool price any pair of its own assets from one shared state, and reduces divergence loss for pools of three or more assets relative to the same exposure run as separate two-asset pools, a theoretical property of the geometric-mean invariant stated in the reference (source: `contracts/reciprocity/docs/ports/bitshares/bsip.md`, Discussion, 'Reduced IL'); no quantified bound is cited and the claim is not measured. Inside such a pool, arbitrage pushes each asset's share of pool value toward its declared weight, so an 80 percent BTC and 20 percent CNY basket rebalances itself as prices move (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §14.1).

The second gap is the fee. A constant-product pool charges the rate its creator set, whether the trade is balanced or violently lopsided. The design below prices each swap by imbalance: a fairness curve measures how far the trade leaves the pool from proportion, and the fee follows it quadratically, so balanced trades cost little and lopsided ones carry their own cost (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §2). Two regulators, one on the average fee level and one on its volatility, steer the fee continuously, which puts fee calibration in consensus arithmetic instead of in a scheduled governance vote (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §6.1).

The third gap is publishing. A pool's own trades already contain price information, and this design turns that into a public record: each pool accumulates a time-and-volume weighted average price, or TVWAP, over three fixed windows, computed by every node from the pool's own state (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §7). No external feed is trusted, no committee approves a price, and the oracle costs nothing to run. The record stays a public aggregate for readers and tools; it does not gate smartcoin debt, because the settled rule here is that market-pegged debt is authorized by bonded price oracles, not by pool telemetry (AGENTS.md, section 4, settled decisions 1 and 3; BSIP-1, Specification, section 2).

The formulas above are not inventions of this document. The reference implements them exactly, with identical integer outputs, across 15 blockchain platforms (source: `contracts/reciprocity/README.md`; `contracts/reciprocity/AGENTS.md`, Identity), and its tree contains a worked BitShares port carrying these pools as objects, operations, and evaluators (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, sections 1, 4, and 12). What follows distills that reference for this chain.

Status, stated plainly. The protocol named Reciprocity exists and runs in the reference tree; this program's BitShares core branches do not carry it, they carry the three proposals BSIP-1 through BSIP-3. Nothing in this document switches on, and nothing here touches the `bit*`, `HONEST.*`, or gateway assets (AGENTS.md, section 4, settled decision 1). Before any build or vote, the work begins with its own design spec, test vectors, and the wire-format pinning discipline the other proposals use.

The project name is the settlement rule: whenever value enters the pool, the pool can settle part of it in its own share tokens, and the net share change across a swap is the number the reference calls reciprocity, refused if it would go negative (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §4.2).

# What changes for you

- If you hold `bit*`, `HONEST.*`, or gateway assets, nothing about your balances, markets, or fees changes, and no rule here can move, cap, or re-price them later either.
- If you are a liquidity provider, you can deposit up to eight assets into one pool and hold a share of the whole basket, with fees reinvested rather than collected and withdrawn.
- If you trade, you gain a venue where any two assets inside a pool price against each other from one shared state, with fees that shrink for balanced trades and grow for imbalanced ones.
- If you witness, you gain no new duty: the math runs in the evaluators you already run, behind one hardfork date the committee sets or does not set.
- If you operate a gateway, you are untouched, and pools against your tokens are just ordinary pools against ordinary UIAs, user-issued assets with no special custody.
- If you develop on BitShares, new operations append at fresh wire tags with new object types, old operations and fields stay where they are, and the pool oracle exposes itself through new read-only API calls.

# Specifications

This section is for developers and reviewers. Every formula and constant cites the reference specification it comes from, and the full math lives in `contracts/reciprocity/docs/whitepaper/spec.md`. Readers outside development can stop after the first paragraph of each subsection.

Consensus law throughout, matching the program's rules: no floats, no `unordered_*` containers, head-block time only; operations and extension fields appended, never inserted; every new operation hardfork-gated in its evaluator and in the proposal hardfork visitor; bounded state; 128-bit comparison with declared rounding.

## 1. What ships

Two new object types ship, the N-asset pool itself and the factory provenance record that attests how each pool was created. Six operations ship behind one hardfork flag: pool creation, deposit, withdraw, exchange, and the two fee claims of the royalty model in section 5. New object types take fresh ids appended after the existing registry, and new operations take fresh wire tags appended after the operations the approved proposals already own; the exact numbers belong to the design spec that precedes any build (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, sections 1.3 and 12). The pool asset bounds are 2 to 8 tokens, eight fitting the protocol-level operation size limit that constrains the port (source: `contracts/reciprocity/docs/ports/bitshares/bsip.md`, Rationale, the eight-token maximum; the BitShares per-port limit is committee-set at hardfork rather than runtime-changeable, source: `contracts/reciprocity/docs/whitepaper/max_tokens.md`, table row "BitShares").

The pool's share token, the object representing a proportional claim on the whole basket, is a user-issued asset created with zero initial supply, and market-issued assets are refused as pool shares (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, §1.4). That choice settles the privacy consequence the program has already fixed: pool shares are ordinary visible assets, so a blind liquidity position is impossible by construction, as market and vault paths stay transparent-aggregate permanently (BSIP-2, Specification, permanent non-goals).

## 2. The invariant

The pool prices with one number, called the konstant, K. For a pool of n assets with balances b and genesis balances g, the balances recorded at first capitalization, the equal-weight form is:

    K = s × ( (b1/g1) × (b2/g2) × ... × (bn/gn) )^(1/n)

where s is a scale factor, 10 raised to the digit count of the largest genesis balance plus a fixed buffer of 4, so the integer arithmetic has headroom (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §1.1). Genesis balances are set at first capitalization and updated only on re-initialization after a full exit, because re-anchoring them on every operation would make K path-dependent (source: §1.3). A weighted form lets the creator bias composition, each token a weight over the sum of weights in the exponent; the normalization keeps K homogeneous of degree 1, doubling all balances doubles K, which share accounting requires (source: §1.2).

Consensus computes K without floats through a Padé approximation, a rational-function approximation of the logarithm and exponential accurate over the relevant ranges, evaluated on ratios r = b × BASIS / g with a fixed-point denominator of 10,000, then combined by the log-sum-exp path or an integer nth root in the equal-weight case (source: §1.4). Accuracy, from the reference's own analysis: exact for uniform ratios, within 0.1 percent for ratios between one half and two, within 1 percent up to a two-times single-token deviation, about 13 percent error at a ten-times single-token deviation in the worst case (source: §1.4; `contracts/reciprocity/docs/whitepaper/exploits.md`, §1.2). The matching risk is priced in Risks: what a trade must pay to reach that deviation dwarfs the error.

## 3. The fairness curve: aequitas

Aequitas, from the Latin for fairness, measures how imbalanced a swap leaves the pool. Consensus compares pre-trade and post-trade balances, computes per-token ratios between the smaller and larger state, averages the absolute pairwise distance among those ratios, and applies a quadratic curve:

    dev = max(0, BASIS - avg_err)
    spread = 1 + BASIS - dev² / BASIS

giving a spread from 1 basis point at the floor to roughly the full basis when the trade is maximally imbalanced (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §2.1). A spread near 1 means every token moved proportionally; a spread near the full basis means one token was dumped and another bought. The floor keeps a balanced trade from producing a zero that would stall the regulators (source: §2.1). The curve's monotonicity and convergence are proved in the reference's theory document (source: `contracts/reciprocity/docs/whitepaper/theory.md`, §5, cross-referenced from spec §2.3).

## 4. The fee

The fee rate is the fairness spread passed through the regulators of section 7, and the fee is charged on each output asset leg as

    fee_i = ceil(amount_out_i × spread / BASIS)

with two bounds. The hard bound is the trade itself: the fee can never exceed the received amount. The floor is drip protection: an output of 3 units or fewer is taken entirely as the fee, and otherwise the fee is floored at 3 units, so dust farming cannot strip a pool three units at a time while the royalty split of section 5 keeps at least a unit for each recipient (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §3.1). There is no fee cap constant, on purpose; its absence is defended in Rationale.

## 5. Fee split: reinvestment and the two named takes

Fees split three ways inside consensus arithmetic, with guaranteed units first and percentages second, so tiny fees still pay every recipient at least one unit and nothing can underflow (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §3.2). The reference hard-codes a founder royalty and a promoter royalty at 250 basis points each, 2.5 percent, with no setter operation (source: §3.2 and §16.2). The LP share, the remainder, is not paid out: it is reinjected into the pool, so fee income compounds into every share token, and the accounting identity is that the share supply equals the konstant of the reinvested balances after the split (source: §4.1).

This proposal adopts the reference's fee circuit but maps the two named takes onto BitShares governance, as the reference's own BitShares port already does: the founder-style share routes to the committee account and the promoter-style share to each deposited asset's issuer, with global royalty parameters that are committee-controlled and bounded between 0 and 500 basis points each (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, §5.1). Payment to the two claimants runs through the chain's existing vesting balance system, balances that must age under a vesting policy before they can be claimed; on whether claims create vesting balance objects or transfer directly from factory accumulators the reference's own port notes diverge, and this proposal fixes the vesting route for its on-ledger audit trail (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, §7 note, and §8.2 note). The default at activation is 0 for both, so pools open with LPs keeping 100 percent of swap fees, and any take requires a visible governance action rather than a hidden default. Because asset issuers already earn market fees on their own assets under the existing schedule, the issuer share duplicates an existing lane, and a committee that sets it above zero twice pays the same traders once. That redundancy is part of why the default is 0 and the cap is small.

## 6. The swap and the reciprocity rule

An exchange order carries vectors of assets and amounts to sell and to receive, an optional amount of share tokens to stake, a minimum share floor, and an optional deadline (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §8.2). The pool computes post-trade balances, runs the fairness curve, charges the fee, then settles what remains in shares: reciprocity equals the shares staked plus the reinvested konstant minus the pre-trade konstant, and a negative result rejects the whole trade rather than burning the trader's shares (source: §4.2). The rejection is a griefing defense, since letting a trader destroy share value at zero cost invites pool attacks; the same net position is reachable legitimately by unstaking instead (source: §4.2). The share floor carries slippage protection, slippage being the gap between the shares quoted at submission and the shares settled at execution, and rejects the trade rather than delivering less than promised.

## 7. The governor

Two channels steer the fee. An exponential moving average, EMA, tracks the average spread over 500 observations, and an average true range, ATR, tracks spread volatility over 100. When the average spread sits above its target of 50 basis points the EMA channel lowers a fixed additive by 1, and when below, raises it by 1. When volatility sits above its target of 10 basis points the ATR channel shrinks a multiplicative rate by 1 percent per swap, and when below, grows it by 1 percent. Both channels carry their fractional remainders in dedicated accumulators, so integer division never deadlocks. The first two observations are warmup: the pool prices at the EMA target while the channels seed, and full convergence takes hundreds of swaps, the intended smoothness (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §6.1; `contracts/reciprocity/agents/ref.md`, Spread Governor).

The consequence is a fee policy with no scheduled changes: pools deep in sandwich traffic watch the channels drive fees down to attract stabilizing arbitrage, and calm pools slowly restore revenue, all in deterministic arithmetic every node computes identically (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, §1.1). Targets are set at creation, bounded at 100 basis points, and reconfigurable only by re-initialization after a full exit (source: §8.3).

## 8. Pool operations

Creation sets identity once: the token set, optional weights, the governor targets, and the royalty recipients; it transfers nothing (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §8.1). Deposits mint shares equal to the konstant growth and set genesis balances on first capitalization; a re-deposit into a fully exited pool re-initializes, updating genesis and optionally weights and targets, minting shares so the first depositor after a full exit is not diluted to nothing (source: §8.3). Withdrawals burn shares and pay pro-rata balances.

Full exit has one rule worth stating: an account holding every remaining share sweeps the entire pool with zero dust, the LP royalty accumulator and all oracle state reset, the royalty accumulators of the two named roles persist across the exit because they were segregated at swap time, and the timestamps are restored only after the reset, because restoring before resetting would poison the next window with an astronomically large time delta (source: §8.4, §7.5, and §8.5's full-exit note). Nobody can drain a pool silently: a full exit is visible, and the identity, the token set and weights, survives it (source: §9).

Two invariants hold across every operation (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §9): share supply tracks the konstant of actual balances approximately, with bounded drift from incremental updates instead of full recomputation, since recomputing from K drifts the supply away from the share token's real supply by about 0.03 percent per operation (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, §9); and the once-initialized flag never resets, so creation cannot be replayed into a live pool.

## 9. The pool oracle

Every state-mutating pool operation updates three volume accumulators, one each for 15-minute, 1-hour, and 24-hour windows, measured with head-block time only. The volume metric is the trade's gross inflow: the konstant of balances after the inflow is added but before the outflow is removed, which measures volume by the inbound leg and cannot be understated by choosing a small output (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §7.2). Within one block the time delta is zero, so nothing moves the accumulators inside a block, and a flash loan cannot fabricate a price; pushing the average requires holding an imbalanced position across blocks while paying fees (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, §1.3). Queries are by exact window only, with a revert-on-empty rule for windows that hold no data, and the zero-window query returns the pool's spot reading, its share supply standing in for the konstant of balances (source: §7.4). The BitShares port exposes this through read-only API calls returning pool state and window readings (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, §10).

## 10. Conformance and limits

Everything above is computed in consensus, deterministically, under the standing rules. Arithmetically, every multiplication path carries explicit overflow checks in the reference and gets 128-bit math here (source: `contracts/reciprocity/docs/ports/bitshares/engineering.md`, §3.2). The weights cap at a sum of 100 after normalization by the greatest common divisor of the weight set, which bounds the pairwise arithmetic of the fairness curve (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §14.1 for the cap; `contracts/reciprocity/docs/ports/bitshares/engineering.md`, §9 on why the normalization bound exists).

State is bounded by construction: a pool holds at most 8 asset ids, 8 balance and 8 genesis entries, per-token fee accumulators, and 13 fixed oracle fields; the open growth surface is the pool count, which the creation fee and a committee-set ceiling bound in the design spec, because unbounded object creation is the problem the settlement proposal's own review surfaced first (BSIP-3, Risks, the state-bound finding).

What does not ship matters as much as what does. This proposal creates no blind form of anything: the blind-to-blind pool exchange in BSIP-2 is proved for the constant-product kernel by an explicit value pin, and extending that proof to the geometric-mean invariant is new cryptography this design does not attempt, so the N-asset pools remain a transparent-aggregate path permanently (BSIP-2, Specification, blind pool swaps and permanent non-goals). Pools do not mint smartcoins and do not gate debt; the pool-issued index money class parked among the settled decisions stays parked (AGENTS.md, section 4, settled decision 8). No foreign-chain anything lives in a pool: a pool holds what BitShares holds, and cross-chain settlement stays with the offer book of BSIP-3, designed honestly about what consensus cannot know.

# Rationale

The invariant comes first. For two assets the geometric mean of ratios is the constant product curve, so the proposal generalizes instead of replacing: an existing style of pool becomes the special case where the pool holds two assets. For three or more, the geometric mean prices every pair inside the pool from one shared state, and the arbitrage equilibrium settles each asset's value share at its weight, which the reference states and the weighted extension documents as Balancer-equivalent up to a constant that cancels in pricing (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §14.1; `contracts/reciprocity/docs/ports/bitshares/bsip.md`, Rationale). Running the same exposure as three pairwise pools instead triples the state to keep synchronized and triples the fee events to arbitrage through.

The fairness curve prices the harm a swap does to the pool rather than the size of the swap. A fixed rate taxes both and invites two attacks at once: cheap churn on balanced trades and underpriced imbalanced ones. The quadratic curve separates them, and its floor keeps the regulators alive. Pricing fees by measured imbalance is also what makes the pool defensible against sandwich loops: the reference's red-team analysis finds the attack unprofitable because the governor cuts fees exactly when volatility rises, and the attacker pays on both legs of the sandwich (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, §1.1).

The absence of a fee cap is a choice with a reason and a boundary. The reference states the dual regulator provides continuous steering without an artificial ceiling (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §3.1); a hard cap would create a cliff to cluster against and a governance knob to argue over, while the drip bound in section 4 caps each fee at the amount received, the only cap that survives as fairness rather than as politics. The program's caps-over-uncapped rule is unchanged: it governs mint and burn of smartcoins, and nothing here mints anything but share tokens whose supply tracks actual paid-in balances.

The two-channel regulator beats a single channel because level and volatility are different problems: an average spread above target is a pricing problem, spread spikes are a volume problem, and the reference assigns one channel to each (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §6.1). Genesis anchors were chosen over re-anchoring: the reference rejects re-anchoring genesis on every operation because the approximation error would accumulate in a path-dependent way, and the double-Padé's exactness under proportional growth shows re-anchoring is never needed (source: §1.3).

Rejected outright, with reasons. Migrating or upgrading the existing two-asset pools is rejected because legacy stays untouched permanently and a parallel class costs nothing to defend (AGENTS.md, section 4, settled decisions 1 and 5). Using pool prices as smartcoin collateral is rejected because debt needs bonded oracles, and a pool oracle is unbonded by definition. Per-operation genesis updates are rejected for the path-dependence reason above. Weight inputs from external feeds are rejected because every external number here would be an unbonded feed wearing a costume; weights are set once by the pool's creator, inside validation.

# Risks and failure modes

The pool oracle can be bent across blocks. A flash loan cannot move it, since the time delta inside one block is zero, but a patient attacker holding an imbalanced position across blocks moves the average while paying for the imbalance, and any consumer pricing debt or derivatives off pool TVWAP inherits that risk unless their own gates price it. This document caps its own ambition: the oracle is a published aggregate, never the authorization for issuance, which is reserved to the bonded oracle proposal (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, §1.3; AGENTS.md, section 4, settled decision 3).

The fee is uncapped and thin pools are where that bites. A small pool facing a large trade computes a large fairness spread, and a trader who misquotes receives a large fee; the amount-received bound in section 4 keeps the fee below the receipt, and the share floor of section 6 keeps the trader whole or rejects the trade, but the experience is still expensive. Pools that want depth should not launch from dust, and a creation minimum, proposed and owned by the design spec, keeps the first deposits meaningful.

The approximation error is priced, not zeroed. At a ten-times single-token deviation the share accounting could err about 13 percent of the konstant, and the reference's bound on reaching that state is at least 30 percent of pool value paid in fees on both access and exit (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, §1.2). The residual risk is a market where the deviation is cheap to hold, a deeply illiquid asset nobody arbitrages; the defense is creation-time validation refusing duplicates and refusing weights computed to game the sum cap, and it is stated rather than hidden (source: `contracts/reciprocity/docs/whitepaper/spec.md`, §14.1).

Committee and issuer royalty rates are a lever, controlled and small, with the default at zero and the ceiling at 500 basis points each; a capture that raises them taxes every swap in the class, visibly, and the rate's full history is readable from the fee-parameter records. The counter-lever is that LPs can migrate to a new pool created by anyone, since creation is permissionless, and a royalty rise that exceeds the competitiveness of the asset's obvious pairs splits the pool rather than enriching the takers.

Unbounded pool creation is bounded by the design spec, not by this text: it must file a creation ceiling and per-account pool caps, because an object type whose instances grow with every spam transaction taxes every node forever, the same unbounded-state lesson the settlement proposal recorded (BSIP-3, Risks). The kill criterion is the standing one: no design spec, no vector suite, no wire pinning, no build, and until every one exists and passes, the proposal remains a design document.

# Backwards compatibility

Nothing migrates. The legacy constant-product pools, their operations, their objects, and their fee accounting are untouched, and an account that never touches an N-asset pool sees no change to any transaction it signs. New operations append behind one hardfork flag, new object types append to the id registry, and no existing field moves or reinterprets, so a wallet that does not know the class still prices and trades legacy venues identically. The gates are the committee's to set or not set, and in builds where the flag has not passed, the new operations declare but refuse at apply, the established fail-closed pattern across the program's proposals (BSIP-2, Backwards compatibility). Existing UIAs, including gateway tokens, are eligible as pool assets unchanged, since a pool treats a UIA as an ordinary balance to custody and return.

# Test cases

Each row names behavior to pin before any build, derived from what this document's specification already states and cites, and the last rows are the properties the reference's own adversarial analyses compute. The suite does not exist, and this section claims no recorded result; it claims what any implementation must refuse or match. Every row belongs to the design spec's assertion suite and to the wire-format pinning the Reference implementation section files.

- Invariant homogeneity: a deposit that doubles every balance against its genesis balance doubles K, so new capital prices exactly (source: Specification, section 2; `contracts/reciprocity/docs/whitepaper/spec.md`, §1.2).
- Fee bounds: an output of three units or fewer is taken entirely as the fee, any larger fee stays under the amount received, and the drip floor never strips a pool three units at a time (source: Specification, section 4).
- Split underflow: a fee too small for exact proportions still pays each named recipient at least one unit and reinvests the LP remainder, so nothing underflows into silence (source: Specification, section 5).
- Reciprocity refusal: a swap whose settlement computes negative rejects outright and burns none of the trader's shares (source: Specification, section 6).
- Slippage floor: a trade that would settle below the share floor quoted at submission rejects instead of delivering less (source: Specification, section 6).
- Full exit: one account holding every share sweeps the pool to zero dust, the LP and oracle accumulators reset before their timestamps restore, and the once-initialized flag survives so creation cannot replay into the emptied pool (source: Specification, section 8).
- Weight gate: a weight set normalizes by its greatest common divisor to a sum of 100, and a set built to game the cap refuses at creation (source: Specification, section 10).
- Oracle windows: inside one block the accumulators move zero, a query on a window holding no data reverts, and the zero-window query returns the pool's spot reading (source: Specification, section 9).
- Sandwich property: a sandwich loop nets a loss once the governor prices the volatility it creates (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, §1.1).
- Approximation-error property: the share accounting error stays inside the bound quoted in Risks, and reaching that deviation costs more in fees than the error is worth (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, §1.2).
- Flash property: a sequence inside one block moves no oracle accumulator and no price (source: `contracts/reciprocity/docs/whitepaper/exploits.md`, §1.3).
- Wire pinning: operation tags and object ids append and hold stable across builds with and without the hardfork flag (source: Backwards compatibility).

# Reference implementation

The proposal is a distillation of protocol sources, not a port that has landed in this program's core branches. In the reference tree:

- `contracts/reciprocity/docs/whitepaper/spec.md` carries the mathematical specification quoted throughout: invariant §1, aequitas §2, fees §3, split and reciprocity §4, governor §6, oracle §7, operations §8, accounting invariants §9, constants §10, extensions §14, admin model §16.
- `contracts/reciprocity/docs/whitepaper/exploits.md` carries the adversarial analyses cited in Motivation and Risks: sandwich feasibility §1.1, approximation-error bounds §1.2, flash-loan resistance §1.3.
- `contracts/reciprocity/docs/whitepaper/theory.md` carries the error budget and convergence proofs the spec cross-references at §1.4, §2.3, and §6.1.
- `contracts/reciprocity/docs/ports/bitshares/bsip.md` is the owner's earlier BitShares framing of the protocol, mined for motivation and the eight-asset bound.
- `contracts/reciprocity/docs/ports/bitshares/engineering.md` carries the BitShares-specific mapping this proposal adopts for royalties, share assets, arithmetic, and API: §1.3, §1.4, §3, §5.1, §8, §9, §10.
- `contracts/reciprocity/agents/ref.md` condenses the invariants, fee split, governor, and Padé path used throughout.
- `contracts/reciprocity/contracts/` holds the per-chain ports, with `contracts/reciprocity/contracts/bitshares/` the Graphene port inside the reference tree, and `contracts/reciprocity/demonstrations/python/integer/` the canonical integer reference all ports must match.

What still needs writing before any build: the design spec in `docs/superpowers/specs/` that fixes operation tags, object ids, field order, the royalty defaults, the creation ceiling, and the pool-cap rules; the assertion suite adapted from the reference's property and fuzz invariants; the wire-format pinning tests; and the full-binary evidence gates every other proposal in this program has already run. None of that exists today, and this document claims none of it.

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
