---
description: The Rule Engine and its modules, the parameters each module accepts, the reason codes they return, and how lots are tracked and consumed.
---

# Transfer rules

Every Precursor Share Token has a `RuleEngine`. When tokens move, the Rule Engine runs each module the company has enabled for that share class. A transfer settles only if every module approves it. If one module rejects it, the transfer reverts with that module's reason code.

The company chooses and configures modules in the [Console](../products/console.md). Modules are separate contracts, so a company can add, remove or reconfigure a rule without upgrading the token.

## Order of checks

```
canTransfer(from, to, amount)
  1. Token:            paused?               → TOKEN_PAUSED
                       frozen / locked?      → FROZEN, INSUFFICIENT_UNLOCKED_BALANCE
  2. IdentityRegistry: sender verified?      → SENDER_NOT_VERIFIED, CLAIM_EXPIRED
                       recipient verified?   → RECIPIENT_NOT_VERIFIED, CLAIM_EXPIRED
  3. RuleEngine:       each enabled module, in configured order
  4. OK
```

The first failing check determines the reason code. Agents, meaning the company's `LiquidityWindow` and any venue holding a `VENUE` claim for the token, skip the identity check for their own side, and modules skip their sender-side or recipient-side check for them, so rules apply to the people at each end of a trade.

Each module takes its configuration as ABI-encoded bytes, set when the share class is deployed or later through `configureModule`. The parameter order in each table below is the encoding order. `canTransfer` is a view function, so wallets, venues and the API can call it before a user signs.

## Modules

### RegionModule

Restricts holders by the country in their `REGION` claim.

| Parameter | Type | Meaning |
|---|---|---|
| `mode` | `uint8`: `0` for `ALLOW_LIST`, `1` for `BLOCK_LIST` | Whether `regions` lists permitted or excluded countries |
| `regions` | `bytes2[]` | ISO 3166 alpha-2 codes, up to 250 |

Applies to the recipient. A recipient with no valid `REGION` claim is always blocked. Returns `REGION_BLOCKED`.

### EligibilityModule

Requires the recipient to hold a valid `INVESTOR_ELIGIBLE` claim at or above a set tier.

| Parameter | Type | Meaning |
|---|---|---|
| `minimumTier` | `uint8` | The lowest eligibility tier allowed to receive tokens |
| `exemptEmployees` | `bool` | Whether identities with a valid `EMPLOYEE_OF` claim for this company skip the tier check |

Returns `NOT_ELIGIBLE`.

### HoldingPeriodModule

Applies a lockup to each lot from the moment it is acquired.

| Parameter | Type | Meaning |
|---|---|---|
| `holdingPeriod` | `uint32` (seconds) | Default lockup per lot |
| `sources` | `bytes32[]` | Acquisition sources with their own lockup, for example `OFFERING` or `WINDOW` |
| `periods` | `uint32[]` | The lockup for each entry in `sources` |
| `operators` | `address[]` | Contracts whose transfers label a lot, such as the company's `OfferingVault` and `LiquidityWindow` |
| `operatorSources` | `bytes32[]` | The source label for each entry in `operators` |

A lot created by a labelled operator takes that operator's source. Other mints are `ISSUANCE` and other transfers are `TRANSFER`. Up to 16 overrides of each kind. Periods are read from the current configuration, so changing a period applies to existing lots too. Returns `HOLDING_PERIOD_ACTIVE` if the sender's unlocked lots do not cover the amount.

### HolderCapModule

Caps the number of distinct holders of the share class.

| Parameter | Type | Meaning |
|---|---|---|
| `maxHolders` | `uint32` | Maximum number of identities with a non-zero balance |

The module counts identities rather than wallets. A transfer to a new holder when the cap has been reached returns `HOLDER_CAP_REACHED`. Transfers between existing holders, or transfers that close out the sender's whole position, are always allowed.

### TransferWindowModule

Allows transfers only during company-defined windows.

| Parameter | Type | Meaning |
|---|---|---|
| `windows` | `(uint40 opensAt, uint40 closesAt)[]` | Scheduled periods when transfers are open, up to 64 |
| `allowDuringLiquidityWindows` | `bool` | Whether transfers to or from the company's `LiquidityWindow` are always permitted |
| `liquidityWindow` | `address` | The company's `LiquidityWindow` contract |

Mints always pass. Returns `WINDOW_CLOSED` outside an open window.

### ROFRModule

Gives the company, or a designee, a right of first refusal on transfers.

| Parameter | Type | Meaning |
|---|---|---|
| `holder` | `address` | The party that holds the right, usually the company treasury |
| `noticePeriod` | `uint32` (seconds) | How long the right holder has to exercise after a transfer is proposed |
| `exemptVenues` | `address[]` | Settlement contracts where the right is exercised in-line, such as `LiquidityWindow`, up to 16 |

A holder proposes a direct transfer on the module with `propose(to, amount)`, which emits `TransferProposed`. Until then, and until the notice period ends or the right holder calls `waive(from, to)`, the transfer returns `ROFR_PENDING`. If the right holder calls `exercise(from, to)` instead, only a transfer to the right holder is allowed for that proposal, and the parties settle payment between them, for example through [RFQ](../liquidity/rfq.md). Transfers to or from the right holder always pass. In a [Liquidity Window](../liquidity/liquidity-windows.md), the right is exercised as a priority bid at the clearing price instead.

### MaxOwnershipModule

Caps any single identity's share of the class.

| Parameter | Type | Meaning |
|---|---|---|
| `maxBps` | `uint16` | Maximum holding as basis points of total supply |
| `applyToMints` | `bool` | Whether the cap also applies to mints. Leave it off to import a cap table in any order |
| `exemptIdentities` | `address[]` | Identities exempt from the cap, such as founders or the treasury, up to 32 |

Returns `MAX_OWNERSHIP_EXCEEDED`.

### EmployeeVestingModule

Locks unvested employee shares and releases them on schedule.

| Parameter | Type | Meaning |
|---|---|---|
| `action` | `uint8` | `0` sets `accelerationEnabled`; `1` accelerates one grant to fully vested |
| `payload` | `bytes` | For action `0`, `abi.encode(bool accelerationEnabled)`. For action `1`, `abi.encode(address identity, uint256 index)` |

Grants come from the cap table, so the registrar adds them directly with `addGrants(identity, grants)`, each grant holding a start, cliff, duration and total, up to 16 per identity. Vesting is linear after the cliff. Acceleration runs through the engine and therefore the `IssuerTimelock`, and only while `accelerationEnabled` is set. Vesting is tracked per identity, so an employee with several wallets is one holder. Returns `UNVESTED` when a transfer would draw on shares that have not vested. Vested shares are visible in [Portfolio](../products/portfolio.md) as available, unvested shares as locked.

## Lots

Holding periods and vesting both depend on when tokens were acquired, so Precursor tracks holdings as **lots**. A lot is a quantity of tokens acquired in one event, for example a mint at onboarding, an allocation from an offering, or a purchase in a Liquidity Window. Each lot records its amount, acquisition time and source.

When a holder sends tokens, lots are consumed **first in, first out**. The oldest lot is drawn down first, then the next, and so on. A transfer only passes `HoldingPeriodModule` if the lots it consumes are all past their lockup. On the recipient's side, the transferred amount becomes a new lot with the current timestamp. Lots are tracked per wallet. Balances that existed before the module was added count as the oldest holdings and are unlocked, and a wallet recovery moves lots to the replacement wallet with their original timestamps.

Holder caps, ownership caps and vesting track balances per identity rather than per wallet, so add those modules when the share class is deployed, before the first mint.

```
Holder lots (FIFO)          Transfer 1,500
  Lot 1  1,000  2025-03-01  → consumed in full
  Lot 2  2,000  2026-06-15  → 500 consumed, 1,500 remain
```

You can see a holder's lots with `GET /v1/holders/{id}/lots` or in Portfolio.

## Reason codes

| Code | Source |
|---|---|
| `OK` | All checks passed |
| `TOKEN_PAUSED` | Token |
| `FROZEN` | Token |
| `INSUFFICIENT_UNLOCKED_BALANCE` | Token |
| `SENDER_NOT_VERIFIED` | Identity Registry |
| `RECIPIENT_NOT_VERIFIED` | Identity Registry |
| `CLAIM_EXPIRED` | Identity Registry |
| `REGION_BLOCKED` | `RegionModule` |
| `NOT_ELIGIBLE` | `EligibilityModule` |
| `HOLDING_PERIOD_ACTIVE` | `HoldingPeriodModule` |
| `HOLDER_CAP_REACHED` | `HolderCapModule` |
| `WINDOW_CLOSED` | `TransferWindowModule` |
| `ROFR_PENDING` | `ROFRModule` |
| `MAX_OWNERSHIP_EXCEEDED` | `MaxOwnershipModule` |
| `UNVESTED` | `EmployeeVestingModule` |

Plain-language explanations for each code are in [Reason codes](../developers/reason-codes.md).

## Changing rules

Adding, removing or reconfiguring a module goes through the `IssuerTimelock` with a 24-hour minimum delay. Adding or removing emits `ModuleAdded(module)` or `ModuleRemoved(module)`. A share class can have up to 10 modules, and a module must be bound to the share class's engine when it is created. Reconfiguring a module emits `ModuleConfigured(module, configHash)`. Every module's current configuration can be read on-chain and is shown in the [Explorer](../products/explorer.md).
