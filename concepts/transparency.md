---
description: Which information Precursor records publicly on Robinhood Chain, which it keeps private, and how anyone can verify a company's record.
---

# What is public, what is private

Precursor is built on one principle: anyone can verify **what happened**, and only authorized parties can see **who it happened to**. Every issuance, transfer, corporate action and rule change is recorded on Robinhood Chain and settles to Ethereum. Personal data never goes on-chain.

## The split

| Public, on-chain | Private, access-controlled |
|---|---|
| Total supply of every share class and its holder count | Names, contact details and other personal information |
| Every mint, burn, transfer, freeze, recovery and pause, with its reference | Identity verification documents and the evidence behind each claim |
| Offering terms, amount raised and allocation results | The full cap table with identities, visible to the company and its registrar only |
| Liquidity Window orders, clearing prices and volumes | Financial statements, gated to verified holders unless the company publishes them |
| Corporate actions and the full multiplier history | The contents of anchored documents |
| Hashes of every anchored document | Individual claim values beyond topic and validity |
| The configuration of every rule module | |
| Proposal results and every vote cast, by wallet | |

## What a wallet reveals

A wallet address on Robinhood Chain is public, and so is its token balance. What the chain does not reveal is who controls the address. The `IdentityRegistry` links a wallet to an identity contract, and that contract holds signed claims, such as a two-letter country code, an eligibility tier or a screening timestamp. None of those values identifies a person on its own.

The link between an identity and a real person is held by the verifier that issued the claims and, where needed, in Precursor's access-controlled systems. The company and its registrar can see who holds their shares. The public cannot.

## Documents

Documents stay private, while their fingerprints are public. When a company anchors a document in the `DisclosureRegistry`, the contract records:

| Field | Example |
|---|---|
| `kind` | `SHARE_CLASS_TERMS`, `OFFERING_DOCUMENT`, `FINANCIAL_STATEMENT`, `COMPANY_UPDATE`, `BOARD_RESOLUTION`, `VALUATION_REPORT` |
| `hash` | The SHA-256 hash of the file |
| `uri` | Where verified holders can retrieve it |
| `timestamp` | When it was anchored |

Each anchor emits `DocumentAnchored(documentId, kind, hash, uri)`. A holder who downloads a document in [Portfolio](../products/portfolio.md) can hash it and check that the result matches the on-chain record, which proves the document has not changed since it was anchored. Companies can choose to make particular documents public, for example company updates.

## Verifying a company yourself

You do not need to trust Precursor's interface to check a company's record. Everything below can be confirmed from a Robinhood Chain node or the public block explorer at robinhoodchain.blockscout.com.

1. **Find the contracts.** Every company is deployed by the `PrecursorFactory`, which emits `CompanyDeployed` with the address of each contract. Addresses are also listed in the [Explorer](../products/explorer.md) and returned by `GET /v1/contracts`.
2. **Reconcile supply.** Sum `SharesIssued` minus `SharesRedeemed` for a share token. The result must equal `totalSupply()`.
3. **Check the rules.** Read the `RuleEngine` for enabled modules, and each module's configuration. Compare against `ModuleAdded`, `ModuleRemoved` and `ModuleConfigured` events.
4. **Check pending actions.** Read the `IssuerTimelock` for queued recoveries, multiplier changes and upgrades before they take effect.
5. **Check pricing.** Compare `WindowCleared` events from `LiquidityWindow` against `ValuationPublished` events from the `ValuationOracle`.
6. **Check documents.** Hash a document you received and compare it with the matching `DocumentAnchored` event.

## Invariants Precursor monitors

Precursor runs on-chain monitors that alert on any violation of these invariants. The same checks can be reproduced by anyone from public data.

- Total supply equals the sum of `SharesIssued` minus `SharesRedeemed`.
- No transfer settles to or from a wallet without a valid identity at that block.
- No transfer settles while the token is paused.
- No recovery executes before its timelock expires.
- The multiplier only changes through a scheduled `MultiplierScheduled` action.

{% hint style="success" %}
The aim is zero transfers that break a company's configured rules, and that is provable from on-chain events alone. If you find a counterexample, report it to security@useprecursor.xyz.
{% endhint %}
