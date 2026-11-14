<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-1 — Parallel bonded smartcoin class: bonded oracles, bonded vaults, and tranched issuance

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
8. Max issuance and caps
9. Oracle windows
10. Glossary
11. Provenance
12. Test cases

---

## 1. Narrative

BitShares trades like a modern exchange and trusts like an old one. Its price feeds carry no bond, money locked so lying costs something, and its legacy custodial tokens promise backing no rule enforces. This proposal adds a parallel asset class in which every claim states its capital. An oracle, the on-chain record of published prices, is run by bonded producers paid only for staying online. Vaults custody the backing behind locked bonds. New dollar-tracking smartcoins, tokens built to follow an external price, arrive as two tranches paid in a fixed order, one first, one absorbing losses up to a cap. Coverage, the bonds behind a promise divided by the promise, is public and gates growth. Legacy assets stay untouched, permanently. Nothing turns on until a pilot under $20,000 proves coverage for 90 days and a vote passes. The pitch: every dollar on this exchange states its backing and its bonds.

BitShares' trade machinery has never been the weak part. Deterministic order books, a margin engine, constant-product swap pools, and native hash-time-locked contracts compare well with what most chains run. The weak part is trust without capital behind it. Order books hold about fifteen dollars of depth, so price discovery moved to swap pools, and the pools disagree by about 86 percent: BTS per bitUSD clears 59.7, BTS per bitCNY clears 15.6, implying 3.8 CNY per USD against 7.1 fair. The chain carries 6,608 assets, 438 of them market-issued, and the legacy books total roughly six to eight million dollars against a BTS market capitalization of about 2.9 million. The margin engine consumes each median in the block window it was written, with no delay, no speed limit, and no brake scoped to one market. Feed producers post no bond, so a colluding majority lies at zero cost. Custodial tokens rest on reputation plus revocable permissions, so unbacked printing, lost keys, and silent insolvency surface only at redemption runs. A perfect feed would not prove one custodial BTC token redeems for one BTC.

The gap is not hypothetical. HONEST began as a protest against a witness cartel that fed false prices to shield marginers while breaking forced conversion for holders. The missing input is capital at risk: bonds behind custody and behind prices, a delay before a price may be consumed, and issuance whose variance a junior tranche is paid to absorb.

This proposal rests on one chain of reasoning. Bonded, liveness-paid price oracles, not unbonded voted feeds, can authorize a new class of market-pegged smartcoins redeemable through atomic cross-chain swap. The settlement machinery ships as the separate BSIP-3. This proposal ships the oracles, the vaults, and the issuance layer, and lets coverage, not a committee, decide whether the market migrates.

## 2. Specification

Consensus law throughout: no floats, no unordered containers, head-block time only; operations and extension fields appended, never inserted; every new operation and field hardfork-gated in its evaluator and in the proposal hardfork visitor; bounded state; select-only aggregation; 128-bit comparison with declared rounding.

`vault_coverage = vault_bonds / outstanding`
`oracle_coverage = oracle_bonds / secured_debt`

A bare coverage means bonded divided by secured, where 1.0 or higher is healthy.

The community votes on three protocols and any combination of them. BSIP-3 is the foreign chain transaction oracle plus ACCS orderbook settlement, BSIP-2 is true stealth transfers plus blind pool swaps, and BSIP-1, this proposal, is the price oracle (Phase 1) plus the smartcoins built on it (Phase 2). Any one, any two, or all three may pass, and the build the community runs contains exactly the approved set, so a transaction means the same thing in every configuration.

Operation tags are fixed across configurations: 0 through 77 are the develop baseline and stay untouched, 78 through 82 are Phase 1, 83 through 103 are Phase 2, for 26 new operations. This proposal shares a common oracle and guard substrate with BSIP-3, byte-identical in every build that carries it.

Legacy assets stay untouched, permanently: bit*, HONEST.*, and legacy custodial UIAs keep their semantics, fees, and settlement rules, with no migration path and none planned.

Each hybrid pair is two tokens sharing one reserve: a senior stable such as FX.USD, redeemable at the bonded price and first in line on exits, and a junior reserve coin such as FXR, which absorbs volatility, earns protocol revenue, dilutes first up to an rmax cap, and may go to zero. Both are ordinary user-issued assets created with the standard asset operations, so no new asset-type flags exist. vault_register binds a vault to a pair, requires the registrar to be the pair's issuer, and refuses market-issued or Phase 1-priced pairs at registration, a consensus reject. The vault's oracle binding pins on the pair's first freeze.

Raising a security floor takes a 7-day timelock with new debt frozen at announcement, executed as committee procedure rather than a consensus opcode; lowering one requires global settlement and recreation. A 10 percent holder coalition, measured at a snapshot so flash-borrowed grief fails, can freeze and exit at last-good-minus-haircut without committee approval. Operators and producers must not hold paired FXR above dust, disclosure-enforced at pilot.

The debt_cap_A growth throttle was designed and then retired in favor of pure coexistence: no legacy caps, no migration, no repricing path. Blind positions and transfers are permitted while protocol totals (supply, reserve, queue depth) stay permanently public, and disputes are transparent operations with attributable bonds, so blind-funded challenges are rejected.

### 2.1 Phase 1: bondable oracle base and spike guards

Five operations carry tags 78 through 82: oracle_create, oracle_update, oracle_delete, oracle_publish, and reference_asset_create. Reference assets register an off-chain measuring stick with symbol and precision, for example USD with 4 decimals, and reference_mapping appends last into bitasset_options::ext along with price_oracle_binding, so old software reads old fields at old positions. The activation date lives in a hardfork guard that the committee schedule sets.

Each publish carries up to four series with per-series relation checks (bid <= ask, down <= up); aggregation is same-producer-set with per-producer outlier exclusion and is select-only, meaning sort, exclude, pick the median, with no floating point. State is bounded: 32 producers, 64 history entries, 64 subscribers, 16 pending guard slots. Bindings are validated at bind and at every read, so a drifted mapping yields no price, holds last-good, and blocks new debt rather than following the change, and one pair's drift never poisons another.

Track A guards are per-oracle opt-in flags that default off, so Phase 1 pair alone means nothing and wallets must show the three flags. When enabled, a publish moving more than 5 percent in one interval holds last-good and flags the breach, a median computed at block N becomes consumable only at N plus the delay (chain parameter oracle_guard_delay_sec, default 3600 seconds, maximum 86400), and a freeze on one oracle halts margin for that oracle alone and never liquidates, with unfreeze by vote. Liveness and deviation history are on-chain counters for observability only; reputation is never weight, and coverage reads 0 honestly until Phase 2 bonds exist.

Every explicit price binding carries a consensus expiry height that must be strictly future at bind and is immutable while bound. Past expiry the binding yields no price and blocks new debt, with no extension path, so running uncovered past the published window is impossible by rule rather than promise. Six API calls ship: get_oracles, get_oracle_by_name, list_oracles, get_oracles_by_owner, get_reference_assets, get_reference_asset_by_symbol. History, reputation, and delayed-price endpoints do not exist in this phase, and history is readable from the oracle object. Legacy feed paths execute unmodified code, pinned by a regression test.

### 2.2 Phase 2: bonds, disputes, and vaults

Twenty-one operations carry tags 83 through 103, covering vault lifecycle, bonding, challenges, disputes, freeze, wind-down, seasoning, and issuance. Bonds lock in vesting balances with a 7-day unbond delay, each publication pushes the unlock out, and every bond denominates in USD terms at lock rather than raw units: BASIS raw per cent at the pilot basis of 0.001 per BTS, enforced at all four lock sites as raw >= usd_cents x BASIS in 128-bit comparison. Weights are owner-set or bond-proportional at genesis, never reputation-earned.

The dispute rulebook is built once and shared. A challenger posts 837.50 USD in terms at lock, the fee doubles each round up to three rounds, the vote window is 48 hours, and a claim clears on two-thirds with the published quorum half. Loser pays, and nothing is slashed on deviation alone: 80 percent of forfeits burn and 15 percent feed the reward pool, with the finder fee capped at 83.75. A 24-hour cooldown, three concurrent claims per pair, and fast-reject with full forfeit bound spam, and challenges never halt price consumption unless coverage or the tripwire fires. A bounded authority-graph check bans self-dealing pairings, formal ratification still owed. Every deadline fails closed: abstain or missed means frozen with guilty history, never acquitted.

Rewards pay liveness only, pro-rata on live weight, never agreement. Funding as built is forfeit-only, so honest producer income from forfeits is close to zero at pilot bond sizes until the committee prices a consumer-fee route, a 50/30/20 oracle/vault/reserve split with a 70 percent joint cap left deliberately unbuilt. This document promises no yield.

Vaults custody backing assets in threshold sets, native multi-signature control across several parties, with mint authority at the vault rather than an issuer key. Genesis secure_ratio is 200 percent and mint is refused below it. A mint needs vault signature plus coverage plus a live oracle at once, so a breach fails mint closed while redemptions continue pro-rata per vault, and one pair's failure never touches another's book or BTS. Coverage below 150 percent halts the door, opens a fixed 36-hour top-up, then triggers automatic wind-down and seizure. Reimbursement comes from the segregated bond at last-good-minus-haircut, 15 percent, and the filed exit math is loss = 1 - min(0.85, coverage), so exits pay pro-rata coverage below 85 percent cover. Wind-down pays seniors first under the rmax cap and is exit-only. pair_freeze halts door and debt atomically, because stopping one while the other drains is the classic partial-halt failure, and vault_resolve runs arbitration in consensus, since the committee cannot move funds by procedure alone.

The price leg layers the same discipline onto Phase 1: one-hour delay, rate limit, a fallback series, and a halt when both primary and fallback are stale, with a recovery analog at 150 percent suspending new debt on live prices only. The tripwire watches deviation past 5 percent with a depth qualifier over two windows, halts door first and debt second, and reopens after two clean windows. Every Phase 2 price read needs a non-empty bonded live set, an unslashed bond, a live submission, and a current producer entry, so an unbonded oracle yields no price.

Activation runs in three consensus gates: bonds primitives, then custody, then price bonding, resolution, freeze, and issuance. Custody leg before price is asserted first in every helper, so an inverted schedule refuses all Phase 2 paths. In-tree dates are placeholders the committee replaces.

### 2.3 Issuance: the door and the soft machinery

The door runs on hybrid_mint and hybrid_redeem. Mint enforces a 130 percent floor against exact integer math on bonded reads only, plus a per-block door cap so oracle profit-from-corruption stays bounded. Redeem pays into 24-hour native vesting escrow, never booked as capitalization, and no mint answers its own flow.

Absorption before liquidation is the point of the design. A 7-bucket accounting partition spends center-out in a fixed order, so spending order is consensus while no price edge is ever compared in consensus. The offset pool object holds up to 64 claimants on 7-day vesting, is non-transferable, and absorbs shortfalls below 100 percent. Batch-settle exits carry a 2 percent exit fee that stays in the reserve, suspend junior minting past a queue threshold, and settle at a single clearing haircut, while unvested door conversions revert to the stable leg so settled trades are never repriced.

Sizing is capped by rule: cap = min(fixed ceiling, headroom x k1, depth x k2) with k1 = 10000 and k2 = 5000, over a two-window depth-qualified multi-pool median, exposed read-only. The seasoning clock runs 90 days on head-block time from Phase 2 state only, so an Phase 1 freeze cannot pause or extend it, and any breach restarts the count.

### 2.4 Staging, halts, and constants

Staging runs in order: the Phase 1 acceptance gate, the primitives gate, custody activation, BTC vault tokens in bonded custody at coverage 2.0 and 1.0, 90 days of seasoning, USD issuance debt-capped by the cap function, then scale with proven liquidity. The pilot rule sits inside that order: no USD-scale issuance until a pilot on a book of 20,000 dollars or less proves coverage for 90 days. If the pilot fails, USD issuance does not proceed.

Halts are automatic, each has a re-open rule, none is appeal-proof, and none can censor without a separately authorized operation: the coverage halt (36 hours with post-hoc appeal), the challenge freeze (reopens on resolution plus coverage), the tripwire ladder (two clean windows), and holder exit at the haircut, open throughout. Every gate treats a missed deadline the same way, as freeze rather than acquit.

Pre-deposit disclosure is mandatory before any mint or deposit: the junior-backed dollar is experimental and never fiat-guaranteed, exits may price below face under the haircut, FXR may go to zero and pays variable income rather than promised yield, vault recovery can be partial or zero, prices arrive late and are disputable, and any 10 percent coalition, the committee, or the tripwire can wind the pair down. Deposit only what you can afford to lose.

## 3. Theory

The invariants the implementation must preserve:

- No floats anywhere in consensus; all math is integer or 128-bit fixed point.
- Every new operation and field is appended, never inserted, and hardfork-gated in its evaluator and in the proposal visitor.
- State is bounded: producer sets, history rings, subscriber lists, and claimant queues have fixed caps.
- Aggregation is select-only: sort, exclude outliers, pick the median. No weighted averages that hide a single bad input.
- Bonds denominate in USD terms at lock, so a collateral move changes coverage but never the size of the bond behind a claim.
- Coverage is bonded divided by secured, published, and gates growth.
- The 7-bucket partition spends in a fixed order; no price edge is ever compared in consensus.
- Wind-down pays seniors first under the rmax cap and is exit-only.
- Seasoning runs on head-block time from Phase 2 state only; no external pause.
- Legacy assets are untouched by construction.

Overflow discipline: 128-bit comparison for bond and coverage math, 256-bit conversion where a product of two 128-bit values is needed, and saturation on price conversion so a wrap becomes a refusal rather than a silent value.

## 4. Exploit analysis

- Colluding feed majority: bonded producers face disputes, a 24-hour cooldown, three concurrent claims per pair, and fast-reject with full forfeit. Challenges never halt price consumption unless coverage or the tripwire fires, so spam is bounded.
- Feed sniping with fresh orders: the minimum time gate and top-N filter keep fresh, small orders from diluting established providers.
- Wash trading: self-trade prevention and the TSP proximity weight make wash volume expensive and low-yield.
- Vault insolvency: bonded custody, automatic wind-down at 150 percent coverage, pro-rata exits, and the 15 percent haircut bound the loss.
- Oracle profit from corruption: the per-block door cap bounds how much can be minted on a corrupted price before the delay and freeze react.
- Griefing on challenges: the doubling fee and finder cap make false challenges expensive and limited.
- Flash-borrowed governance grief: holder coalitions are measured at a snapshot, so borrowed tokens cannot vote.
- Repricing through thin pools: bonds lock at a fixed basis, so a thin pool price cannot inflate a bond.
- Partial halt failure: pair_freeze halts door and debt atomically.

## 5. Gas and cost analysis

Operations are consensus objects, not smart contract calls, so cost is measured in transaction fees and block weight rather than gas. The per-block door cap and the verification budget bound how much new work any block can carry. Blind work on the BSIP-2 carries its own per-block budget of 750000 microseconds; the BSIP-1 does not add verification work of that kind. Pool and order book math reuses shared helpers to avoid duplicated computation.

## 6. Error code registry

Errors are bare integers matching the exceptions module. New BSIP-1 errors occupy a reserved range appended after the existing registry. Each error maps to one refusal condition: bond too small, coverage too low, oracle stale, vault not registered, door capped, seasoning incomplete, challenge bond insufficient, wind-down in progress. Legacy error codes keep their numbers.

## 7. Admin architecture

Admin is split between consensus operations and committee procedure. Consensus owns bonds, disputes, vaults, mint, redeem, freeze, wind-down, and issuance. The committee owns hardfork dates, security floor changes, fee schedule, and consumer-fee routing. No admin key can move funds by procedure alone; vault_resolve runs arbitration in consensus. Founder and promoter roles, where they exist in the pool layer, are locked or transferred by operation, not by key.

## 8. Max issuance and caps

Issuance is capped by three mechanisms: the 130 percent floor on mint, the per-block door cap, and the sizing cap function min(fixed ceiling, headroom x 10000, depth x 5000). The fixed ceiling is a 2000 dollar proposal at pilot basis. The offset pool absorbs shortfalls below 100 percent. No uncapped mint or burn path exists.

## 9. Oracle windows

Phase 1 guards use a one-hour default delay with a maximum of one day. The tripwire uses two windows and reopens after two clean windows. TVWAP-style pool oracles, where used, carry three windows of fifteen minutes, one hour, and twenty-four hours. All windows run on head-block time.

## 10. Glossary

- Coverage: bonded divided by secured.
- Bonded oracle: a price producer who posts a bond and faces disputes.
- Vault: bonded custody of backing assets with mint authority.
- Senior tranche: the stable token, first in line on exits.
- Junior tranche: the reserve coin, absorbs losses up to the cap.
- Offset pool: absorbs shortfalls below 100 percent.
- Seasoning: 90 days of proven coverage before USD issuance.
- Tripwire: automatic halt on deviation past 5 percent.
- Wind-down: automatic exit-only settlement at a clearing haircut.
- Door cap: per-block limit on mint volume.
- rmax: cap on junior dilution in wind-down.

## 11. Provenance

This whitepaper adapts the structure of the Prism and Reciprocity whitepapers in the salvaged reference tree. The concepts are original to the BitShares Track B program. The reference implementations live under docs/reference/contracts-prism and docs/reference/contracts-reciprocity.

## 12. Test cases

- Tag stability across builds with and without the hardfork flag.
- Oracle aggregation select-only with per-producer outlier exclusion.
- Oracle delay enforced at bind and read.
- Oracle expiry strictly future at bind, no extension.
- Bond USD basis enforced at all four lock sites.
- Challenge bond floor, doubling fee, 48-hour window, two-thirds clear.
- Wind-down haircut and pro-rata exits.
- Issuance floor and door cap.
- Band spending center-out in fixed order.
- Cap function over two-window depth.
- Seasoning 90 days on head-block time.
- Custody before price asserted in every helper.
- Legacy assets untouched.

# Copyright

This document is placed in the public domain.
