---
description: Every reason code a Precursor transfer check can return, where it comes from, what it means and how to resolve it.
---

# Reason codes

When a transfer of a Precursor Share Token would fail, the token tells you why. `canTransfer(from, to, amount)` returns `(false, reasonCode)`, where `reasonCode` is a `bytes32` string. The same value appears as `reason_code` from `POST /v1/transfers/check`, as `code` on API errors of type `rule_violation`, and on-chain as the argument of the custom error `TransferNotAllowed(bytes32 reasonCode)` when a transfer, transfer from or mint fails.

Checks run in a fixed order: token state first, then identity, then each enabled module in the order configured by the issuer. The first failure is returned, so a single transfer can have more than one problem, and fixing one may reveal the next.

## Reference

| Code | Source | Meaning | How to resolve | Suggested user message |
|---|---|---|---|---|
| `OK` | Token | The transfer passes every check | Nothing to resolve | |
| `TOKEN_PAUSED` | Token | The issuer or registrar has paused the share class | Wait for the token to be unpaused. The pause reason is published on-chain | "Transfers of this share class are paused by the company." |
| `FROZEN` | Token | Some of the sender's tokens are frozen and the unfrozen balance does not cover the amount, or the wallet has a scheduled recovery | Contact the company. Frozen tokens stay in the wallet but cannot move | "Some of your shares are frozen. Contact the company for details." |
| `INSUFFICIENT_UNLOCKED_BALANCE` | Token | The amount is larger than the sender's balance | Reduce the amount. `unlockedBalanceOf(from)` shows what can move once freezes, holding periods and vesting are also taken into account | "You can transfer up to {unlocked} shares right now." |
| `SENDER_NOT_VERIFIED` | Identity | The sending wallet has no linked identity, or the identity lacks an `IDENTITY_VERIFIED` or `SCREENED` claim | Complete verification in Portfolio, or link the wallet to an existing identity | "Verify your identity to transfer shares from this wallet." |
| `RECIPIENT_NOT_VERIFIED` | Identity | The receiving wallet has no linked identity, or the identity lacks an `IDENTITY_VERIFIED` or `SCREENED` claim | The recipient must complete verification first | "The recipient needs to verify their identity before they can receive shares." |
| `CLAIM_EXPIRED` | Identity | An `IDENTITY_VERIFIED` or `SCREENED` claim for the sender or recipient has expired, or its verifier is no longer trusted | Renew verification. Holders are notified 30 days before expiry | "Your verification has expired. Renew it to continue." |
| `REGION_BLOCKED` | `RegionModule` | The recipient's region is not allowed for this share class, or the recipient has no valid `REGION` claim | No resolution through the platform. The company sets which regions are allowed | "This transfer is not available in the recipient's region." |
| `NOT_ELIGIBLE` | `EligibilityModule` | The recipient does not hold a valid `INVESTOR_ELIGIBLE` claim at the required tier | The recipient completes the eligibility check in Portfolio | "The recipient isn't eligible to hold this share class yet." |
| `HOLDING_PERIOD_ACTIVE` | `HoldingPeriodModule` | The tokens being sent are still within their holding period | Wait until the lots unlock. `GET /v1/holders/{id}/lots` shows each lot's unlock date | "These shares can be transferred from {date}." |
| `HOLDER_CAP_REACHED` | `HolderCapModule` | The transfer would add a new holder beyond the share class's maximum | Transfer to an existing holder, or wait for the company to change the cap | "This share class has reached its maximum number of holders." |
| `WINDOW_CLOSED` | `TransferWindowModule` | Transfers are only allowed during an open Liquidity Window or approved trading period | Wait for the next window. Upcoming windows are listed in Portfolio | "Transfers open during the company's next Liquidity Window." |
| `ROFR_PENDING` | `ROFRModule` | No proposal covers this transfer, the notice period has not ended, or the right holder exercised its right | Propose the transfer with `propose(to, amount)` and wait for the notice period to end or for a waiver | "The company has first right to buy these shares until {date}." |
| `MAX_OWNERSHIP_EXCEEDED` | `MaxOwnershipModule` | The recipient would hold more than the per-holder limit | Reduce the amount | "This would take the recipient above the maximum holding for this share class." |
| `UNVESTED` | `EmployeeVestingModule` | The transfer would draw on shares that have not vested yet across the sender's identity | Wait for the vesting schedule. Unvested shares are shown in Portfolio | "These shares haven't vested yet." |

## Handling reason codes in your app

**Check before you build.** Call `canTransfer` or `POST /v1/transfers/check` before asking a user to sign. A failed transfer on-chain still costs gas and gives the user a worse experience than an explanation up front.

**Show the message, not the code.** The API returns a plain-language `message` with every check. Use it, or use the suggested messages above with your own wording. Show the raw code only in developer tools and logs.

**Re-check close to submission.** Rule state changes over time. Claims expire, windows close, holder counts move. If a user leaves a transfer screen open, check again just before submitting.

**Expect new codes.** New modules may introduce new reason codes. Treat any unrecognized code as a generic failure and show the API's `message`.

## Decoding on-chain

Reason codes are ASCII strings right-padded with zeros to 32 bytes. With viem:

```ts
import { hexToString } from "viem";

const code = hexToString(reasonCode, { size: 32 }).replace(/\0+$/, "");
```

The SDK exports the same logic as `decodeReasonCode`, along with every code as a constant. See [TypeScript SDK](sdk.md). The modules behind each code are explained in [Transfer rules](../concepts/transfer-rules.md).
