---
description: Permissioned pools let a company opt into continuous AMM trading on Robinhood Chain, with the PoolGate hook enforcing transfer rules on every swap and liquidity change.
---

# Permissioned pools

Some companies want continuous trading for a share class, not just scheduled windows and bilateral quotes. Permissioned pools provide this. They are AMM pools on Robinhood Chain DEX infrastructure that pair a Precursor Share Token with a stablecoin, and they enforce the share class's [transfer rules](../concepts/transfer-rules.md) on every interaction through a hook contract called `PoolGate`.

Pools are opt-in. A company enables them per share class in the [Console](../products/console.md), and can disable them at any time.

## How a pool becomes eligible

A Precursor Share Token can only be transferred to an address with a verified identity. A pool contract has no person behind it, so it needs a different kind of identity.

When a company approves a pool, the pool's address is registered in the company's `IdentityRegistry` with a `VENUE` claim. The claim records that the address is an approved venue for that share class. The `RuleEngine` treats an address with a `VENUE` claim as a pass-through: transfers into and out of the pool are allowed, provided the wallet on the other side of the trade passes every rule.

A `VENUE` claim is issued per share class and can be revoked by the company. Revoking it stops all trading in that pool, although liquidity providers can still withdraw.

## What PoolGate checks

`PoolGate` runs before every swap and every liquidity change in an approved pool. It identifies the end wallet, meaning the address that will send or receive shares, and runs `canTransfer` for that wallet against the pool.

| Action | Check |
| --- | --- |
| Buy shares | The pool calls `beforeSwap(recipient, sharesOut)`, which runs `canTransfer(pool, recipient, sharesOut)` |
| Sell shares | The share token itself runs the sender's rules when shares move into the pool |
| Add liquidity | The pool calls `beforeAddLiquidity(provider)`, which runs every recipient-side rule on the provider, as if they were receiving shares |
| Transfer an LP position | The pool calls `beforeTransferPosition(from, to)`, which runs the same check on the new owner |

If a check fails, the hook reverts with `TransferNotAllowed(reasonCode)`, and the swap or liquidity change does not happen. `checkSwap` and `checkLiquidityProvider` return the same reason code as a view, without reverting. Shares removed from a pool are ordinary transfers from the pool, so the share token checks the recipient.

Routers and aggregators can route through permissioned pools, but they must pass the end recipient through to the hook. `PoolGate` checks the recipient, not the router. A router that cannot name the recipient cannot use the pool.

## Rules still apply

A pool does not loosen any rule. In particular:

- **Holding periods.** Shares in a locked lot cannot be sold into a pool. Shares bought from a pool form a new lot, which starts any holding period configured for the class.
- **Holder caps.** A purchase that would add a new holder beyond the `HolderCapModule` limit fails with `HOLDER_CAP_REACHED`.
- **Ownership caps.** A purchase that would take a wallet over the `MaxOwnershipModule` limit fails with `MAX_OWNERSHIP_EXCEEDED`.
- **Transfer windows.** If the class uses a `TransferWindowModule`, the pool only trades while a window is open.
- **ROFR.** Share classes with a `ROFRModule` cannot enable pools, because a right of first refusal cannot be exercised against an instant swap.
- **Pause.** If the token is paused, every pool for that class stops trading.

## Liquidity providers

Liquidity positions carry a claim on the shares in the pool, so they are restricted in the same way as the shares themselves. Only verified holders who pass the transfer rules can provide liquidity, and LP positions can only be transferred to other eligible holders. To limit liquidity provision to a named set of providers, such as a market maker it has engaged, the company can require a claim that only those providers hold through the share class's rules.

## Pricing

Pool prices are market prices, set by the pool's curve and the trades against it. They are not Reference Valuations, and Precursor does not publish them as such. Portfolio and the Explorer show the pool price next to the latest Reference Valuation, each clearly labeled, so holders can see both.

## Fees

The pool's own swap fee goes to its liquidity providers. `PoolGate` only checks the rules and does not take a fee. See [Settlement](settlement.md) and [Pricing](../resources/pricing.md).

{% hint style="warning" %}
Pools suit share classes with enough holders and enough float to support continuous trading. For most private companies, [Liquidity Windows](liquidity-windows.md) give better price discovery, and pools work best as a complement to them.
{% endhint %}
