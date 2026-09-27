---
description: The Precursor Share Token, the ERC-20 that represents one share class of a company, and the design rules that govern supply, recovery, pausing and upgrades.
---

# Share tokens

A Precursor Share Token is the on-chain representation of one share class of one company. If a company has Common, Series A Preferred and Series B Preferred shares, it has three share tokens. Each token is its own contract, with its own rules, its own multiplier and its own Reference Valuation.

## Standards

Every share token implements three standards.

| Standard | What it provides |
|---|---|
| **ERC-20** | Balances, transfers and allowances that standard wallets and libraries understand. 18 decimals. |
| **ERC-3643** | A permissioned token model. Transfers check holder identity and compliance rules before they settle. |
| **ERC-8056** | `uiMultiplier()`, which scales the number of shares each token represents without rebasing balances. |

When the multiplier is `1e18` (1.0), one token equals one share. If the company runs a 2-for-1 split, the multiplier becomes `2e18`, and every holder's existing balance now represents twice as many shares. Raw balances do not change. See [Corporate actions](corporate-actions.md).

## Interface

```solidity
interface IPrecursorShareToken is IERC20, IERC3643, IERC8056 {
    // Identity and transfer rules
    function identityRegistry() external view returns (IIdentityRegistry);
    function ruleEngine() external view returns (IRuleEngine);
    function canTransfer(address from, address to, uint256 amount)
        external view returns (bool ok, bytes32 reasonCode);

    // Corporate actions (ERC-8056)
    function uiMultiplier() external view returns (uint256); // 1e18 = 1.0

    function isAgent(address account) external view returns (bool);

    // Issuer controls (role-gated, timelocked where noted, all emit events)
    function mint(address to, uint256 amount, bytes32 ref) external;
    function burn(address from, uint256 amount, bytes32 ref) external;
    function approveRecovery(address lost, address replacement, bytes32 ref) external;
    function cancelRecovery(address lost) external;
    function recover(address lost, address replacement, bytes32 ref) external;
    function freezePartial(address holder, uint256 amount, bytes32 reason) external;
    function unfreezePartial(address holder, uint256 amount) external;
    function pause(bytes32 reason) external;
    function unpause() external;
    function scheduleMultiplier(uint256 newMultiplier, uint64 effectiveAt) external;
    function applyMultiplier() external;

    // Metadata
    function shareClass() external view returns (bytes32);
    function companyId() external view returns (bytes32);
    function documentsRoot() external view returns (bytes32);
    function unlockedBalanceOf(address holder) external view returns (uint256);

    // Snapshots
    function clock() external view returns (uint48);
    function balanceOfAt(address account, uint48 timepoint) external view returns (uint256);
    function totalSupplyAt(uint48 timepoint) external view returns (uint256);
}
```

The full reference, including every event, is in the [Contract reference](../developers/contracts.md).

## How a transfer works

Every movement of tokens, whether through `transfer`, `transferFrom`, a Liquidity Window settlement or an RFQ fill, passes through the same path.

1. The token checks it is not paused, that the sender's wallet is not under a scheduled recovery, and that the sender's balance, less any frozen amount, covers the transfer. Tokens placed in a Liquidity Window order have already moved into the window's escrow, so they are no longer in the sender's balance.
2. The [Identity Registry](identity-and-claims.md) confirms that both sender and recipient are linked to identities with valid, unexpired `IDENTITY_VERIFIED` and `SCREENED` claims. Agents skip this step for their own side: the company's `LiquidityWindow` and any venue holding a `VENUE` claim for the token, so the rules apply to the people at each end of a trade rather than to the escrow in between.
3. The [Rule Engine](transfer-rules.md) runs every module the company has enabled for that share class.
4. If every check passes, balances move and the modules update their state, for example holder counts and lot records.

If any check fails, the transfer reverts with `TransferNotAllowed(reasonCode)`. The same logic is available as a view through `canTransfer`, which returns `(false, reasonCode)` without spending gas. Portfolio and the API call it before asking anyone to sign, so a user sees a clear explanation such as `HOLDING_PERIOD_ACTIVE` rather than a failed transaction. The full list is in [Reason codes](../developers/reason-codes.md).

## Design rules

### One token per share class

Rights, rules and valuation differ between classes, so each class gets its own contract. This also matches Robinhood Chain's one-token-per-asset convention, which keeps price feeds and multipliers simple for integrators.

### No silent supply changes

Supply only changes through `mint` and `burn`. Both take a `ref` argument. `burn` requires the `REGISTRAR` role; `mint` requires `REGISTRAR` or `MINTER`, which only the company's `OfferingVault` holds, and runs the full rule check on the recipient. Each call emits `SharesIssued(to, amount, ref)` or `SharesRedeemed(from, amount, ref)`. The `ref` points to a record anchored in the `DisclosureRegistry`, such as the board resolution or offering that authorized the change. Anyone can reconcile total supply against those events.

### Recovery, not custody

Precursor never holds a holder's tokens. If a holder loses access to a wallet, the tokens can be moved to a replacement wallet after the holder re-verifies their identity. Two of three parties (`ISSUER_ADMIN`, `REGISTRAR` and `OPERATOR`) each call `approveRecovery(lost, replacement, ref)`, which emits `RecoveryApproved`. The replacement wallet must already be verified. The second approval from a different party schedules the recovery after a delay of 72 hours by default (never less than 48) and emits `RecoveryScheduled(lost, replacement, eta, ref)`, so the pending recovery is publicly visible before it executes. While it is pending, the lost wallet cannot transfer. The holder of the original wallet can still raise an objection during that period, and any of the three parties can call `cancelRecovery`, which emits `RecoveryCancelled`. After the delay, the registrar calls `recover` to move the balance, frozen amount and lot history.

### Freezes are partial and visible

`freezePartial` locks a specific amount in a specific wallet, for example while a dispute is resolved. It emits `TokensFrozen(holder, amount, reason)`. Frozen tokens stay in the holder's wallet and keep receiving distributions, but cannot be transferred until `unfreezePartial` is called.

### Pausing is immediate and explained

A `PAUSER` (the issuer or the registrar) can pause a share token instantly, for example during a corporate action or an incident. `pause` takes a reason, emits `Paused(reason)`, and causes every transfer to fail with `TOKEN_PAUSED`. The `ValuationOracle` holds its last published value while the token is paused.

### Limited upgradeability

Token logic sits behind a UUPS proxy. Upgrades go through the `IssuerTimelock`, a multi-sig that includes the issuer, with a delay of 7 days. Rule modules are separate contracts, so changing a rule never requires upgrading the token.

## Metadata

| Function | Returns |
|---|---|
| `companyId()` | The company's identifier, shared by all its share classes |
| `shareClass()` | The class name as `bytes32`, for example `SERIES_B_PREFERRED` |
| `documentsRoot()` | The Merkle root of the class's governing documents, anchored in the `DisclosureRegistry` |
| `uiMultiplier()` | Shares per token, scaled by `1e18` |
| `unlockedBalanceOf(holder)` | The balance the holder can transfer right now |

Unlocked balance is the wallet's balance minus frozen tokens and the amounts the Rule Engine's modules report as locked (holding periods and unvested shares). It is zero while a recovery is scheduled. It is a conservative figure: other rules, such as region or holder cap, still apply at transfer time. The API endpoint `GET /v1/holders/{id}/lots` shows the full picture for a holder.
