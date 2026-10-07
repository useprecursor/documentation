---
description: How the on-chain contracts and the Precursor platform services fit together, and how a single transfer and a single offering close move through the system.
---

# System overview

Precursor has two layers. The **on-chain layer** is a set of contracts on Robinhood Chain that holds every balance, enforces every transfer rule and records every corporate action. The **platform layer** is a set of services run by Precursor that make those contracts usable: onboarding, transaction submission, indexing, the Explorer and webhooks.

The split is deliberate. Anything that decides who owns what lives on-chain. The platform can prepare, submit and display transactions, but it cannot move a holder's tokens on its own or override a rule the company has configured.

```
┌───────────────────────────── Precursor platform ─────────────────────────────┐
│                                                                              │
│  Console      Portfolio      API / SDK            Explorer      Webhooks     │
│     │             │              │                    ▲             ▲        │
│     └─────────────┴──────┬───────┘                    │             │        │
│                          ▼                            │             │        │
│   Issuance · Rules · Offerings · Identity orchestration · Corporate actions  │
│                          │                            │             │        │
│                          ▼                            │             │        │
│              Transaction queue ──► Signing (HSM / MPC relayers)    │        │
│                          │                            │             │        │
│                          │               Indexer (archive RPC, failover)     │
└──────────────────────────┼────────────────────────────▲─────────────────────┘
                           │ signed transactions        │ blocks and events
                           ▼                            │
┌──────────────────────── Robinhood Chain (4663) ─────────────────────────────┐
│  PrecursorFactory ─► per company: IdentityRegistry, TrustedVerifiers,        │
│  per share class: PrecursorShareToken, RuleEngine + modules                  │
│  OfferingVault · LiquidityWindow · RFQSettlement · PoolGate                  │
│  DistributionVault · Governance · DisclosureRegistry · ValuationOracle       │
│  IssuerTimelock                                                              │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    ▼
                            Ethereum (settlement)
```

## The on-chain layer

`PrecursorFactory` is the single root address. When a company is deployed, the factory creates its full contract set in one transaction and emits `CompanyDeployed`. From that event alone you can find every contract that belongs to the company.

Each company gets one `IdentityRegistry` and one `TrustedVerifiers` contract. Each share class gets its own `PrecursorShareToken` and its own `RuleEngine`, with the modules the company has enabled. The shared operational contracts handle the work that happens around the token: offerings in `OfferingVault`, liquidity in `LiquidityWindow`, `RFQSettlement` and `PoolGate`, payouts in `DistributionVault`, votes in `Governance`, document hashes in `DisclosureRegistry`, and valuations in `ValuationOracle`. Sensitive changes pass through the company's `IssuerTimelock`.

The [Contract reference](../developers/contracts.md) documents each contract's interface, and [On-chain events](../developers/events.md) lists everything they emit.

## The platform layer

| Service | Responsibility |
|---|---|
| **Issuance** | Turns cap table imports and Console actions into factory deployments, mints and conversions. Reconciles on-chain balances against the registrar's records. |
| **Rules** | Builds module configurations from the Console, runs `canTransfer` simulations before a user signs, and explains failures in plain language. |
| **Offerings** | Creates and closes offerings and Liquidity Windows, and tracks subscriptions and orders. |
| **Identity orchestration** | Routes investors to verification providers, receives their results by webhook, and writes the resulting claims to the `IdentityRegistry` through a claim-signing service. Schedules re-screening and revokes claims that lapse. |
| **Corporate actions** | Schedules multiplier changes, funds distributions, opens votes and anchors documents. |
| **Indexer** | Reads every block from an archive RPC endpoint and stores decoded events in Postgres. It runs against Alchemy as the primary provider and fails over to Chainstack, comparing block hashes between providers to detect divergence. |
| **Explorer** | Serves the public record at [explorer.useprecursor.xyz](https://explorer.useprecursor.xyz), built entirely from indexed chain data. |
| **Webhooks** | Delivers signed event notifications to the endpoints that companies and partners register. See [Webhooks](../developers/webhooks.md). |

### Signing

Precursor never holds a holder's keys. Holders sign with their own wallet, either an external wallet or an embedded wallet created during onboarding.

Precursor does hold the `OPERATOR` keys that submit transactions and sponsor gas. Each company has its own operator signer, held in hardware security modules or split across an MPC cluster, so no single machine ever has a complete key. The operator role is scoped narrowly. It can relay and pay for transactions, and co-sign recoveries, but it cannot move a holder's tokens alone. [Roles and permissions](roles-and-permissions.md) covers exactly what each signer can do.

### Transaction queue

Every write goes through an idempotent queue. Each job carries a deterministic key, so a retried API call or a restarted worker never submits the same action twice. The queue assigns nonces per signer, estimates gas, and submits to the Robinhood Chain sequencer.

Workers subscribe to the sequencer feed, which publishes ordered transactions before they are batched to Ethereum. That gives the platform a soft confirmation in well under a second, which is what the Console and Portfolio show as "confirmed". The queue keeps tracking each transaction until its batch is posted to Ethereum and records both milestones. If the primary feed drops, workers switch to the delayed feed, which runs about 500 ms behind on separate capacity. [Robinhood Chain](robinhood-chain.md) explains the two confirmation levels.

## Data flow: one transfer

A holder sends 1,000 shares of a company's Common class to another verified holder through Portfolio.

1. Portfolio calls the rules service, which runs `canTransfer(from, to, 1000e18)` against the latest block. The check returns `OK`, or a reason code such as `HOLDING_PERIOD_ACTIVE` that Portfolio shows before the holder signs anything.
2. The holder signs the transfer in their wallet. Gas is sponsored, so they don't need ETH.
3. The transaction reaches the sequencer. Inside `transfer`, the share token asks the `RuleEngine` to run every enabled module again. On-chain is the only check that counts. The earlier simulation exists only to give a clear message up front.
4. If any module fails, the transaction reverts and no balance changes. If all pass, the token updates balances, consumes the sender's oldest unlocked lots first, and emits `Transfer`.
5. The indexer picks up the event from the sequencer feed. The Explorer shows the transfer with both wallet addresses and no identities, and webhook endpoints receive `transfer.settled`.

## Data flow: one offering close

An offering has passed its `closesAt` time with subscriptions above its minimum raise.

1. The company, or the offerings service on its behalf, calls `OfferingVault.close(offeringId)`.
2. The vault checks the result against the configured minimum and maximum. If the offering is oversubscribed, it allocates pro-rata and sets aside the excess for refund.
3. In the same transaction, the vault releases USDG to the company treasury minus the platform fee, mints shares to each investor as a new lot, and emits `SharesIssued` for each mint and `OfferingClosed` for the offering.
4. The issuance service matches each `SharesIssued` event to the registrar's records, so the cap table and the chain agree.
5. Webhook endpoints receive `offering.closed`, Portfolio shows investors their new position, and the offering's final figures appear in the Explorer.

If the minimum was not met, the vault emits `OfferingFailed` instead and every investor can call `refund`. See [Primary offerings](../liquidity/offerings.md) for the full lifecycle.
