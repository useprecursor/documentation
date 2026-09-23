---
description: The three ways equity is represented on-chain, the stock token conventions Robinhood Chain has standardized, and where Precursor Share Tokens fit.
---

# How stock tokens work

A stock token is an on-chain token whose value is tied to a company's equity. The term covers very different instruments, and the difference that matters most is what the holder actually owns. This page sets out the three common models, the conventions Robinhood Chain uses for stock tokens, and how Precursor builds on them.

## Three models

| Model | What the holder owns | Company involved | Holder rights |
|---|---|---|---|
| **Third-party token** | A claim on the token's issuer, whose value tracks the price of a stock | No | Price exposure only. No voting, information or direct economic rights in the company. |
| **Vehicle token** | An interest in a vehicle that holds real shares | Sometimes | Indirect. Economics pass through the vehicle, and voting usually sits with its manager. |
| **Issuer-sponsored token** | The share itself, issued by the company | Yes, always | The rights of the share class: economic, information and voting, as the class defines them. |

### Third-party tokens

A third party buys or hedges exposure to a stock and issues a token that tracks its price. The token can trade freely, but it is a claim on the third party, not on the company. For private companies this model has a further problem: the company did not choose to participate, has no control over who holds the exposure, and may disagree with how the product is presented.

### Vehicle tokens

A vehicle, typically a fund or a special-purpose entity, buys shares in a company and issues tokens representing interests in itself. The holder gets closer to the underlying economics, but there is still a layer in between. Fees stack, information flows through the vehicle manager, and the company often only learns about transfers indirectly.

### Issuer-sponsored tokens

The company issues the token directly as a representation of its own shares. The token balance is the ownership record. The company configures which wallets can hold the token and under what conditions, and corporate actions such as dividends, splits and votes run through the token itself.

**Precursor only issues issuer-sponsored tokens.** No Precursor token exists without the company it represents being the issuer.

## Conventions on Robinhood Chain

Robinhood Stock Tokens are the reference stock token implementation on Robinhood Chain. Their design has established conventions that wallets, DEXs, lending markets and oracles on the chain already support.

**One ERC-20 per asset.** Each stock is its own ERC-20 contract with 18 decimals, mapped to a single underlying security.

**Corporate actions through a multiplier.** Splits and dividends do not rebase balances. Instead, each token implements ERC-8056 (Scaled UI Amount Extension) and exposes `uiMultiplier()`. The multiplier changes the number of shares each token represents, while raw balances stay exactly the same. AMM pools and lending positions keep working through a corporate action because nothing they hold has moved.

**Per-asset price feeds.** Each stock token has its own Chainlink feed. The feed incorporates the multiplier and respects a pause flag on the token contract, holding the last good value while the token is paused.

**A restricted primary market.** New tokens are created only by authorized parties. End users acquire tokens on secondary venues.

**A trading-status check.** Applications check whether an asset is currently tradable before they execute.

## How Precursor tokens relate

A [Precursor Share Token](../concepts/share-tokens.md) follows every one of those conventions, so an integration built for Robinhood Stock Tokens reads a Precursor token the same way.

| Convention | Robinhood Stock Tokens | Precursor Share Tokens |
|---|---|---|
| Token standard | ERC-20, 18 decimals | ERC-20, 18 decimals, ERC-3643-compatible |
| Granularity | One contract per stock | One contract per company share class |
| Corporate actions | ERC-8056 `uiMultiplier()` | ERC-8056 `uiMultiplier()`, scheduled through a timelock |
| Price data | Per-asset Chainlink feed | Per-class `ValuationOracle`, Chainlink `AggregatorV3` compatible |
| Pause | On-contract pause flag | `pause(reason)` with the reason published on-chain |
| Primary market | Authorized participants | Company offerings through `OfferingVault` |
| Tradability check | Trading-status lookup | `canTransfer(from, to, amount)` with a reason code |

Precursor then adds what a company's own shares need and a third-party token does not.

**Identity-gated transfers.** Every holder wallet is linked to an on-chain identity carrying signed claims. A transfer settles only if both parties are verified and every rule the company has enabled passes. See [Identity and claims](../concepts/identity-and-claims.md) and [Transfer rules](../concepts/transfer-rules.md).

**Issuer governance.** The company controls configuration through a timelocked multi-sig, and token holders can vote on-chain. See [Governance and voting](../concepts/governance.md).

**Dated valuation rather than a live price.** Private companies have no continuous market price. Precursor publishes a [Reference Valuation](../concepts/reference-valuation.md) that always carries its source and date, and never presents it as a live market price.

**Company-controlled liquidity.** Secondary trading happens through [Liquidity Windows](../liquidity/liquidity-windows.md), [RFQ trading](../liquidity/rfq.md) and [permissioned pools](../liquidity/permissioned-pools.md), each of which the company enables explicitly.

{% hint style="info" %}
Because a Precursor Share Token is permissioned, a plain ERC-20 `transfer` to an unverified wallet reverts. Integrations should call `canTransfer` first. The [Integrating Precursor assets](../developers/integrating-precursor-assets.md) guide covers this in detail.
{% endhint %}
