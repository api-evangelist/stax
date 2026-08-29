---
name: stax-reconcile-deposits
description: Reconcile Stax settlements — match deposits to transactions, pull statement reports, and handle ACH rejects and disputes.
api: Stax API
operations:
  - get-list-of-deposits
  - get-detail-of-specific-deposit
  - list-and-filter-all-transactions
  - get-a-transactions-information
  - get-a-transactions-funding-instructions
  - daily-sales
  - card-processing
  - volumes-by-card-type
  - surcharging-1
  - refund-and-adjustments
  - ach-rejections
  - disputes
  - list-disputes
  - upload-evidence-for-dispute
  - submit-dispute-evidence-files-for-review
---

# Reconcile deposits and settlements

Base URL `https://apiprod.fattlabs.com`, merchant API key.

## 1. Deposits

- `GET /query/deposit` (`get-list-of-deposits`) — the settlement batches
- `GET /query/depositDetail` (`get-detail-of-specific-deposit`) — what is in one

The `create_deposit` webhook carries the settlement record; its `external_id`
is the join key back to deposit detail. In the sandbox that key does **not**
match `batch_id`, so treat the sandbox event as a trigger to call the API rather
than as data.

## 2. Transactions

`GET /transaction` (`list-and-filter-all-transactions`) with the paginated
envelope (`total`, `per_page`, `current_page`, `last_page`, `next_page_url`,
`data`). Join a transaction to its batch through `batch_id`;
`GET /transaction/{id}/funding` (`get-a-transactions-funding-instructions`)
explains how a single transaction was funded.

For ACH, wait for `update_transaction_settled`: on settlement `settled_at` is
populated, and on a clawback the void appears in `child_transactions`.

## 3. Statement reports

All under `/query/statement/v3/`:

| Report | operationId |
|---|---|
| Daily sales | `daily-sales` |
| Card processing volumes | `card-processing` |
| Volumes by card type | `volumes-by-card-type` |
| Surcharges | `surcharging-1` |
| Refunds and adjustments | `refund-and-adjustments` |
| ACH rejects | `ach-rejections` |
| Disputes | `disputes` |

`fee_statement_ready` is the webhook that tells you a statement exists.

## 4. ACH rejects

Codes returned on ACH processing are numeric and documented — see
`errors/stax-ach-error-codes.yml` in this repo. A few worth branching on:
`140` invalid routing number (validate the 9-digit ABA before sending),
`1007` SEC code not authorized, `814` provisioning deactivated,
`29` standalone ACH not permitted for this merchant.

## 5. Disputes

`GET /underwriting/disputes` (`list-disputes`) and
`GET /underwriting/disputes/{merchantId}` (`list-disputes-copy`, 10 per page).
Attach evidence with `POST /file/dispute` (`upload-evidence-for-dispute`),
remove it with `DELETE /file/dispute/{evidenceFileId}`, then commit with
`POST /underwriting/dispute/{disputeId}/submit`
(`submit-dispute-evidence-files-for-review`). Submission is the point of no
return — there is no documented un-submit.

Subscribe `create_dispute` and `update_dispute` so you are not polling.

## Rate discipline

Reporting pulls are the easiest way to trip the limit: 200 requests/minute per
IP (the Overview page says 100 — assume 100), and only 10 **failed** requests
per minute before a one-hour block. Page through results rather than fanning
out concurrent requests.
