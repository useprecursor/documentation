---
description: How Precursor secures its contracts and infrastructure, including audits, formally verified invariants, role separation, pause controls, key management, monitoring and incident response.
---

# Security

Precursor contracts hold the record of who owns a company. The security model starts from that fact: any balance change must be authorized, rule-checked and visible, and no single party, Precursor included, can change ownership on its own.

## Audits and formal verification

Independent firms audit every Precursor contract before it is deployed to mainnet, and they audit every upgrade before it is proposed. Audit reports are published in full and linked from the Explorer page for each contract version.

The `RuleEngine` and the share token's transfer path are also formally verified. Rather than relying only on tests, the verification proves that the following properties hold for every possible sequence of calls.

| Invariant | What it guarantees |
|---|---|
| No unchecked transfer | Every change to a holder's balance passes through the `RuleEngine`, except mints, burns and recoveries, which have their own role checks. |
| Verified holders only | No balance can ever be credited to a wallet without a registered identity in the company's `IdentityRegistry`. |
| Supply accounting | Total supply always equals the sum of all balances, and it changes only through `mint` and `burn`. |
| Lots are conserved | The sum of a holder's lots always equals their balance, and holding periods cannot be shortened by a transfer. |
| Frozen means frozen | A holder can never transfer more than `unlockedBalanceOf`, which excludes frozen and locked amounts. |
| Pause is total | While a token is paused, no transfer, mint, burn or settlement involving it can succeed. |
| Atomic settlement | In `OfferingVault`, `LiquidityWindow` and `RFQSettlement`, tokens and stablecoins move in the same transaction, or neither moves. |
| Recovery is delayed | A recovery can execute only after two of the three required approvals and the full timelock. |

The test suite adds unit, fuzz and stateful invariant tests on top of these proofs. It runs on every change in continuous integration.

## Bug bounty

Precursor runs a public bug bounty covering the contracts, the API and the hosted applications. Report vulnerabilities to **security@useprecursor.xyz**. Include a description, the steps to reproduce, and the affected contract addresses or endpoints. We acknowledge reports within one business day.

Please don't test against mainnet contracts or other people's accounts. Use the sandbox on Robinhood Chain testnet, described in [Authentication and environments](../developers/authentication.md).

## Role separation

No party holds every permission. The company's admin multi-sig, the registrar and Precursor's operator are separate signers with separate keys. The actions that matter most need agreement between them:

* Wallet recoveries need two of the three, then a timelock.
* Upgrades to a share token need the company's multi-sig and a seven-day timelock.
* Transfer rule changes need the company's multi-sig and a timelock.

The full matrix is in [Roles and permissions](roles-and-permissions.md).

## Circuit breaker

Every share token can be paused by its `PAUSER` role holders, meaning the company and the registrar. A pause takes effect in the next block and stops all transfers and settlements for that share class. The call takes a `reason` code, which is published in the `Paused` event and shown in the Explorer. When a token is paused, the `ValuationOracle` stops publishing new values and holds the last one, so integrations that read it see a stable price.

Pausing one share class doesn't affect a company's other classes or any other company.

## Testnet rehearsal

Every company deployment is rehearsed on Robinhood Chain testnet (chain ID `46630`) before mainnet. The rehearsal uses the identical configuration, meaning the same modules, parameters, roles and timelocks. Precursor replays the company's cap table, a sample of transfers that should pass and fail, an offering close and a corporate action. Mainnet deployment starts only after the company signs off on the rehearsal results in the Console.

## Upgrade policy

Share tokens sit behind UUPS proxies, and the `IssuerTimelock` controls them, with the company's multi-sig among its signers. Precursor can propose an upgrade, but only the company can approve it, and it waits seven days before it executes. During that window the new implementation, its audit report and a diff are published in the Explorer.

Transfer rule modules can be replaced without upgrading the token. Swapping a module also goes through the timelock. The other Precursor contracts are not upgradeable. New versions are deployed alongside old ones, and companies migrate to them when they choose.

## Key management

* **Operator keys** are generated and kept in hardware security modules or split across an MPC cluster. No complete key ever exists on a single machine. Each company has its own operator signer, so a problem with one cannot affect another.
* **Claim-signing keys** used to write identity claims are isolated from operator keys, rate-limited, and scoped to the claim topics they are trusted for in `TrustedVerifiers`.
* **Company keys** belong to the company's multi-sig. Precursor never sees or holds them.
* **Holder keys** belong to holders. Embedded wallets are non-custodial, and their key shares are not held by Precursor alone.

Keys are rotated on a fixed schedule and immediately after any suspected compromise.

## Monitoring

Precursor watches its own contracts continuously:

* **On-chain invariant monitors** check every block for conditions that should never happen, for example a balance held by an unverified wallet, or a mismatch between total supply and the registrar's record.
* **Role and configuration watchers** alert on every role change, module change, pause and scheduled recovery, including ones Precursor did not initiate.
* **Infrastructure monitors** track RPC provider health, sequencer feed lag, queue depth and webhook delivery.

Live system status, including chain connectivity and API availability, is published at the public status page linked from [useprecursor.xyz](https://useprecursor.xyz).

## Incident response

When a monitor fires or a report comes in, an on-call engineer triages it at once. If holder assets could be at risk:

1. Precursor contacts the affected company's `PAUSER` holders, who can pause the share class immediately. Precursor cannot pause a company's tokens on its own.
2. The issue is contained and fixed. A fix that changes contract logic goes through the normal audit and timelock process. The timelock is never bypassed.
3. Affected companies are told directly, and holders see a notice in Portfolio.
4. A written post-incident report is published, covering the timeline, root cause and remediation, and it is linked from the status page.
