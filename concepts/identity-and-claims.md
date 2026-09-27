---
description: How Precursor links wallets to verified identities using signed, expiring claims, without putting any personal data on-chain.
---

# Identity and claims

A Precursor Share Token can only be held by a wallet that belongs to a verified identity. Precursor handles this with on-chain identities that carry signed claims from trusted verifiers. The chain records that a wallet has been verified, and to what level, but never who the person is.

## The pieces

| Component | Role |
|---|---|
| **Identity** | An address representing one person or entity, either a dedicated identity contract or a stable identifier Precursor assigns. One identity can be linked to several wallets. |
| **Claim** | A signed attestation attached to an identity, with a topic, a value and an expiry. |
| **Verifier** | An identity provider whose signing key is trusted to issue claims on particular topics. |
| `IdentityRegistry` | One per company. Maps wallets to identities, stores the claims about each identity, and answers whether a wallet is verified. |
| `TrustedVerifiers` | One per company. Lists which verifier keys may sign which claim topics. |

Every company gets its own `IdentityRegistry` and `TrustedVerifiers`, deployed by the `PrecursorFactory`. Precursor maintains a default set of verifiers, and the company can add or remove verifiers for its own registry.

## Claim topics

| Topic | Value | Typical issuer | Used by |
|---|---|---|---|
| `IDENTITY_VERIFIED` | `true`, with expiry | Identity verification provider | Every transfer |
| `INVESTOR_ELIGIBLE` | Eligibility tier, with expiry | Eligibility verification provider | `EligibilityModule` |
| `REGION` | ISO 3166 alpha-2 country code | Identity verification provider | `RegionModule` |
| `EMPLOYEE_OF` | The `companyId` of the employer | The company | `EligibilityModule` employee exemption, Liquidity Window seller eligibility |
| `SCREENED` | Last-screened timestamp | Screening provider | Every transfer |
| `VENUE` | The share token's address | Precursor, on the company's approval | Pool and venue contracts, see [Permissioned pools](../liquidity/permissioned-pools.md) |

The registry stores each claim as the verifier's address, an expiry timestamp and the value, keyed by identity and topic. A verifier can submit a claim with `addClaim` directly, or sign it off-chain and let anyone, usually Precursor's relayer, submit it with the signature. Signatures use EIP-712 with the domain name `Precursor`, version `1`, and the type `Claim(address identity,bytes32 topic,bytes32 dataHash,uint64 expiry,bytes32 companyId,address registry)`, so a signature is bound to one company's registry. Each signature can be used only once, so a revoked claim cannot be replayed. Verifiers re-screen identities on a schedule. If re-screening fails, the verifier revokes the claim and the identity can no longer send or receive tokens until it is verified again.

## What gets checked on a transfer

On every transfer the token asks its `IdentityRegistry` two questions, once for the sender and once for the recipient.

1. **Is this wallet linked to an identity?** If not, the transfer fails with `SENDER_NOT_VERIFIED` or `RECIPIENT_NOT_VERIFIED`.
2. **Does that identity hold valid `IDENTITY_VERIFIED` and `SCREENED` claims from a trusted verifier?** An expired claim, or one whose verifier has since been removed from `TrustedVerifiers`, fails with `CLAIM_EXPIRED`. A missing or revoked one fails with the relevant `_NOT_VERIFIED` code. `status(wallet)` returns the same answer as `NOT_VERIFIED`, `CLAIM_EXPIRED` or `VERIFIED`.

Once identity passes, the [Rule Engine](transfer-rules.md) reads any further claims its modules need, such as `REGION` or `INVESTOR_ELIGIBLE`.

Mints are checked against the same registry, so a company cannot mint tokens to an unverified wallet. Burns are not, so the registrar can always redeem shares. Agents, meaning the company's `LiquidityWindow` and contracts holding a `VENUE` claim for the token, skip these checks for their own side of a transfer.

## What stays off-chain

No personal data is ever written to Robinhood Chain. The registry holds only claim topics, values that carry no personal information on their own (a country code, a tier, a timestamp), verifier addresses and expiries.

Names, contact details, documents and the evidence behind each claim stay with the verifier and, where the company needs it, in Precursor's access-controlled systems. The company and its registrar can see who is behind each identity. The public cannot. The full split is described in [What is public, what is private](transparency.md).

## Linking wallets

An identity can hold more than one wallet. This is useful for an investor who keeps tokens in self-custody and in an institutional custody account, or for a person who wants a separate wallet per device.

- **Embedded wallets.** Investors and employees who do not have a wallet get one when they onboard in [Portfolio](../products/portfolio.md). It is linked to their identity automatically.
- **External wallets.** Any EVM wallet can be linked. The holder signs a message proving control of the address, and the `OPERATOR` registers it.
- **Custodial wallets.** Institutional holders can link addresses held with a qualified custodian such as BitGo.

Each link emits `IdentityRegistered(wallet, identity)`, and removing one emits `IdentityRemoved(wallet)`. Holder caps count identities rather than wallets, so linking a second wallet never uses up an additional holder slot.

## Lifecycle events

| Event | Emitted when |
|---|---|
| `IdentityRegistered(wallet, identity)` | A wallet is linked to an identity |
| `IdentityRemoved(wallet)` | A wallet is unlinked |
| `ClaimAdded(identity, topic, verifier, expiry)` | A verifier attaches or renews a claim |
| `ClaimRevoked(identity, topic)` | A verifier revokes a claim |

The API emits `holder.verified` when an invited holder's identity becomes valid, and `holder.claim_expiring` ahead of a claim's expiry, so a company or integrator can prompt the holder to renew in good time. See [Webhooks](../developers/webhooks.md).

{% hint style="warning" %}
A claim that expires does not freeze the holder's tokens. The holder keeps their balance and keeps receiving distributions, but cannot send or receive tokens until the claim is renewed.
{% endhint %}

## Lost wallets

If a holder loses access to a linked wallet, they re-verify their identity and request recovery. The tokens move to a new wallet linked to the same identity through the timelocked recovery process described in [Share tokens](share-tokens.md).
