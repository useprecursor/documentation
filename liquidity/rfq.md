---
description: RFQ trading lets eligible holders trade directly with each other through EIP-712 signed quotes, settled atomically by the RFQSettlement contract against the share class's transfer rules.
---

# RFQ trading

RFQ (request for quote) trading gives eligible holders a continuous, bilateral way to trade between Liquidity Windows. One party signs a quote off-chain. The other fills it on-chain through `RFQSettlement`, which checks the transfer rules and swaps the shares for stablecoin in a single transaction.

RFQ is disabled by default. The company enables it per share class in the [Console](../products/console.md), which calls `setEnabled(shareToken, true)` from `ISSUER_ADMIN`. Because every fill is an ordinary share token transfer, a `TransferWindowModule` on the class limits RFQ fills to its scheduled trading periods too.

## The quote

A quote is EIP-712 typed data signed by the maker. The domain is name `Precursor`, version `1`, the chain id, and the `RFQSettlement` address as `verifyingContract`.

{% code title="Quote.sol" %}
```solidity
struct Quote {
    address maker;       // the signer of the quote
    address shareToken;  // the Precursor Share Token being traded
    uint8   side;        // 0 = maker sells shares, 1 = maker buys shares
    uint256 quantity;    // maximum shares, in raw token units
    uint256 price;       // stablecoin per share
    address stablecoin;  // settlement asset, USDG by default
    address taker;       // a specific counterparty, or address(0) for any eligible holder
    uint256 nonce;       // unique per maker, used for cancellation
    uint64  expiry;      // unix timestamp after which the quote cannot be filled
}
```
{% endcode %}

Quotes are shared off-chain. Portfolio delivers them between holders, and venues can relay them through their own channels. A quote with a `taker` of `address(0)` can be filled by any wallet that passes the transfer rules. A quote with a named `taker` can only be filled by that address.

## Filling a quote

The taker calls:

```solidity
function fill(Quote calldata quote, bytes calldata signature, uint256 quantity) external;
```

`quantity` can be less than `quote.quantity`, so a quote can be filled in parts until it is exhausted. `RFQSettlement` then:

1. checks that RFQ is enabled for the share class, the quote has not expired, its nonce has not been cancelled, the caller is the named taker if there is one, and enough quantity remains
2. verifies the maker's signature against the EIP-712 digest from `hashQuote(quote)`. Smart contract wallets are supported through EIP-1271
3. moves the shares from seller to buyer with `transferFrom`, so the share token runs its full rule check and reverts with `TransferNotAllowed(reasonCode)` if it fails
4. moves the stablecoin from buyer to seller in the same transaction
5. charges fees on the stablecoin leg: the buyer pays the price rounded up plus the buyer fee, the seller receives the price rounded down less the seller fee, and the difference goes to the fee recipient
6. emits `QuoteFilled(quoteHash, maker, taker, quantity, price)`

Both sides must have approved `RFQSettlement` to move their shares or stablecoin. Portfolio handles approvals as part of the flow, and gas is sponsored for holders.

Shares bought through RFQ form a new lot for the buyer, and any holding period configured for the class starts at the fill.

## Cancelling quotes

A maker cancels a single quote by cancelling its nonce:

```solidity
function cancelQuote(uint256 nonce) external;
```

To cancel many quotes at once, a maker can raise their minimum valid nonce. Every quote with a lower nonce becomes unfillable:

```solidity
function invalidateNoncesBelow(uint256 newMinNonce) external;
```

Both functions emit `QuoteCancelled(maker, nonce)`. For `invalidateNoncesBelow`, the event carries the new minimum. Quotes also stop being fillable at `expiry`, so short expiries are a simple way to limit exposure without any cancellation transaction.

## Pricing and transparency

RFQ prices are agreed between two parties, so they are not published as Reference Valuations. Every fill is still public: the Explorer shows each `QuoteFilled` event with its price and quantity, attributed by wallet address only. Companies can see RFQ activity for their share classes in the Console alongside Liquidity Window results.

## Checking before you sign

Before signing or filling a quote, check the transfer with `POST /v1/transfers/check` or by calling `canTransfer` directly. If the seller's shares are in a holding period, or the buyer has reached an ownership cap, the check returns the relevant [reason code](../developers/reason-codes.md) without anyone spending gas.

## Events

| Event | Emitted when |
| --- | --- |
| `QuoteFilled(quoteHash, maker, taker, quantity, price)` | A quote is filled in full or in part |
| `QuoteCancelled(maker, nonce)` | A maker cancels a quote or invalidates a range of nonces |
| `Transfer` | Shares move between seller and buyer (emitted by the share token) |

For how settlement and fees work across every liquidity route, see [Settlement](settlement.md).
