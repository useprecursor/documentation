---
description: How Precursor settles every trade and issuance as atomic delivery-versus-payment on Robinhood Chain, which stablecoins it uses, what confirmation means, and how fees are deducted.
---

# Settlement

Every movement of value on Precursor settles on-chain as **delivery-versus-payment**. Shares and stablecoin move in the same transaction, or neither moves. There is no clearing period, no settlement agent and no point at which one side has delivered and the other has not.

This applies to every route: [primary offerings](offerings.md), [Liquidity Windows](liquidity-windows.md), [RFQ trades](rfq.md), [permissioned pools](permissioned-pools.md) and [distributions](../guides/pay-a-distribution.md).

## How atomic settlement works

Each settlement contract holds or pulls both legs of a trade, then performs every transfer in one transaction:

| Route | Shares | Stablecoin |
| --- | --- | --- |
| Offering close | Minted to each investor | Escrow released to the company treasury |
| Liquidity Window | Locked sell orders move to buyers | Locked bids move to sellers, excess returned |
| RFQ fill | Seller to buyer | Buyer to seller |
| Pool swap | Pool to buyer, or seller to pool | The opposite direction |

Every share transfer in these flows goes through the share token's `canTransfer` check. If any check fails, or any transfer reverts, the whole transaction reverts. Nothing is partially settled.

For Liquidity Windows, `clear(windowId)` settles every matched order in the window in one transaction. A window accepts at most 200 orders, which keeps that transaction within a block's gas limit.

## Stablecoins

USDG is the default settlement asset. A company can enable other stablecoins that are live on Robinhood Chain for a share class, and each offering or window names exactly one. The supported list is shown in the Console and returned by `GET /v1/share_classes/{id}`.

Stablecoin amounts are always handled in the stablecoin's own decimals. Share amounts are in the token's 18 decimals. Prices are stablecoin per share, with the share count adjusted by the current `uiMultiplier`.

## Confirmation

Robinhood Chain has two levels of confirmation, and Precursor reports both.

**Sequencer confirmation.** Robinhood Chain's sequencer orders transactions as they arrive and publishes them on its feed, typically in well under a second. Once a settlement transaction appears on the feed, its outcome is determined. Portfolio and the Console show a trade as settled at this point.

**L1 confirmation.** The sequencer posts batches of transactions to Ethereum, using blobs for data availability. Once the batch containing a transaction is posted, the transaction inherits Ethereum's ordering guarantees.

API responses and webhook events include a `confirmation` field with the value `sequencer` or `l1`, alongside `block_number` and `tx_hash`. Integrations that move value off-chain in response to a settlement, such as releasing goods or wiring funds, should wait for `l1`.

## When settlement fails

Settlement can fail for a few reasons: a holder's claim expired between placing an order and clearing, the token was paused, or a holder's balance was frozen. For a single trade, the transaction reverts and no value moves. For settlements that involve many people, the contracts are built so one holder's problem never blocks everyone else.

How each route handles a failure:

- **Offerings.** Each subscription is checked again at close. If one investor no longer passes the rules, their subscription becomes refundable instead of minting, and the rest of the offering closes normally.
- **Liquidity Windows.** Bids from buyers who no longer pass the rules at clearing are excluded and refunded, and the clearing price is calculated from the remaining orders. If a token delivery still fails, for example returning unfilled shares to a seller whose claim has expired, the window holds those tokens for the owner and emits `DeliveryDeferred` instead of reverting. The owner claims them with `claimDeferred` once they pass the rules again.
- **RFQ and pools.** The fill or swap reverts with the reason code. The quote remains fillable if it has not expired.

The [reason codes](../developers/reason-codes.md) returned in each case are the same ones `canTransfer` returns.

## Fees

Platform fees are deducted from the stablecoin leg inside the settlement transaction. There is no separate invoice for transaction fees, and every fee is visible on-chain as a stablecoin transfer to Precursor's fee recipient in the same transaction.

| Route | How the fee is taken |
| --- | --- |
| Offering | A percentage of capital raised, deducted from the proceeds at close. No fee on a failed offering |
| Liquidity Window | A percentage of notional, split between buyer and seller |
| RFQ | A percentage of notional, split between buyer and seller |
| Pool | Set by the pool itself. `PoolGate` checks the rules and takes no fee |
| Distribution | No transaction fee |

In windows and RFQ, the buyer pays the price rounded up plus the buyer fee and the seller receives the price rounded down less the seller fee, so rounding dust also goes to the fee recipient. Rates are set per company when its contracts are deployed and can only change through the company's `IssuerTimelock`. On-chain caps limit them to 5% of an offering and 2.5% per side of a window or RFQ trade. See [Pricing](../resources/pricing.md).

## Gas

Gas on Robinhood Chain is paid in ETH. Precursor sponsors gas for investors and employees on every action in Portfolio, so holders never need to hold ETH. Companies can sponsor gas for their own integrations through the API, or submit transactions from their own wallets.
