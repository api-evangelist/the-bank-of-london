---
name: the-bank-of-london-virtual-account-hierarchy
description: Build and query a Bank of London virtual account hierarchy — aggregator nodes over virtual accounts pinned to a physical header account — the structure used for safeguarding and client-money segregation.
api: Bank of London API
version: v2/v3
generated: '2026-08-30'
method: generated
source: openapi/the-bank-of-london-api-openapi.json + https://developer.bankoflondon.com/docs/overviews/virtual-account-management
operations:
  - CreateVirtualAggregatorNode
  - GetVirtualAggregatorNodes
  - GetVirtualAggregatorNode
  - UpdateVirtualAggregatorNode
  - CreateVirtualAccountHierarchy
  - GetHierarchy
  - CreateVirtualAccount
  - GetVirtualAccounts
  - GetAVirtualAccount
  - UpdateAVirtualAccount
  - CloseVirtualAccount
  - GetVirtualAccountTransactionsV3
---

# Build a virtual account hierarchy

## The model

Three levels, and the direction of the references matters:

- A **physical account** (`AccountId`) is the real bank account. Everything below it is ledger
  structure, not a separate bank account.
- An **aggregator node** (`AggregatorNodeId`) sits under a physical account via `headerAccountId`,
  and under another node via `parentVirtualAggregatorNodeId`. That self-reference is what makes the
  structure a tree.
- A **virtual account** points at its physical account via `headerAccountId` and at its node via
  `virtualAggregatorNodeId`, and carries an `organisationId`.

This is how you segregate per-end-client balances inside one safeguarding or client-money account
without opening a real account per client.

## Steps

1. **Pick the header account.** `GET /v2/accounts` (`GetAccounts`) — take the physical account that
   will hold the pooled funds.

2. **Create the tree.** Either build it node by node with `POST /v2/virtual/aggregator-nodes`
   (`CreateVirtualAggregatorNode`), setting `headerAccountId` and, for anything below the root,
   `parentVirtualAggregatorNodeId`; or create a whole structure in one call with
   `POST /v2/virtual/hierarchy` (`CreateVirtualAccountHierarchy`).

3. **Create the virtual accounts.** `POST /v2/virtual/accounts` (`CreateVirtualAccount`) with
   `headerAccountId` and `virtualAggregatorNodeId`.

4. **Read the structure back.** `GET /v2/virtual/hierarchy` (`GetHierarchy`) returns the tree;
   `GET /v2/virtual/aggregator-nodes` and `GET /v2/virtual/accounts` list the parts with
   `page`/`pageSize` pagination.

5. **Reconcile.** `GET /v3/virtual/transactions` (`GetVirtualAccountTransactionsV3`) returns
   transactions at virtual-account granularity — this is the operation reconciliation should read,
   not the physical account's transaction feed.

6. **Re-parent or close.** `PATCH /v2/virtual/accounts/{id}` (`UpdateAVirtualAccount`) can move a
   virtual account to a different `virtualAggregatorNodeId`; `PATCH /v2/virtual/aggregator-nodes/{id}`
   (`UpdateVirtualAggregatorNode`) can re-parent a node.
   `POST /v2/virtual/accounts/{id}/close` (`CloseVirtualAccount`) closes a virtual account and has no
   inverse.

## Constraints the contract states

- **Virtual accounts cannot be used for CHAPS payments.** The CHAPS operation description says so
  explicitly. Route CHAPS from the physical account.
- `CloseVirtualAccount` applies only to `VIRTUAL_ACCOUNT` type accounts; `CloseAccount` applies only
  to `BAAS_PHYSICAL_ACCOUNT`.
- None of these creates accept an `idempotencyId` — see the idempotency warning in
  `the-bank-of-london-onboard-customer-and-open-account`.
