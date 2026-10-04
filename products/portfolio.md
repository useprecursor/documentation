---
description: Portfolio is where investors and employees verify their identity, hold Precursor Share Tokens, join offerings and Liquidity Windows, vote and read company documents.
---

# Precursor Portfolio

Portfolio is the holder's side of Precursor. Investors, employees and early shareholders use it to see what they own, take part in offerings and liquidity events, vote, and read what the company has shared with them. It runs at [app.useprecursor.xyz/portfolio](https://app.useprecursor.xyz/portfolio).

## Getting started

A holder arrives in Portfolio in one of two ways: through an invitation from a company that has tokenized its cap table, or through the link to an offering.

1. **Verify your identity.** Portfolio walks you through identity verification with one of Precursor's verification providers. Most people are verified the same day. The result is written to the company's `IdentityRegistry` as signed [claims](../concepts/identity-and-claims.md). Your documents and personal details are never written on-chain.
2. **Set up a wallet.** You can connect any EVM wallet that supports Robinhood Chain, or let Portfolio create an embedded wallet for you. Embedded wallets are secured by passkeys and can be exported at any time.
3. **Receive your shares.** If you already hold shares in the company, they appear as tokens once the company completes its migration.

Precursor sponsors gas for every holder action in Portfolio, so you never need to hold ETH.

## Positions

The main view lists every share class you hold, across every company. For each position you see:

- your balance in shares, adjusted by the current `uiMultiplier`
- the latest [Reference Valuation](../concepts/reference-valuation.md), with its source and date
- your lots, with acquisition date and whether each is locked by a holding period, vesting schedule or freeze
- your unlocked balance, which is the amount you could transfer today if every other rule allows it

Unvested employee shares are shown, but marked as locked until each tranche vests.

## Offerings

When a company opens an offering you are eligible for, it appears here. You can read the offering documents, see their hashes in the `DisclosureRegistry`, and subscribe by depositing USDG. Your subscription stays in escrow on-chain until the offering closes. If the offering does not reach its minimum, you can withdraw your full deposit. See [Invest in an offering](../guides/invest-in-an-offering.md).

## Selling

Portfolio supports every liquidity route the company has enabled:

- **Liquidity Windows.** Place a sell order during an open window. Your tokens lock until the window clears, and USDG arrives in the same transaction as the sale. See [Sell in a Liquidity Window](../guides/sell-in-a-liquidity-window.md).
- **RFQ.** Request or accept quotes from other eligible holders where the company allows it.
- **Permissioned pools.** Swap through an approved pool if the company has opted in.

Before you sign anything, Portfolio runs the same `canTransfer` check the contract will run. If a rule would block the transfer, you see the reason in plain language, such as "Your shares from this lot are in a holding period until 14 March", instead of a failed transaction.

## Distributions

When a company pays a dividend or other distribution, your share is calculated from your balance at the record block. Portfolio shows unclaimed distributions and lets you claim them in one step.

## Voting

Open proposals appear with their options, the snapshot block, the closing date and whether the result is binding or advisory. Your voting weight is your balance at the snapshot, multiplied by the share class multiplier. You can vote directly or delegate your vote to another holder.

## Documents and updates

Companies share financial statements, company updates and other documents with verified holders. Each document shows its hash and the transaction that anchored it, so you can confirm it has not changed since it was published.

## Account security

- Embedded wallets use passkeys. You can add more than one device.
- If you lose access to a wallet, the company can restore your tokens to a new wallet through [recovery](../concepts/share-tokens.md). Recovery requires re-verifying your identity, approval from two of three parties, and a 72 hour delay that is visible on-chain.
- Claims expire. Portfolio reminds you before a claim such as `SCREENED` needs to be refreshed, so your shares stay transferable.

{% hint style="info" %}
Portfolio is a view over on-chain state. Your tokens are held in your wallet, not by Precursor.
{% endhint %}
