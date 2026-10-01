---
description: How shareholder votes run on-chain through the Governance contract, and how the IssuerTimelock controls changes to a company's configuration.
---

# Governance and voting

Governance on Precursor has two layers. **Shareholder governance** covers votes by token holders on matters the share class allows them to vote on. **Configuration governance** covers changes to the contracts themselves: rules, multipliers, upgrades and recoveries. The first runs through the `Governance` contract. The second runs through the `IssuerTimelock`.

## Shareholder votes

### Proposals

The company creates a proposal in the [Console](../products/console.md) or through `POST /v1/proposals`. A proposal defines:

| Field | Meaning |
|---|---|
| `shareTokens` | Which share classes vote, up to 8 |
| `snapshotBlock` | The Robinhood Chain L2 block at which voting power is measured. It must be in the past when the proposal is created |
| `endsAt` | When voting closes |
| `quorumBps` | Minimum participation, in basis points of eligible voting power |
| `options` | The number of choices, from 2 to 16, for example For, Against and Abstain |
| `binding` | Whether the result is binding or advisory |
| `documentHash` | The anchored proposal document |

Creating a proposal emits `ProposalCreated(proposalId, snapshotBlock, endsAt)`.

### Voting power

A holder's voting power is their balance at the snapshot block multiplied by each share token's multiplier, read when the proposal is created, so votes are counted in shares rather than tokens. Snapshot blocks are L2 block numbers from the ArbSys precompile, because `block.number` on Arbitrum chains tracks L1 and moves far more slowly. Taking a snapshot means tokens bought after the snapshot do not carry votes, and nobody can vote the same shares twice by moving them between wallets.

Frozen tokens and tokens in a holding period or unvested still carry votes. Voting power belongs to the shares, not to their transferability. Tokens escrowed in an open Liquidity Window order at the snapshot do not, because at that block they sit in the window contract.

### Delegation

A holder can delegate their voting power to another verified identity, for example a fund delegating to its investment committee. Delegation is set in Portfolio, or on-chain with `delegate(delegatee)`, and is checkpointed, so it is judged at each proposal's snapshot block. A delegate votes for the holders who had delegated to them at the snapshot with `castVoteFor(proposalId, option, delegators)`. A holder who had a delegate at the snapshot cannot vote directly on that proposal, and a holder who had none votes with `castVote`. Either way, each holder's weight counts once.

### Casting and finalizing

Holders vote in [Portfolio](../products/portfolio.md). Each vote emits `VoteCast(proposalId, voter, support, weight)`, and gas is sponsored. Once `endsAt` passes, anyone can finalize the proposal. `ProposalFinalized(proposalId, result)` records the outcome: the index of the winning option, or `255` (`NO_QUORUM`) if quorum was not met.

Results are public: the totals, the weight behind each option and each `VoteCast` event are all visible in the [Explorer](../products/explorer.md). The identity behind each voting wallet is not.

### Binding and advisory votes

The company sets each proposal as binding or advisory, depending on what the share class terms allow. The result of a binding vote is recorded on-chain, and the company then carries it out through its usual controls, for example by scheduling a multiplier change through the timelock. An advisory vote records holder sentiment. In both cases `Governance` itself executes nothing.

## Configuration governance

Changes to a company's contracts go through the `IssuerTimelock`, a timelocked multi-sig whose signers always include the company. Precursor cannot change a company's configuration on its own.

| Action | Default delay | Minimum |
|---|---|---|
| Recovery of a lost wallet | 72 hours | 48 hours |
| Multiplier change | 48 hours | 48 hours |
| Add or remove a rule module | 24 hours | 24 hours |
| Share token upgrade | 7 days | 7 days |
| Pause and unpause | Immediate | Immediate |

The company can lengthen any delay. It cannot shorten one below its minimum.

Every scheduled action is visible on-chain from the moment it is queued until it executes, which gives holders time to see a change coming. Pausing is the one action without a delay, because it needs to work during an incident. A pause blocks transfers but never moves tokens.

## Who can do what

| Action | `ISSUER_ADMIN` | `REGISTRAR` | `OPERATOR` | Holders |
|---|---|---|---|---|
| Create a proposal | Yes | | | |
| Vote | | | | Yes |
| Finalize a proposal | Anyone | Anyone | Anyone | Anyone |
| Propose a configuration change | Yes | | | |
| Approve a recovery | Yes | Yes | Yes | |
| Pause a token | Yes | Yes | | |

Recovery needs two of the three approving roles. The full permission model is in [Roles and permissions](../architecture/roles-and-permissions.md).

{% hint style="info" %}
Precursor's `OPERATOR` role submits transactions on behalf of companies and holders so that they do not need ETH for gas. It cannot cast votes, move holder tokens or execute configuration changes by itself.
{% endhint %}
