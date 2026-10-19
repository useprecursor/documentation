---
description: Make your first Precursor API calls in the sandbox, from checking a transfer to deploying a test share class and receiving a webhook.
---

# Quickstart

This guide takes you from an empty project to a working integration against the Precursor sandbox. The sandbox runs on Robinhood Chain testnet (chain ID `46630`), so everything you do here produces real on-chain transactions without touching real shares or real money.

By the end you will have:

* authenticated with a sandbox secret key
* installed the TypeScript SDK
* checked whether a transfer would pass a share class's rules
* deployed a test share class to testnet
* received and verified a webhook

## Before you begin

You need a Precursor account with sandbox access. Sign in to the [Console](https://app.useprecursor.xyz/console), open **Developers → API keys**, and create a sandbox secret key. Sandbox secret keys start with `sk_test_`.

{% hint style="warning" %}
Secret keys can create offerings, invite holders and move company configuration. Keep them on your server, never in a browser or a mobile app, and never in source control.
{% endhint %}

Store the key in an environment variable:

```bash
export PRECURSOR_SECRET_KEY="sk_test_51Hx...redacted"
```

{% stepper %}
{% step %}
### Install the SDK

The SDK requires Node.js 18 or later.

```bash
npm install @precursor/sdk
```

Create a client. The SDK reads the environment from the key prefix, so a `sk_test_` key always talks to `https://api.sandbox.useprecursor.xyz/v1`.

{% code title="precursor.ts" %}
```ts
import { Precursor } from "@precursor/sdk";

export const precursor = new Precursor(process.env.PRECURSOR_SECRET_KEY!);
```
{% endcode %}
{% endstep %}

{% step %}
### Find your sandbox company

Every sandbox account comes with a demo company that already has a share class, a handful of verified holders and an open Liquidity Window. List your companies to get its id.

{% tabs %}
{% tab title="curl" %}
```bash
curl https://api.sandbox.useprecursor.xyz/v1/companies \
  -H "Authorization: Bearer $PRECURSOR_SECRET_KEY" \
  -H "Precursor-Version: 2026-10-01"
```
{% endtab %}

{% tab title="TypeScript" %}
```ts
const companies = await precursor.companies.list({ limit: 10 });
const demo = companies.data[0];
console.log(demo.id, demo.name); // cmp_8f2kQ1xZ "Northwind Robotics (Sandbox)"
```
{% endtab %}
{% endtabs %}

The response follows the standard list format:

```json
{
  "data": [
    {
      "id": "cmp_8f2kQ1xZ",
      "object": "company",
      "name": "Northwind Robotics (Sandbox)",
      "network": "robinhood_chain_testnet",
      "chain_id": 46630,
      "company_id_onchain": "0x6e6f727468...",
      "share_classes": ["cls_3VbT9pLm"],
      "created_at": "2026-10-01T14:02:11Z"
    }
  ],
  "has_more": false
}
```
{% endstep %}

{% step %}
### Check a transfer

Before you build any transaction that moves a Precursor Share Token, ask whether it would pass. The check runs the same rules the token enforces on-chain, so a result of `ok: true` means the transfer will succeed if nothing changes before it lands.

{% tabs %}
{% tab title="curl" %}
```bash
curl https://api.sandbox.useprecursor.xyz/v1/transfers/check \
  -H "Authorization: Bearer $PRECURSOR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 7d1c2f0e-quickstart-check" \
  -d '{
    "share_class": "cls_3VbT9pLm",
    "from": "0x1111111111111111111111111111111111111111",
    "to": "0x2222222222222222222222222222222222222222",
    "amount": "250"
  }'
```
{% endtab %}

{% tab title="TypeScript" %}
```ts
const check = await precursor.transfers.check({
  share_class: "cls_3VbT9pLm",
  from: "0x1111111111111111111111111111111111111111",
  to: "0x2222222222222222222222222222222222222222",
  amount: "250",
});

if (!check.ok) {
  console.log(check.reason_code, check.message);
}
```
{% endtab %}
{% endtabs %}

The recipient in this example has no registered identity, so the check fails:

```json
{
  "object": "transfer_check",
  "ok": false,
  "reason_code": "RECIPIENT_NOT_VERIFIED",
  "message": "The receiving wallet has no verified identity for this company.",
  "evaluated_at_block": 18204417
}
```

Every possible value of `reason_code` is listed in [Reason codes](reason-codes.md).
{% endstep %}

{% step %}
### Deploy a test share class

Create a new share class on the demo company. Creating a share class deploys a `PrecursorShareToken` and its `RuleEngine` through the factory, so the response comes back with `status: "deploying"` and moves to `deployed` once the transaction is included.

{% tabs %}
{% tab title="curl" %}
```bash
curl https://api.sandbox.useprecursor.xyz/v1/share_classes \
  -H "Authorization: Bearer $PRECURSOR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 4a9e-quickstart-series-a" \
  -d '{
    "company": "cmp_8f2kQ1xZ",
    "name": "Series A Preferred",
    "symbol": "NWR-A",
    "modules": {
      "region": { "blocked": ["KP", "IR"] },
      "holding_period": { "days": 365 },
      "holder_cap": { "max_holders": 500 }
    }
  }'
```
{% endtab %}

{% tab title="TypeScript" %}
```ts
const seriesA = await precursor.shareClasses.create(
  {
    company: "cmp_8f2kQ1xZ",
    name: "Series A Preferred",
    symbol: "NWR-A",
    modules: {
      region: { blocked: ["KP", "IR"] },
      holding_period: { days: 365 },
      holder_cap: { max_holders: 500 },
    },
  },
  { idempotencyKey: "4a9e-quickstart-series-a" },
);

console.log(seriesA.id, seriesA.status); // cls_9QwE2rTy "deploying"
```
{% endtab %}
{% endtabs %}

When deployment finishes, the share class carries its token address, and you can open it on the testnet block explorer or the Precursor Explorer.
{% endstep %}

{% step %}
### Receive a webhook

Register an endpoint so you hear about deployments, transfers and offerings as they happen. For local development, expose your server with a tunnel of your choice and register its public URL.

```ts
const endpoint = await precursor.webhookEndpoints.create({
  url: "https://example.ngrok.app/precursor/webhooks",
  events: ["share_class.deployed", "transfer.settled", "transfer.blocked"],
});

console.log(endpoint.secret); // whsec_... shown once, store it securely
```

Then verify each delivery before trusting it:

{% code title="server.ts" %}
```ts
import express from "express";
import { precursor } from "./precursor";

const app = express();

app.post("/precursor/webhooks", express.raw({ type: "application/json" }), (req, res) => {
  let event;
  try {
    event = precursor.webhooks.constructEvent(
      req.body,
      req.header("Precursor-Signature")!,
      process.env.PRECURSOR_WEBHOOK_SECRET!,
    );
  } catch {
    return res.status(400).send("Invalid signature");
  }

  if (event.type === "share_class.deployed") {
    console.log("Deployed at", event.data.object.token_address);
  }
  res.sendStatus(200);
});

app.listen(3000);
```
{% endcode %}

The share class you created in the previous step will trigger `share_class.deployed` within a few seconds. Delivery details, retries and the full event list are in [Webhooks](webhooks.md).
{% endstep %}
{% endstepper %}

## Next steps

* Read [Authentication and environments](authentication.md) before you move to live keys.
* Browse the full [API reference](api-reference.md).
* If you are building a wallet, venue or data product, go straight to [Integrating Precursor assets](integrating-precursor-assets.md).
