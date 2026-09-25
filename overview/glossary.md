---
description: Definitions of the terms, contracts, roles and identifiers used throughout the Precursor documentation.
---

# Glossary

| Term | Definition |
|---|---|
| **Claim** | A signed attestation attached to an identity, with a topic, a value and an expiry. Claims are issued by verifiers listed in `TrustedVerifiers`. See [Identity and claims](../concepts/identity-and-claims.md). |
| **Claim topic** | The kind of fact a claim attests to: `IDENTITY_VERIFIED`, `INVESTOR_ELIGIBLE`, `REGION`, `EMPLOYEE_OF`, `SCREENED` or `VENUE`. |
| **Clearing price** | The single price at which a Liquidity Window settles every matched order. It is published as a Reference Valuation with source `WINDOW_CLEARING`. |
| **Company** | An issuer onboarded to Precursor. Identified by a `companyId` on-chain (`bytes32`) and a `cmp_` id in the API. |
| **Console** | The web application companies use to configure share classes, rules, offerings, Liquidity Windows and corporate actions. See [Precursor Console](../products/console.md). |
| **Delivery versus payment (DvP)** | Settlement in which tokens and stablecoins move in the same transaction, or neither moves. Every Precursor trade settles this way. See [Settlement](../liquidity/settlement.md). |
| **`DisclosureRegistry`** | The contract that anchors document hashes with their kind, URI and timestamp. |
| **Distribution** | A payment to holders, such as a dividend or acquisition proceeds, paid through the `DistributionVault`. |
| **`DistributionVault`** | The contract that snapshots balances at a record block and lets holders claim their pro-rata share of a distribution. |
| **EIP-712** | The Ethereum standard for signing typed structured data. Precursor's signing domain uses the name `Precursor` and version `1`. |
| **Embedded wallet** | A wallet created for an investor or employee during onboarding in Portfolio, linked to their identity automatically. |
| **ERC-3643** | A permissioned token standard that checks holder identity and transfer rules before a transfer settles, while staying ERC-20 compatible. |
| **ERC-8056** | The Scaled UI Amount Extension. It exposes `uiMultiplier()` so corporate actions can change shares per token without rebasing balances. |
| **Explorer** | Precursor's public view of every tokenized company, at explorer.useprecursor.xyz. See [Precursor Explorer](../products/explorer.md). |
| **Holder** | A wallet with a registered identity that holds, or is eligible to hold, a share token. Identified by an `hld_` id in the API. |
| **Holder cap** | A maximum number of distinct identities that may hold a share class, enforced by `HolderCapModule`. |
| **Holding period** | A lockup applied to each lot from the time it is acquired, enforced by `HoldingPeriodModule`. |
| **Identity** | An on-chain identity contract representing one person or entity, linked to one or more wallets and holding claims. It contains no personal data. |
| **`IdentityRegistry`** | The per-company contract that maps wallets to identities and checks whether each has valid claims. |
| **Issuer-sponsored token** | A token issued by the company whose shares it represents. Every Precursor token is issuer-sponsored. |
| **`IssuerTimelock`** | The timelocked multi-sig, with the company as a signer, through which sensitive configuration changes pass. |
| **Liquidity Window** | A time-boxed, company-run auction or tender in which eligible sellers and buyers trade at a single clearing price. See [Liquidity Windows](../liquidity/liquidity-windows.md). |
| **`LiquidityWindow`** | The contract that runs Liquidity Windows, locks orders and settles them atomically on `clear`. |
| **Lot** | A quantity of tokens acquired in one event, with its own acquisition time and source. Lots are consumed first in, first out on transfer. |
| **Module** | A contract plugged into a `RuleEngine` that enforces one transfer rule, such as `RegionModule` or `ROFRModule`. |
| **Multiplier** | The value returned by `uiMultiplier()`, the number of shares each token represents, scaled by `1e18`. |
| **`OfferingVault`** | The contract that runs primary offerings, escrows subscriptions and mints shares at close or refunds investors. |
| **`OPERATOR`** | The role held by Precursor's relayers. It submits sponsored transactions and co-signs recoveries, but cannot move holder tokens alone. |
| **`PAUSER`** | The role, held by the company and the registrar, that can pause and unpause a share token immediately. |
| **Permissioned pool** | An AMM pool on Robinhood Chain whose swaps and liquidity changes are checked against a share token's rules by `PoolGate`. |
| **`PoolGate`** | The hook contract that enforces `canTransfer` on permissioned pools. |
| **Portfolio** | The web application investors and employees use to onboard, hold, trade, vote and read company documents. See [Precursor Portfolio](../products/portfolio.md). |
| **`PrecursorFactory`** | The single root contract that deploys a company's full contract set and emits `CompanyDeployed`. |
| **Precursor Share Token** | The ERC-20, ERC-3643 and ERC-8056 token that represents one share class of one company. See [Share tokens](../concepts/share-tokens.md). |
| **Reason code** | The `bytes32` string `canTransfer` returns to explain why a transfer would fail, such as `HOLDING_PERIOD_ACTIVE`. See [Reason codes](../developers/reason-codes.md). |
| **Record block** | The block at which balances are snapshotted for a distribution. |
| **Recovery** | Moving a holder's tokens from a lost wallet to a replacement after re-verification, with two-of-three approval and a timelock. |
| **Reference Valuation** | A dated, sourced price per share for a share class, with source `PRICED_ROUND`, `INDEPENDENT_VALUATION` or `WINDOW_CLEARING`. See [Reference Valuation](../concepts/reference-valuation.md). |
| **`REGISTRAR`** | The role held by the registrar. It mints and burns against cap table records, executes conversions and approves recoveries. |
| **Registrar** | The party that maintains the authoritative cap table for a company, either the company itself or an administrator it appoints. |
| **RFQ** | Request for quote. Bilateral trading in which a maker signs a quote and a taker fills it through `RFQSettlement`. See [RFQ trading](../liquidity/rfq.md). |
| **`RFQSettlement`** | The contract that verifies a signed quote, runs transfer checks and settles the trade atomically. |
| **ROFR** | Right of first refusal. The company's right, enforced by `ROFRModule`, to buy shares before they transfer to someone else. |
| **`RuleEngine`** | The per-share-class contract that runs every enabled module when tokens move. |
| **Share class** | One class of a company's equity, such as Common or Series B Preferred. Each share class is one share token and has a `cls_` id in the API. |
| **Snapshot block** | The block at which voting power is measured for a proposal. |
| **`TrustedVerifiers`** | The per-company contract listing which verifier keys may sign which claim topics. |
| **Unlocked balance** | The part of a holder's balance that is not frozen or committed to an open order, returned by `unlockedBalanceOf`. |
| **USDG** | The stablecoin Precursor uses by default for offerings, trades and distributions on Robinhood Chain. |
| **`ValuationOracle`** | The per-share-class contract that publishes Reference Valuations, readable through a Chainlink-compatible interface. |
| **Verifier** | An identity provider whose key is trusted to sign claims on particular topics. |
| **Vesting** | The schedule by which an employee's shares become transferable, enforced by `EmployeeVestingModule`. |
