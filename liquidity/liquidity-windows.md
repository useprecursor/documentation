---
description: Liquidity Windows are issuer-run, time-boxed batch auctions and tenders that let shareholders sell to eligible buyers at a single clearing price, settled atomically on-chain.
---

# Liquidity Windows

A Liquidity Window is a scheduled liquidity event that the company controls. For a set period, eligible holders can offer shares for sale and eligible buyers can bid for them. When the window closes, the `LiquidityWindow` contract settles every matched order at once, at one price, in a single transaction.

Windows replace the traditional tender offer. The company decides when they happen, who can take part and what limits apply. Price discovery happens in the open, and every participant is verified before they can place an order.

## Configuration

| Parameter | Description |
| --- | --- |
| `shareToken` | The share class being traded |
| `mode` | `AUCTION` (uniform-price batch auction) or `TENDER` (fixed price) |
| `opensAt` / `closesAt` | The order period |
| `stablecoin` | The settlement asset. USDG by default |
| `reservePrice` | Optional. No shares trade below this price |
| `tenderPrice` | `TENDER` mode only. The fixed price per share |
| `maxVolume` | Optional. The most shares that can trade in this window, in raw token units. `0` means no limit |
| `perSellerCap` | Optional. The most shares one seller can offer, in raw token units. `0` means no limit |
| `rofrHolder` | Optional. The party with a right of first refusal in this window, usually the company treasury |
| `rofrPeriod` | Optional. How long `rofrHolder` has to exercise after `closesAt` |
| `sellerClaims` | Claim topics a seller must hold, for example `EMPLOYEE_OF` for an employee-only window |
| `buyerClaims` | Claim topics a buyer must hold |

Beyond the claims listed here, every seller and buyer must pass the share class's own transfer rules, so holding periods, vesting and region rules decide who can sell and buy. A window accepts at most 200 orders (`MAX_ORDERS`), which keeps settlement within a single transaction.

If the share class has a `TransferWindowModule` configured with `allowDuringLiquidityWindows`, transfers to and from the `LiquidityWindow` contract pass even when the module's own trading periods are closed. See [Transfer rules](../concepts/transfer-rules.md).

## Auction mode

In auction mode, both sides name their price.

- **Sellers** call `placeOrder(windowId, SELL, quantity, limitPrice)`, where the limit is the lowest price they will accept. The tokens move into the contract and stay locked until the window clears or the order is cancelled.
- **Buyers** call `placeOrder(windowId, BUY, quantity, limitPrice)`, where the limit is the highest price they will pay. The contract locks `quantity × limitPrice`, rounded up, plus the buyer fee in stablecoin.

Every order is checked with `canTransfer` when it is placed, so a seller can only offer unlocked shares and a buyer must be able to receive them. Orders can be cancelled with `cancelOrder(windowId, orderId)` at any time before `closesAt`. Between close and clearing, the company can remove a single problem order with `removeOrder`, which returns its escrow.

### How the clearing price is set

After `closesAt`, anyone can call `clear(windowId)`. The contract:

1. finds the range of prices that maximizes the number of shares matched, where supply at a price is every sell order with a limit at or below it, and demand is every buy order with a limit at or above it
2. picks the price in that range closest to the share class's latest [Reference Valuation](../concepts/reference-valuation.md), or to the midpoint of the best ask and the best bid if there is no valuation yet
3. raises the price to `reservePrice` if it falls below it
4. fills orders by price priority, then pro-rata among orders at the same limit price
5. settles every fill atomically and returns everything that did not fill

Every matched buyer pays the same clearing price, and every matched seller receives it, less fees. Buyers get back any locked stablecoin above what they paid. Rounding always favors the contract: buyers pay rounded up, sellers receive rounded down, and the dust goes to the fee recipient, so the window never pays out more than it holds. Bids from buyers who no longer pass the share class rules at clearing are refunded rather than filled.

If a token delivery at clearing fails the recipient's rules, for example because a seller's claim expired while their unfilled shares were in escrow, the window does not revert. It keeps those tokens for the owner and emits `DeliveryDeferred`, and the owner calls `claimDeferred(shareToken)` once they pass the rules again. `previewClearing(windowId)` returns the price and volume `clear` would produce at any moment.

### Worked example

A window for Series B Preferred receives these orders. The latest Reference Valuation is 20.40 USDG per share.

| Sellers | Quantity | Min price |
| --- | --- | --- |
| A | 4,000 | 18.00 |
| B | 3,000 | 20.00 |
| C | 5,000 | 22.00 |
| D | 2,000 | 25.00 |

| Buyers | Quantity | Max price |
| --- | --- | --- |
| X | 6,000 | 24.00 |
| Y | 4,000 | 21.00 |
| Z | 3,000 | 19.00 |

The matched volume at each candidate price:

| Price | Supply | Demand | Matched |
| --- | --- | --- | --- |
| 19.00 | 4,000 | 13,000 | 4,000 |
| 20.00 | 7,000 | 10,000 | 7,000 |
| 21.00 | 7,000 | 10,000 | 7,000 |
| 22.00 | 12,000 | 6,000 | 6,000 |

Volume peaks at 7,000 shares for any price from 20.00 to 21.00. The Reference Valuation of 20.40 falls in that range, so the window clears at **20.40 USDG**.

Sellers A and B sell all 7,000 shares. On the buy side, X has the higher limit and fills all 6,000 shares. Y receives the remaining 1,000. Z's limit is below the clearing price, so Z does not fill. C and D get their tokens back. In total 7,000 shares trade for 142,800 USDG.

## Tender mode

In tender mode the price is fixed in advance at `tenderPrice`. Holders tender shares with `placeOrder(windowId, SELL, quantity, 0)`, and the company or other buyers commit with `placeOrder(windowId, BUY, quantity, 0)`. Any limit price passed in tender mode is replaced by `tenderPrice`. Whichever side is oversubscribed is filled pro-rata. Tender mode suits company buybacks and structured secondary sales where the price has already been agreed.

## Right of first refusal

If the window names a `rofrHolder`, it enters a ROFR period after `closesAt`. The indicative clearing result is visible on-chain through `previewClearing(windowId)`. During `rofrPeriod`, the right holder can call `exerciseRofr(windowId, quantity)` once, depositing stablecoin for that quantity at the previewed price. Once the period ends, `clear` fills the right holder first at the clearing price, up to the clearing volume and what the deposit covers. ROFR fills displace other buyers rather than adding volume, and any unused deposit is returned. Because the right is exercised here, a share class's `ROFRModule` usually lists the `LiquidityWindow` among its `exemptVenues`.

## Reserve price and buybacks

A `reservePrice` protects existing holders from a disorderly result. If the chosen price is below it, the window clears at the reserve instead, and if no volume can trade there, it clears with no fills and every order is returned.

For a buyback, the company takes part as a buyer. Its bid is placed from its treasury like any other order and is labeled as a company order in the Explorer. Shares bought back land in the treasury wallet, which must be verified like any other holder, and the registrar can burn them afterwards.

## After clearing

If any volume traded, the clearing price is published to the `ValuationOracle` as a Reference Valuation with source `WINDOW_CLEARING`. Each buyer's shares form a new lot, which starts any holding period configured for the class.

## Events

| Event | Emitted when |
| --- | --- |
| `WindowOpened(windowId, opensAt, closesAt)` | The company opens a window |
| `OrderPlaced(windowId, orderId, side, quantity, limitPrice)` | An order is accepted |
| `OrderCancelled(windowId, orderId)` | An order is withdrawn before close, or removed by the company before clearing |
| `WindowCleared(windowId, clearingPrice, volume)` | The window settles |
| `WindowCancelled(windowId)` | The company cancels the window before clearing. Every order is returned |
| `RofrExercised(windowId, holder, quantity, price)` | The right holder commits during the ROFR period |
| `DeliveryDeferred(shareToken, owner, amount)` | A token delivery failed the recipient's rules and is held for them |
| `DeferredClaimed(shareToken, owner, amount)` | An owner claims deferred tokens |

See [Open a Liquidity Window](../guides/open-a-liquidity-window.md) and [Sell in a Liquidity Window](../guides/sell-in-a-liquidity-window.md) for the step-by-step guides.
