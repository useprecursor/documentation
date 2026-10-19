---
description: API key types, the live and sandbox environments, versioning, rate limits, idempotency and key rotation.
---

# Authentication and environments

The Precursor API authenticates every request with an API key passed as a bearer token. The key you use decides which environment you reach and what you are allowed to do.

```bash
curl https://api.useprecursor.xyz/v1/companies \
  -H "Authorization: Bearer sk_live_..."
```

Requests without a valid key return `401` with an error of type `authentication`.

## Key types

| Prefix | Type | Where it belongs | What it can do |
|---|---|---|---|
| `sk_live_` | Live secret key | Your server only | Full access to your companies on Robinhood Chain mainnet |
| `sk_test_` | Sandbox secret key | Your server only | Full access to your sandbox companies on Robinhood Chain testnet |
| `pk_live_` | Live publishable key | Browsers, mobile apps | Read-only public data and transfer checks on mainnet |
| `pk_test_` | Sandbox publishable key | Browsers, mobile apps | The same, in the sandbox |

Publishable keys are safe to embed in client code. They can call `POST /v1/transfers/check` and the read endpoints that expose data already public on-chain (share class metadata, valuations, Explorer data, contract addresses). They cannot read holder identities, create resources or change configuration.

Secret keys carry the permissions of the Console member who created them, scoped to the companies that member can access. If you need a narrower key, create a **restricted key** in the Console and choose resources and read or write access per resource. Restricted keys use the `rk_live_` and `rk_test_` prefixes and behave like secret keys everywhere else.

{% hint style="warning" %}
An API key never holds on-chain authority on its own. Actions that need the issuer's multi-sig or a timelock, such as recoveries, multiplier changes and token upgrades, are proposed through the API and then approved by the required signers. See [Roles and permissions](../architecture/roles-and-permissions.md).
{% endhint %}

## Environments

| | Live | Sandbox |
|---|---|---|
| Base URL | `https://api.useprecursor.xyz/v1` | `https://api.sandbox.useprecursor.xyz/v1` |
| Network | Robinhood Chain mainnet | Robinhood Chain testnet |
| Chain ID | `4663` | `46630` |
| Key prefixes | `sk_live_`, `pk_live_`, `rk_live_` | `sk_test_`, `pk_test_`, `rk_test_` |
| Settlement asset | USDG and other supported stablecoins | Test USDG, minted from the Console faucet |
| Identity verification | Real verification providers | Simulated, with test identities you control |
| Rate limit | 100 requests per second per key | 25 requests per second per key |

The two environments are fully separate. Companies, holders, webhook endpoints and keys created in one do not exist in the other. Every issuer deployment is rehearsed in the sandbox with an identical configuration before it goes to mainnet, so most integrations are built and tested against the sandbox end to end.

In the sandbox you can create test holders with any combination of claims from **Developers → Test identities**, which makes it easy to exercise each [reason code](reason-codes.md).

## Versioning

The API is versioned by date. Send the version you built against in the `Precursor-Version` header:

```bash
-H "Precursor-Version: 2026-10-01"
```

If you omit the header, the request uses the default version pinned to your account, which you can see and upgrade in the Console. We add fields and endpoints without changing the version, so your code should ignore fields it does not recognize. Anything that would break an existing integration (removing a field, changing a type, changing default behavior) ships under a new version date.

Every response includes the version that served it in a `Precursor-Version` response header.

## Rate limits

Limits apply per key. When you exceed them, the API returns `429` with an error of type `rate_limited` and a `Retry-After` header in seconds. Each response also carries:

| Header | Meaning |
|---|---|
| `RateLimit-Limit` | Requests allowed in the current one-second window |
| `RateLimit-Remaining` | Requests left in the window |
| `RateLimit-Reset` | Seconds until the window resets |

The SDK retries `429` responses automatically with backoff. If you need sustained throughput above the default, contact support@useprecursor.xyz.

## Idempotency

Every `POST` request accepts an `Idempotency-Key` header, and you should send one on every write. If a request fails because of a network error or a timeout, retry it with the same key and the API returns the result of the original request instead of performing the action twice. This matters most for anything that produces an on-chain transaction, such as closing an offering or creating a share class.

```bash
curl https://api.useprecursor.xyz/v1/offerings/off_5kLp2Wq/close \
  -X POST \
  -H "Authorization: Bearer sk_live_..." \
  -H "Idempotency-Key: close-off_5kLp2Wq-2026-11-04"
```

Rules for keys:

* Use a value with enough entropy to be unique, such as a UUID, or a deterministic value derived from your own record ids.
* Keys are stored for 24 hours. After that, a reused key is treated as a new request.
* Reusing a key with a different request body returns `400` with an error of type `invalid_request`.

## Request ids

Every response includes a `Request-Id` header, and every error object contains the same value as `request_id`. Include it when you contact support, as it lets us trace the request through the API and to any transaction it produced.

## Rotating keys

Rotate secret keys on a regular schedule and immediately if you suspect one has leaked.

1. In the Console, open **Developers → API keys** and choose **Roll key** on the key you want to replace.
2. Choose how long the old key stays valid, from immediately up to seven days. An overlap lets you deploy the new key without downtime.
3. Deploy the new key to your servers and confirm traffic has moved by watching the old key's **Last used** time.
4. Let the old key expire, or revoke it manually once traffic has stopped.

Every key creation, roll and revocation is recorded in the Console audit log with the member who performed it. Webhook signing secrets rotate the same way from the endpoint's settings page, and during the overlap each delivery is signed with both secrets. See [Webhooks](webhooks.md).
