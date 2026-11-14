<!-- SPDX-License-Identifier: MIT -->
# BitShares BSIP-5 — Open order rewards on the native markets (lightpaper)

> **Lightpaper**
> **Version 1.0 — October 2026**
> **License:** Public domain
> **Status:** Draft for community review

## Abstract

Resting an order on BitShares earns nothing. The maker, the party whose order sits in the book waiting, gives the market time, size, and a price the next trader can reach, and is paid nothing for it, so books sit thin and price discovery drifts to the swap pools. Each taker's fee funds a maker rebate, paid when his order fills, and a provider pool, distributed to open orders weighted by three measured qualities: how long the order has rested, capped at an hour; how much quantity remains; and how close its price sits to the market's own recent price. Gating, capping, and self-trade refusal keep the rewards unforfeitable and unpumpable. Participation is opt-in per asset, everything routed or paid stays on the ledger, and nothing changes for markets that never opt in. The pitch: waiting in the book becomes a paid service, priced in public by formula.

## What changes for you

- Legacy asset holders: nothing changes; markets your assets never opt into behave exactly as they do now.
- Makers: a fill of your resting order pays the rebate at once, and surviving orders share a daily pool by measured time, size, and proximity instead of earning nothing.
- Takers: your cost structure grows only where the assets you trade have opted in, at a rate the issuer sets once, in public, and your fills still match in the same price-time order as today.
- Witnesses: the pools accumulate and distribute inside the already scheduled maintenance sweep, bounded by a per-sweep order count.
- Gateway operators: your IOUs are ordinary assets here too, eligible only if you opt in, and deposits and withdrawals stay what they are.
- Developers: a market reward object appends, new per-asset fields append to asset options, and the sweep gains one bounded loop per participating market, with tags and fields pinned by tests before any hardfork.

## The design in one page

An asset opts in through a single new field in its options, an open order fee from 0 to 500 basis points, charged on the taker's received amount. On each fill the fee splits: a maker rebate, proposed at 70 percent, pays the filled maker into a vesting balance immediately, and the rest, 30 percent, accumulates in the market's per-asset provider pool. At each maintenance sweep, qualifying open orders compute a TSP weight: time credit capped at one hour and floored at five minutes, times remaining quantity, times a linear proximity taper that zeroes orders ten percent or more from price. The sweep keeps the largest standing orders up to a cap of 256 before any weight arithmetic, then distributes the pool among survivors by weight, with the integer remainder staying in the pool. Amendments reset the time credit. Self-trading within one account is refused on participating markets. Non-participating markets keep every behavior they have.

## Key numbers

- Open order fee range: 0 to 500 basis points.
- Maker rebate: 70 percent (proposed).
- Provider pool: 30 percent (proposed).
- Minimum time gate: 100 blocks (5 minutes at 3-second blocks).
- Maximum time credit: 1200 blocks (1 hour).
- Proximity threshold: 0.9.
- Per-sweep top-N cap: 256.
- Reference price window: 1 hour.
- Time weight ratio cap: 12 to 1 oldest to youngest.

## Risks

- The time leg saturates: most mature orders sit at the cap, so the working gradient becomes size and proximity.
- A hostile issuer controls the tap on their own asset; the empty-book rule prevents repricing live orders, and the cap and default bound the damage.
- Wash trading is not impossible, only unprofitable in aggregate; a monopolist of a thin market can run near break-even before the chain's own transaction fees.
- State growth lives in the reward field per open order and in the pool accumulator per market; the design spec must cap participating markets per asset.
- Cold start: an opted-in market with no recent fills has no proximity reference and pays nothing that sweep.

## Non-goals

- No change to matching: price-time priority, same fill accounting, no new order types.
- No rewards reaching margin calls, force settlement, or the ACCS offer book.
- No per-fill provider rewards.
- No taker rewards.
- No proximity against an external feed.
- No order-book blindness: resting orders are public pseudonyms by design.

## Copyright

This document is placed in the public domain.
path: /workspace/docs/lightpapers/bsip-5-open-order-rewards.md