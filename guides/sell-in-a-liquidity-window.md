---
description: How employees and other shareholders sell vested, unlocked shares in a company-run Liquidity Window and receive USDG at settlement.
---

# Sell in a Liquidity Window

When your company opens a Liquidity Window, you can sell some or all of your unlocked shares to eligible buyers. Every order clears at one price, at the same moment, in one transaction. This guide covers the seller's side. If you're running the window, see [Open a Liquidity Window](open-a-liquidity-window.md).

## Understand what you can sell

In [Portfolio](../products/portfolio.md), your holdings are split into lots. Each lot records how many shares you acquired, when and how. Some lots can be sold now and some can't.

| Lot state | Can you sell it? | Why |
|---|---|---|
| Vested and unlocked | Yes | No rule restricts it |
| Unvested | No | `EmployeeVestingModule` locks it until it vests |
| Inside a holding period | No | `HoldingPeriodModule` locks it until the period ends |
| Frozen | No | The company or registrar has frozen it, with a recorded reason |

Your sellable amount is your **unlocked balance**, which the token reports through `unlockedBalanceOf`. Portfolio shows it on every holding, and the vesting timeline shows when more shares unlock. When you sell, lots are consumed oldest first.

{% stepper %}
{% step %}
### Check the window terms

You'll get an email when a window opens for a class you hold. The window page shows:

* Whether it's a batch auction or a fixed-price tender.
* The open and close times.
* Any cap on how much each seller can sell.
* Any reserve price, below which nothing trades.
* The latest [Reference Valuation](../concepts/reference-valuation.md) for the class, with its source and date.
{% endstep %}

{% step %}
### Place your order

Enter the number of shares to sell. In a batch auction, you also set your **minimum price**, the lowest price per share you'll accept. In a tender, the price is already set, so you only choose a quantity.

When you confirm, your shares are locked in the `LiquidityWindow` contract and an `OrderPlaced` event is recorded. Gas is sponsored.
{% endstep %}

{% step %}
### Adjust or cancel if needed

While the window is open, you can cancel your order and place a new one. Cancelling unlocks your shares immediately and emits `OrderCancelled`. After the window closes, orders can't be changed.
{% endstep %}

{% step %}
### Settlement

When the window clears, a single clearing price is set for everyone:

* **Clearing price at or above your minimum.** Your order fills, either in full or pro-rata if there were more sellers than buyers at the margin. USDG arrives in your wallet in the same transaction that moves your shares to the buyer.
* **Clearing price below your minimum.** Your order doesn't fill, and your shares are unlocked.

Anything that doesn't fill returns to your wallet automatically. Portfolio shows the result, the clearing price and the transaction hash.
{% endstep %}
{% endstepper %}

## Common questions from sellers

**Why can't I select some of my shares?** They're either unvested or still inside a holding period. Portfolio shows the date each lot unlocks.

**Does a low minimum price mean I'll get a low price?** No. Everyone who fills gets the same clearing price. Your minimum only decides whether your order is included.

**Can the company buy my shares?** Yes, if the company places a buyback bid from its treasury or exercises a right of first refusal. Either way, the company pays the same clearing price as any other buyer.

{% hint style="info" %}
Every fill is public in the Explorer as part of the `WindowCleared` settlement, showing wallets and amounts but not names.
{% endhint %}
