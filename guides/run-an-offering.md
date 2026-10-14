---
description: Configure, open and close a primary offering that escrows USDG on-chain and mints shares to investors at close.
---

# Run an offering

A primary offering sells new shares of a share class to eligible investors. Investor funds sit in escrow in the `OfferingVault` until the offering closes. If the minimum raise is met, the vault releases funds to the company treasury and mints shares in the same transaction. If it isn't, every investor can take their money back. For the mechanics in full, see [Primary offerings](../liquidity/offerings.md).

You'll need a live share class (see [Tokenize your company](tokenize-your-company.md)) and approval from your signers.

{% stepper %}
{% step %}
### Draft the offering

Open **Offerings** in the Console and select **New offering**. Set the terms:

| Term | What it controls |
|---|---|
| Share class | The class being issued |
| Price per share | Fixed for the life of the offering |
| Minimum raise | Below this, the offering fails and everyone is refunded |
| Maximum raise | Above this, allocations are pro-rata and the excess is refunded |
| Minimum ticket | Smallest subscription accepted |
| Opens at, closes at | The subscription period |
| Stablecoin | USDG by default |
| Eligible claim topics | For example `IDENTITY_VERIFIED` and `INVESTOR_ELIGIBLE` |

The shares you can issue are capped by the authorized shares for the class. The Console shows how much headroom is left.
{% endstep %}

{% step %}
### Attach the offering documents

Upload the documents investors will review. Each one is anchored in the `DisclosureRegistry` as an `OFFERING_DOCUMENT`, and investors see the hash in Portfolio next to the file. If you replace a document before the offering opens, the new hash replaces the old one. After the offering opens, the documents are fixed.
{% endstep %}

{% step %}
### Approve and publish

Submit the draft to **Approvals**. Once your signers approve it, `OfferingVault` emits `OfferingCreated` and the offering appears in Portfolio for eligible investors. You can share a direct link, or invite specific investors from **Holders**.

You can also create the offering through the API with `POST /v1/offerings`. Console and API offerings behave identically.
{% endstep %}

{% step %}
### Monitor subscriptions

While the offering is open, **Offerings** shows the amount raised, the number of investors and progress toward the minimum. Every subscription is a `Subscribed` event, so the same figures are public in the Explorer. If you've set up [webhooks](../developers/webhooks.md), you'll receive `offering.subscribed` for each one.

Investors who haven't finished verification can't subscribe, because the vault checks their claims on-chain. The **Holders** view lists anyone who started onboarding and hasn't completed it.
{% endstep %}

{% step %}
### Close the offering

After the close time, select **Close offering** or call `POST /v1/offerings/{id}/close`. Closing is a single on-chain transaction:

* **Minimum met.** Funds move to the treasury, minus the platform fee. Shares are minted to each investor, scaled pro-rata if the offering was oversubscribed. Excess subscriptions are returned. The vault emits `OfferingClosed` with the amount raised and shares issued.
* **Minimum not met.** The vault emits `OfferingFailed`, and each investor calls `refund` to withdraw their full deposit. Portfolio prompts them to do so.

New lots start their holding periods from the close date, according to the rules on the class.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
The offering price is published as a Reference Valuation with source `PRICED_ROUND` when the offering closes. See [Reference Valuation](../concepts/reference-valuation.md).
{% endhint %}
