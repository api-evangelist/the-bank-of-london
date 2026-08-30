---
name: the-bank-of-london-send-faster-payment
description: Send a UK Faster Payment from a Bank of London account, with a Confirmation of Payee check first, an idempotency key, and correct handling of the asynchronous 202 result and the FPS scheme reason codes.
api: Bank of London API
version: v2
generated: '2026-08-30'
method: generated
source: openapi/the-bank-of-london-api-openapi.json + https://developer.bankoflondon.com/docs/guides/getting-started-guide
operations:
  - ConfirmPayee
  - CreateAFasterPayment
  - GetPayment
  - GetPayments
---

# Send a Faster Payment

Moving real money. Read the reversibility note at the bottom **before** you call `CreateAFasterPayment`.

## Preconditions

- An API key + secret created in the Developer Studio, bound to the environment you are calling
  (Live or Sandbox — a key works in exactly one and cannot be moved).
- Every request must carry a freshly generated `x-jws-signature` header. See
  `authentication/the-bank-of-london-authentication.yml`. The signature is valid for 5 minutes and
  its `nonce` may not be reused inside a 5-minute interval, so you cannot cache one.

## Steps

1. **Find the sending account.** `GET /v2/accounts` (`GetAccounts`). Paginate with `page` /
   `pageSize` (max 50) and read `metaData.totalRecords`. Take the `id` of the account you will debit —
   it is the value for `sender.accountId`.

2. **Check the payee.** `POST /v2/confirmation-of-payee` (`ConfirmPayee`) with the recipient's name,
   sort code and account number. This is Pay.UK Confirmation of Payee: a `MATCHED` result means the
   name matches the account; anything else means it does not, and sending anyway is how APP fraud
   losses happen. Do not auto-proceed on a non-match — surface it.

3. **Mint an idempotency id.** Generate a UUID and put it in the request body as `idempotencyId`
   (string, 5–72 characters). Persist it against your own payment record *before* you send. This is
   the only thing standing between a timeout and a double payment.

4. **Create the payment.** `POST /v2/payments/faster-payment` (`CreateAFasterPayment`) with
   `type: FPS_DEBIT`, `sender.accountId`, `recipient` (accountHolderName, sortCode, accountNumber),
   `amount` (`currency: GBP`, `value`), `reference`, and `idempotencyId`.

5. **Handle the response.**
   - **202 Accepted** — the payment is *accepted for processing*, not settled. Store the returned
     payment `id`. Record the `x-correlation-id` response header.
   - **409 `IDEMPOTENCY_ID_CONFLICT`** — you already sent this. Do **not** retry with a new id; look
     the original up with `GET /v2/payments?idempotencyId=<yours>` (`GetPayments`).
   - **422** — validation. The `code` tells you which rule fired:
     `PAYMENT_AMOUNT_EXCEEDS_LIMIT`, `PAYMENT_AMOUNT_EXCEEDS_AVAILABLE_BALANCE`,
     `EXECUTION_DATE_TODAY`, `EXECUTION_DATE_ON_WEEKEND`, `EXECUTION_DATE_ON_PUBLIC_HOLIDAY`,
     `INVALID_SORT_CODE_PROVIDED`, `PAYMENT_REFERENCE_VIOLATES_PROFANITY_FILTER`, and others. None of
     these are retryable without changing the request.
   - **401 `BANK_ACCOUNT_LOCKED` / `BANK_ACCOUNT_CLOSED`** — the sending account cannot transact.
   - **403 `PAYEE_IS_NOT_A_NOMINATED_PAYEE`** — the recipient is not on the account's nominated list.
   - **429** — you hit 1,000 requests in a 10-minute window on this key. There is no `Retry-After`
     header; back off exponentially.

6. **Track it to a terminal state.** Either poll `GET /v2/payments/{paymentId}` (`GetPayment`) or —
   better — subscribe to the `PAYMENT_SUCCESSFUL` and `PAYMENT_FAILED` webhooks (see
   `the-bank-of-london-receive-webhooks`). `status.identifier` moves through
   `PENDING` → `SUCCESSFUL` | `FAILED`, and `PENDING_APPROVAL` → `APPROVED` | `REJECTED` when the
   account requires approval in the TBOL Core web interface.

7. **Read the failure reason, not just the status.** On `FAILED`, `status.detailedStatusIdentifier`
   carries the FPS scheme reason — `FPS_BENEFICIARY_ACCOUNT_CLOSED`,
   `FPS_BENEFICIARY_ACCOUNT_NAME_AND_NUMBER_MISMATCH`, `FPS_FUNDS_NOT_AVAILABLE`,
   `FPS_SENDING_AGENCY_SORT_CODE_ACCOUNT_UNKNOWN`, `FPS_SCHEME_IS_DOWN`, and others. The full list is
   in `errors/the-bank-of-london-decline-codes.yml`. These values are **extensible** — handle unknown
   ones with a default case rather than throwing.

## Reversibility — read before step 4

**There is no recall or reversal path for a Faster Payment.** Once `CreateAFasterPayment` returns
202, the instruction is out. Confirmation of Payee in step 2 and the idempotency id in step 3 are the
only two safety nets that exist. If you need a reversible rail, Bacs can be recalled — but only up to
5.30PM on the day of initiation, and only by contacting the bank out-of-band
(`uksupport@thebankoflondon.com`, `+44 3301 659 131`). CHAPS cannot be reversed at all.
