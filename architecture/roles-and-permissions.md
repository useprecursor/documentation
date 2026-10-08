---
description: The roles that can act on a company's contracts, what each one can and cannot do, how wallet recovery works, and the timelocks that sit in front of sensitive changes.
---

# Roles and permissions

Every privileged action on a Precursor contract is tied to a role, and every role belongs to a specific party. No party holds every role. The company controls configuration, the registrar controls the share record, and Precursor operates the infrastructure. Actions that affect a holder's balance need more than one of them to agree.

## The roles

| Role | Held by | Can | Cannot |
|---|---|---|---|
| `DEFAULT_ADMIN_ROLE` | The company's `IssuerTimelock` | Add, remove and configure transfer rule modules, upgrade share tokens, change verifiers, fees, agents and delays, grant and revoke roles | Act without its signers' approval and the delay for that function |
| `ISSUER_ADMIN` | The company's operational multi-sig | Create offerings, Liquidity Windows, distributions and votes, schedule multiplier changes, enable RFQ, anchor documents, publish valuations, approve recoveries | Mint or burn shares outside an offering, change transfer rules directly, or move a holder's tokens alone |
| `REGISTRAR` | The registrar, meaning the company itself or an administrator it appoints | Mint and burn against cap table records, execute share class conversions, freeze and unfreeze, add vesting grants, approve and execute recoveries | Change transfer rules or offering terms |
| `OPERATOR` | Precursor, through a per-company HSM or MPC relayer | Register and unlink holder identities, revoke claims, co-approve recoveries, relay transactions and sponsor gas | Move a holder's tokens alone, mint, freeze, change any configuration, or bypass the `RuleEngine` |
| `PAUSER` | The company (`ISSUER_ADMIN`) and the registrar | Call `pause` and `unpause` on a share token | Do anything else |
| `MINTER` | The company's `OfferingVault` | Mint shares to subscribers when an offering closes | Burn, freeze or mint outside an offering's settlement |
| Verifier | Identity providers listed in `TrustedVerifiers` | Sign claims for the topics they are trusted for | Hold, move or configure tokens |

Role assignments live on-chain and can be read by anyone. The Explorer lists the current holders of each role for every company, and every change emits an event.

## Scope of the operator role

The `OPERATOR` role exists so that holders never have to think about gas and companies never have to run infrastructure. It is limited by design:

* Operator transactions go through the same `RuleEngine` checks as any other transaction.
* The operator cannot mint, burn, freeze or reconfigure anything. Its on-chain powers are linking wallets to identities, revoking claims and one of three recovery approvals.
* The operator's only part in a recovery is one of three approvals, and a recovery still waits out its timelock.

If Precursor's infrastructure were unavailable, holders could still transfer by calling the share token directly from their own wallets, paying their own gas.

## Wallet recovery

Holders sometimes lose access to a wallet. Precursor restores their position by moving it to a new wallet linked to the same identity. Nobody takes custody of it along the way.

{% stepper %}
{% step %}
### Re-verify the holder
The holder confirms their identity again with a verification provider. The new wallet is registered to their existing identity in the `IdentityRegistry`.
{% endstep %}

{% step %}
### Collect two of three approvals
Any two of `ISSUER_ADMIN`, `REGISTRAR` and `OPERATOR` call `approveRecovery(lost, replacement, ref)` on the share token. Each call emits `RecoveryApproved`. The second approval from a different party emits `RecoveryScheduled` with the time it can execute. The same account cannot approve twice, even if it holds more than one role.
{% endstep %}

{% step %}
### Wait out the timelock
The recovery is visible on-chain for the full delay, 72 hours by default. If the original wallet is still in the holder's control, any transfer it makes during this period is blocked, and any of the three parties can cancel the recovery, which emits `RecoveryCancelled`.
{% endstep %}

{% step %}
### Execute
After the delay, the registrar calls `recover(lost, replacement, ref)`. The full balance, any frozen amount and the lot history move to the replacement wallet. `RecoveryExecuted` records the amount. Holding periods and vesting schedules carry over unchanged.
{% endstep %}
{% endstepper %}

The `ref` field links the recovery to the registrar's record, and the record's hash is anchored in the `DisclosureRegistry`.

## Timelocks

Changes that could surprise a holder wait before they take effect. Most go through the `IssuerTimelock`, which needs M-of-N signer approval and then applies a delay that depends on the function being called. Recoveries and multiplier changes are delayed by the share token itself. Each company can lengthen the defaults, but never go below the minimum.

| Action | Default delay | Bounds |
|---|---|---|
| Wallet recovery | 72 hours | Minimum 48 hours |
| Multiplier change (split or reverse split) | 48 hours | Minimum 48 hours |
| Adding, removing or configuring a transfer rule module | 24 hours | Minimum 24 hours |
| Share token upgrade | 7 days | Minimum 7 days |
| Any other `IssuerTimelock` action, such as role grants, verifier, fee and agent changes, signer changes | 48 hours | Minimum 48 hours |
| Pause | None, takes effect immediately | Not configurable |
| Unpause | None, takes effect immediately | Not configurable |

Pausing is immediate because it is a safety control. Every other delay exists so that holders, venues and integrators can see a change coming and react before it applies. Pending actions appear in the Explorer with their execution time, and webhook endpoints receive events such as `multiplier.scheduled` and `recovery.scheduled`.

## Rotating signers

Companies change: officers leave, board members rotate, and multi-sig keys need replacing. The `IssuerTimelock` keeps its own signer list and threshold, which change only through the timelock itself (`addSigner`, `removeSigner`, `setThreshold`). Removing a signer also removes their approvals from every pending action. The `ISSUER_ADMIN` multi-sig is a separate wallet, so rotating one of its signers is a change to that multi-sig, not to the Precursor contracts.

1. In the Console, open **Approvals → Signers** and propose the change: add a signer, remove one, or change the approval threshold.
2. The existing signers approve the change at the current threshold.
3. The change goes through the `IssuerTimelock` and executes after its delay. The Explorer shows the pending change while it waits.

Rotating the registrar works the same way, but the outgoing and incoming registrar both have to approve before the `REGISTRAR` role moves. That way the share record never sits with a party that hasn't accepted it.

Precursor rotates operator keys on a fixed schedule and immediately after any suspected compromise. Rotation replaces the `OPERATOR` assignment for the affected company. Companies don't need to take any action, and the change is recorded on-chain like every other role update.

For how these controls fit into Precursor's wider security model, see [Security](security.md).
