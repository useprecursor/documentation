---
description: Verify your identity, fund your wallet with USDG and subscribe to a company's primary offering from Precursor Portfolio.
---

# Invest in an offering

This guide is for investors subscribing to a primary offering on Precursor. You'll verify your identity once, and then subscribe to any offering you're eligible for. Your funds stay in an on-chain escrow until the offering closes, and you can check every step in the Explorer.

{% stepper %}
{% step %}
### Open the offering

Companies share offerings by direct link or by inviting you by email. Either takes you to the offering page in [Portfolio](../products/portfolio.md), which shows the price per share, the minimum and maximum raise, the minimum ticket, the close date and the offering documents.

Each document shows its hash. The same hash is recorded on-chain in the company's `DisclosureRegistry`, so you can confirm that what you're reading is exactly what the company published.
{% endstep %}

{% step %}
### Verify your identity

If this is your first time on Precursor, you'll complete identity verification. It usually finishes the same day. When it does, signed claims are written to your on-chain identity:

* `IDENTITY_VERIFIED`, confirming verification.
* `REGION`, recording your country.
* `INVESTOR_ELIGIBLE`, recording your eligibility tier, if the offering requires it.

None of your personal information goes on-chain. The claims are signed attestations that say a check passed, with an expiry date. See [Identity and claims](../concepts/identity-and-claims.md).
{% endstep %}

{% step %}
### Set up a wallet

You can connect a wallet you already use on Robinhood Chain, or let Portfolio create an embedded wallet for you. Either way, the wallet is registered to your identity. Precursor sponsors gas, so you don't need ETH.

If you use institutional custody, your custodian's wallet can be registered to your identity in the same way.
{% endstep %}

{% step %}
### Fund and subscribe

Deposit USDG (or the stablecoin the offering accepts) to your wallet, enter the amount you want to invest and confirm. Portfolio asks you to approve the stablecoin and then calls `subscribe` on the `OfferingVault`. Your deposit is held in escrow, and a `Subscribed` event records it on-chain.

Your subscription can't fall below the minimum ticket. If the offering is close to its maximum, Portfolio shows how much room is left.
{% endstep %}

{% step %}
### Receive your shares at close

When the offering closes, one of two things happens:

| Outcome | What you receive |
|---|---|
| Minimum raise met | Share tokens in your wallet, in the same transaction that releases funds to the company. If the offering was oversubscribed, your allocation is scaled pro-rata and the rest of your deposit is returned. |
| Minimum raise not met | Nothing is issued. Portfolio prompts you to call `refund`, which returns your full deposit. |

Your new shares appear in Portfolio as a lot with its acquisition date. Any holding period on the class starts from that date.
{% endstep %}
{% endstepper %}

## After you invest

As a holder, you can:

* Read company updates and financial statements the company shares with verified holders.
* Vote on proposals during their voting period. See [Governance and voting](../concepts/governance.md).
* Claim distributions when the company pays them.
* Sell during a Liquidity Window, once any holding period on your lots has ended. See [Sell in a Liquidity Window](sell-in-a-liquidity-window.md).

{% hint style="warning" %}
Claims expire. Portfolio reminds you before your verification needs renewing. An expired claim blocks transfers and subscriptions with the reason code `CLAIM_EXPIRED` until you renew it.
{% endhint %}
