---
name: the-bank-of-london-reconcile-transactions
description: Reconcile a Bank of London account — query the transaction ledger, run an asynchronous transaction export, and pull statements — without confusing a payment instruction with its settled ledger entry.
api: Bank of London API
version: v2/v3
generated: '2026-08-30'
method: generated
source: openapi/the-bank-of-london-api-openapi.json
operations:
  - GetTransactionsV3
  - GetTransactionV3
  - InitiateTransactionExportV3
  - GetTransactionExportV3
  - GetTransactionExportsByAccountIdV3
  - GetVirtualAccountTransactionsV3
  - GetStatements
  - DownloadStatements
  - GetPayments
---

# Reconcile an account

## The distinction that matters

A **Payment** is an instruction. A **Transaction** is the settled ledger entry it produced. They have
different ids and different lifecycles, and they are joined by `paymentOrderId`. Reconciling against
`GetPayments` alone will miss fees, interest and inbound credits, which only ever appear as
Transactions.

## Steps

1. **Query the ledger.** `GET /v3/transactions` (`GetTransactionsV3`) filtered by `accountId` and a
   `fromDate`/`toDate` window. Other useful filters: `isCredit`, `type`, `amount`, `reference`,
   `paymentOrderId`, `search`, `orderBy`. Paginate with `page`/`pageSize` (max 50) and read
   `metaData.totalRecords`.

2. **For a virtual-account ledger, use the virtual operation.** `GET /v3/virtual/transactions`
   (`GetVirtualAccountTransactionsV3`) — the physical account's feed will not give you per-virtual-
   account granularity.

3. **For a large window, export instead of paging.** `POST /v3/transaction-export`
   (`InitiateTransactionExportV3`) with the `accountId` and range starts an asynchronous job. Do not
   poll it in a tight loop — you have 1,000 requests per 10 minutes.

4. **Wait for the export by webhook.** Subscribe to `TRANSACTION_EXPORT_SUCCESSFUL` and
   `TRANSACTION_EXPORT_FAILED` (see `the-bank-of-london-receive-webhooks`). On success, fetch it with
   `GET /v3/transaction-export/{transactionExportId}` (`GetTransactionExportV3`). List prior exports
   with `GetTransactionExportsByAccountIdV3`.

5. **Statements for the formal record.** `GET /v2/statements` (`GetStatements`) lists them;
   `GET /v2/statements/{statementId}/download` (`DownloadStatements`) returns
   **`application/pdf`** — the only non-JSON response in the entire contract. Do not try to parse it
   as JSON.

6. **Join payments to transactions.** Match `Transaction.paymentOrderId` to `Payment.paymentOrderId`.
   Use `GET /v2/payments?idempotencyId=<yours>` when you need to find a payment from your own
   correlation key rather than the bank's id.

## Operational notes

- Every response you got an error from carries `x-correlation-id`. Log it against the reconciliation
  run; it is what the bank's support team will ask for.
- Transaction and payment status enums are extensible — a reconciliation job that throws on an
  unknown status will break on a routine, non-breaking release.
- Rate limit is 1,000 requests per rolling 10-minute window per API key, with no headers telling you
  how much is left. Size batch reconciliation around that, and prefer exports over deep pagination.
