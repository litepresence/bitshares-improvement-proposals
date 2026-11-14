     BSIP: (number assigned at filing; house label BSIP-8)
     Title: Forex-style PAMM (Percentage Allocation Management Module) accounts
     Authors: litepresence (HONEST) (identity confirmation pending per AGENTS.md section 3)
     Status: Draft
     Type: Protocol
     Created: 2026-10-26

# Abstract

PAMM (Percentage Allocation Management Module) is the dominant account model in retail forex: an investor pools funds under a money manager, the manager trades a single master account, and profits and losses are distributed to investors in proportion to their share of the pool. This proposal adds a first-class PAMM account type to BitShares by reusing two existing primitives: BSIP-72 Tanks and Taps for bounded, auditable fund custody, and BitShares custom active authorities for per-investor withdrawal and reporting rights. It does not touch legacy accounts, does not add a new VM, and does not change core asset or pool semantics. The tank holds all investor deposits plus accrued performance fees; taps encode the manager's trading limit, the performance-fee waterfall, the investor withdrawal right, and the emergency drain, each with its own flow limit and delay; shares are a user-issued asset minted and burned at current NAV; and a rolling 365-entry NAV buffer makes every pool comparable on standardised metrics. NAV is sourced from a BSIP-1 bonded oracle when available, and from manager self-report with a challenge window otherwise. The pitch: managed accounts on chain, with the manager's power to move money written on screen and enforced by consensus, rather than by a custodian's word.

# Motivation

BitShares has no native asset-management account. Traders who want to manage third-party capital today must either take custody, which is counterparty risk with no audit trail, or build on a trustless counterparty settlement protocol, which is heavy and does not fit the retail-forex shape most managers and investors expect. A PAMM built on tanks and taps gives investors a cryptographically enforced, on-screen view of exactly how much the manager can move, when, and to where, with an emergency drain tap requiring a supermajority. Custom active authorities let an investor retain a withdrawal right that the manager cannot revoke, and let the platform enforce reporting and fee waterfalls as authority predicates rather than as code.

The demand is real and the failure mode is known. Managed accounts fail when the manager can move more than the investor agreed, when the investor cannot see what the manager moved, or when the platform cannot stop a runaway manager without shutting the whole venue down. The existing primitives already solve the first two: a tank is a bounded custody object, a tap is a flow limit with a delay, and an emergency tap is a destructor requiring a supermajority. What is missing is the share accounting that turns a tank into a pool, the fee waterfall that turns performance into manager income, and the reporting that turns a pool into a comparable product.

The program's settled rules shape this proposal. No new VM is added, because the chain has no general smart contracts and this proposal does not pretend otherwise. No new consensus math beyond share and NAV arithmetic is introduced, because the chain's existing primitives already carry the heavy lifting. No legacy account, asset, or pool is touched, because the proposal is a new account type that sits beside the existing ones. And no foreign-chain attestation is involved, because a PAMM trades BitShares assets on BitShares markets; the attestor ceiling of the program never applies.

Status, stated plainly. The proposal is a design. No code exists today for this proposal, though its building blocks, tanks and taps and custom active authorities, already exist in the base tree. The open decisions in section 10 are filed and unresolved, and nothing here activates anything.

# What changes for you

- If you hold `bit*`, `HONEST.*`, or gateway assets, nothing about your balances, markets, or fees changes, and no rule here can move, cap, or re-price them later either.
- If you are a manager, you can create a PAMM pool with a tap topology that bounds your trading power, accrue a performance fee only above a high-water mark, and publish a standardised performance record that investors can compare.
- If you are an investor, you can deposit into a pool, hold a share denominated in the pool asset, withdraw subject to a delay and a per-investor limit, and retain a withdrawal right the manager cannot revoke.
- If you are a platform or auditor, you hold the emergency-tap authority at supermajority, and you enforce reporting and large-flow review as authority predicates.
- If you witness, you gain no new duty: the proposal reuses primitives you already run, behind one hardfork date the committee sets or does not set.
- If you develop on BitShares, new operations append at fresh wire tags with new object types, old operations and fields stay where they are, and the tank and tap primitives are reused rather than duplicated.

# Specifications

This section is for developers and reviewers. Every formula and field cites the primitive it comes from, and the full field layout lives in the design spec that must precede any build. Readers outside development can stop after the first paragraph of each subsection.

Consensus law throughout, matching the program's rules: no floats, no `unordered_*` containers, head-block time only; operations and extension fields appended, never inserted; every new operation hardfork-gated in its evaluator and in the proposal hardfork visitor; bounded state, including the 365-entry NAV buffer and the per-investor limits; 128-bit comparison with declared rounding.

## 1. Roles

Three roles participate, and each is a BitShares account, not a new account type.

- Manager: operates the master trading account and the PAMM tank. Creates the pool, sets the tap topology within the bounds of section 2, requests performance-fee tap openings above the high-water mark, tops up the trading tap, and publishes commentary. Cannot lower the minimum tank level floor or shorten withdrawal delays unilaterally.
- Investor: deposits into the PAMM tank, holds a share denominated in the pool asset, and may withdraw subject to tap limits. Retains a withdrawal right via a custom active authority that the manager cannot revoke.
- Platform or auditor: holds the emergency-tap authority at supermajority, enforces reporting and large-flow review as authority predicates, and gates the `pamm_snapshot` operation.

## 2. Fund custody via tanks and taps

One tank per PAMM pool, denominated in the pool asset, for example bitUSD or a UIA. The tank holds all investor deposits plus accrued performance fees. The tank is an existing primitive and is reused unchanged; this proposal adds the share accounting and fee waterfall on top of it.

Four tap types are defined, each with its own flow limit, delay, and authority:

- Trading tap: the manager opens it to move funds to the master trading account, subject to a `periodic_flow_limit` and a `minimum_tank_level`, so investors always retain a floor. The limit is per period and resets with the period; the floor is absolute and cannot be lowered by the manager.
- Performance-fee tap: opens only after a high-water mark is recorded. The amount is `fee_rate * (new_equity - high_water_mark)`, routed to the manager account. The tap refuses to open when new equity is at or below the high-water mark, so a manager who loses money cannot charge a fee for recovering it.
- Withdrawal tap: an investor opens it to redeem shares at current NAV, gated by a `delay_requirement` (anti-run) and a `per-investor cumulative_flow_limit`. The delay is a minimum age before the tap opens; the cumulative limit caps how much one investor can withdraw per period.
- Emergency tap: a supermajority authority, destructor tap, drains the tank to a renegotiation tank. Required on every tank, with no exceptions and no way to remove it.

Field-level detail for the tank attachment, as proposed and to be fixed by the design spec:

- `pool_asset`: the asset id the tank is denominated in.
- `share_asset`: the user-issued asset id representing shares, created with zero initial supply.
- `manager_account`: the manager's BitShares account.
- `platform_authority`: the supermajority authority id for the emergency tap.
- `trading_tap`: tap id, periodic flow limit, minimum tank level, period length.
- `performance_fee_tap`: tap id, fee rate in basis points, high-water mark.
- `withdrawal_tap_template`: delay requirement, per-investor cumulative flow limit.
- `emergency_tap`: tap id, supermajority threshold.
- `nav_buffer`: the rolling 365-entry array described in section 5.

The tank itself enforces the flow limits and delays; this proposal does not reimplement them. What this proposal adds is the share accounting that sits beside the tank and the authority wiring that scopes the taps.

## 3. Share accounting

Shares are a user-issued asset created by the PAMM contract account with zero initial supply, and market-issued assets are refused as share assets, consistent with the program's rule that pool and vault shares are ordinary visible assets. Deposits mint shares at current NAV; withdrawals burn shares and pay out at current NAV.

NAV is computed as `tank_balance / share_supply`, with virtual offsets to avoid division by zero on first deposit. The virtual offsets are a small constant added to both numerator and denominator so that the first depositor does not receive an infinite share count and cannot be diluted to nothing by a second depositor. The constant is proposed as 1 unit of the pool asset and 1 virtual share, and is filed by the design spec.

High-water mark is stored on the tank attachment. The performance fee only accrues above it, and it resets on a new high. A withdrawal that reduces the tank balance below the high-water mark does not lower the high-water mark; only a new equity high moves it. This prevents a manager from charging a fee for recovering a loss.

Deposit and withdrawal mechanics:

- Deposit: the investor transfers pool asset to the tank, the share asset is minted to the investor at `deposit_amount / NAV`, and the tank balance increases. The first deposit sets NAV to 1 via the virtual offsets, so the first depositor receives shares equal to their deposit.
- Withdrawal: the investor opens the withdrawal tap, waits the delay, and receives `shares_burned * NAV` in pool asset. The shares are burned on completion, not on request, so a pending withdrawal does not reduce share supply until it settles.
- Full exit by a single investor is allowed and burns all their shares; the tank balance decreases accordingly, and the minimum tank level floor still applies.

## 4. Custom active authorities

Each investor gets a custom active authority on the PAMM tank's withdrawal tap scoped to their own deposit, so only that investor or their delegate can open it, and only up to their cumulative flow limit. The authority is a BitShares custom active authority, not a new permission type, and it is created at deposit time and removed when the investor's share balance reaches zero.

- The manager's trading tap authority is a separate custom authority that can be revoked by the emergency supermajority. Revocation is immediate and does not require the manager's consent.
- Reporting: a `documentation_requirement` on every tap forces a reason string, and a `review_requirement` routes large flows to an auditor. Both are existing tap features reused here.
- The emergency tap authority is a supermajority of the platform or auditor set, with a threshold proposed at two-thirds. The threshold is filed by the design spec and is not changeable by the manager.

The authority model is the security story. An investor's withdrawal right lives in their own authority, not in the manager's goodwill, and the manager's trading power lives in an authority the platform can revoke. No single party holds both sides, and no party can unilaterally change the limits.

## 5. Performance reporting

Investors must be able to judge PAMM products against each other, so NAV history is a first-class, on-chain, bounded artifact.

- Rolling buffer of 365 `(day, NAV)` entries in the `pamm_pool` attachment. One entry per day; older entries fall off. The buffer is bounded at 365 by construction, so state growth is capped regardless of pool age.
- `pamm_snapshot` operation appends today's entry if missing, gated by the platform or auditor authority, not the manager. The manager cannot write their own performance record.
- NAV source: a BSIP-1 bonded oracle when available, else manager self-report with a challenge window before lock. If the oracle is available, the snapshot uses the oracle's mark-to-market price for the pool's held assets; otherwise the manager submits an equity figure and a challenge window opens, during which any investor or the platform can dispute it. If unchallenged, the figure locks and becomes the day's NAV.
- Standardised client-computed metrics: cumulative return, annualised return, maximum drawdown, current drawdown, over 30-day, 90-day, 365-day, and since-inception windows. These are computed off chain from the buffer and are not consensus state, so the metric definitions can evolve without a hardfork.

The buffer uses head-block time for day boundaries, not wall clock, so two nodes agree on which day an entry belongs to. A snapshot submitted within the same day as an existing entry is refused rather than overwriting it, so a manager cannot rewrite history by resubmitting.

## 6. Operation types

The following operations are proposed, appended at fresh wire tags after the operations the approved proposals already own. Tag numbers are filed by the design spec, not by this document, consistent with the program's rule that no BSIP allocates tag numbers.

- `pamm_pool_create`: create a PAMM pool with a tap topology, share asset, manager, and platform authority. Transfers nothing; sets identity once.
- `pamm_deposit`: deposit pool asset into the tank and mint shares at current NAV.
- `pamm_withdraw_request`: request a withdrawal tap opening, subject to delay and cumulative limit.
- `pamm_withdraw_settle`: settle a matured withdrawal, burning shares and paying out pool asset.
- `pamm_trade_tap_open`: manager opens the trading tap within its flow limit.
- `pamm_performance_fee_claim`: manager claims the performance fee above the high-water mark.
- `pamm_snapshot`: platform or auditor appends a daily NAV entry.
- `pamm_emergency_drain`: supermajority drains the tank to a renegotiation tank.
- `pamm_pool_close`: close a pool after all shares are redeemed and taps are settled.

Each operation is hardfork-gated and refuses at apply when the flag has not passed. Each validates its inputs at validate, so a malformed request fails before it reaches consensus state.

## 7. Object types

Two new object types are proposed, appended to the id registry after the existing objects.

- `pamm_pool_object`: the pool record, holding the tank id, share asset id, manager, platform authority, tap topology, high-water mark, and the 365-entry NAV buffer.
- `pamm_investor_position`: per-investor state, holding the investor account, share balance, deposit history summary, cumulative withdrawal flow, and the custom active authority id. Created at first deposit and removed when share balance reaches zero.

No other object types are introduced. The tank and tap objects are existing primitives reused unchanged, and the share asset is an ordinary user-issued asset.

## 8. Hardfork gate

One flag, `HARDFORK_PAMM_PASSED`, gates tank creation, share issuance, and all tap operations above. Dummy date only, proposed as a committee-set parameter. Evaluators refuse at apply when the flag has not passed, using the established fail-closed pattern. The gate is all-or-nothing: a build that has the operations but not the gate refuses every PAMM operation.

## 9. State bounds

State growth is bounded by construction:

- The NAV buffer is fixed at 365 entries per pool, with older entries falling off.
- Per-investor positions are removed when share balance reaches zero, so a pool with no investors carries no investor state.
- Tap flow limits cap how much can move per period, which bounds the rate of state change but not the number of operations; the operation count is bounded by the block size limit and the per-transaction operation cap, which are existing protocol limits.
- Pool creation is permissionless but costs a creation fee, which bounds spam. A committee-set creation ceiling and per-account pool cap are filed as open items in the design spec, consistent with the unbounded-state lesson the settlement proposal recorded.
- The share asset is a normal UIA, so its supply tracking is handled by the existing asset machinery.

## 10. Out of scope by design

- No new VM. The proposal uses existing primitives only.
- No new consensus math beyond share and NAV arithmetic.
- No foreign-chain attestation. A PAMM trades BitShares assets on BitShares markets.
- No legacy account, asset, or pool is touched.
- No blind form of anything. Share balances and NAV are public by construction, consistent with the program's settled rule that protocol totals stay public.
- No manager co-investment floor unless open decision 3 is closed in favour.

# Rationale

Tanks and taps already solve bounded custody, flow limits, and emergency drainage; PAMM only adds share accounting and a fee waterfall on top. Custom active authorities give per-investor rights without new account types. No new VM, no new consensus math beyond share and NAV arithmetic. The reuse is the point: every primitive this proposal depends on already exists, has already been audited, and already has tests, so the new surface area is small and reviewable.

The fee waterfall is the economic core. A performance fee that accrues only above a high-water mark aligns the manager with the investor: the manager earns nothing for recovering a loss, and earns only for new value. The high-water mark is stored on the tank, not in the manager's records, so it cannot be quietly reset. The fee rate is set at pool creation and is immutable, so a manager cannot raise it after investors have deposited.

The reporting layer is what makes the product comparable. A pool with no standardised history is a pool whose marketing is unverifiable; a pool with a 365-day on-chain buffer is a pool whose numbers any client can compute independently. The buffer is bounded so it cannot grow forever, and the snapshot is gated by the platform so the manager cannot write their own record.

Rejected alternatives, each with its reason. Building a PAMM on a custodial model is rejected because custody is the problem. Building it on a trustless counterparty settlement protocol is rejected because that protocol is designed for orderbook settlement, not for managed accounts, and adapting it would add complexity without adding safety. Adding a new VM is rejected because the chain has no general smart contracts and this proposal does not pretend otherwise. Allowing the manager to set NAV is rejected because the manager is the party with the incentive to overstate it; the oracle or the challenge window is the backstop. Letting the manager lower the minimum tank level is rejected because that floor is the investor's guarantee, and a manager who can lower it can drain the tank.

# Risks and failure modes

- The manager can still trade badly. PAMM does not guarantee returns; it guarantees bounded custody and transparent reporting. An investor who deposits into a badly managed pool loses money, and no primitive here prevents that. The mitigation is the reporting layer and the market, not the protocol.
- NAV oracle risk. Master account equity must be reported honestly. A bonded oracle is preferred, and where only manager self-report is available, the challenge window is the defence. A manager who overstates NAV and is not challenged locks in a false record, and subsequent deposits mint shares at the inflated price. The mitigation is the challenge window and the platform's gate on `pamm_snapshot`, not a proof.
- Run risk. The delay requirement on withdrawals mitigates but does not eliminate a run: a pool whose NAV is falling still faces withdrawal requests that queue up and settle at the falling NAV, and an investor who exits early gets a better price than one who exits late. The delay spreads the exit over time but does not change the arithmetic.
- Share dilution. The first deposit must set NAV to 1 via the virtual offsets, or a first-depositor attack is possible: a depositor who puts in 1 unit, receives 1 share, then a second depositor puts in 1,000,000 units and receives 1,000,000 shares, leaving the first depositor with a share worth 500,000 units instead of 1. The virtual offsets close this by making the first deposit price finite and the second deposit price close to it.
- Manager capture of the platform authority. If the platform authority is captured, the emergency tap can be misused and the snapshot gate can be bypassed. The mitigation is the supermajority threshold and the on-chain visibility of every snapshot and emergency action.
- State growth if pool creation is unbounded. Mitigated by the creation fee and the proposed per-account and global caps, filed as open items.

# Backwards compatibility

No migration. Tanks, taps, and custom authorities are additive. Legacy accounts, assets, and pools are untouched. New operations append behind one hardfork flag, new object types append to the id registry, and no existing field moves or reinterprets. A wallet that does not know the PAMM class keeps working on every other market. The gate dates are the committee's to set or not set, and where the flag has not passed, the opt-in and the tap operations refuse at apply, using the program's established fail-closed pattern. The rebate and fee payment targets are existing systems, tanks and vesting balances, which need no new type.

# Test cases

Each row names behaviour to pin before any build, derived from what this document's specification already states. The suite does not exist, and this section claims no recorded result.

- Deposit minting: a deposit mints shares at current NAV, and the first deposit sets NAV to 1 via the virtual offsets.
- First-depositor safety: the virtual offsets prevent a first-depositor dilution attack.
- Withdrawal delay: a withdrawal tap refuses to open before the delay elapses.
- Cumulative limit: a withdrawal that would exceed the per-investor cumulative flow limit refuses.
- Minimum tank level: a trading tap opening that would push the tank below the floor refuses.
- Performance fee gate: a performance-fee tap refuses to open when new equity is at or below the high-water mark.
- High-water mark reset: a new equity high moves the high-water mark; a withdrawal does not lower it.
- Emergency drain: requires the supermajority threshold; a sub-threshold request refuses.
- Authority scoping: an investor's withdrawal tap refuses to open for any account other than the scoped investor.
- Manager revocation: the platform can revoke the manager's trading tap authority without the manager's consent.
- Snapshot gating: `pamm_snapshot` refuses when submitted by the manager or any non-platform account.
- Snapshot deduplication: a second snapshot in the same day refuses rather than overwriting.
- Buffer bound: the NAV buffer never exceeds 365 entries; older entries fall off.
- Day boundary: day boundaries use head-block time, not wall clock.
- Challenge window: a disputed self-report does not lock until the window closes unchallenged.
- Share asset refusal: a market-issued asset refuses as a share asset at pool creation.
- Wire pinning: operation tags and object ids append and hold stable across builds with and without the flag.
- Close: `pamm_pool_close` refuses while shares remain outstanding or taps are unsettled.

# Reference implementation

The proposal is a design. No code exists today for this proposal, though its building blocks already exist in the base tree.

What still needs writing before any build: the design spec in `docs/superpowers/specs/` fixing the tag numbers, object ids, field order, the virtual offset constant, the supermajority threshold, the creation ceiling and per-account pool cap, the fee rate bounds, and the challenge window length; the assertion suite with the adversarial vectors above; wire-format pinning across builds with and without this feature; and full-binary evidence gates before any replay or pilot discussion. None exists today, and this document claims none of it.

# Open decisions

1. Whether PAMM pools may be denominated in foreign assets (F track) or only BitShares-native assets.
2. Performance-fee high-water mark: global versus per-investor.
3. Whether to allow manager co-investment (skin in the game) as a minimum tank level floor.
4. Mark-to-market source: bonded oracle versus manager self-report plus challenge.
5. The supermajority threshold for the emergency tap.
6. The creation ceiling and per-account pool cap.

# Build requirements

- Design spec finalized with open decisions closed or filed.
- Tank and tap primitives available (BSIP-72) in the base tree.
- Custom authority evaluator present (already in core).
- Share and NAV math helper with tests.
- Hardfork gate, bounded state, wire pinning, lane re-audit.

# What this spec does not close

- It does not implement BSIP-8.
- It does not allocate tag numbers or object ids.
- It does not set a hardfork date.
- It does not activate anything.

# Discussion

Contested threads, rejected alternatives, and the audit must-close list for this draft:

- Rejected alternatives and why: custodial PAMM, trustless counterparty settlement, a new VM, manager-set NAV, and manager-lowered minimum tank level. See the Rationale section for reasons.
- Contested threads: the open decisions above, the hardfork date, the pilot scale, and whether the first PAMM pool is included in the $20,000 face cap.
- Audit must-close list: `build/audit-round-bsip-readiness/bsip-readiness-review.json` (BSIPR-01/03/12) and `build/audit-round-bsip8/`. Activation remains unauthorized until every row of `build/ACTIVATION-CHECKLIST.md` clears.

# Summary for Shareholders

- Cost: nothing to existing holders by construction. Legacy assets (`bit*`, `HONEST.*`, gateway UIAs) keep their semantics, fees, and settlement rules; no migration, no caps, no repricing.
- Who pays: new work is funded by opt-in fees, forfeits, pilot-capped issuance, and bonded producer stakes on the new asset class only, not by a standing BTS subsidy. PAMM pools charge ordinary trading fees on the underlying assets, and the performance fee is a share of gains, not a standing charge.
- What changes for existing holders: nothing, unless they opt into the new class. Every new operation, object, and field is appended and hardfork-gated, so pre-fork nodes keep their current behaviour.
- Review date: any standing policy is re-reviewed against pilot evidence before activation; see `build/ACTIVATION-CHECKLIST.md`.

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