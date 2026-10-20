---
description: Every endpoint in the Precursor REST API, with request parameters, example responses, pagination and the error format.
---

# API reference

The Precursor API is a REST API over HTTPS. It accepts JSON request bodies, returns JSON responses, and uses standard HTTP status codes. Authentication, environments, versioning and idempotency are covered in [Authentication and environments](authentication.md).

| Environment | Base URL |
|---|---|
| Live | `https://api.useprecursor.xyz/v1` |
| Sandbox | `https://api.sandbox.useprecursor.xyz/v1` |

## Conventions

* **Field names** are `snake_case`.
* **Ids** are prefixed strings: `cmp_` for companies, `cls_` for share classes, `hld_` for holders, `off_` for offerings, `lqw_` for Liquidity Windows, `dst_` for distributions, `prp_` for proposals, `doc_` for documents, `val_` for valuations, `evt_` for events and `we_` for webhook endpoints.
* **Amounts** are decimal strings in token units, such as `"1250.5"`. Where precision matters, objects also include a `raw` field with the integer amount at 18 decimals as a string. Stablecoin amounts follow the same pattern with the stablecoin's own decimals.
* **Addresses** are lowercase hex strings with the `0x` prefix.
* **Timestamps** are ISO 8601 in UTC, such as `"2026-11-04T16:00:00Z"`.
* **On-chain references.** Objects that correspond to a transaction include `tx_hash`, `block_number` and `chain_id`.

## Pagination

List endpoints return a page of results and a flag that tells you whether more exist.

```bash
curl "https://api.useprecursor.xyz/v1/holders?share_class=cls_3VbT9pLm&limit=50" \
  -H "Authorization: Bearer sk_live_..."
```

```json
{
  "object": "list",
  "data": [ { "id": "hld_A1b2C3d4", "object": "holder" } ],
  "has_more": true
}
```

| Parameter | Description |
|---|---|
| `limit` | Number of objects to return, from 1 to 100. Defaults to 50. |
| `starting_after` | An object id. Returns the page that begins after this object. |
| `ending_before` | An object id. Returns the page that ends before this object. |

To fetch the next page, pass the id of the last object you received as `starting_after`. The [SDK](sdk.md) does this for you with auto-pagination.

## Errors

Successful requests return `2xx`. Errors return `4xx` or `5xx` with a single error object:

```json
{
  "error": {
    "type": "rule_violation",
    "code": "HOLDING_PERIOD_ACTIVE",
    "message": "The sender's tokens are still within the holding period set for this share class.",
    "request_id": "req_Qm7vX2pLk9"
  }
}
```

| Type | HTTP status | Meaning |
|---|---|---|
| `invalid_request` | 400, 404, 409 | A parameter is missing or invalid, or the request conflicts with the current state |
| `authentication` | 401, 403 | The key is missing, invalid, revoked, or not allowed to perform this action |
| `rule_violation` | 422 | The action would break a transfer rule. `code` holds the [reason code](reason-codes.md) |
| `not_found` | 404 | The object does not exist, or the key cannot see it |
| `rate_limited` | 429 | Too many requests. Retry after the `Retry-After` interval |
| `chain_error` | 502, 503 | The transaction could not be submitted or was reverted on Robinhood Chain |

For `invalid_request` errors, `code` names the problem (such as `parameter_missing`) and an extra `param` field names the parameter.

## Companies

A company is an issuer onboarded to Precursor. Each one has an off-chain id (`cmp_...`) and an on-chain `companyId` (bytes32), returned as `company_id_onchain`.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/companies` | List companies your key can access |
| `POST` | `/v1/companies` | Create a company and deploy its contract set through `PrecursorFactory` |
| `GET` | `/v1/companies/{id}` | Retrieve a company |

Key parameters for `POST`: `name`, `legal_name`, `country` (ISO 3166 alpha-2), `issuer_multisig` (address of the company multi-sig that will hold `ISSUER_ADMIN`), `registrar` (address that will hold `REGISTRAR`), `timelocks` (optional overrides within the allowed bounds).

```json
{
  "id": "cmp_8f2kQ1xZ",
  "object": "company",
  "name": "Northwind Robotics",
  "status": "deployed",
  "chain_id": 4663,
  "company_id_onchain": "0x6e6f7274...",
  "contracts": {
    "identity_registry": "0x...",
    "trusted_verifiers": "0x...",
    "offering_vault": "0x...",
    "liquidity_window": "0x...",
    "distribution_vault": "0x...",
    "governance": "0x...",
    "disclosure_registry": "0x...",
    "issuer_timelock": "0x..."
  },
  "share_classes": ["cls_3VbT9pLm"],
  "created_at": "2026-09-14T10:21:44Z"
}
```

## Share classes

Each share class is one `PrecursorShareToken` with its own `RuleEngine` and `ValuationOracle`.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/share_classes` | List share classes, filterable by `company` |
| `POST` | `/v1/share_classes` | Create and deploy a share class |
| `GET` | `/v1/share_classes/{id}` | Retrieve a share class, including supply, holder count and module configuration |

Key parameters for `POST`: `company`, `name`, `symbol`, `modules` (an object keyed by module name: `region`, `eligibility`, `holding_period`, `holder_cap`, `transfer_window`, `rofr`, `max_ownership`, `employee_vesting`), `rfq_enabled` (boolean, default `false`).

Retrieved objects include `token_address`, `rule_engine_address`, `valuation_oracle_address`, `total_supply`, `holder_count`, `ui_multiplier` (decimal string, `"1.0"` by default), `paused` and `status` (`deploying`, `deployed`, `failed`).

## Holders

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/holders` | List holders, filterable by `company`, `share_class` and `status` |
| `POST` | `/v1/holders` | Invite a holder by email. Precursor handles identity verification and wallet setup |
| `GET` | `/v1/holders/{id}` | Retrieve a holder, their wallets, claims and balances |
| `GET` | `/v1/holders/{id}/lots` | List a holder's lots with acquisition time, holding period end and vesting state |

Key parameters for `POST`: `company`, `email`, `name`, `wallet` (optional, an existing address), `claims` (optional, for example `{ "employee_of": true }`), `allocations` (optional array of `{ share_class, amount, reference }` to mint after verification).

Holder objects include `status` (`invited`, `verifying`, `verified`, `restricted`), `wallets`, and `claims`, an array of `{ topic, value, verifier, expires_at }`. Personal data returned here never leaves Precursor's access-controlled systems and is never written on-chain.

## Transfer checks

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/transfers/check` | Evaluate whether a transfer would pass every rule, without sending it |

Works with publishable keys. Parameters: `share_class`, `from`, `to`, `amount`.

```json
{
  "object": "transfer_check",
  "ok": false,
  "reason_code": "WINDOW_CLOSED",
  "message": "Transfers in this share class are only allowed during an open Liquidity Window.",
  "evaluated_at_block": 18204417
}
```

The result mirrors `canTransfer` on the token at `evaluated_at_block`.

## Offerings

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/offerings` | List offerings, filterable by `company`, `share_class` and `status` |
| `POST` | `/v1/offerings` | Create an offering in `OfferingVault` |
| `POST` | `/v1/offerings/{id}/close` | Close an offering after `closes_at`. Mints shares or marks it failed |

Key parameters for `POST /v1/offerings`: `share_class`, `price_per_share`, `min_raise`, `max_raise`, `min_ticket`, `opens_at`, `closes_at`, `stablecoin` (defaults to `usdg`), `eligible_claims` (array of claim topics), `documents` (array of `doc_` ids anchored in `DisclosureRegistry`).

Offering objects include `status` (`scheduled`, `open`, `closed`, `failed`), `raised`, `subscriber_count` and, after close, `shares_issued` and `close_tx_hash`. See [Primary offerings](../liquidity/offerings.md).

## Liquidity Windows

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/liquidity_windows` | List windows |
| `POST` | `/v1/liquidity_windows` | Schedule a window |
| `POST` | `/v1/liquidity_windows/{id}/cancel` | Cancel a window before it clears. Locked tokens and stablecoins are released |
| `GET` | `/v1/liquidity_windows/{id}/orders` | List the orders placed in a window |

Key parameters for `POST`: `share_class`, `mode` (`auction` or `tender`), `opens_at`, `closes_at`, `seller_claims` and `buyer_claims` (claim topic filters), `reserve_price` (optional), `max_volume` (optional), `per_seller_cap` (optional), `rofr_holder` and `rofr_period` (optional), `tender_price` (required in `tender` mode).

Cleared windows include `clearing_price`, `volume` and `clear_tx_hash`. See [Liquidity Windows](../liquidity/liquidity-windows.md).

## Distributions

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/distributions` | List distributions |
| `POST` | `/v1/distributions` | Fund a distribution in `DistributionVault` |

Key parameters: `share_class`, `asset` (defaults to `usdg`), `amount`, `record_block` (or `record_at`, which resolves to the first block at or after that time), `claim_deadline` (optional, at least 12 months after funding), `reference`.

## Proposals

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/proposals` | List proposals and their results |
| `POST` | `/v1/proposals` | Create a token-weighted vote in `Governance` |

Key parameters: `share_classes`, `title`, `options`, `snapshot_block` (defaults to the current block), `ends_at`, `quorum`, `binding` (boolean), `documents`.

## Documents

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/documents` | List anchored documents |
| `POST` | `/v1/documents` | Upload a document and anchor its hash in `DisclosureRegistry` |

Key parameters: `company`, `kind` (`SHARE_CLASS_TERMS`, `OFFERING_DOCUMENT`, `FINANCIAL_STATEMENT`, `COMPANY_UPDATE`, `BOARD_RESOLUTION`, `VALUATION_REPORT`), `file` (multipart upload), `visibility` (`holders` or `public`). The response includes the SHA-256 `hash` that was anchored and the `document_id` on-chain.

## Valuations

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/valuations` | List Reference Valuations for a share class. Works with publishable keys |
| `POST` | `/v1/valuations` | Publish a Reference Valuation through `ValuationOracle` |

Key parameters for `POST`: `share_class`, `price` (USD per share), `source` (`PRICED_ROUND` or `INDEPENDENT_VALUATION`), `as_of`, `document` (a `doc_` id). Valuations with source `WINDOW_CLEARING` are published automatically when a Liquidity Window clears. See [Reference Valuation](../concepts/reference-valuation.md).

## Events

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/events` | List indexed on-chain events |

Filters: `company`, `share_class`, `type` (an [on-chain event](events.md) name such as `SharesIssued`), `from_block`, `to_block`. Each event includes `contract`, `args`, `tx_hash`, `block_number`, `log_index` and `confirmation` (`sequencer` or `l1`).

## Contracts

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/contracts` | Deployed contract addresses for the current network. Works with publishable keys |

Without parameters, it returns the `PrecursorFactory` address and the module implementations. With `company`, it returns every contract in that company's set. This endpoint and the Explorer are the authoritative sources for addresses. See [Contract reference](contracts.md).

## Webhook endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/webhook_endpoints` | List endpoints |
| `POST` | `/v1/webhook_endpoints` | Register an endpoint. The response includes the signing `secret`, shown once |
| `DELETE` | `/v1/webhook_endpoints/{id}` | Delete an endpoint |

Parameters for `POST`: `url`, `events` (array of event types, or `["*"]`), `description`. See [Webhooks](webhooks.md).
