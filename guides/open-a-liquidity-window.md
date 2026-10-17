---
description: Give shareholders a controlled way to sell by running a time-boxed batch auction or fixed-price tender that settles atomically on-chain.
---

# Open a Liquidity Window

A Liquidity Window is a scheduled, time-boxed market for one share class. Eligible sellers post asks, eligible buyers post bids, and when the window closes everything clears at a single price in one transaction. It replaces the traditional tender offer, and you decide when it happens, who can take part and how much can trade. The full mechanics are in [Liquidity Windows](../liquidity/liquidity-windows.md).

{% stepper %}
{% step %}
### Choose a format

Open **Liquidity** in the Console and select **New window**. There are two formats:

* **Batch auction.** Sellers set a minimum price and buyers set a maximum price. The window clears at the single price that matches the most volume.
* **Fixed-price tender.** You (or a named buyer) set the price, and sellers decide how much to tender at it. Allocations are pro-rata if more is tendered than the buyer wants.

Most companies run batch auctions for routine liquidity and tenders when a specific buyer, or the company itself, wants to buy back a set amount.
{% endstep %}

{% step %}
### Set participation

Decide who can sell and who can buy.

| Setting | Examples |
|---|---|
| Eligible sellers | All holders of the class, employees with vested lots, a named list |
| Eligible buyers | Existing holders, invited investors, any wallet with an `INVESTOR_ELIGIBLE` claim |
| Per-seller cap | Each seller can sell up to a share of their holding |
| Maximum volume | A ceiling on total shares traded |

Every order is still checked against the class's [transfer rules](../concepts/transfer-rules.md). A seller with unvested or locked lots can only offer their unlocked balance.
{% endstep %}

{% step %}
### Set pricing controls

For a batch auction, you can set:

* **A reserve price.** No trade clears below it.
* **A per-seller cap.** The most shares any one seller can offer.
* **Right of first refusal.** Name a right holder and a period, and the company or its designee gets a priority period after close to buy at the clearing price.

To buy back shares, place a bid from the company treasury like any other buyer once the window opens. If no volume can trade at the reserve price, the window closes with no trades and every order is returned.
{% endstep %}

{% step %}
### Schedule and approve

Set the open and close times. Two to five business days is typical, which gives employees time to decide. Submit to **Approvals**. Once approved, `LiquidityWindow` emits `WindowOpened`, eligible holders see the window in Portfolio, and they are notified by email.

If the class uses `TransferWindowModule`, transfers are only possible while a window is open. That keeps all trading inside windows you control.
{% endstep %}

{% step %}
### Watch the book

During the window, **Liquidity** shows aggregate demand and supply at each price level. Individual orders appear as `OrderPlaced` events in the Explorer, with the wallet shown and no identity attached. Sellers' tokens and buyers' USDG are locked when they place orders, so every order on the book is fully funded.

You can cancel a window before it clears. That emits `WindowCancelled` and unlocks every order.
{% endstep %}

{% step %}
### Clear the window

After the close time, anyone can call `clear(windowId)`. Precursor's operator does this automatically. In one transaction, the contract:

1. Computes the clearing price that matches the most volume.
2. Fills orders at that price, pro-rata at the margin.
3. Moves tokens to buyers and USDG to sellers, minus fees.
4. Unlocks everything that didn't fill.

The contract emits `WindowCleared` with the price and volume. The price is also published as a Reference Valuation with source `WINDOW_CLEARING`.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
For the seller's side of the same process, share [Sell in a Liquidity Window](sell-in-a-liquidity-window.md) with your employees before the window opens.
{% endhint %}
