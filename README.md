---
description: Precursor lets private companies issue stock tokens on Robinhood Chain, with every issuance, transfer and corporate action settled transparently on-chain.
---

# Precursor at a glance

Precursor is equity infrastructure for private companies. A company uses it to issue its shares as stock tokens on Robinhood Chain, run its cap table on-chain, raise capital from eligible investors, and give shareholders liquidity on terms the company sets.

Every Precursor token is **issuer-sponsored**. The company it represents issues it, configures the rules it follows, and stays in control of who can hold it. Every issuance, transfer, corporate action and rule decision is recorded on Robinhood Chain and settles to Ethereum. Anyone can verify what happened, while the identities of the people involved stay private.

## What you can do with Precursor

| | |
|---|---|
| **Tokenize** | Issue each share class as a Precursor Share Token, an ERC-20 that follows Robinhood Chain's stock token conventions. |
| **Enforce rules by construction** | Every transfer is checked on-chain against holder identities and the rules the company configures: regions, eligibility, holding periods, holder caps, vesting and right of first refusal. |
| **Raise** | Run primary offerings from an on-chain vault. Funds are escrowed in USDG, and shares are minted to investors in the same transaction that releases the funds. |
| **Administer** | Run splits, dividends, conversions, shareholder votes and disclosures as on-chain transactions. |
| **Provide liquidity** | Open time-boxed Liquidity Windows, enable RFQ trading between eligible holders, or opt into permissioned pools. |
| **Prove it** | The Precursor Explorer shows every company's supply, holder count, transfers and corporate actions, all verifiable on-chain. |

## How it fits together

```
 Company ─► Console ──┐
 Investor ─► Portfolio ┼─► Precursor platform ─► Robinhood Chain (4663) ─► Ethereum
 Developer ─► API/SDK ─┘        │                     │
                                └──── Explorer ◄──────┘
```

The company works in the [Console](products/console.md). Investors and employees hold, buy and sell in [Portfolio](products/portfolio.md). Developers and venues integrate through the [API and SDK](products/api-and-sdk.md). The [Explorer](products/explorer.md) gives anyone a view of the public record.

## Where to start

* **New to Precursor.** Read [Why Precursor](overview/why-precursor.md) and [How stock tokens work](overview/stock-tokens.md), then the core concept of [share tokens](concepts/share-tokens.md).
* **Tokenizing a company.** Follow [Tokenize your company](guides/tokenize-your-company.md), then [Run an offering](guides/run-an-offering.md) or [Open a Liquidity Window](guides/open-a-liquidity-window.md).
* **Investing or holding shares.** See [Invest in an offering](guides/invest-in-an-offering.md) and [Sell in a Liquidity Window](guides/sell-in-a-liquidity-window.md).
* **Building on Precursor.** Begin with the [Quickstart](developers/quickstart.md), then the [Contract reference](developers/contracts.md) and [Integrating Precursor assets](developers/integrating-precursor-assets.md).

{% hint style="info" %}
Precursor is an independent company built on Robinhood Chain. Contract addresses for every network are listed in the [Explorer](https://explorer.useprecursor.xyz) and returned by `GET /v1/contracts`. Treat any address published anywhere else with caution.
{% endhint %}
