---
description: The problem Precursor solves for private companies, their shareholders and their investors, and why issuer-sponsored stock tokens on Robinhood Chain are the right foundation.
---

# Why Precursor

Private companies now raise more capital and stay private for longer than at any point in history. The infrastructure behind their equity has not kept up. Ownership lives in spreadsheets and cap-table software, transfers take weeks of manual review, and most secondary sales happen bilaterally at prices no one else can see.

Precursor replaces that process with a programmable one. A company issues its shares as stock tokens on Robinhood Chain, configures the rules those tokens must follow, and runs issuance, transfers, corporate actions and liquidity as on-chain transactions. The company stays in control throughout, and the public record shows exactly what happened.

## The problem, by participant

### Companies

A growing private company has three recurring problems with its equity.

**The cap table is fragmented.** Records are spread across cap-table software, board consents, investment-vehicle statements and spreadsheets maintained by different people. Reconciling them before a financing or a tender offer is slow and error-prone.

**Liquidity is ad hoc.** Employees and early investors want to sell some of their shares long before an exit. Tender offers are the usual answer, and they are expensive to run, infrequent, and hard to price. Between tenders, shareholders look for buyers on their own, and the company finds out about transfers after the fact.

**Others create exposure without consent.** Third-party products already track the price of well-known private companies. The company has no say in who holds that exposure, what they are told, or what price signals the product sends to the market.

### Shareholders

For an employee with vested shares, there is rarely a reliable way to sell. A secondary sale can take weeks, needs company approval at several points, and clears at a price that is hard to benchmark. There is also no verifiable view of the cap table, of past transactions, or of what other holders paid.

### Investors

Access to pre-IPO companies is mostly a matter of relationships, and minimum tickets are large. Exposure through intermediaries adds fees and adds a layer between the investor and the information they need.

## How Precursor addresses it

| Problem | What Precursor does |
|---|---|
| Fragmented cap table | Each share class is a [share token](../concepts/share-tokens.md). Token balances are the cap table, and the registrar reconciles them once at onboarding. |
| Manual transfer review | [Transfer rules](../concepts/transfer-rules.md) run inside the token. A transfer that breaks a rule never settles, and the reason is returned before anyone signs. |
| Ad hoc liquidity | [Liquidity Windows](../liquidity/liquidity-windows.md), [RFQ trading](../liquidity/rfq.md) and [permissioned pools](../liquidity/permissioned-pools.md) give the company controlled ways to open liquidity, on its schedule. |
| Opaque pricing | Clearing prices are public and feed the [Reference Valuation](../concepts/reference-valuation.md), which always shows its source and date. |
| Third-party exposure | Every Precursor token is issued by the company it represents. Nothing is tokenized without the issuer. |
| No verifiable record | The [Explorer](../products/explorer.md) shows every issuance, transfer and corporate action. Identities stay private, as described in [What is public, what is private](../concepts/transparency.md). |

## Issuer-sponsored by design

The defining choice in Precursor is that the company is always the issuer. That choice shapes the rest of the system:

- **The token is the share.** A Precursor Share Token represents a share in a specific class of the company, with the economic, information and voting rights that class carries. It is not a claim on a third party that happens to track a price.
- **The company sets the rules.** Regions, eligibility, holding periods, holder caps, vesting and right of first refusal are configured by the company in the [Console](../products/console.md) and enforced by contracts it controls.
- **The company decides when liquidity happens.** Liquidity is opened in windows the company schedules, or through venues it explicitly approves.
- **Sensitive actions include the company.** Upgrades, recoveries and corporate actions pass through a timelocked multi-sig on which the issuer is a signer. Precursor cannot act alone on a holder's tokens.

## Why Robinhood Chain

Precursor runs on Robinhood Chain, an Arbitrum-based Ethereum L2 built for tokenized real-world assets. Four properties matter:

1. **It is where stock tokens already live.** Wallets, DEXs and oracles on the chain already understand the stock token conventions Precursor follows, so Precursor tokens work with existing infrastructure.
2. **The primitives are in place.** Chainlink feeds, institutional custody through BitGo, and the USDG stablecoin are all available on the chain.
3. **It settles to Ethereum.** The chain inherits Ethereum security and is fully EVM compatible, so standard tooling and audit practice apply.
4. **Fees are low and confirmation is fast.** That makes it practical to run a full rule check on every single transfer.

Robinhood Chain is permissionless, so Precursor enforces every rule at the token contract level rather than relying on the network. Network details are on the [Robinhood Chain](../architecture/robinhood-chain.md) page.

## Who uses Precursor

- **Companies** from Series B to pre-IPO with active secondary demand, large employee option pools, and boards that prefer to manage liquidity rather than suppress it.
- **Funds and vehicle managers** moving existing holdings on-chain.
- **Venues and wallets on Robinhood Chain** that want to list private-company assets that come with issuer participation and enforceable rules.

To see how Precursor tokens compare with other ways of putting equity on-chain, continue to [How stock tokens work](stock-tokens.md).
