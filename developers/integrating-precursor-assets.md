---
description: How wallets, venues, lending protocols and data providers support Precursor Share Tokens correctly on Robinhood Chain.
---

# Integrating Precursor assets

Precursor Share Tokens are designed to work with existing Robinhood Chain infrastructure. They are ERC-20 tokens with 18 decimals, they follow the same multiplier and oracle conventions as Robinhood Stock Tokens, and they expose Chainlink-compatible price reads. Most integrations need only a few additions on top of standard ERC-20 support.

This page covers what those additions are, for each kind of integration.

## What is different from a plain ERC-20

| Property | Plain ERC-20 | Precursor Share Token |
|---|---|---|
| Transfers | Always succeed with sufficient balance | Succeed only if `canTransfer` returns `OK` for both parties |
| Balance meaning | Units of the token | Raw units. Multiply by `uiMultiplier() / 1e18` for shares |
| Price | From markets | Reference Valuation from `ValuationOracle`, dated and sourced |
| Pausing | Usually none | The share class can be paused, blocking all transfers |
| Holders | Any address | Wallets linked to a verified identity for the company |

Everything else, including `balanceOf`, `transfer`, `approve`, `allowance`, `transferFrom` and the standard events, behaves as you expect.

## Check before you build a transaction

Every transfer runs the share class's rules on-chain. A transfer that fails a rule reverts with a [reason code](reason-codes.md). Call `canTransfer` first so you never ask a user to sign something that will revert.

```ts
const [ok, reasonCode] = await client.readContract({
  address: token,
  abi: shareTokenAbi,
  functionName: "canTransfer",
  args: [from, to, amount],
});
```

If you cannot call the chain directly, `POST /v1/transfers/check` returns the same result with a plain-language message, and it works with publishable keys from a browser. Use `unlockedBalanceOf(holder)` to show how much of a balance can move right now. It excludes frozen tokens, unvested tokens and tokens still in a holding period.

## Display balances with the multiplier

Splits and other corporate actions change `uiMultiplier`, never raw balances. This keeps pools, lending positions and approvals intact through a split. Your interface has to apply the multiplier to show the right number of shares.

```ts
const shares = (rawBalance * multiplier) / 10n ** 18n; // still at 18 decimals
```

Read `uiMultiplier()` when you display a balance rather than caching it indefinitely, or subscribe to `UIMultiplierUpdated`. A change is always announced in advance with `MultiplierScheduled`, which gives integrators at least 48 hours of notice. The SDK's `toDisplayAmount` helper does this conversion for you.

Amounts a user types in should be converted the other way, from shares to raw units, before you build the transaction.

## Read prices from the ValuationOracle

Each share class has a `ValuationOracle` that implements the Chainlink `AggregatorV3Interface` read functions. Existing code that reads a Chainlink feed with `latestRoundData()` works unchanged. Prices are in USD per share with 8 decimals. After a split the company publishes a new valuation per post-split share, so pair a price with the multiplier in force on its `asOf` date.

```solidity
(, int256 price, , uint256 updatedAt, ) = AggregatorV3Interface(oracle).latestRoundData();
```

For richer data, `latestValuation()` returns the price, its source (`PRICED_ROUND`, `INDEPENDENT_VALUATION` or `WINDOW_CLEARING`), the date it applies to and the hash of the supporting document.

{% hint style="warning" %}
A Reference Valuation is not a live market price. Private company valuations update at funding rounds, independent valuations and Liquidity Window clears, which can be months apart. Always show the date and source next to the price, and if you use it for risk, such as collateral value, apply limits that account for how old it is. See [Reference Valuation](../concepts/reference-valuation.md).
{% endhint %}

## Handle pauses

When a share class is paused, every transfer reverts with `TOKEN_PAUSED` and the valuation oracle holds its last value without publishing new ones. Watch for `Paused(reason)` and `Unpaused()` on the token, and reflect the state in your interface. Companies usually pause the token around a multiplier change, which also holds the oracle, the same convention Robinhood Chain uses for its stock tokens.

## Guidance by integration type

### Wallets

* Show balances in shares, using the multiplier, with the share class name and company.
* Run `canTransfer` as the user fills in a send form and show the reason before they sign.
* Show unlocked and locked balances separately.
* Link each token to its page on the [Explorer](https://explorer.useprecursor.xyz) for documents, supply and history.

### Venues and AMMs

A pool or venue contract that holds Precursor tokens needs a registered identity in the company's `IdentityRegistry` holding a valid `VENUE` claim whose data is the share token's address, and the issuer must opt in. That claim makes the contract an agent: the token checks the people on either side of a trade rather than the venue itself (`isAgent(venue)` returns true). Without it, transfers into the venue revert with `RECIPIENT_NOT_VERIFIED`. For AMM pools, Precursor provides `PoolGate`, a hook that checks the end recipient on every swap and each liquidity provider. See [Permissioned pools](../liquidity/permissioned-pools.md) for how issuers enable this and how a venue requests a `VENUE` claim.

Contact hello@useprecursor.xyz to list Precursor assets on your venue.

### Lending protocols

* The lending contract that holds collateral needs a `VENUE` claim, just as a pool does.
* Liquidations must deliver tokens only to eligible wallets. Check `canTransfer` for the liquidator before you allow a liquidation path.
* Account for pauses, because a paused share class cannot be liquidated until it is unpaused.
* Use conservative collateral factors and staleness limits that reflect how often a Reference Valuation updates.

### Data providers and indexers

* Index from the factory's `CompanyDeployed` and `ShareClassDeployed` events. See [On-chain events](events.md).
* Store raw balances and apply the multiplier at read time.
* Label every price with its source and date.
* Use `GET /v1/contracts` and `GET /v1/valuations` with a publishable key for metadata.

## Labeling

Describe Precursor tokens as **issuer-sponsored stock tokens**, issued by the company they represent. They are distinct from third-party stock tokens that track a company's price without its participation, and users deserve to know which one they are looking at. Use the share class name and company name from the token metadata, and do not describe a Reference Valuation as a market price.

## Testing

Every integration can be tested end to end in the sandbox on Robinhood Chain testnet (`46630`), where you can create share classes, test identities with any claims, and venue identities. Start with the [Quickstart](quickstart.md).
