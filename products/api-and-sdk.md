---
description: The Precursor API and TypeScript SDK give companies, venues and wallets programmatic access to issuance, transfer checks, offerings, liquidity, corporate actions and the indexed on-chain record.
---

# Precursor API and SDK

Everything you can do in the Console, and everything the Explorer shows, is available programmatically. The API is a REST interface over Precursor's contracts and indexer. The TypeScript SDK wraps it, and also ships the ABIs and helpers you need to work with Precursor contracts directly.

## Who uses it

| Integrator | Typical use |
| --- | --- |
| **Companies** | Sync the cap table with internal systems, automate employee onboarding, trigger distributions from treasury tooling, and receive events in their own data warehouse. |
| **Venues and wallets** | List Precursor assets, run `canTransfer` before submitting a trade, read multipliers and Reference Valuations, and explain blocked transfers to users. |
| **Fund administrators** | Reconcile holdings across many companies and generate reports from indexed events. |
| **Service providers** | Build on top of offerings and Liquidity Windows for their own clients. |

## What it covers

- **Companies and share classes.** Create companies, deploy share classes, read supply and configuration.
- **Holders.** Invite holders, read balances and lots, and check identity status.
- **Transfer checks.** `POST /v1/transfers/check` returns the same result the contract would, with a reason code and a plain-language message.
- **Offerings and Liquidity Windows.** Create, monitor and close them, and list their orders.
- **Distributions and proposals.** Fund payouts and run votes.
- **Documents and valuations.** Anchor documents and publish Reference Valuations.
- **Events.** Query the indexed on-chain record with `GET /v1/events`.
- **Contracts.** Look up deployed addresses for each network with `GET /v1/contracts`.
- **Webhooks.** Receive signed notifications when something happens on-chain.

## Design principles

**The chain is the source of truth.** The API never holds state that contradicts the chain. Every write becomes an on-chain transaction, and every read either comes from the chain or from an index of it. Responses include the `block_number` and `tx_hash` that a value came from.

**The same rules everywhere.** A transfer check through the API runs the same modules, with the same configuration, as the on-chain `RuleEngine`. A venue can rely on it to predict whether a trade will settle.

**Approvals still apply.** API keys can prepare actions, but sensitive changes still need signatures in the company's `IssuerTimelock`. An API key cannot pause a token, move holder balances or change rules by itself.

**Predictable behavior.** All write requests accept an `Idempotency-Key`. Lists use cursor pagination. Errors are typed and carry a request id. The API is versioned by date through the `Precursor-Version` header.

## Environments

| Environment | Base URL | Network | Keys |
| --- | --- | --- | --- |
| Live | `https://api.useprecursor.xyz/v1` | Robinhood Chain mainnet (4663) | `sk_live_...`, `pk_live_...` |
| Sandbox | `https://api.sandbox.useprecursor.xyz/v1` | Robinhood Chain testnet (46630) | `sk_test_...` |

The sandbox mirrors the live API on testnet, with test verifiers that issue claims instantly. Companies rehearse every deployment there with an identical configuration before going live.

## A first request

{% tabs %}
{% tab title="TypeScript" %}
```typescript
import { Precursor } from "@precursor/sdk";

const precursor = new Precursor(process.env.PRECURSOR_SECRET_KEY);

const result = await precursor.transfers.check({
  share_class: "cls_8KdT2rPq",
  from: "0x4b2a...91c0",
  to: "0x7e19...0a3d",
  amount: "250",
});

console.log(result.ok, result.reason_code);
```
{% endtab %}

{% tab title="curl" %}
```bash
curl https://api.useprecursor.xyz/v1/transfers/check \
  -H "Authorization: Bearer $PRECURSOR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "share_class": "cls_8KdT2rPq",
    "from": "0x4b2a...91c0",
    "to": "0x7e19...0a3d",
    "amount": "250"
  }'
```
{% endtab %}
{% endtabs %}

## Where to go next

- [Quickstart](../developers/quickstart.md) gets you from an API key to your first sandbox share class.
- [Authentication and environments](../developers/authentication.md) covers keys, versioning and rate limits.
- [API reference](../developers/api-reference.md) documents every endpoint.
- [Webhooks](../developers/webhooks.md) explains event delivery and signature verification.
- [TypeScript SDK](../developers/sdk.md) covers the client and the on-chain helpers.
- [Integrating Precursor assets](../developers/integrating-precursor-assets.md) is the guide for venues and wallets.
