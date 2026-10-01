---
description: How splits, dividends, conversions and exit events run on-chain through the multiplier, the DistributionVault and registrar actions.
---

# Corporate actions

Corporate actions on Precursor are on-chain transactions. Each one is scheduled or executed through a contract, emits events that anyone can read, and is backed by a document anchored in the `DisclosureRegistry`. This page covers splits, dividends and other distributions, conversions between share classes, and exit events.

## Splits and reverse splits

Splits use the ERC-8056 multiplier. A share token's `uiMultiplier()` returns the number of shares each token represents, scaled by `1e18`. Changing the multiplier changes what every balance is worth in shares, without changing any raw balance.

| Action | Multiplier before | Multiplier after | A holder with 1,000 tokens holds |
|---|---|---|---|
| None | `1e18` | `1e18` | 1,000 shares |
| 2-for-1 split | `1e18` | `2e18` | 2,000 shares |
| 1-for-4 reverse split | `2e18` | `0.5e18` | 500 shares |

Because raw balances never move, anything that holds tokens keeps working through the change: pool reserves, open orders, lots and vesting schedules all stay valid.

### How a split is executed

1. The company proposes the new multiplier in the [Console](../products/console.md) and attaches the board resolution. The resolution's hash is anchored with `DocumentAnchored`.
2. The change is scheduled through the `IssuerTimelock` with a 48-hour delay, emitting `MultiplierScheduled(newMultiplier, effectiveAt)`.
3. If the company pauses the token around `effectiveAt`, the `ValuationOracle` stops accepting new values and holds its last one until the token is unpaused.
4. At `effectiveAt`, the multiplier updates and the token emits `UIMultiplierUpdated(oldMultiplier, newMultiplier)`.
5. The oracle publishes an adjusted Reference Valuation and resumes.

Integrators should always read `uiMultiplier()` when converting a token balance to a share count. See [Integrating Precursor assets](../developers/integrating-precursor-assets.md).

## Dividends and distributions

Cash returned to holders, whether a dividend, a special distribution or proceeds from a sale, goes through the `DistributionVault`.

1. The company funds a distribution with USDG or any other ERC-20 and sets a **record block**, which must already be in the past. Record blocks are Robinhood Chain L2 block numbers, read from the ArbSys precompile. The company can also exclude up to 20 accounts, such as treasury shares or the `LiquidityWindow` escrow. The vault emits `DistributionFunded(distributionId, asset, amount, recordBlock)`.
2. Entitlements come from balances at the record block. Each holder's entitlement is their balance at that block as a share of the eligible supply, meaning total supply minus excluded balances.
3. Holders call `claim(distributionId)`, or Portfolio claims on their behalf with `claimFor`, which pays the holder and uses sponsored gas. Each claim emits `DistributionClaimed(distributionId, holder, amount)`.

Distributions are pull-based. Holders claim their share rather than the company pushing payments to every wallet, which keeps distributions cheap and fully auditable no matter how many holders there are.

| Question | Answer |
|---|---|
| Do frozen tokens receive distributions? | Yes. A freeze blocks transfers, not entitlements. |
| What if a holder's claim has expired? | Their entitlement is preserved. They can claim once verified again. |
| What happens to unclaimed amounts? | They remain claimable. The company can set a claim deadline of at least 12 months, after which unclaimed amounts return to the treasury. |
| Can a distribution be paid in tokens? | Yes, any ERC-20 the company funds the vault with, including another share token. |

The step-by-step process is in [Pay a distribution](../guides/pay-a-distribution.md).

## Conversions

Some events move holders from one share class to another, for example preferred shares converting to common. A conversion is a burn and mint across two share tokens, executed by the `REGISTRAR` in a single transaction at the ratio set out in the class terms.

- The source token emits `SharesRedeemed(from, amount, ref)` for each holder.
- The destination token emits `SharesIssued(to, amount, ref)` for each holder.
- Both events share the same `ref`, so the conversion can be traced from one class to the other.

Converted shares arrive as new lots in the destination class. The original acquisition time carries over, so a conversion does not restart a holding period.

## Exit events

### IPO

When a company lists, its tokenized shares convert into the listed security. The company can either convert tokens into conventional shares held through the listing venue, or continue a tokenized program for its public shares. In both cases the conversion is recorded with `SharesRedeemed` events and a reference to the conversion record.

### Acquisition

When a company is acquired, the consideration is paid through the `DistributionVault`. The company funds a distribution with the purchase consideration at a record block, holders claim, and the registrar burns the tokens. The full sequence, from funding to the final burn, is visible on-chain.

## Corporate action timeline

| Action | Delay | Pauses token | Key events |
|---|---|---|---|
| Split or reverse split | 48 hours | Oracle only | `MultiplierScheduled`, `UIMultiplierUpdated` |
| Dividend | None, record block chosen by company | No | `DistributionFunded`, `DistributionClaimed` |
| Conversion | Set by the company | Typically yes, for the duration | `SharesRedeemed`, `SharesIssued` |
| Acquisition payout | Set by the company | Yes, from the record block | `DistributionFunded`, `SharesRedeemed` |

{% hint style="info" %}
Every corporate action references an anchored document. You can open the document hash from the event in the [Explorer](../products/explorer.md) and compare it against the copy the company provides in Portfolio.
{% endhint %}
