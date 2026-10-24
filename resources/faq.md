---
description: Answers to common questions from companies, investors and developers using Precursor.
---

# Frequently asked questions

## For companies

**What does "issuer-sponsored" mean?**
The company issues the token itself. It configures the rules, approves every sensitive action and keeps control of who can hold its shares. Precursor never creates a token for a company that hasn't onboarded. See [How stock tokens work](../overview/stock-tokens.md).

**How long does onboarding take?**
Most companies go from signed agreement to mainnet in under four weeks. The longest step is usually reconciling the cap table. [Tokenize your company](../guides/tokenize-your-company.md) covers each step.

**Do we have to tokenize every share class?**
No. Each class is its own token, and you can add classes whenever you're ready. Many companies start with common stock and one preferred series.

**Can we control who buys our shares?**
Yes. Every transfer is checked against your [transfer rules](../concepts/transfer-rules.md). You decide the regions, the eligibility requirements, the holding periods, the holder cap and the maximum ownership per holder, and whether transfers can only happen during windows you open.

**Will our shares trade around the clock?**
Only if you want them to. Most companies start with periodic [Liquidity Windows](../liquidity/liquidity-windows.md) and add [RFQ trading](../liquidity/rfq.md) or [permissioned pools](../liquidity/permissioned-pools.md) later.

**What if a holder loses access to their wallet?**
The holder re-verifies their identity and a recovery is scheduled. It needs two of three approvals from the issuer, the registrar and Precursor, and then waits out a 72 hour timelock. The pending recovery is public on-chain throughout. See [Roles and permissions](../architecture/roles-and-permissions.md).

**Can Precursor move our holders' tokens?**
No. The `OPERATOR` role submits transactions and sponsors gas, but it can't move holder tokens on its own. Recoveries need the issuer or the registrar as well, and the timelock gives everyone time to see them.

**How do splits work?**
Through the `uiMultiplier` defined by ERC-8056. Raw balances never change, so pools and integrations keep working. See [Corporate actions](../concepts/corporate-actions.md).

**What happens at an IPO or acquisition?**
At an IPO, tokens convert into the listed security or continue as a tokenized program. At an acquisition, the consideration is paid through the `DistributionVault` and the tokens are burned.

## For investors and employees

**Do I need crypto experience?**
No. Portfolio can create an embedded wallet for you, and Precursor sponsors gas, so you never need ETH. You'll need USDG to invest, and you receive USDG when you sell.

**Is my personal information on the blockchain?**
No. Only signed claims go on-chain, stating that a check passed and when it expires. Your name and documents stay off-chain. See [What is public, what is private](../concepts/transparency.md).

**Why can't I sell all of my shares?**
Some lots may be unvested or still inside a holding period. Portfolio shows your unlocked balance and the date each lot unlocks.

**What price will I get in a Liquidity Window?**
Everyone who fills gets the same clearing price. Your minimum price only decides whether your order is included.

**What happens if an offering doesn't reach its minimum?**
Nothing is issued and you can withdraw your full deposit with `refund`. Your funds stay in the on-chain escrow until you do.

**How do I know the documents I read are genuine?**
Every document's hash is anchored in the company's `DisclosureRegistry`. Portfolio and the Explorer show the hash next to each file.

**Do I get to vote?**
Yes, on proposals the company puts to holders of your class. Your vote weight is your balance at the snapshot block. See [Governance and voting](../concepts/governance.md).

**How do I receive dividends?**
When the company funds a distribution, you claim your share in Portfolio. See [Pay a distribution](../guides/pay-a-distribution.md) for how it works on the company's side.

**Can I use my own custodian?**
Yes. A custodial wallet can be registered to your identity like any other wallet.

## For developers

**Is there a sandbox?**
Yes. `https://api.sandbox.useprecursor.xyz/v1` runs against Robinhood Chain testnet (chain ID `46630`) with `sk_test_` keys. See [Authentication and environments](../developers/authentication.md).

**How do I check whether a transfer will succeed?**
Call `canTransfer(from, to, amount)` on the share token, or `POST /v1/transfers/check`. Both return a reason code. See [Reason codes](../developers/reason-codes.md).

**Do Precursor tokens work with standard ERC-20 tooling?**
Yes. They're ERC-20 tokens with 18 decimals. Transfers that fail a rule revert with a reason code, so check first. See [Integrating Precursor assets](../developers/integrating-precursor-assets.md).

**Where do I find contract addresses?**
In the [Explorer](https://explorer.useprecursor.xyz) and from `GET /v1/contracts`. Every company's contracts can also be traced from the `CompanyDeployed` events of `PrecursorFactory`.

**How do I read a price for a Precursor token?**
Read the company's `ValuationOracle`. It exposes `latestRoundData` for Chainlink-compatible consumers and `latestValuation` for the source and date. See [Reference Valuation](../concepts/reference-valuation.md).

**How are webhooks secured?**
Each delivery is signed in the `Precursor-Signature` header with HMAC-SHA256 using your endpoint secret. See [Webhooks](../developers/webhooks.md).

**How do I report a vulnerability?**
Email [security@useprecursor.xyz](mailto:security@useprecursor.xyz). See [Security](../architecture/security.md) for the bug bounty.
