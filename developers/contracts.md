---
description: Every Precursor contract on Robinhood Chain, what it does, its key functions, and how to discover the addresses for a company.
---

# Contract reference

Precursor is a set of Solidity contracts on Robinhood Chain. One factory deploys a complete, isolated set of contracts for each company, and each share class within a company gets its own token, rule engine and valuation oracle. No company's contracts can read or change another's state.

```
PrecursorFactory
 └── per company
      ├── IdentityRegistry ◄── TrustedVerifiers
      ├── OfferingVault
      ├── LiquidityWindow
      ├── RFQSettlement
      ├── DistributionVault
      ├── Governance
      ├── DisclosureRegistry
      ├── IssuerTimelock
      └── per share class
           ├── PrecursorShareToken
           ├── RuleEngine ──► modules
           ├── ValuationOracle
           └── PoolGate (if permissioned pools are enabled)
```

## Finding addresses

Precursor does not publish addresses on this page. The two authoritative sources are:

* **The API.** `GET /v1/contracts` returns the factory address and module implementations for the network you are calling. `GET /v1/contracts?company=cmp_...` returns every contract in that company's set.
* **The [Explorer](https://explorer.useprecursor.xyz).** Every company page lists its contracts, each linked to the verified source on Robinhood Chain's block explorer.

You can also derive everything from the chain. The factory emits `CompanyDeployed` for each company and `ShareClassDeployed` for each new share class, so indexing those two events from the factory address gives you the full contract set for every issuer.

```solidity
event CompanyDeployed(
    bytes32 indexed companyId,
    address identityRegistry,
    address trustedVerifiers,
    address offeringVault,
    address liquidityWindow,
    address rfqSettlement,
    address distributionVault,
    address governance,
    address disclosureRegistry,
    address issuerTimelock
);

event ShareClassDeployed(
    bytes32 indexed companyId,
    bytes32 indexed shareClass,
    address token,
    address ruleEngine,
    address valuationOracle
);

event PoolGateDeployed(bytes32 indexed companyId, address indexed token, address poolGate);
```

Only the factory owner (Precursor) can call `deployCompany` and `deployShareClass`. The factory holds no role on any contract it deploys. You can also read a share class's engine, oracle and gate with `shareClassOf(token)`.

All contracts are verified on Blockscout at `robinhoodchain.blockscout.com`.

## PrecursorShareToken

One token per share class. It is a standard ERC-20 with 18 decimals, compatible with ERC-3643 for permissioned transfers and with ERC-8056 for corporate-action scaling. Every transfer, whether through `transfer`, `transferFrom` or a Precursor settlement contract, calls `canTransfer` and reverts with the returned reason code if the check fails.

The token sits behind a UUPS proxy. Upgrades are proposed through `IssuerTimelock` and carry a seven-day delay.

```solidity
interface IPrecursorShareToken is IERC20, IERC3643, IERC8056 {
    // Identity & transfer rules
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

    // Snapshots (L2 block numbers on Robinhood Chain)
    function clock() external view returns (uint48);
    function balanceOfAt(address account, uint48 timepoint) external view returns (uint256);
    function totalSupplyAt(uint48 timepoint) external view returns (uint256);
}
```

`ref` is a free-form `bytes32` that links an action to the off-chain record behind it, such as a cap table entry or a document hash. The name is `ref` because `reference` is a reserved word in Solidity.

| Function | Who can call | Notes |
|---|---|---|
| `mint` | `REGISTRAR`, or `MINTER` (the `OfferingVault`) | Runs the full rule check on the recipient |
| `burn` | `REGISTRAR` | `ref` links to the cap table record and its document hash |
| `approveRecovery` | `ISSUER_ADMIN`, `REGISTRAR` or `OPERATOR` | The second approval from a different party schedules the recovery after a 72-hour default delay (48-hour floor) |
| `cancelRecovery` | `ISSUER_ADMIN`, `REGISTRAR` or `OPERATOR` | Cancels a pending recovery at any point before execution |
| `recover` | `REGISTRAR` | Executes a scheduled recovery once the delay has passed |
| `freezePartial`, `unfreezePartial` | `REGISTRAR` | Frozen tokens stay in the wallet but cannot move |
| `pause`, `unpause` | `PAUSER` | Immediate. Pausing blocks every transfer and halts the valuation oracle |
| `scheduleMultiplier`, `cancelMultiplier` | `ISSUER_ADMIN` | The effective time must be at least 48 hours out |
| `applyMultiplier` | Anyone | Finalizes a multiplier that is already in effect and emits `UIMultiplierUpdated` |
| `setAgent`, `setDocumentsRoot`, `setRecoveryDelay`, `setMultiplierDelay`, `upgradeToAndCall` | `IssuerTimelock` | Delays below the floors are rejected |
| `unlockedBalanceOf` | Anyone | Balance not held back by freezes, holding periods or vesting |
| `isAgent` | Anyone | True for the company's `LiquidityWindow` and for contracts holding a valid `VENUE` claim for this token |

Raw balances never change during a corporate action. Multiply by `uiMultiplier() / 1e18` to get shares. See [Corporate actions](../concepts/corporate-actions.md).

## IdentityRegistry

Maps wallets to on-chain identities for one company. An identity holds signed claims, and the registry accepts a claim only if it is signed by a verifier listed in `TrustedVerifiers` for that topic.

| Function | Purpose |
|---|---|
| `registerIdentity(address wallet, address identity)` | Links a wallet to an identity. Called by `OPERATOR` after verification |
| `removeIdentity(address wallet)` | Unlinks a wallet. Called by `OPERATOR` |
| `identityOf(address wallet) returns (address)` | The identity for a wallet, or zero |
| `addClaim(address identity, bytes32 topic, bytes data, uint64 expiry, address verifier, bytes signature)` | Stores a claim. The verifier can submit it directly with an empty signature, or anyone can relay it with the verifier's EIP-712 signature. Each signature can be used once |
| `revokeClaim(address identity, bytes32 topic)` | Deletes a claim. Called by the claim's verifier or by `OPERATOR` |
| `claimOf(address identity, bytes32 topic) returns (Claim)` | The stored verifier, expiry and data |
| `claimDigest(address identity, bytes32 topic, bytes data, uint64 expiry) returns (bytes32)` | The EIP-712 digest a verifier signs |
| `status(address wallet) returns (Status)` | `NOT_VERIFIED`, `CLAIM_EXPIRED` or `VERIFIED`, the identity check every transfer runs |
| `isVerified(address wallet) returns (bool)` | True if the identity holds valid `IDENTITY_VERIFIED` and `SCREENED` claims |
| `hasValidClaim(address wallet, bytes32 topic) returns (bool)` | True if the claim exists, is signed by a trusted verifier and has not expired |
| `claimData(address wallet, bytes32 topic) returns (bytes)` | The claim value, such as an ISO 3166 code for `REGION` |

Signed claims use the EIP-712 domain name `Precursor`, version `1`, and this type:

```solidity
Claim(address identity,bytes32 topic,bytes32 dataHash,uint64 expiry,bytes32 companyId,address registry)
```

`dataHash` is the keccak256 of the claim data. A claim whose verifier has since been removed from `TrustedVerifiers` reports `CLAIM_EXPIRED`.

## TrustedVerifiers

Lists which verifier keys may sign which claim topics for a company. Precursor maintains a default set of identity providers, and the issuer can add or remove verifiers through `IssuerTimelock`. Main functions: `addVerifier(address verifier, bytes32[] topics)`, `removeVerifier(address verifier)` and `isTrusted(address verifier, bytes32 topic)`.

## RuleEngine and modules

One rule engine per share class. `check(from, to, amount)` runs each enabled module in order and returns the first failure. Modules are separate contracts, so the issuer can add, remove or reconfigure them without upgrading the token. Changes go through `IssuerTimelock` with a 24-hour default delay.

| Module | Reason code |
|---|---|
| `RegionModule` | `REGION_BLOCKED` |
| `EligibilityModule` | `NOT_ELIGIBLE` |
| `HoldingPeriodModule` | `HOLDING_PERIOD_ACTIVE` |
| `HolderCapModule` | `HOLDER_CAP_REACHED` |
| `TransferWindowModule` | `WINDOW_CLOSED` |
| `ROFRModule` | `ROFR_PENDING` |
| `MaxOwnershipModule` | `MAX_OWNERSHIP_EXCEEDED` |
| `EmployeeVestingModule` | `UNVESTED` |

Main functions: `modules() returns (address[])`, `addModule(address module, bytes config)`, `removeModule(address module)`, `configureModule(address module, bytes config)`. Module behavior is described in [Transfer rules](../concepts/transfer-rules.md).

## OfferingVault

Runs primary offerings. Investors deposit stablecoins, funds stay escrowed, and at close the vault either releases funds and mints shares in one transaction or opens refunds.

`createOffering(OfferingParams params) returns (uint256 offeringId)`, `subscribe(uint256 offeringId, uint256 amount)`, `close(uint256 offeringId)`, `cancelOffering(uint256 offeringId)`, `refund(uint256 offeringId)`, `state(uint256 offeringId) returns (State)`, `offering(uint256 offeringId) returns (Offering)`. An offering accepts at most 500 subscribers (`MAX_SUBSCRIBERS`), and every subscriber is checked against the share class rules again at close. See [Primary offerings](../liquidity/offerings.md).

## LiquidityWindow

Runs issuer-controlled uniform-price batch auctions and fixed-price tenders.

`openWindow(WindowParams params) returns (uint256 windowId)`, `placeOrder(uint256 windowId, uint8 side, uint256 quantity, uint256 limitPrice) returns (uint256 orderId)` where `side` is `0` for `SELL` and `1` for `BUY`, `cancelOrder(uint256 windowId, uint256 orderId)`, `removeOrder(uint256 windowId, uint256 orderId)` (company only, between close and clearing), `previewClearing(uint256 windowId) returns (uint256 clearingPrice, uint256 volume)`, `exerciseRofr(uint256 windowId, uint256 quantity)`, `clear(uint256 windowId)`, `cancelWindow(uint256 windowId)`, `claimDeferred(address shareToken)`. Anyone can call `clear` once the window and any ROFR period have ended. A window accepts at most 200 orders (`MAX_ORDERS`). See [Liquidity Windows](../liquidity/liquidity-windows.md).

## RFQSettlement

Settles EIP-712 signed quotes between eligible holders with atomic delivery versus payment. The domain uses name `Precursor`, version `1`, the chain id and the contract address.

```solidity
struct Quote {
    address maker;
    address shareToken;
    uint8 side;          // 0 = maker sells, 1 = maker buys
    uint256 quantity;
    uint256 price;       // stablecoin units per share
    address stablecoin;
    address taker;       // zero address = any eligible taker
    uint256 nonce;
    uint64 expiry;
}

function fill(Quote calldata quote, bytes calldata signature, uint256 quantity) external;
function cancelQuote(uint256 nonce) external;
function invalidateNoncesBelow(uint256 newMinNonce) external;
function hashQuote(Quote calldata quote) external view returns (bytes32);
function setEnabled(address shareToken, bool on) external; // ISSUER_ADMIN
```

The EIP-712 type string is `Quote(address maker,address shareToken,uint8 side,uint256 quantity,uint256 price,address stablecoin,address taker,uint256 nonce,uint64 expiry)`.

See [RFQ trading](../liquidity/rfq.md).

## PoolGate

A hook contract for permissioned AMM pools. A pool calls `beforeSwap(recipient, sharesOut)`, `beforeAddLiquidity(provider)` and `beforeTransferPosition(from, to)`, which revert with `TransferNotAllowed(reasonCode)` if the end recipient or provider fails the share class rules. `checkSwap` and `checkLiquidityProvider` return the reason code without reverting. The company registers pools with `registerPool`, through `IssuerTimelock`; a pool must already hold a `VENUE` claim, and share classes with a `ROFRModule` cannot register pools. See [Permissioned pools](../liquidity/permissioned-pools.md).

## DistributionVault

Pays dividends and other proceeds. `fund(address shareToken, address asset, uint256 amount, uint48 recordBlock, uint64 claimDeadline, address[] excludedAccounts) returns (uint256 distributionId)`, `claim(uint256 distributionId)`, `claimFor(uint256 distributionId, address holder)`, `claimable(uint256 distributionId, address holder) returns (uint256)`, `sweep(uint256 distributionId)`. Entitlements are computed from balances at the record block, which is a Robinhood Chain L2 block number and must be in the past. Excluded accounts (up to 20, such as treasury shares or the `LiquidityWindow` escrow) are removed from the eligible supply. Holders claim at any time before the deadline; a deadline, if set, is at least 12 months out, and after it passes `ISSUER_ADMIN` can `sweep` the remainder to the treasury.

## Governance

Token-weighted votes with a snapshot block. `propose(ProposalParams params) returns (uint256 proposalId)`, `castVote(uint256 proposalId, uint8 option)`, `castVoteFor(uint256 proposalId, uint8 option, address[] delegators)`, `delegate(address delegatee)`, `finalize(uint256 proposalId)`. `ProposalParams` holds `shareTokens`, `snapshotBlock`, `endsAt`, `quorumBps`, `options`, `binding` and `documentHash`. A result of `255` (`NO_QUORUM`) means quorum was not met. See [Governance and voting](../concepts/governance.md).

## DisclosureRegistry

Anchors document hashes. `anchor(bytes32 kind, bytes32 hash, string uri) returns (uint256 documentId)` and `document(uint256 documentId) returns (Document)`. The registry never holds document contents.

## ValuationOracle

Publishes [Reference Valuations](../concepts/reference-valuation.md) per share class. It exposes the Chainlink `AggregatorV3Interface` read functions (`latestRoundData`, `decimals`, `description`) so existing integrations can read it unchanged, plus a richer call:

```solidity
function latestValuation()
    external view
    returns (int256 price, uint8 source, uint64 asOf, bytes32 documentHash);
```

Prices are in USD with 8 decimals. `source` is `0` for `PRICED_ROUND`, `1` for `INDEPENDENT_VALUATION` and `2` for `WINDOW_CLEARING`.

## IssuerTimelock

A timelocked multi-sig that executes sensitive actions for a company. The issuer's signers are always part of it. Default delays: recovery 72 hours, multiplier change 48 hours, module changes 24 hours, token upgrade seven days. Pausing never waits. See [Roles and permissions](../architecture/roles-and-permissions.md).

Event signatures for every contract are listed in [On-chain events](events.md).
