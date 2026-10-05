---
description: The Explorer is the public record of every company on Precursor, showing supply, holder counts, transfers, corporate actions and anchored documents, all verifiable on Robinhood Chain.
---

# Precursor Explorer

The Explorer is a public, read-only view of everything Precursor records on-chain. Anyone can use it, without an account, at [explorer.useprecursor.xyz](https://explorer.useprecursor.xyz).

It is built on the same indexed event stream that powers the Console, Portfolio and API. The Explorer adds no information that the chain does not already hold. It organizes that information around companies and share classes instead of raw transactions, and it links every figure back to the transactions behind it.

## Views

### Companies

A directory of every company that has deployed through the `PrecursorFactory`. Each entry shows the company name, its share classes, total holders and the date it deployed. The directory is generated from `CompanyDeployed` events, so a company appears here only if the factory deployed it.

### Company page

For one company:

- the addresses of its contract set: `IdentityRegistry`, `TrustedVerifiers`, one `PrecursorShareToken` and `RuleEngine` per share class, and its vaults, `Governance`, `DisclosureRegistry`, `ValuationOracle` and `IssuerTimelock`
- every share class with its supply, holder count and current multiplier
- the latest [Reference Valuation](../concepts/reference-valuation.md) per class, with its source and date
- a timeline of offerings, Liquidity Windows, distributions, votes and corporate actions

### Share class page

For one share class:

- **Supply.** Raw token supply, the current `uiMultiplier`, and the share count they imply.
- **Holders.** The number of holder wallets and identities. Individual holders are shown by wallet address only.
- **Transfers.** Every `Transfer`, `SharesIssued` and `SharesRedeemed` event, with its reference.
- **Rules.** The modules enabled in the `RuleEngine` and the hash of each module's configuration.
- **Controls.** Every pause, freeze, recovery and multiplier change, including scheduled actions that have not executed yet.
- **Valuation history.** Every `ValuationPublished` event, charted over time and labeled by source.

### Offerings and windows

Each offering page shows its terms, the amount raised against the minimum and maximum, the number of subscribers and the outcome. Each Liquidity Window page shows its mode, dates, reserve price if one was set, the number of orders, the clearing price and the volume that settled.

### Documents

A list of every document anchored in the `DisclosureRegistry`, with its kind, hash, URI and timestamp. If a document is public, you can download it and the Explorer will hash it in your browser and compare the result with the on-chain value. If it is restricted to holders, the hash is still shown, so holders can verify the copy they receive in Portfolio.

### Pending actions

Every action queued in an `IssuerTimelock` across all companies: recoveries, multiplier changes, module changes and upgrades. Each entry shows when it becomes executable. This view exists so that no sensitive change can take effect without public notice.

## What the Explorer does not show

Names, contact details, verification documents and the details inside individual claims stay private. The Explorer shows that a wallet holds a verified identity, but not who the person is. See [What is public, what is private](../concepts/transparency.md).

## Verifying against Blockscout

The Explorer is a convenience. You should be able to reach the same conclusions without trusting it. Every value it displays links to the underlying transaction or contract on [robinhoodchain.blockscout.com](https://robinhoodchain.blockscout.com).

To check a figure yourself:

1. Open the share class page and copy the `PrecursorShareToken` address.
2. On Blockscout, read `totalSupply()` and `uiMultiplier()` from the contract. The share count is `totalSupply × uiMultiplier / 1e18`.
3. Filter the token's event logs for `SharesIssued`, `SharesRedeemed` and `RecoveryExecuted` to reconstruct every supply change.
4. Read `ruleEngine()` from the token, then the list of modules from the `RuleEngine`, and compare it with the Rules tab.
5. Confirm the token was deployed by the `PrecursorFactory` by finding its address in the factory's `CompanyDeployed` event.

The factory address for each network is published in the Explorer footer and returned by `GET /v1/contracts`. It is the single root of trust for every other address.

{% hint style="info" %}
For programmatic access to the same data, use `GET /v1/events` in the [API](api-and-sdk.md) or index the events listed in [On-chain events](../developers/events.md) directly.
{% endhint %}
