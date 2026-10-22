---
description: The official TypeScript SDK for the Precursor API, with typed resources, auto-pagination, error classes and viem-compatible on-chain helpers.
---

# TypeScript SDK

`@precursor/sdk` is the official client for the Precursor API. It wraps every endpoint in the [API reference](api-reference.md) with typed methods, handles retries and pagination, verifies webhook signatures, and ships the ABIs and helpers you need to read Precursor contracts directly from Robinhood Chain.

## Installation

```bash
npm install @precursor/sdk
```

The SDK supports Node.js 18 and later, Bun and Deno. The API client is server-side only because it uses secret keys. The on-chain helpers in `@precursor/sdk/onchain` contain no keys and are safe in the browser.

## Initialization

```ts
import { Precursor } from "@precursor/sdk";

const precursor = new Precursor(process.env.PRECURSOR_SECRET_KEY!);
```

The environment follows the key prefix: `sk_live_` keys call `https://api.useprecursor.xyz/v1`, and `sk_test_` keys call the sandbox. You can pass options as a second argument:

```ts
const precursor = new Precursor(process.env.PRECURSOR_SECRET_KEY!, {
  apiVersion: "2026-10-01", // sent as Precursor-Version
  maxRetries: 3,            // network errors, 429 and 5xx
  timeoutMs: 30_000,
});
```

Pin `apiVersion` in production so an account-level default change never alters the shape of responses your code receives. Versioning is explained in [Authentication and environments](authentication.md).

## Resources

Methods mirror the REST resources. Request parameters use the same `snake_case` names as the API, and responses are returned as typed objects.

| SDK property | Endpoints |
|---|---|
| `precursor.companies` | `list`, `create`, `retrieve` |
| `precursor.shareClasses` | `list`, `create`, `retrieve` |
| `precursor.holders` | `list`, `create`, `retrieve`, `listLots` |
| `precursor.transfers` | `check` |
| `precursor.offerings` | `list`, `create`, `close` |
| `precursor.liquidityWindows` | `list`, `create`, `cancel`, `listOrders` |
| `precursor.distributions` | `list`, `create` |
| `precursor.proposals` | `list`, `create` |
| `precursor.documents` | `list`, `create` |
| `precursor.valuations` | `list`, `create` |
| `precursor.events` | `list` |
| `precursor.contracts` | `retrieve` |
| `precursor.webhookEndpoints` | `list`, `create`, `del` |
| `precursor.webhooks` | `constructEvent` |

A typical server workflow:

```ts
const offering = await precursor.offerings.create(
  {
    share_class: "cls_3VbT9pLm",
    price_per_share: "18.40",
    min_raise: "2000000",
    max_raise: "5000000",
    min_ticket: "25000",
    opens_at: "2026-11-10T14:00:00Z",
    closes_at: "2026-11-24T22:00:00Z",
    eligible_claims: ["IDENTITY_VERIFIED", "INVESTOR_ELIGIBLE"],
    documents: ["doc_Lk2m9QpZ"],
  },
  { idempotencyKey: "series-b-extension-2026-11" },
);
```

Every write method accepts `{ idempotencyKey }` in its options. If you omit it, the SDK generates one per call and reuses it across its own retries, so automatic retries never duplicate an action.

## Pagination

List methods return a single page by default. To walk every object, iterate the result with `for await`, and the SDK fetches further pages as you go.

```ts
for await (const holder of precursor.holders.list({ share_class: "cls_3VbT9pLm", limit: 100 })) {
  console.log(holder.id, holder.status);
}
```

To collect into an array with a ceiling:

```ts
const holders = await precursor.holders
  .list({ share_class: "cls_3VbT9pLm" })
  .autoPagingToArray({ limit: 5000 });
```

## Errors

Failed requests throw a subclass of `PrecursorError`. Each one exposes `type`, `code`, `message`, `statusCode` and `requestId`, matching the [error object](api-reference.md#errors).

| Class | Error type |
|---|---|
| `PrecursorInvalidRequestError` | `invalid_request` |
| `PrecursorAuthenticationError` | `authentication` |
| `PrecursorRuleViolationError` | `rule_violation` |
| `PrecursorNotFoundError` | `not_found` |
| `PrecursorRateLimitError` | `rate_limited` |
| `PrecursorChainError` | `chain_error` |
| `PrecursorConnectionError` | No response was received |

```ts
import { PrecursorRuleViolationError } from "@precursor/sdk";

try {
  await precursor.offerings.close("off_5kLp2Wq");
} catch (err) {
  if (err instanceof PrecursorRuleViolationError) {
    console.warn(`Blocked by ${err.code}`, err.requestId);
  } else {
    throw err;
  }
}
```

`PrecursorChainError` additionally carries `txHash` when a transaction was submitted and reverted.

## Webhooks

`precursor.webhooks.constructEvent(rawBody, signatureHeader, secret)` verifies the `Precursor-Signature` header, checks the timestamp tolerance (five minutes by default), and returns a typed event. It throws `PrecursorSignatureVerificationError` when verification fails. Pass the raw body as a `Buffer` or string, not parsed JSON. See [Webhooks](webhooks.md) for a full handler.

## On-chain helpers

`@precursor/sdk/onchain` exports ABIs, reason code constants and small helpers that work with [viem](https://viem.sh). Use them when you want to read state directly from Robinhood Chain rather than through the API, for example in a wallet or a venue front end.

| Export | Contents |
|---|---|
| `shareTokenAbi` | ABI for `PrecursorShareToken`, including ERC-20, ERC-3643 and ERC-8056 functions |
| `ruleEngineAbi`, `identityRegistryAbi`, `valuationOracleAbi`, `factoryAbi` | ABIs for the other core contracts |
| `reasonCodes` | Every [reason code](reason-codes.md) as a `bytes32` constant, plus `decodeReasonCode(bytes32)` |
| `robinhoodChain`, `robinhoodChainTestnet` | viem chain definitions for chain IDs `4663` and `46630` |
| `toDisplayAmount(raw, multiplier)` | Converts a raw balance to shares using the current multiplier |

Reading `canTransfer` and `uiMultiplier` before building a transfer:

{% code title="check-transfer.ts" %}
```ts
import { createPublicClient, http, parseUnits } from "viem";
import {
  shareTokenAbi,
  decodeReasonCode,
  robinhoodChain,
  toDisplayAmount,
} from "@precursor/sdk/onchain";

const client = createPublicClient({
  chain: robinhoodChain,
  transport: http(process.env.ROBINHOOD_RPC_URL),
});

const token = "0x..." as const; // from GET /v1/contracts or the Explorer
const from = "0x1111111111111111111111111111111111111111" as const;
const to = "0x2222222222222222222222222222222222222222" as const;

const [[ok, reason], multiplier, raw] = await Promise.all([
  client.readContract({
    address: token,
    abi: shareTokenAbi,
    functionName: "canTransfer",
    args: [from, to, parseUnits("100", 18)],
  }),
  client.readContract({ address: token, abi: shareTokenAbi, functionName: "uiMultiplier" }),
  client.readContract({ address: token, abi: shareTokenAbi, functionName: "balanceOf", args: [from] }),
]);

console.log("Shares held:", toDisplayAmount(raw, multiplier));

if (!ok) {
  console.log("Transfer would fail:", decodeReasonCode(reason)); // e.g. "HOLDING_PERIOD_ACTIVE"
}
```
{% endcode %}

Use a dedicated RPC provider in production. The public Robinhood Chain endpoint is rate-limited, as described in [Robinhood Chain](../architecture/robinhood-chain.md).

## Versioning and support

The SDK follows semantic versioning. Each major version targets a specific API version date, listed in the changelog on [GitHub](https://github.com/useprecursor). Report bugs through GitHub issues or to support@useprecursor.xyz.
