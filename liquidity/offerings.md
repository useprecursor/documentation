---
description: How the OfferingVault runs a primary offering, from configuration and escrowed subscriptions to minting at close, allocation when oversubscribed, and refunds.
---

# Primary offerings

A primary offering is how a company raises new capital on Precursor. Each offering runs in the company's `OfferingVault`. Investors deposit stablecoins into escrow, and when the offering closes, the vault releases the funds to the company and mints shares to investors in the same transaction. If the offering does not reach its minimum, every investor can take their full deposit back.

No one, including the company and Precursor, can move escrowed funds anywhere except to the company treasury at a successful close or back to the investor who deposited them.

## Parameters

The company sets these when it creates the offering in the [Console](../products/console.md) or through `POST /v1/offerings`. They are fixed once the offering opens.

| Parameter | Description |
| --- | --- |
| `shareToken` | The share class being issued |
| `pricePerShare` | Price of one share in the accepted stablecoin |
| `minRaise` | The amount that must be raised for the offering to succeed |
| `maxRaise` | The most the offering will accept. Subscriptions above this are allocated pro-rata |
| `minTicket` | The smallest subscription accepted from one investor |
| `opensAt` / `closesAt` | The subscription period |
| `stablecoin` | The settlement asset. USDG by default |
| `requiredClaims` | The claim topics an investor must hold, such as `IDENTITY_VERIFIED`, `INVESTOR_ELIGIBLE` and `REGION` |
| `treasury` | The company address that receives the proceeds |
| `documentId` | The offering document anchored in the `DisclosureRegistry` |

The vault also checks every subscription against the share class's [transfer rules](../concepts/transfer-rules.md) as if the shares were being transferred to the investor. An investor who could not receive the shares cannot subscribe for them, so an offering never mints to a wallet that the rules would block.

## Lifecycle

| State | Meaning |
| --- | --- |
| `SCHEDULED` | Created, waiting for `opensAt`. Terms and documents are visible. |
| `OPEN` | Accepting subscriptions. |
| `CLOSED` | Reached at least `minRaise`. Funds released, shares minted. |
| `FAILED` | Did not reach `minRaise` by `closesAt`, or was cancelled by the company before close with `cancelOffering`. Every deposit is refundable. |

An offering reports `OPEN` after `closesAt` until it is settled, but no longer accepts subscriptions. An offering moves from `OPEN` to `CLOSED` or `FAILED` when `close()` is called after `closesAt`. Anyone can call it, and Precursor's operator calls it automatically. The company may also close early once `maxRaise` has been reached.

## Subscribing

An investor approves the vault to spend their stablecoin and calls `subscribe(offeringId, amount)`. In Portfolio this is a single step, and gas is sponsored. The vault checks that:

1. the offering is `OPEN`
2. the investor's identity holds every claim in `requiredClaims`, and none has expired
3. the investor's total subscription is at least `minTicket`
4. a transfer of the resulting shares to the investor would pass `canTransfer`

If every check passes, the stablecoin moves into escrow and the vault emits `Subscribed`. An investor can subscribe more than once. Their subscriptions add together. An offering accepts up to 500 distinct subscribers (`MAX_SUBSCRIBERS`), so that close fits in one transaction.

## Closing

At close, the vault compares total subscriptions with `minRaise` and `maxRaise`.

**Successful close.** If total subscriptions are at least `minRaise`, the vault:

1. calculates each investor's allocation
2. checks each investor against the share class rules again. An investor who no longer passes, for example because a claim expired, is not minted and their whole subscription becomes refundable
3. mints shares to each remaining investor with `SharesIssued`, using the offering id as the `ref`
4. sends the proceeds to `treasury`, minus the platform fee
5. records any unallocated amount as refundable to its investor
6. publishes `pricePerShare` to the share class's `ValuationOracle` as a `PRICED_ROUND` Reference Valuation
7. emits `OfferingClosed(offeringId, raised, sharesIssued)`, where `raised` is the amount actually accepted

The shares minted in an offering form a new lot for each investor. If the share class has a `HoldingPeriodModule`, the holding period for that lot starts at close.

**Failed close.** If total subscriptions are below `minRaise`, the vault emits `OfferingFailed` and every deposit becomes refundable in full. No fee is charged.

## Allocation when oversubscribed

If total subscriptions exceed `maxRaise`, each investor receives the same fraction of their subscription:

```
fill ratio  = maxRaise / totalSubscribed
allocation  = subscription × fill ratio
shares      = allocation / pricePerShare
refund      = subscription - allocation
```

For example, an offering with a `maxRaise` of 5,000,000 USDG receives 6,250,000 USDG in subscriptions. The fill ratio is 0.8. An investor who subscribed 100,000 USDG at 20 USDG per share receives 80,000 USDG worth of shares, or 4,000 shares, and can reclaim 20,000 USDG.

Investors are charged only for the whole raw token units they receive, so any rounding remainder is added to their refund. Allocation is pro-rata only. Subscription time does not affect it, so there is no advantage to subscribing in the first minutes of an offering.

## Refunds

Refundable amounts stay in the vault until the investor calls `refund(offeringId)`, which transfers the full refundable balance and emits `Refunded`. Portfolio prompts investors to claim, with gas sponsored. Refunds never expire.

## Events

| Event | Emitted when |
| --- | --- |
| `OfferingCreated(offeringId)` | The company creates an offering |
| `Subscribed(offeringId, investor, amount)` | An investor's subscription is accepted |
| `OfferingClosed(offeringId, raised, sharesIssued)` | The offering closes successfully |
| `OfferingFailed(offeringId)` | The offering misses its minimum or is cancelled |
| `Refunded(offeringId, investor, amount)` | An investor withdraws a refundable amount |
| `SharesIssued(to, amount, ref)` | Shares are minted to an investor at close (emitted by the share token) |

Webhooks for `offering.subscribed`, `offering.closed` and `offering.failed` mirror these events. See [Webhooks](../developers/webhooks.md).

{% hint style="info" %}
To set up and run an offering step by step, see [Run an offering](../guides/run-an-offering.md). Investors can follow [Invest in an offering](../guides/invest-in-an-offering.md).
{% endhint %}
