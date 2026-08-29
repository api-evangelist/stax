---
name: stax-invoice-and-collect
description: Create, send and collect on a Stax invoice, including recurring schedules and payment links.
api: Stax API
operations:
  - create-customer
  - create-catalog-item
  - create-an-invoice
  - send-an-invoice-via-email
  - send-an-invoice-via-sms
  - pay-an-invoice
  - mark-an-invoice-paid-in-cash-or-check
  - createScheduledInvoice
  - editInvoiceSchedule
  - delete-invoice-schedule
  - create-a-payment-link
---

# Invoice and collect

Base URL `https://apiprod.fattlabs.com`, `Authorization: Bearer <merchant API key>`.

## 1. Customer and catalog

`POST /customer` (`create-customer`) for the payer. Optional but useful:
`POST /item` (`create-catalog-item`) for reusable products and services —
stock moves with `PUT /item/{id}/increment` and `PUT /item/{id}/decrement`.

## 2. Create the invoice

`POST /invoice` (`create-an-invoice`) referencing `customer_id` and the line
items. Read it back with `GET /invoice/{id}` (`get-an-invoice`), amend with
`PUT /invoice/{id}` (`update-an-invoice`).

## 3. Deliver it

- `PUT /invoice/{id}/send/email` (`send-an-invoice-via-email`)
- `PUT /invoice/{id}/send/sms` (`send-an-invoice-via-sms`)

## 4. Take the money

- `POST /invoice/{id}/pay` (`pay-an-invoice`) — accepts the same
  `idempotency_id` semantics as `POST /charge`. **Always send one.**
- `POST /invoice/{id}/pay/{method}` (`mark-an-invoice-paid-in-cash-or-check`)
  to record an out-of-band payment.
- Or issue `POST /query/payment-links` (`create-a-payment-link`) for a hosted
  link, button or QR code with custom input fields.

## 5. Recurring

`POST /invoice/schedule/` (`createScheduledInvoice`) emits invoices on a
recurrence rule. A `422` here means DTSTART is in the past, the recurrence rule
is malformed, or a field value is invalid. Amend with
`PUT /invoice/schedule/{id}`, stop with `DELETE /invoice/schedule/{id}`.

## 6. Watch it land

Subscribe merchant webhooks (`POST /webhook`) for `create_invoice`,
`send_invoice`, `update_invoice`, `create_transaction` and
`update_transaction_settled`. The event name arrives in the `stax-event-name`
header; the body is the object itself. Your receiver must serve a public-CA
certificate and negotiate TLS 1.2 or 1.3 — self-signed certificates are
rejected. Retry behaviour is configured per webhook through
`meta.response_body`, `meta.webhook_retry_frequency` and
`meta.webhook_retry_count`; Stax signs nothing, so verify by re-reading the
object through the API rather than trusting the payload.

## Reversing

An invoice payment is a transaction: void it before settlement
(`POST /transaction/{id}/void`), refund it after
(`POST /transaction/{id}/refund`), or let Stax choose with
`POST /transaction/{id}/void-or-refund`.
