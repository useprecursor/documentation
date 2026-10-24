---
description: How Precursor charges companies, investors and integrators, and what each fee covers.
---

# Pricing

Precursor charges for work it does: tokenizing a company, running its share register and settling trades. Fees are set out in a company's agreement, deducted transparently on-chain where they apply, and never hidden in a spread. For a quote, contact [hello@useprecursor.xyz](mailto:hello@useprecursor.xyz).

## Fees at a glance

| Fee | Who pays | How it is charged | What it covers |
|---|---|---|---|
| Tokenization setup | Company | One-time, tiered by company stage | Verification, cap table import and reconciliation, rule configuration, testnet rehearsal, mainnet deployment |
| Platform administration | Company | Annual, in basis points of tokenized value, with a floor | Registrar tooling, identity orchestration, corporate actions, governance, disclosures, Explorer listing, support |
| Offerings | Company | Percentage of capital raised, deducted at close | The `OfferingVault`, investor onboarding and settlement |
| Liquidity Windows and RFQ | Buyer and seller | Percentage of notional traded, split between the two sides | Order handling, clearing and atomic settlement |
| API and partner access | Venues, wallets, integrators | Usage-based | Indexed data, transfer checks at volume, webhooks |

## How fees are collected

**Setup and administration** are invoiced to the company under its agreement.

**Offering fees** are deducted by the `OfferingVault` when an offering closes successfully. The amount sent to the treasury and the fee appear in the same transaction, and the `OfferingClosed` event records the total raised. A failed offering costs nothing: investors are refunded in full and no fee is taken.

**Liquidity Window and RFQ fees** are deducted at settlement. Buyers and sellers see the fee before they confirm an order, and the settled amounts in the transaction already reflect it. An order that doesn't fill pays nothing.

**API access** for issuers on their own companies is included in platform administration. Usage pricing applies to venues and integrators that read or check transfers across many companies.

## What is free

* **Gas for investors and employees.** Precursor sponsors transaction fees for holders in Portfolio, so nobody needs to hold ETH to subscribe, vote, claim or sell.
* **Onboarding for holders.** Identity verification for invited holders and investors is included.
* **The Explorer.** The public record is open to anyone, with no account.
* **The sandbox.** Testing on Robinhood Chain testnet through `https://api.sandbox.useprecursor.xyz/v1` is free.

## Changes to pricing

Pricing for an existing company only changes under the terms of its agreement, with notice. Fee parameters on settlement contracts are visible on-chain, and any change to them goes through the same timelock as other sensitive actions, so you'll see it before it takes effect.
