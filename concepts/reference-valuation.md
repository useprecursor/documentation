---
description: How Precursor publishes a dated, sourced valuation for each share class through the ValuationOracle, and how to read it on-chain.
---

# Reference Valuation

Shares in a private company do not have a continuous market price. Precursor publishes a **Reference Valuation** for each share class instead: a price per share with its source, its date and the document behind it. It is a reference point for portfolio values, collateral calculations and price discovery, and it is never presented as a live market price.

## Sources

Every Reference Valuation has one of three sources.

| Source | What it is | Published by |
|---|---|---|
| `PRICED_ROUND` | The price per share in the company's most recent priced financing, including offerings run through `OfferingVault` | The company, or automatically when an offering closes |
| `INDEPENDENT_VALUATION` | A per-share value from an independent valuation of the company | The company |
| `WINDOW_CLEARING` | The clearing price of a [Liquidity Window](../liquidity/liquidity-windows.md) | Automatically, when the window clears |

The oracle exposes the most recent valuation of any source. The full history, with every source, is available on-chain through events and through `GET /v1/valuations`.

## The ValuationOracle

Each share class has a `ValuationOracle`. It has two read interfaces.

### Chainlink-compatible

The oracle implements the Chainlink `AggregatorV3Interface`, so any contract or tool that already reads Chainlink feeds can read a Precursor valuation.

```solidity
function decimals() external view returns (uint8); // 8
function latestRoundData() external view returns (
    uint80 roundId,
    int256 answer,      // USD per share, 8 decimals
    uint256 startedAt,
    uint256 updatedAt,  // equals asOf of the valuation
    uint80 answeredInRound
);
```

### Full valuation

```solidity
function latestValuation() external view returns (
    int256 price,         // USD per share, 8 decimals
    uint8 source,         // 0 PRICED_ROUND, 1 INDEPENDENT_VALUATION, 2 WINDOW_CLEARING
    uint64 asOf,          // timestamp the valuation refers to
    bytes32 documentHash  // anchored supporting document
);
```

Integrations should use `latestValuation()` wherever they can, because it carries the source and date that make the number meaningful.

Each new valuation emits `ValuationPublished(price, source, asOf, documentHash)`.

## Per share, not per token

Valuations are quoted **per share**. To value a holder's position, convert their token balance into shares using the token's multiplier first.

```
shares = balance × uiMultiplier / 1e18
value  = shares × price / 1e8
```

This keeps valuations stable through splits. After a 2-for-1 split the multiplier doubles and the oracle publishes a halved per-share price, so position values do not jump. See [Corporate actions](corporate-actions.md).

## Pausing

The oracle holds its last published value while its share token is paused. Companies that want feeds to hold steady across a split pause the token around the multiplier change. During a pause, `latestRoundData()` continues to return the last valuation with its original `updatedAt`, so consumers can see that the value is not being refreshed.

## How valuations appear in the product

Precursor always shows a Reference Valuation with its source and date. Portfolio might show "Reference Valuation $42.10 per share, Liquidity Window clearing, 14 September 2026" rather than a bare number. The Console warns a company before it publishes a valuation that is older than the latest Liquidity Window clearing price.

| Where | What is shown |
|---|---|
| Portfolio | Position value at the latest Reference Valuation, with source and date |
| Console | Full valuation history by source, and the gap between each source |
| Explorer | Public valuation history for each share class |
| API | `GET /v1/valuations` with source, date and document hash |

{% hint style="warning" %}
A Reference Valuation is a dated reference, not a quote. Nobody is obliged to buy or sell at it. Any integration that uses it, for example to value collateral, should check `asOf` and apply its own staleness limit.
{% endhint %}

## Publishing a valuation

A company publishes a `PRICED_ROUND` or `INDEPENDENT_VALUATION` from the Console or with `POST /v1/valuations`, attaching the supporting document. The document's hash is anchored in the `DisclosureRegistry` first, and its hash is referenced by the valuation. `WINDOW_CLEARING` valuations are published automatically by the `LiquidityWindow` contract when a window clears with volume, and an offering that closes publishes its price as a `PRICED_ROUND`. Only accounts with the `PUBLISHER` role can publish: the company's `ISSUER_ADMIN`, the `OfferingVault` and the `LiquidityWindow`. Publishing is blocked while the share token is paused.
