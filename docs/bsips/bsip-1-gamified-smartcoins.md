    BSIP: (number assigned at filing; house label BSIP-1)
    Title: Parallel bonded smartcoin class: bonded oracles, bonded vaults, and tranched issuance
    Authors: litepresence (HONEST)
    Status: Draft
    Type: Protocol
    Created: 2026-10-25

# Abstract

BitShares trades like a modern exchange and trusts like an old one. Its price feeds carry no bond, money locked so lying costs something, and its legacy custodial tokens promise backing no rule enforces. This proposal adds a parallel asset class in which every claim states its capital. An oracle, the on-chain record of published prices, is run by bonded producers paid only for staying online. Vaults custody the backing behind locked bonds. New dollar-tracking smartcoins, tokens built to follow an external price, arrive as two tranches paid in a fixed order, one first, one absorbing losses up to a cap. Coverage, the bonds behind a promise divided by the promise, is public and gates growth. Legacy assets stay untouched, permanently. Nothing turns on until a pilot under $20,000 proves coverage for 90 days and a vote passes. The pitch: every dollar on this exchange states its backing and its bonds.

# Motivation

BitShares' trade machinery has never been the weak part. Deterministic order books, a margin engine, constant-product swap pools, and native hash-time-locked contracts compare well with what most chains run. The weak part is trust without capital behind it. Every figure below was measured on 2026-09-26 against a public API node (`wss://api.bts.mobi/wss`) with the snapshot script `tooling/bts_chain_snapshot.py`, at head block ~114706236.

- Order books hold about $15 of depth, so price discovery moved to swap pools, and the pools disagree by about 86 percent: BTS/USD clears 59.7 BTS per bitUSD, BTS/CNY clears 15.6, implying 3.8 CNY per USD against 7.1 fair.
- The chain carries 6,608 assets, 438 of them market-issued, and the legacy books total roughly $6 to $8 million against a BTS market capitalization of about $2.9 million.
- The margin engine consumes each median in the block window it was written, with no delay, no speed limit, and no brake scoped to one market. Feed producers post no bond, so a colluding majority lies at zero cost.
- Custodial tokens rest on reputation plus revocable permissions, so unbacked printing, lost keys, and silent insolvency surface only at redemption runs. A perfect feed would not prove one custodial BTC token redeems for one BTC.

The gap is not hypothetical. HONEST began as a protest against a witness cartel that fed false prices to shield marginers while breaking forced conversion for holders, and the documented fake-feed episode is why every rule here assumes vote capture is possible and prices it (`docs/superpowers/specs/2026-09-29-bsip-h1-rev2-oracle-base.md`, section "Why plumbing first").

Fee tweaks, including the BSIP-0042 era, move these numbers cosmetically. The missing input is capital at risk: bonds behind custody and behind prices, a delay before a price may be consumed, and issuance whose variance a junior tranche is paid to absorb.

This proposal rests on one chain of reasoning. Bonded, liveness-paid price oracles, not unbonded voted feeds, can authorize a new class of market-pegged smartcoins redeemable through atomic cross-chain swap, ACCS, a swap across two chains that completes on both sides or neither. The settlement machinery ships as the separate BSIP-3. This proposal ships the oracles, the vaults, and the issuance layer, and lets coverage, not a committee, decide whether the market migrates.

# What changes for you

- If you hold `bit*`, `HONEST.*`, or legacy custodial UIAs, nothing about your balances, fees, or settlement rules changes, and no rule here can move, cap, or re-price them later either.
- If you trade, new bonded pairs sit beside the legacy books with prices that arrive one hour late and move at most five percent per interval, harder to push around.
- If you witness, you keep producing blocks as today and gain no new duty unless you choose to run a bonded oracle slot, where posting a bond and facing disputes is the price of publishing prices.
- If you issue a custodial token, you can keep it as is or move to bonded custody, where a locked bond, keys shared across several parties, published deposit IDs, and on-screen coverage replace reputation as what your users check.
- If you develop, you get 26 new operations at wire numbers 78 through 103, appended objects, and new API calls, while every existing operation, object, and fee keeps today's wire format.

Nothing turns on automatically. Activation requires the pilot to prove coverage for 90 days, the remaining specialist and replay gates to clear, committee-set hardfork dates, and a witness-majority adoption at those dates, a hardfork being an upgrade the whole chain switches to at a scheduled block. None of those gates is authorized today.

# Specifications

Consensus law throughout: no floats, no `unordered_*` containers, head-block time only; operations and extension fields appended, never inserted; every new operation and field hardfork-gated in its evaluator and in the proposal hardfork visitor, so a multi-operation proposal cannot smuggle it past the date; bounded state; select-only aggregation; 128-bit comparison with declared rounding.

`vault_coverage = vault_bonds / outstanding`
`oracle_coverage = oracle_bonds / secured_debt`

A bare `coverage` means bonded divided by secured, where 1.0 or higher is healthy. Every constant below is a proposal the activation vote owns, except the four marked built in section 7.

## 0. Three independent proposals

The community votes on three protocols and any combination of them (`docs/superpowers/specs/2026-10-24-three-track-split-design.md`, sections 1 and 2). BSIP-3 is the foreign chain transaction oracle plus ACCS orderbook settlement, BSIP-2 is true stealth transfers plus blind pool swaps, and BSIP-1, this proposal, is the price oracle (Phase 1) plus the smartcoins built on it (Phase 2). Any one, any two, or all three may pass, and the build the community runs contains exactly the approved set, so a transaction means the same thing in every configuration.

Operation tags are fixed across configurations: 0 through 77 are the develop baseline and stay untouched, 78 through 82 are Phase 1, 83 through 103 are Phase 2, for 26 new operations. This proposal shares a common oracle and guard substrate with BSIP-3, byte-identical in every build that carries it.

Legacy assets stay untouched, permanently: `bit*`, `HONEST.*`, and legacy custodial UIAs keep their semantics, fees, and settlement rules, with no migration path and none planned.

## 1. Asset model and coexistence

Each hybrid pair is two tokens sharing one reserve: a senior stable such as `FX.USD`, redeemable at the bonded price and first in line on exits, and a junior reserve coin such as `FXR`, which absorbs volatility, earns protocol revenue, dilutes first up to an `rmax` cap, and may go to zero. Both are ordinary user-issued assets created with the standard asset operations, so no new asset-type flags exist. `vault_register` binds a vault to a pair, requires the registrar to be the pair's issuer, and refuses market-issued or Phase 1-priced pairs at registration, a consensus reject. The vault's oracle binding pins on the pair's first freeze.

Raising a security floor takes a 7-day timelock with new debt frozen at announcement, executed as committee procedure rather than a consensus opcode; lowering one requires global settlement and recreation. A 10 percent holder coalition, measured at a snapshot so flash-borrowed grief fails, can freeze and exit at last-good-minus-haircut without committee approval, a mechanism specified and unbuilt that no safety depends on today. Operators and producers must not hold paired `FXR` above dust, disclosure-enforced at pilot.

The `debt_cap_A` growth throttle was designed and then retired in favor of pure coexistence: no legacy caps, no migration, no repricing path. Blind positions and transfers are permitted while protocol totals (supply, reserve, queue depth) stay permanently public, and disputes are transparent operations with attributable bonds, so blind-funded challenges are rejected.

## 2. Phase 1: bondable oracle base and spike guards

Five operations carry tags 78 through 82: `oracle_create`, `oracle_update`, `oracle_delete`, `oracle_publish`, and `reference_asset_create`. Reference assets register an off-chain measuring stick with symbol and precision, for example USD with 4 decimals, and `reference_mapping{reference, asset_amount, reference_amount}` appends last into `bitasset_options::ext` along with `price_oracle_binding`, so old software reads old fields at old positions. `hardfork.d/ORACLE.hf` carries the activation date, which the committee schedule sets.

Each publish carries up to four series with per-series relation checks (`bid <= ask`, `down <= up`); aggregation is same-producer-set with per-producer outlier exclusion and is select-only, meaning sort, exclude, pick the median, with no floating point. State is bounded: 32 producers, 64 history entries, 64 subscribers, 16 pending guard slots (`oracle.hpp`). Bindings are validated at bind and at every read, so a drifted mapping yields no price, holds last-good, and blocks new debt rather than following the change, and one pair's drift never poisons another.

Track A guards are per-oracle opt-in flags that default off, so "Phase 1 pair" alone means nothing and wallets must show the three flags. When enabled, a publish moving more than 5 percent in one interval (`GRAPHENE_ORACLE_GUARD_MAX_MOVE_PPM=50000`) holds last-good and flags the breach, a median computed at block N becomes consumable only at N plus the delay (chain parameter `oracle_guard_delay_sec`, default 3600 seconds, maximum 86400), and a freeze on one oracle halts margin for that oracle alone and never liquidates, with unfreeze by vote. Liveness and deviation history are on-chain counters for observability only; reputation is never weight, and coverage reads 0 honestly until Phase 2 bonds exist.

Every explicit price binding carries a consensus expiry height that must be strictly future at bind and is immutable while bound. Past expiry the binding yields no price and blocks new debt, with no extension path, so running uncovered past the published window is impossible by rule rather than promise. Six API calls ship: `get_oracles`, `get_oracle_by_name`, `list_oracles`, `get_oracles_by_owner`, `get_reference_assets`, `get_reference_asset_by_symbol`. History, reputation, and delayed-price endpoints do not exist in this phase, and history is readable from the oracle object. Legacy feed paths execute unmodified code, pinned by a regression test. Oracle detail sits in `docs/superpowers/specs/2026-09-29-bsip-h1-rev2-oracle-base.md`, sections 1 to 6.

## 3. Phase 2: bonds, disputes, and vaults

Twenty-one operations carry tags 83 through 103, covering vault lifecycle, bonding, challenges, disputes, freeze, wind-down, seasoning, and issuance (`operations.hpp`). Bonds lock in vesting balances with a 7-day unbond delay (`GRAPHENE_H2_BOND_UNBOND_DELAY_SEC=604800`), each publication pushes the unlock out, and every bond denominates in USD terms at lock rather than raw units: `GRAPHENE_H2_USD_BASIS_RAW_PER_CENT=1000000` at the pilot basis of $0.001 per BTS, enforced at all four lock sites as `raw >= usd_cents x BASIS` in 128-bit comparison. Weights are owner-set or bond-proportional at genesis, never reputation-earned.

The dispute rulebook is built once and shared. A challenger posts $837.50 in USD terms at lock (`GRAPHENE_H2_CHALLENGE_BOND_FLOOR_USD_CENTS=83750`), the fee doubles each round up to three rounds, the vote window is 48 hours (`GRAPHENE_H2_VOTE_WINDOW_SEC=172800`), and a claim clears on two-thirds (`GRAPHENE_H2_SUPERMAJORITY_BPS=6667`) with the published quorum half (`GRAPHENE_H2_QUORUM_BPS=5000`). Loser pays, and nothing is slashed on deviation alone: 80 percent of forfeits burn and 15 percent feed the reward pool (`GRAPHENE_H2_FORFEIT_BURN_BPS`, `GRAPHENE_H2_FORFEIT_POOL_BPS`), with the finder fee capped at $83.75. A 24-hour cooldown, three concurrent claims per pair, and fast-reject with full forfeit bound spam, and challenges never halt price consumption unless coverage or the tripwire fires. A bounded authority-graph check bans self-dealing pairings, formal ratification still owed. Every deadline fails closed: abstain or missed means frozen with guilty history, never acquitted.

Rewards pay liveness only, pro-rata on live weight, never agreement. Funding as built is forfeit-only, so honest producer income from forfeits is close to zero at pilot bond sizes until the committee prices a consumer-fee route, a 50/30/20 oracle/vault/reserve split with a 70 percent joint cap left deliberately unbuilt. Review trigger and owner: the consumer-fee route and the 50/30/20 split are priced by committee vote, gated on pilot coverage evidence and a review of producer cost at pilot bond sizes; until then honest producer income stays near zero and no yield is promised. This document promises no yield.

Vaults custody backing assets in threshold sets, native multi-signature control across several parties, with mint authority at the vault rather than an issuer key. Genesis `secure_ratio` is 200 percent (`vault_ops.hpp`) and mint is refused below it. A mint needs vault signature plus coverage plus a live oracle at once, so a breach fails mint closed while redemptions continue pro-rata per vault, and one pair's failure never touches another's book or BTS. Coverage below 150 percent (`GRAPHENE_H2_MAINTENANCE_TRIGGER_BPS=15000`) halts the door, opens a fixed 36-hour top-up (`GRAPHENE_H2_TOP_UP_WINDOW_SEC=129600`), then triggers automatic wind-down and seizure. Reimbursement comes from the segregated bond at last-good-minus-haircut, 15 percent (`GRAPHENE_H2_WINDDOWN_HAIRCUT_BPS=1500`), and the filed exit math is `loss = 1 - min(0.85, coverage)`, so exits pay pro-rata coverage below 85 percent cover. Wind-down pays seniors first under the `rmax` cap (`GRAPHENE_H2_WINDDOWN_RMAX_JUNIOR_X=2`) and is exit-only. `pair_freeze` halts door and debt atomically, because stopping one while the other drains is the classic partial-halt failure, and `vault_resolve` runs arbitration in consensus, since the committee cannot move funds by procedure alone.

The price leg layers the same discipline onto Phase 1: one-hour delay, rate limit, a fallback series, and a halt when both primary and fallback are stale, with a recovery analog at 150 percent suspending new debt on live prices only. The tripwire watches deviation past 5 percent (`GRAPHENE_H2E_TRIPWIRE_DEVIATION_PPM=50000`) with a depth qualifier over two windows, halts door first and debt second, and reopens after two clean windows. Every Phase 2 price read needs a non-empty bonded live set, an unslashed bond, a live submission, and a current producer entry, so an unbonded oracle yields no price.

Activation runs in three consensus gates: `HARDFORK_H2_BONDS_TIME` for primitives, `HARDFORK_H2_GATEWAY_TIME` for the vault leg, `HARDFORK_H2_PRICE_TIME` for bonding, resolution, freeze, and issuance. Custody leg before price is asserted first in every helper, so an inverted schedule refuses all Phase 2 paths. In-tree dates are placeholders the committee replaces. Bond, dispute, and vault detail sits in `docs/superpowers/specs/2026-09-29-bsip-h2-bonds-vaults-issuance.md`, sections 1 to 5.

## 4. Issuance: the door and the soft machinery

The door runs on tags 99 and 100, `hybrid_mint` and `hybrid_redeem`. Mint enforces a 130 percent floor (`GRAPHENE_H2_ISSUE_RMIN_BPS=13000`) against exact integer math on bonded reads only, plus a per-block door cap (`GRAPHENE_H2_ISSUE_DOOR_CAP_BTS`, a $2,000 proposal at the pilot basis) so oracle profit-from-corruption stays bounded. Redeem pays into 24-hour native vesting escrow, never booked as capitalization, and no mint answers its own flow.

Absorption before liquidation is the point of the design. A 7-bucket accounting partition (`GRAPHENE_H2_ISSUE_N_BANDS=7`, `GRAPHENE_H2_ISSUE_BAND_TICK_BPS=250`) spends center-out in a fixed order, so spending order is consensus while no price edge is ever compared in consensus. The offset pool object holds up to 64 claimants on 7-day vesting, is non-transferable, and absorbs shortfalls below 100 percent. Batch-settle exits on tag 103 carry a 2 percent exit fee that stays in the reserve, suspend junior minting past a queue threshold, and settle at a single clearing haircut, while unvested door conversions revert to the stable leg so settled trades are never repriced.

Sizing is capped by rule: `cap = min($2,000 fixed ceiling, headroom x k1, depth x k2)` with `k1 = 10000` and `k2 = 5000`, over a two-window depth-qualified multi-pool median, exposed read-only through `get_h2_qualified_depth`. The seasoning clock on tag 98 runs 90 days (`GRAPHENE_H2_SEASONING_WINDOW_SEC=7776000`) on head-block time from Phase 2 state only, so a Phase 1 freeze cannot pause or extend it, and any breach restarts the count. Issuance detail sits in the same spec, sections 6 and 7.

## 5. Staging, halts, and constants

Staging runs in order: the Phase 1 acceptance gate, the primitives gate, custody activation, BTC vault tokens in bonded custody at coverage 2.0 and 1.0, 90 days of seasoning, USD issuance debt-capped by the cap function, then scale with proven liquidity. The pilot rule sits inside that order: no USD-scale issuance until a pilot on a book of $20,000 or less proves coverage for 90 days (`docs/superpowers/specs/2026-10-17-pilot-decision.md`, sections 1 and 3). If the pilot fails, USD issuance does not proceed.

Halts are automatic, each has a re-open rule, none is appeal-proof, and none can censor without a separately authorized operation: the coverage halt (36 hours with post-hoc appeal), the challenge freeze (reopens on resolution plus coverage), the tripwire ladder (two clean windows), and holder exit at the haircut, open throughout. Every gate treats a missed deadline the same way, as freeze rather than acquit: the maintenance top-up lapses into wind-down and seizure, and the unbond delay refuses early withdrawal.

Pre-deposit disclosure is mandatory before any mint or deposit: the junior-backed dollar is experimental and never fiat-guaranteed, exits may price below face under the haircut, `FXR` may go to zero and pays variable income rather than promised yield, vault recovery can be partial or zero, prices arrive late and are disputable, and any 10 percent coalition, the committee, or the tripwire can wind the pair down. Deposit only what you can afford to lose.

## 6. Escrow basis

The bond floor exists so the USD face locked at a claim cannot be inflated by a thin pool price on the day of lock. Only the fixed pilot basis enters bond math, so a collateral move changes coverage but never the size of the bond behind a claim.

## 7. Parameter derivation appendix

| Constant | Filed value | Code symbol | Status |
|---|---|---|---|
| Shared fee-diversion cap | 70 percent, shared denominator | `GRAPHENE_H2E_FEE_DIVERSION_CAP_BPS=7000` | vote |
| Vault minimum bond | $13,400 | `GRAPHENE_H2_VAULT_MIN_BOND_USD_CENTS=1340000` | vote |
| Producer minimum bond | $10,050 | `GRAPHENE_H2_ORACLE_MIN_BOND_USD_CENTS=1005000` | vote |
| Griefing bond and finder cap | $83.75 | `GRAPHENE_H2_FINDER_CAP_USD_CENTS=8375` | vote |
| Wind-down haircut | 15 percent | `GRAPHENE_H2_WINDDOWN_HAIRCUT_BPS=1500` | vote, 10 to 20 percent band |
| Top-up window | 36 hours | `GRAPHENE_H2_TOP_UP_WINDOW_SEC=129600` | built |
| Unbond delay | 7 days | `GRAPHENE_H2_BOND_UNBOND_DELAY_SEC=604800` | built |
| Vote window | 48 hours | `GRAPHENE_H2_VOTE_WINDOW_SEC=172800` | built |
| Oracle rate limit | 5 percent per interval | `GRAPHENE_ORACLE_GUARD_MAX_MOVE_PPM=50000` | built |

Constants whose value and code symbol already appear in sections 2 to 5 are not repeated here.

# Rationale

Where the 30-plus papers on the `/workspace/papers` shelf converge, this design follows them, and it states the ceiling where BitShares differs.

Delay, freeze, and shutdown come before settlement, following Maker's Oracle Security Module and Reflexer's auto-shutdown, so no price is consumed in the window it was written. The dispute path, not the happy path, carries the cost, following UMA's DVM, Augur's escalation bonds, and Tellor's rounds. No single push feed is trusted, following Pyth's confidence and Liquity's fallback, so both are wired from day one. Over-collateralization pairs with a pooled absorber, following Liquity's stability pool and Djed's reserve gates. Security scales with value secured, in the UMA and Chainlink 2.0 framing where cost of corruption must exceed profit from corruption, which is why debt growth stops below coverage 1.0 and why the per-block conversion cap is load-bearing.

Rejected with reasons. Uncapped reflexive mint and burn, because Terra, Haven, and PegNet broke the peg. Unbonded voted feeds as an end state, because collusion among them is free. Rewards paid for agreement with the median, because they teach copy-farming. Blind supply totals, because hiding totals removes the public audit. Automatic slash on deviation, because it hands a confiscation tool to any majority. Reputation-earned weights, because a patient attacker farms them. Per-account caps, because Sybil splits defeat them. Mandated blindness for market, vault, and oracle paths, because it would kill supply audit, listings, and DEX transparency. Verifier-model claims, because Graphene has no virtual machine, so no in-consensus SPV relay and no programmatic slashing are possible. BitShares gets bonds, transparency, and vote arbitration with fail-closed rules, the attestor model, and nothing beyond it.

Two ordering choices earn their place. Custody comes before price because vault-backed tokens carry no repricing risk to existing debt and need only caps, challenges, and wind-down, so custody is fixed first and the fight over bonding elected feeders comes second. A parallel class comes before touching legacy smartcoins because coexistence costs nothing to defend.

The author's DFX protocol draft is a pointer, not gospel. It supplied decaying federated reputation at genesis, advancing bond periods, fee-funded pool deepening, a time-weighted pool oracle as tripwire input, and HTLC griefing deposits, and it was rejected on uncapped mint and burn, agreement-paid rewards, and blind supply.

# Risks and failure modes

- If the new class fails, your existing assets are not touched. Losses come from the new class's own bonds and reserves, and no rule here can move, cap, or re-price `bit*`, `HONEST.*`, or legacy custodial UIAs.
- Collusion can win votes. Upholding a claim needs two-thirds with the published quorum, an inconclusive vote freezes the pair instead of clearing it, and visibility, freezes, and the threat of a community software split limit the damage. Nothing here removes the risk, and holders of the new class bear it.
- BTS-correlated collateral crashes together until the collateral mix widens beyond BTS, planned after the pilot. The filed exit math is harsh in a deep drawdown: on a $20,000 book at 200 percent bonds, a 70 percent crash leaves coverage near 0.60, so seniors get about 60 cents on the dollar and the junior gets nothing.
- Silence is free, so a challenger drought is possible. If nobody posts the bond to challenge a bad price, it stands until coverage breaks or the tripwire halts new debt, and apathy is the residual no rule removes.
- Custodial insolvency is bounded by bonded custody and automatic wind-down, but recovery is partial or zero by design, so do not deposit more than you can afford to lose.
- The junior token can go to zero. It absorbs losses first up to its cap and is never a promise, and no yield may be marketed against it.
- Phase 1 guards alone do not stop a colluding feed majority or a single custodian, the accepted risk published with the oracle base (`2026-09-29-bsip-h1-rev2-oracle-base.md`, section 7). Consensus expiry bounds how long an uncovered pair may run, and bonds arrive with Phase 2.
- Pilot scale can fail, and it is meant to: no USD-scale issuance until a book of $20,000 or less proves coverage for 90 days. The sizing risk is published in the pilot decision (`2026-10-17-pilot-decision.md`, section 2): at the minimum bond shape the value an attacker could seize exceeds the $20,000 face, so the owner must pick a sizing option and publish the at-risk cap before launch.
- Realized haircuts can exceed the filed 15 percent when exits crowd thin books, and arbitration stays political: every vote is on-chain, no freeze is appeal-proof, and a missed deadline freezes rather than acquits.
- Activation is not authorized. The protocol is implemented and independently audited, and the open gates fall into four categories: pilot evidence, specialist audits, activation votes, and committee hardfork dates. Owners, pass predicates, status, and kill criteria live in `build/ACTIVATION-CHECKLIST.md`, including a funding stop if coverage stays ungated for 90 days and no USD issuance on a missed gate.

# Backwards compatibility

- Nothing breaks for existing holders, traders, or witnesses: `bit*`, `HONEST.*`, and legacy custodial UIAs keep their semantics, fees, and settlement rules, with no migration path in either direction and none planned.
- Every new operation, object, and extension field is appended, never inserted, and hardfork-gated in its evaluator and in the proposal hardfork visitor.
- The one legacy touch is disclosed and gated: past the bonds gate, new hash-time-locked contracts that use the sha1 hash are refused while contracts created before the fork are untouched and in-flight locks drain through expiry.
- Wallets and tools that do not know the new operations keep working on legacy paths, and new operations apply only behind their hardfork dates in the order BONDS, then CUSTODY, then PRICE.
- Rollback is clean before seizure: a primitives-only rollback returns the chain to its prior state, while past seizure the doctrine is forward-fix plus fresh challenge windows, written down before activation.

# Test cases

Each row names behavior to pin before any build, derived from what this document's specification already states and cites, and the last rows are the invariants the audit rounds and the pilot gate depend on. The suite does not exist, and this section claims no recorded result; it claims what any implementation must refuse or match. Every row belongs to the design spec's assertion suite and to the wire-format pinning the Reference implementation section files.

- Tag stability: every operation tag from 78 to 103 and every object id from `1.23.x` to `1.38.x` holds its number across builds with and without the hardfork flag, so a transaction built against one configuration deserializes identically against any other (source: Specification, section 0; Backwards compatibility).
- Oracle aggregation: a publish with more than 5 percent move in one interval holds last-good and flags the breach, and aggregation is select-only with per-producer outlier exclusion, no floating point (source: Specification, section 2).
- Oracle delay: a median computed at block N becomes consumable only at N plus the delay, and a frozen oracle halts margin for that oracle alone without liquidating (source: Specification, section 2).
- Oracle expiry: a binding whose consensus expiry height is not strictly future at bind is refused, and past expiry the binding yields no price and blocks new debt with no extension path (source: Specification, section 2).
- Bond USD basis: every bond locks at least `usd_cents x BASIS` raw units at all four lock sites, in 128-bit comparison, so a thin pool price on the day of lock cannot inflate the bond (source: Specification, section 3; section 6).
- Challenge bond floor: a challenger posts $837.50 in USD terms at lock, the fee doubles each round up to three rounds, and the vote window is 48 hours with a two-thirds clear (source: Specification, section 3).
- Wind-down haircut: exits pay pro-rata coverage below 85 percent cover, seniors settle first under the rmax cap, and the filed exit math is `loss = 1 - min(0.85, coverage)` (source: Specification, section 3).
- Issuance floor: mint is refused below 130 percent against bonded reads only, with the per-block door cap, and redeem pays into 24-hour native vesting escrow, never booked as capitalization (source: Specification, section 4).
- Band spending: the 7-bucket partition spends center-out in a fixed order, and no price edge is ever compared in consensus (source: Specification, section 4).
- Cap function: `cap = min($2,000, headroom x 10000, depth x 5000)` over the two-window depth-qualified multi-pool median, exposed read-only (source: Specification, section 4).
- Seasoning clock: 90 days on head-block time from Phase 2 state only, so a Phase 1 freeze cannot pause or extend it, and any breach restarts the count (source: Specification, section 4).
- Custody-before-price ordering: custody leg before price is asserted first in every helper, so an inverted schedule refuses all Phase 2 paths (source: Specification, section 3).
- Legacy untouched: no new operation, object, or field moves, caps, or re-prices `bit*`, `HONEST.*`, or legacy custodial UIAs (source: Specification, section 1; Backwards compatibility).
- Wire pinning: operation tags and object ids append and hold stable across builds with and without the hardfork flag (source: Backwards compatibility).

# Reference implementation

Operations run from tag 78 to tag 103 and objects from `1.23.x` to `1.38.x`, both pinned by wire-format tests so a tag never shifts. Where a companion design document disagrees with this one, this document governs: `docs/superpowers/specs/2026-09-29-bsip-h1-rev2-oracle-base.md` for the oracle base, `docs/superpowers/specs/2026-09-29-bsip-h2-bonds-vaults-issuance.md` for bonds, vaults, and issuance, `docs/superpowers/specs/2026-10-24-three-track-split-design.md` for the split and tag map, and `docs/superpowers/specs/2026-10-17-pilot-decision.md` for the pilot gates. The implementation is audited and gated, and activation remains unauthorized until every row of `build/ACTIVATION-CHECKLIST.md` clears.

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
