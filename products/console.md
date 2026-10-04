---
description: The Console is where a company configures its share classes, transfer rules, offerings, liquidity and corporate actions, with every sensitive change routed through multi-sig approval.
---

# Precursor Console

The Console is the company's workspace on Precursor. Finance, legal operations and the board use it to manage everything that happens to the company's tokenized equity. It runs at [app.useprecursor.xyz/console](https://app.useprecursor.xyz/console).

Every action in the Console maps to an on-chain transaction or a signed record. Nothing is changed silently. When you edit a rule, schedule a split or open a Liquidity Window, the Console prepares the transaction, collects the approvals it needs, and submits it to Robinhood Chain. The result appears in the [Explorer](explorer.md) as soon as it is sequenced.

## Sections

### Overview

The landing view summarizes the company's on-chain state: total supply per share class, holder count, the current [Reference Valuation](../concepts/reference-valuation.md), open offerings and windows, pending approvals, and upcoming timelocked actions. A feed lists recent events with links to the transactions behind them.

### Share classes

Each share class is one [Precursor Share Token](../concepts/share-tokens.md). This section lists every class with its token address, supply, holder count, current `uiMultiplier` and status (active or paused). From here you can create a new class, view its document root, schedule a multiplier change, or pause and unpause the token.

### Holders

A searchable view of the cap table. For each holder you see their verified identity, linked wallets, balance per class, lots with their acquisition dates and lock status, and the claims recorded against their identity. You can invite new holders, start a [recovery](../concepts/share-tokens.md) for a lost wallet, or freeze a specific amount. Personal details are visible only to Console members with the right permissions and never leave the platform's access-controlled storage.

### Rules

Configure the [transfer rules](../concepts/transfer-rules.md) for each share class. You enable or disable modules, set their parameters (allowed regions, holding periods, holder cap, ownership cap, transfer windows, ROFR terms, vesting schedules) and preview the effect before submitting. A built-in checker lets you test any proposed transfer between two holders and see the exact reason code it would return.

### Offerings

Create and manage [primary offerings](../liquidity/offerings.md). You set the price per share, minimum and maximum raise, minimum ticket, dates, accepted stablecoin and which claims investors must hold. While an offering is open, the view tracks subscriptions in real time against the minimum and maximum.

### Liquidity

Set up [Liquidity Windows](../liquidity/liquidity-windows.md), enable [RFQ trading](../liquidity/rfq.md) for a class, and opt into [permissioned pools](../liquidity/permissioned-pools.md). For each window you choose the mode (batch auction or tender), the claims sellers and buyers must hold, reserve price, volume cap, per-seller cap and any right of first refusal holder. Past windows show their clearing price, volume and participant counts.

### Distributions

Fund dividends and other payouts through the `DistributionVault`. You choose the asset, the total amount, the record block and an optional claim deadline. The Console shows claim progress per distribution. See [Pay a distribution](../guides/pay-a-distribution.md).

### Governance

Create proposals for token-weighted votes, set the snapshot block, voting period, quorum and options, and mark each proposal as binding or advisory. Results finalize on-chain. See [Governance and voting](../concepts/governance.md).

### Documents

Upload share class terms, offering documents, financial statements, board resolutions, valuation reports and company updates. The Console anchors each document's hash in the `DisclosureRegistry` and lets you choose whether its contents are public or visible only to verified holders.

### Approvals

Sensitive actions are controlled by the company's `IssuerTimelock`, a multi-sig with a delay. This section shows every pending action, who has signed, how many signatures are still needed, and when the timelock expires. Signers approve from here with their own wallet or hardware key, and the **Signers** tab is where the company adds or removes signers and changes the approval threshold.

| Action | Default delay |
| --- | --- |
| Pause or unpause | None |
| Add or remove a rule module | 24 hours |
| Multiplier change | 48 hours |
| Wallet recovery | 72 hours |
| Token upgrade | 7 days |

### Developers

Create and roll API keys, register webhook endpoints, view request logs and switch between live and sandbox data. The [API and SDK](api-and-sdk.md) page covers what you can build.

## Team roles

Console membership is separate from on-chain roles. A member's Console role decides what they can see and prepare, while the on-chain role held by their wallet decides what they can sign.

| Console role | Can |
| --- | --- |
| Owner | Everything, including managing members and signers |
| Admin | Configure share classes, rules, offerings, liquidity, distributions and governance; prepare actions for approval |
| Signer | Approve pending actions in the `IssuerTimelock` with their wallet |
| Cap table manager | View and manage holders, lots and documents |
| Viewer | Read-only access to everything except personal details |
| Developer | Manage API keys, webhooks and logs |

Every member action is recorded in an audit log, which the Owner can export at any time.

{% hint style="info" %}
Anything a member prepares in the Console still needs the approvals configured on-chain. A compromised Console account cannot move tokens or change rules on its own.
{% endhint %}
