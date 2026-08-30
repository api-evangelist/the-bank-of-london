---
name: the-bank-of-london-onboard-customer-and-open-account
description: Onboard an organisation or individual as a Bank of London customer and open a physical account for them, using the v3 organisation surface and the v2 account surface.
api: Bank of London API
version: v2/v3
generated: '2026-08-30'
method: generated
source: openapi/the-bank-of-london-api-openapi.json
operations:
  - CreateOrganisationV3
  - GetOrganisationsV3
  - GetOrganisationV3
  - PatchOrganisationV3
  - CreateIndividual
  - GetIndividuals
  - GetIndividual
  - PatchIndividual
  - CreateAccount
  - GetAccounts
  - GetAccount
  - UpdateAccount
  - CloseAccount
---

# Onboard a customer and open an account

## Which version to call

Organisations are on **v3** (`/v3/organisations`). Individuals are on **v2** (`/v2/individuals`).
Accounts are on **v2** (`/v2/accounts`). This is not a mistake in the contract — the two surfaces are
versioned independently and both are current.

## Steps

1. **Create the organisation.** `POST /v3/organisations` (`CreateOrganisationV3`). Store the returned
   `id` — it is the `organisationId` every downstream object hangs off.

2. **Create the individuals under it.** `POST /v2/individuals` (`CreateIndividual`) with
   `organisationId` set to the organisation from step 1. Store each returned `id` (a `PersonId`).

3. **Open the account.** `POST /v2/accounts` (`CreateAccount`). The account is created against a
   `salesProductId` — the product the bank has agreed with you commercially (operating account,
   safeguarding account, client-money account, notice deposit, easy-access deposit). If you do not
   know your product ids, list existing accounts with `GetAccounts` and read the `salesProductId` on
   one, or ask your account manager. Do not guess one.

4. **Verify.** `GET /v2/accounts/{id}` (`GetAccount`). The account `id` is in the documented form
   `GB-<sortcode>-<accountnumber>` (for example `GB-040075-12345678`), which is what you will pass as
   `sender.accountId` on every payment.

5. **Amend, don't recreate.** `PATCH /v2/accounts/{id}` (`UpdateAccount`) and
   `PATCH /v3/organisations/{id}` (`PatchOrganisationV3`) exist for corrections.

## Idempotency warning

**None of the operations in this skill accept an `idempotencyId`.** That field exists only on payment
and standing-order creates. If `CreateOrganisationV3`, `CreateIndividual` or `CreateAccount` times
out, you have no safe replay key — you must call the corresponding `Get…` list operation and search
for the record before retrying, or you will create a duplicate customer or a duplicate account.

## Closing

`POST /v2/accounts/{id}/close` (`CloseAccount`) applies only to `BAAS_PHYSICAL_ACCOUNT` type
accounts. There is **no reopen or restore operation** — closing is one-way. Close virtual accounts
with `CloseVirtualAccount` before closing the physical header account they hang off.
