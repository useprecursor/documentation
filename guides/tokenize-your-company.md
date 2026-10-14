---
description: Take a company from signed agreement to live share tokens on Robinhood Chain mainnet, usually in under four weeks.
---

# Tokenize your company

This guide walks a company through onboarding, from verification to the first tokens minted on mainnet. Most companies finish in under four weeks. The longest step is usually reconciling the cap table, so it pays to start that early.

You will need:

* A company multi-sig, or a set of signers who will form one, to hold the `ISSUER_ADMIN` role.
* An agreed registrar. This is the company itself or the administrator that keeps its authoritative cap table, and it holds the `REGISTRAR` role.
* A current cap table export, either as CSV or from your cap-table software.

{% stepper %}
{% step %}
### Verify the company

Sign in to the [Console](../products/console.md) and open **Overview**. Precursor asks for the company's legal name, registered address, officers and signers. Verification usually completes within a few business days. When it does, the company receives a `companyId`, which appears on-chain as a `bytes32` and in the API as `cmp_...`.

Add each person who will approve actions under **Approvals**. Every sensitive action in the Console needs the approval threshold you set here, so pick it with care. Two of three signers is a common starting point.
{% endstep %}

{% step %}
### Choose the share classes to tokenize

Open **Share classes** and add each class you want on-chain, such as Common or Series B Preferred. Each class becomes its own [share token](../concepts/share-tokens.md). For each class, set:

| Field | Notes |
|---|---|
| Name and symbol | Shown in wallets and in the Explorer |
| Authorized shares | The ceiling the registrar can mint up to |
| Conversion terms | Used later if the class converts into another class |
| Share class terms | Uploaded as a `SHARE_CLASS_TERMS` document and anchored on-chain |

You don't have to tokenize every class on day one. Many companies start with common stock and a single preferred series.
{% endstep %}

{% step %}
### Import and reconcile the cap table

Open **Holders** and upload the cap table. Each row is one lot, so a holder who acquired shares on three dates has three rows. Holding periods and vesting are tracked per lot, which is why dates matter.

{% code title="cap-table.csv" %}
```csv
holder_email,holder_name,share_class,quantity,acquired_at,lot_type,vesting_schedule
ana@example.com,Ana Ruiz,COMMON,120000,2022-03-01,founder,
ana@example.com,Ana Ruiz,COMMON,40000,2024-06-15,option_exercise,
li@example.com,Li Chen,COMMON,25000,2025-01-10,employee_grant,4y_1y_cliff_monthly
fund@example.com,Northfield Ventures II,SERIES_B,850000,2025-09-30,purchase,
```
{% endcode %}

Precursor checks the file against the authorized shares for each class and flags anything that doesn't add up, such as duplicate holders, totals over the authorized amount or missing dates. The registrar signs off on the reconciled table in **Approvals**. Nothing is minted until that sign-off exists.
{% endstep %}

{% step %}
### Invite holders

When the cap table is approved, Precursor emails every holder an invitation to [Portfolio](../products/portfolio.md). Each holder completes identity verification, and the resulting claims are written to the company's `IdentityRegistry`. Holders can connect an existing wallet or use an embedded wallet that Portfolio creates for them. Gas is sponsored, so holders never need ETH.

Tokens for holders who haven't finished onboarding are minted later, to the wallet they register. Their position is held in the reconciled cap table until then.
{% endstep %}

{% step %}
### Configure transfer rules

Open **Rules** and enable the modules each class needs. See [Transfer rules](../concepts/transfer-rules.md) for the full list. A typical starting setup:

* `RegionModule` with the regions you serve.
* `EligibilityModule` requiring an `INVESTOR_ELIGIBLE` claim for incoming transfers.
* `HoldingPeriodModule` with a per-lot lockup.
* `EmployeeVestingModule` linked to the vesting schedules from your import.
* `TransferWindowModule`, so transfers only happen during Liquidity Windows you open.
* `ROFRModule`, if the company holds a right of first refusal.

Rule changes on a live class go through the `IssuerTimelock` with a 24 hour delay, so it's worth getting the configuration right before mainnet.
{% endstep %}

{% step %}
### Rehearse on testnet

Precursor deploys your exact configuration to Robinhood Chain testnet (chain ID `46630`) through the sandbox. Use the rehearsal to:

* Confirm balances match the reconciled cap table.
* Run `canTransfer` checks for a few real scenarios and confirm the [reason codes](../developers/reason-codes.md) you expect.
* Walk your signers through an approval, a pause and a scheduled recovery.

The Console shows a rehearsal checklist. Every item has to pass before the mainnet deployment unlocks.
{% endstep %}

{% step %}
### Deploy to mainnet

Approve the deployment in **Approvals**. `PrecursorFactory` deploys the company's contract set in a single transaction and emits `CompanyDeployed`. The registrar then mints tokens to each verified holder against the reconciled cap table. Each mint emits `SharesIssued` with a reference back to the cap table record.

Your company now appears in the [Explorer](../products/explorer.md), with supply, holder count and every event since deployment.
{% endstep %}
{% endstepper %}

{% hint style="success" %}
Next, you can [run an offering](run-an-offering.md) to raise capital or [open a Liquidity Window](open-a-liquidity-window.md) to give existing holders a way to sell.
{% endhint %}
