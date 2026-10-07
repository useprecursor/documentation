---
description: Why Precursor is built on Robinhood Chain, the network parameters for mainnet and testnet, how confirmation and finality work, and what happens if the sequencer is unavailable.
---

# Robinhood Chain

Precursor runs on Robinhood Chain, an Ethereum layer 2 built on Arbitrum technology and designed for tokenized real-world assets. Precursor is an independent company. It builds on Robinhood Chain the way any developer can, because the network is permissionless.

## Why Precursor is built here

**Stock tokens already live here.** Robinhood Chain is where users and integrators already expect to find stock tokens. Wallets, DEXs and price feeds on the chain understand the conventions Robinhood Stock Tokens established: one ERC-20 per asset, corporate actions through an ERC-8056 multiplier, and per-asset price feeds that respect a pause flag. Precursor Share Tokens follow the same conventions, so they work with that infrastructure from day one.

**The primitives are in place.** Chainlink feeds, institutional custody through BitGo, the USDG stablecoin and established AMMs are all live on the network. Precursor builds on them rather than replacing them.

**It settles to Ethereum.** Transaction data is posted to Ethereum as blobs, and the chain inherits Ethereum's security for settlement. It is fully EVM-equivalent, so standard Solidity tooling, audit practices and wallets all work unchanged.

**Per-transfer checks are economical.** Every Precursor transfer runs the company's transfer rules on-chain. Low fees and sub-second soft confirmation make those checks practical, even for small transfers.

The chain is permissionless, so Precursor doesn't rely on the network to restrict who can hold a token. Every rule is enforced in the token contract itself. See [Transfer rules](../concepts/transfer-rules.md).

## Network parameters

| | Mainnet | Testnet |
|---|---|---|
| Chain ID | `4663` | `46630` |
| Gas token | ETH | ETH |
| Public RPC | `https://rpc.mainnet.chain.robinhood.com` | `https://rpc.testnet.chain.robinhood.com` |
| Block explorer | `https://robinhoodchain.blockscout.com` | `https://explorer.testnet.chain.robinhood.com` |
| Sequencer feed | `wss://feed.mainnet.chain.robinhood.com` | `wss://feed.testnet.chain.robinhood.com` |
| Delayed sequencer feed | `wss://delayed-feed.mainnet.chain.robinhood.com` | Not available |
| Precursor environment | Live (`api.useprecursor.xyz`) | Sandbox (`api.sandbox.useprecursor.xyz`) |

The public RPC endpoints are rate-limited and are not meant for production traffic. For production integrations, use a dedicated provider such as Alchemy, Chainstack or QuickNode. Precursor itself uses Alchemy as its primary provider, with Chainstack as failover. Historical reads and event indexing need an archive endpoint.

Precursor contract addresses for both networks are published in the [Explorer](https://explorer.useprecursor.xyz) and returned by `GET /v1/contracts`.

## Wallet setup

Most investors and employees never configure a network. Portfolio creates an embedded wallet during onboarding, and Precursor sponsors gas for every holder transaction, so nobody needs to hold ETH.

To use an external wallet, add Robinhood Chain as a custom network with the mainnet values above: chain ID `4663`, currency symbol `ETH`, an RPC URL and the block explorer URL. Portfolio prompts compatible wallets to add the network automatically when you connect. Any EVM wallet that supports custom networks will work, as will institutional custody through BitGo.

## Confirmation and finality

A transaction on Robinhood Chain reaches two milestones, and Precursor reports both.

| Milestone | When | What it means |
|---|---|---|
| **Sequencer confirmation** | Typically under a second | The sequencer has ordered the transaction and published it on the feed. The result is known and, in normal operation, final. |
| **Ethereum confirmation** | After the batch containing it is posted to Ethereum | The transaction is part of data recorded on Ethereum and inherits Ethereum's settlement guarantees. |

The Console and Portfolio show a transaction as confirmed at sequencer confirmation, which is what makes a Liquidity Window clearing or an offering close feel instant. The Explorer shows both milestones for every transaction. Webhook events include a `confirmation` field set to `sequencer` or `l1`, so integrators can decide which level they need before acting. See [Webhooks](../developers/webhooks.md).

The sequencer orders transactions first-come, first-served, by arrival time. It does not run a priority fee auction, which keeps ordering predictable for settlement transactions.

## If the sequencer is unavailable

Robinhood Chain inherits Arbitrum's force-inclusion mechanism. If the sequencer stops including transactions, anyone can submit a transaction to the chain's inbox contract on Ethereum. After a delay set by the protocol, the transaction is included whether or not the sequencer is cooperating. A sequencer outage delays transactions, but it cannot censor a holder or freeze the chain permanently.

During an outage, Precursor behaves conservatively:

* **Holder balances are unaffected.** Balances live in contract state, and the state up to the last posted batch is recorded on Ethereum.
* **Time-sensitive actions are held.** The transaction queue holds new submissions rather than routing them to force inclusion by default, because the order and timing of forced transactions are harder to predict. A company can choose to force-include a specific transaction from the Console.
* **Liquidity Windows get more time.** If a window's close falls during an outage, Precursor recommends that the company extend it, and the Console offers that as a one-step action.
* **Valuations stay put.** The `ValuationOracle` doesn't publish updates during the outage, and integrations reading it see the last value with its original timestamp.

When the sequencer resumes, the queue drains in submission order, the indexer catches up from the archive endpoint, and the status page records the incident.

## Ecosystem integrations

| Integration | How Precursor uses it |
|---|---|
| **USDG** | The default settlement asset for offerings, Liquidity Windows, RFQ trades and distributions. Other stablecoins can be enabled per company. |
| **Chainlink** | The `ValuationOracle` exposes a Chainlink-compatible `latestRoundData` interface, so integrations that already read Chainlink feeds on the chain can read Precursor's Reference Valuations without new code. |
| **BitGo** | Institutional custody for companies, funds and investors who prefer a qualified custodian to self-custody. |
| **AMMs** | Companies that opt in can run permissioned pools on the chain's DEX infrastructure through `PoolGate`. See [Permissioned pools](../liquidity/permissioned-pools.md). |

For how these pieces connect inside Precursor, see the [System overview](system-overview.md).
