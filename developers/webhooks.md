---
description: Receive signed notifications when share classes deploy, transfers settle, offerings close and other events happen on Robinhood Chain.
---

# Webhooks

Webhooks let your systems react to what happens on Precursor without polling. When an event occurs, such as a transfer settling or a Liquidity Window clearing, Precursor sends an HTTPS `POST` to every endpoint you registered for that event type.

Most events originate on Robinhood Chain. Precursor's indexer watches every company's contracts, turns each relevant log into an event, and delivers it once the block has reached the confirmation level stated in the payload.

## Registering an endpoint

Register endpoints in the Console under **Developers → Webhooks**, or through the API:

```bash
curl https://api.useprecursor.xyz/v1/webhook_endpoints \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: we-cap-table-sync" \
  -d '{
    "url": "https://hooks.example.com/precursor",
    "events": ["transfer.settled", "offering.closed", "distribution.funded"],
    "description": "Cap table sync"
  }'
```

The response includes a `secret` beginning with `whsec_`. It is shown only once, so store it with your other secrets. Endpoints must use HTTPS and respond within 10 seconds.

## Event types

| Type | Sent when |
|---|---|
| `share_class.deployed` | A share class's token, rule engine and oracle are deployed |
| `holder.verified` | A holder completes verification and their claims are written on-chain |
| `holder.claim_expiring` | One of a holder's claims expires in 30 days or less |
| `transfer.settled` | A share token transfer is included on-chain |
| `transfer.blocked` | A transfer attempted through the API or Precursor apps failed a rule check |
| `offering.subscribed` | An investor subscribes to an offering |
| `offering.closed` | An offering closes successfully and shares are minted |
| `offering.failed` | An offering closes below its minimum raise |
| `liquidity_window.opened` | A Liquidity Window opens for orders |
| `liquidity_window.cleared` | A window clears and settles at its clearing price |
| `distribution.funded` | A distribution is funded in `DistributionVault` |
| `distribution.claimed` | A holder claims from a distribution |
| `proposal.finalized` | A vote ends and its result is recorded |
| `recovery.scheduled` | A wallet recovery enters its timelock |
| `recovery.executed` | A wallet recovery completes |
| `token.paused` | A share token is paused |
| `multiplier.scheduled` | A change to `uiMultiplier` is scheduled |

Subscribe to `"*"` to receive every type. New types may be added over time, so ignore any type you do not handle.

## Payload

Every delivery has the same envelope. The changed object sits under `data.object`.

```json
{
  "id": "evt_2Hk9sPq7Xc",
  "object": "event",
  "type": "transfer.settled",
  "created_at": "2026-11-04T16:02:41Z",
  "api_version": "2026-10-01",
  "livemode": true,
  "chain_id": 4663,
  "block_number": 18311902,
  "tx_hash": "0x9c1e...4b7a",
  "log_index": 3,
  "confirmation": "l1",
  "data": {
    "object": {
      "object": "transfer",
      "share_class": "cls_3VbT9pLm",
      "from": "0x1111111111111111111111111111111111111111",
      "to": "0x2222222222222222222222222222222222222222",
      "amount": "250",
      "raw": "250000000000000000000",
      "via": "liquidity_window",
      "liquidity_window": "lqw_7Tn4Vb1s"
    }
  }
}
```

### The confirmation field

Robinhood Chain gives two levels of finality, and each event tells you which one it was delivered at.

| Value | Meaning | Typical delay |
|---|---|---|
| `sequencer` | The transaction is included by the Robinhood Chain sequencer (soft confirmation) | Under a second |
| `l1` | The batch containing the transaction is posted to Ethereum | Minutes |

By default, settlement events (`transfer.settled`, `offering.closed`, `liquidity_window.cleared`, `distribution.claimed`, `recovery.executed`) are delivered at `l1`, and everything else at `sequencer`. You can change the level per endpoint in the Console. Use `sequencer` for interfaces where speed matters, and `l1` for anything that updates books of record.

## Verifying signatures

Each delivery carries a `Precursor-Signature` header:

```
Precursor-Signature: t=1793894561,v1=5f2b8e...c91a
```

`t` is the Unix timestamp of the delivery and `v1` is a hex HMAC-SHA256 of the string `{t}.{raw body}`, keyed with your endpoint secret. Always verify the signature against the **raw** request body, before any JSON parsing, and reject deliveries whose timestamp is more than five minutes old.

{% tabs %}
{% tab title="With the SDK" %}
```ts
import express from "express";
import { Precursor } from "@precursor/sdk";

const precursor = new Precursor(process.env.PRECURSOR_SECRET_KEY!);
const app = express();

app.post("/precursor", express.raw({ type: "application/json" }), (req, res) => {
  let event;
  try {
    event = precursor.webhooks.constructEvent(
      req.body,
      req.header("Precursor-Signature")!,
      process.env.PRECURSOR_WEBHOOK_SECRET!,
    );
  } catch (err) {
    return res.status(400).send("Invalid signature");
  }

  switch (event.type) {
    case "transfer.settled":
      // Update your internal cap table view.
      break;
    case "offering.closed":
      // Mark subscriptions as allocated.
      break;
  }

  res.sendStatus(200);
});
```
{% endtab %}

{% tab title="With node:crypto" %}
```ts
import { createHmac, timingSafeEqual } from "node:crypto";

const TOLERANCE_SECONDS = 300;

export function verifyPrecursorSignature(rawBody: Buffer, header: string, secret: string): boolean {
  const parts = Object.fromEntries(
    header.split(",").map((part) => part.split("=") as [string, string]),
  );
  const timestamp = Number(parts.t);
  const signatures = header
    .split(",")
    .filter((part) => part.startsWith("v1="))
    .map((part) => part.slice(3));

  if (!timestamp || signatures.length === 0) return false;
  if (Math.abs(Date.now() / 1000 - timestamp) > TOLERANCE_SECONDS) return false;

  const expected = createHmac("sha256", secret)
    .update(`${timestamp}.${rawBody.toString("utf8")}`)
    .digest();

  return signatures.some((sig) => {
    const received = Buffer.from(sig, "hex");
    return received.length === expected.length && timingSafeEqual(received, expected);
  });
}
```
{% endtab %}
{% endtabs %}

During a secret rotation the header contains two `v1` values, one for each active secret. The manual example above accepts either.

## Delivery and retries

A delivery succeeds when your endpoint returns any `2xx` status within 10 seconds. Anything else counts as a failure, and Precursor retries with exponential backoff for up to three days. After three days of failures the event is marked as failed, and you can resend it from the Console.

Build your handler with these properties in mind:

* **Respond quickly.** Acknowledge with `200` first, then do slow work in a background job.
* **Handle duplicates.** An event can be delivered more than once. Use the event `id`, or the pair `tx_hash` and `log_index`, to deduplicate.
* **Do not rely on order.** Retries can arrive after later events. Compare `block_number` and `log_index` when order matters, or fetch the current object from the API.
* **Reconcile periodically.** For books of record, combine webhooks with a periodic read of `GET /v1/events` from your last processed block. That endpoint returns the same events and is the backstop if your endpoint was unavailable.

Endpoints that fail continuously for three days are disabled, and the Console members who own them are notified by email.

## Testing

From an endpoint's page in the Console you can send a test delivery of any event type, view every recent delivery with its request, response and timing, and resend individual events. In the sandbox, every action you take on testnet produces the same webhooks as it would on mainnet.
