---
description: Every event Precursor contracts emit on Robinhood Chain, with full parameter signatures and guidance for indexing them.
---

# On-chain events

Every state change in Precursor emits an event. Together they form the complete public record of each company: supply, transfers, rule changes, offerings, liquidity, corporate actions and disclosures. The [Explorer](../products/explorer.md) and `GET /v1/events` are both built from these logs, and you can index them yourself from any Robinhood Chain archive node.

Parameters marked `indexed` can be used as topic filters.

## PrecursorFactory

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
event ShareClassDeployed(bytes32 indexed companyId, bytes32 indexed shareClass, address token, address ruleEngine, address valuationOracle);
event PoolGateDeployed(bytes32 indexed companyId, address indexed token, address poolGate);
```

## PrecursorShareToken

```solidity
event Transfer(address indexed from, address indexed to, uint256 value);
event Approval(address indexed owner, address indexed spender, uint256 value);
event SharesIssued(address indexed to, uint256 amount, bytes32 indexed ref);
event SharesRedeemed(address indexed from, uint256 amount, bytes32 indexed ref);
event RecoveryApproved(address indexed lost, address indexed replacement, address indexed approver);
event RecoveryScheduled(address indexed lost, address indexed replacement, uint64 eta, bytes32 ref);
event RecoveryExecuted(address indexed lost, address indexed replacement, uint256 amount);
event RecoveryCancelled(address indexed lost);
event TokensFrozen(address indexed holder, uint256 amount, bytes32 reason);
event TokensUnfrozen(address indexed holder, uint256 amount);
event Paused(bytes32 reason);
event Unpaused();
event MultiplierScheduled(uint256 newMultiplier, uint64 effectiveAt);
event UIMultiplierUpdated(uint256 oldMultiplier, uint256 newMultiplier);
event AgentSet(address indexed agent, bool enabled);
event DocumentsRootUpdated(bytes32 documentsRoot);
```

`SharesIssued` and `SharesRedeemed` are emitted alongside the ERC-20 `Transfer` from or to the zero address. A share class conversion produces a `SharesRedeemed` on one token and a `SharesIssued` on another in the same transaction, with the same `ref`. A recovery produces a `RecoveryExecuted` alongside a `Transfer` from the lost wallet to the replacement.

## RuleEngine

```solidity
event ModuleAdded(address indexed module);
event ModuleRemoved(address indexed module);
event ModuleConfigured(address indexed module, bytes32 configHash);
```

`configHash` is the keccak256 of the encoded module configuration. The decoded configuration is readable from the module itself and shown in the Explorer.

## IdentityRegistry

```solidity
event IdentityRegistered(address indexed wallet, address indexed identity);
event IdentityRemoved(address indexed wallet);
event ClaimAdded(address indexed identity, bytes32 indexed topic, address verifier, uint64 expiry);
event ClaimRevoked(address indexed identity, bytes32 indexed topic);
```

Claim events never contain personal data. They record that a trusted verifier attested to a topic for an identity, and until when.

## OfferingVault

```solidity
event OfferingCreated(uint256 indexed offeringId);
event Subscribed(uint256 indexed offeringId, address indexed investor, uint256 amount);
event OfferingClosed(uint256 indexed offeringId, uint256 raised, uint256 sharesIssued);
event OfferingFailed(uint256 indexed offeringId);
event Refunded(uint256 indexed offeringId, address indexed investor, uint256 amount);
```

`amount` and `raised` are in stablecoin units. `sharesIssued` is in raw token units at 18 decimals.

## LiquidityWindow

```solidity
event WindowOpened(uint256 indexed windowId, uint64 opensAt, uint64 closesAt);
event OrderPlaced(uint256 indexed windowId, uint256 indexed orderId, uint8 side, uint256 quantity, uint256 limitPrice);
event OrderCancelled(uint256 indexed windowId, uint256 indexed orderId);
event WindowCleared(uint256 indexed windowId, uint256 clearingPrice, uint256 volume);
event WindowCancelled(uint256 indexed windowId);
event RofrExercised(uint256 indexed windowId, address indexed holder, uint256 quantity, uint256 price);
event DeliveryDeferred(address indexed shareToken, address indexed owner, uint256 amount);
event DeferredClaimed(address indexed shareToken, address indexed owner, uint256 amount);
```

`side` is `0` for an ask (sell) and `1` for a bid (buy). Individual fills at clearing appear as `Transfer` events on the share token and on the stablecoin within the same transaction as `WindowCleared`. If a token delivery fails the recipient's rules at clearing, the window keeps the tokens for that owner and emits `DeliveryDeferred` instead of reverting; the owner claims them later with `claimDeferred`, which emits `DeferredClaimed`.

## RFQSettlement

```solidity
event QuoteFilled(bytes32 indexed quoteHash, address indexed maker, address indexed taker, uint256 quantity, uint256 price);
event QuoteCancelled(address indexed maker, uint256 nonce);
```

## DistributionVault

```solidity
event DistributionFunded(uint256 indexed distributionId, address asset, uint256 amount, uint256 recordBlock);
event DistributionClaimed(uint256 indexed distributionId, address indexed holder, uint256 amount);
event DistributionSwept(uint256 indexed distributionId, uint256 amount);
```

`recordBlock` is a Robinhood Chain L2 block number, read from the ArbSys precompile, not the L1-derived value `block.number` returns on Arbitrum chains.

## Governance

```solidity
event ProposalCreated(uint256 indexed proposalId, uint256 snapshotBlock, uint64 endsAt);
event VoteCast(uint256 indexed proposalId, address indexed voter, uint8 support, uint256 weight);
event ProposalFinalized(uint256 indexed proposalId, uint8 result);
event DelegateChanged(address indexed delegator, address indexed delegatee);
```

`weight` already includes each share class's multiplier, read when the proposal was created. A `result` of `255` (`NO_QUORUM`) means quorum was not met; otherwise it is the winning option index.

## DisclosureRegistry

```solidity
event DocumentAnchored(uint256 indexed documentId, bytes32 indexed kind, bytes32 hash, string uri);
```

`kind` is one of `SHARE_CLASS_TERMS`, `OFFERING_DOCUMENT`, `FINANCIAL_STATEMENT`, `COMPANY_UPDATE`, `BOARD_RESOLUTION` or `VALUATION_REPORT`, encoded as a `bytes32` string. `hash` is the SHA-256 of the document file.

## ValuationOracle

```solidity
event ValuationPublished(int256 price, uint8 source, uint64 asOf, bytes32 documentHash);
```

## Administrative events

These record configuration changes. Index them if you show a company's rule and signer history.

```solidity
// IssuerTimelock
event ActionProposed(bytes32 indexed id, address indexed proposer, address target, uint256 value, bytes data);
event ActionApproved(bytes32 indexed id, address indexed signer);
event ActionQueued(bytes32 indexed id, uint64 eta);
event ActionExecuted(bytes32 indexed id);
event ActionCancelled(bytes32 indexed id);
event SignerAdded(address indexed signer);
event SignerRemoved(address indexed signer);
event ThresholdChanged(uint256 threshold);
event DelayChanged(bytes4 indexed selector, uint64 delay);

// TrustedVerifiers
event VerifierAdded(address indexed verifier, bytes32[] topics);
event VerifierRemoved(address indexed verifier);

// ROFRModule
event TransferProposed(address indexed from, address indexed to, uint256 amount, uint64 noticeEndsAt);
event RightWaived(address indexed from, address indexed to);
event RightExercised(address indexed from, address indexed to, uint256 amount);

// EmployeeVestingModule
event GrantAdded(address indexed identity, uint256 index, uint64 start, uint64 cliff, uint64 duration, uint256 total);
event GrantAccelerated(address indexed identity, uint256 index);

// PoolGate
event PoolRegistered(address indexed pool);
event PoolRemoved(address indexed pool);

// Fee and venue settings
event FeeUpdated(address recipient, uint16 feeBps);                              // OfferingVault
event FeesUpdated(address recipient, uint16 sellerFeeBps, uint16 buyerFeeBps);   // LiquidityWindow, RFQSettlement
event ShareClassEnabled(address indexed shareToken, bool enabled);               // RFQSettlement
```

## Indexing advice

**Start from the factory.** Index `CompanyDeployed` and `ShareClassDeployed` from the `PrecursorFactory` address returned by `GET /v1/contracts`. Add each new contract to your watch list as it appears. This keeps you complete without hardcoding any company addresses.

**Use an archive endpoint.** Historical reads and backfills need an archive node. The public Robinhood Chain RPC is rate-limited and not suitable for indexing. Use a dedicated provider, as described in [Robinhood Chain](../architecture/robinhood-chain.md).

**Track confirmation.** Logs are available as soon as the sequencer includes a transaction, and become final once the batch is posted to Ethereum. If you serve data from sequencer-confirmed blocks, be ready to handle the rare case where a block is not what was soft-confirmed. Precursor's own API labels each event with `confirmation: "sequencer"` or `"l1"`.

**Key by transaction and log index.** The pair `(tx_hash, log_index)` uniquely identifies every event. Use it for deduplication and idempotent writes.

**Read balances in raw units.** Store raw balances from `Transfer` events and apply `uiMultiplier` at read time. Raw balances never change during a split, so an index built this way stays correct across corporate actions. Record each `UIMultiplierUpdated` so you can show historical share counts at the multiplier in force at the time.

**Join events within a transaction.** Several Precursor actions emit more than one event in a single transaction: offering closes, window clears, conversions and recoveries. Process each transaction's logs together so your view never shows half of an atomic action.

**Reconcile against the API.** If you also consume [webhooks](webhooks.md), `GET /v1/events` with `from_block` is a convenient way to fill gaps. It returns the same events, decoded and labeled with Precursor ids.
