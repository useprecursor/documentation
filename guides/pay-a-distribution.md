---
description: Fund a dividend or other distribution in the DistributionVault, fix a record block and let holders claim pro-rata on-chain.
---

# Pay a distribution

Distributions pay holders of a share class in proportion to their holdings at a record block. They cover dividends, proceeds from a sale of the company, and any other pro-rata payout. The company funds the `DistributionVault` once, and each holder claims their share directly. Payouts are pull-based, so gas costs stay predictable however many holders a class has, and every claim is auditable. For the mechanics in full, see [Corporate actions](../concepts/corporate-actions.md).

{% stepper %}
{% step %}
### Create the distribution

Open **Distributions** in the Console and select **New distribution**. Choose:

| Setting | Notes |
|---|---|
| Share class | The class being paid |
| Asset | USDG by default; other supported assets are listed in the Console |
| Total amount | Paid out pro-rata across the class |
| Record block | A block that has already passed, by default the block just before funding. Record blocks are Robinhood Chain L2 block numbers |
| Excluded accounts | Optional; up to 20 accounts whose shares don't receive the distribution, such as treasury shares or the `LiquidityWindow` escrow |
| Claim deadline | Optional; minimum 12 months after funding |

The Console previews the amount per share, using the current supply and the class's `uiMultiplier`.
{% endstep %}

{% step %}
### Attach the supporting record

Upload the board resolution that approves the distribution. It's anchored in the `DisclosureRegistry` as a `BOARD_RESOLUTION`, so holders can match the payout to the decision behind it.
{% endstep %}

{% step %}
### Approve and fund

Submit to **Approvals**. Once your signers approve, the treasury transfers the total amount into the `DistributionVault`. The vault emits `DistributionFunded` with the distribution ID, asset, amount and record block. Funding is final: the amount can't be withdrawn while the distribution is active.

You can also create and fund distributions through the API with `POST /v1/distributions`.
{% endstep %}

{% step %}
### Holders claim

Entitlements come from balances at the record block. Each holder's entitlement is their balance at that block as a share of the eligible supply, meaning total supply minus excluded balances. Portfolio can also claim on a holder's behalf with `claimFor`, which always pays the holder. Holders see the distribution in [Portfolio](../products/portfolio.md) and claim with one action. Each claim emits `DistributionClaimed`, and gas is sponsored.

Holders who move their tokens after the record block still keep their entitlement. Holders who acquire tokens after it don't receive one.
{% endstep %}

{% step %}
### Track and close out

**Distributions** shows how much has been claimed and by how many holders. Portfolio reminds holders who haven't claimed. Unclaimed amounts stay claimable until the deadline, if you set one. After the deadline, the remainder returns to the treasury.
{% endstep %}
{% endstepper %}

## Payouts at an acquisition

When the company is acquired, the purchase consideration goes through the same vault. Each holder claims their share of the consideration, and the registrar burns the class's tokens. Each burn emits `SharesRedeemed` with a reference to the transaction record, so the full exit is visible from start to finish in the [Explorer](../products/explorer.md).

{% hint style="info" %}
If you subscribe to the `distribution.funded` and `distribution.claimed` [webhooks](../developers/webhooks.md), you can reconcile payouts in your own systems as they happen.
{% endhint %}
