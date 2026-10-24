---
description: External standards, network documentation and infrastructure that Precursor builds on.
---

# References

Precursor is built on open standards and public infrastructure. These are the external sources behind the conventions these docs use.

## Robinhood Chain

| Resource | What it covers |
|---|---|
| [Connecting to Robinhood Chain](https://docs.robinhood.com/chain/connecting) | Chain IDs, RPC endpoints, sequencer feeds and the block explorer for mainnet and testnet |
| [Add network to your wallet](https://docs.robinhood.com/chain/add-network-to-wallet) | Wallet configuration for both networks |
| [Stock Tokens](https://docs.robinhood.com/chain/stock-tokens/) | The chain's reference stock token conventions: one ERC-20 per asset, the `uiMultiplier` for corporate actions, per-asset price feeds and trading status checks |
| [Robinhood Chain explorer](https://robinhoodchain.blockscout.com) | Blockscout explorer for mainnet transactions and contracts |

Precursor share tokens follow the Robinhood Chain stock token conventions so that they behave predictably in wallets, oracles and DeFi integrations. See [Robinhood Chain](../architecture/robinhood-chain.md) for network details.

## Token standards

| Standard | How Precursor uses it |
|---|---|
| [ERC-20](https://eips.ethereum.org/EIPS/eip-20) | Base interface for every Precursor Share Token |
| [ERC-3643 (T-REX)](https://www.quicknode.com/guides/real-world-assets/erc-3643) | Permissioned token standard: identity registry, claim-based eligibility and modular transfer rules. See also the [T-REX whitepaper](https://tokeny.com/wp-content/uploads/2023/05/ERC3643-Whitepaper-T-REX-v4.pdf). |
| ERC-8056 (Scaled UI Amount Extension) | The `uiMultiplier()` interface for splits and other corporate actions, as used by Robinhood Chain stock tokens. Described in the [Robinhood Chain Stock Tokens docs](https://docs.robinhood.com/chain/stock-tokens/). |
| [EIP-712](https://eips.ethereum.org/EIPS/eip-712) | Typed structured data signing, used for RFQ quotes under the domain name `Precursor`, version `1` |
| [ERC-1967 and UUPS (ERC-1822)](https://eips.ethereum.org/EIPS/eip-1822) | Proxy pattern for share tokens, with upgrades gated by `IssuerTimelock` |

## Oracles and pricing

| Resource | What it covers |
|---|---|
| [Chainlink: Robinhood Tokenized Equity Feeds](https://docs.chain.link/data-feeds/tokenized-equity-feeds/robinhood) | Per-asset price feeds on Robinhood Chain, how they incorporate the multiplier and how they behave when a token is paused |
| [Chainlink Data Feeds API](https://docs.chain.link/data-feeds/api-reference) | The `AggregatorV3Interface`, which the `ValuationOracle` implements for compatible consumers |

## Network and infrastructure

| Resource | What it covers |
|---|---|
| [Arbitrum documentation](https://docs.arbitrum.io) | The rollup stack Robinhood Chain is built on, including sequencing and force inclusion through Ethereum |
| [Blockscout documentation](https://docs.blockscout.com) | Explorer APIs used for contract verification and indexing |
| [Alchemy: Robinhood Chain RPC](https://www.alchemy.com/rpc/robinhood) | Production RPC, WebSocket and archive access for mainnet |
| [Alchemy: Robinhood Chain testnet RPC](https://www.alchemy.com/rpc/robinhood-testnet) | RPC access for testnet (chain ID `46630`) |
| [Chainstack: Robinhood Chain](https://docs.chainstack.com/reference/robinhood-getting-started) | Alternative RPC provider used for failover |

## Libraries

| Library | Use |
|---|---|
| [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts) | Base implementations for ERC-20, access control and proxies |
| [Foundry](https://book.getfoundry.sh) | Contract development, testing and deployment |
| [viem](https://viem.sh) | The on-chain helpers in `@precursor/sdk/onchain` are viem-compatible |
