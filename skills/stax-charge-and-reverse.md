---
name: stax-charge-and-reverse
description: Take a card or ACH payment through the Stax API with an idempotency key, then reverse it correctly — void before settlement, refund after.
api: Stax API
operations:
  - create-customer
  - create-a-payment-method
  - charge-a-payment-method
  - get-a-transactions-information
  - void-transaction
  - refund-transaction
  - void-or-refund-transaction
---

# Charge and reverse a payment

Base URL `https://apiprod.fattlabs.com`. Every request carries
`Authorization: Bearer <merchant API key>` and `Content-Type: application/json`.
A sandbox key uses this same base URL and is bound to a test gateway.

## 1. Make sure there is a customer

`POST /customer` (`create-customer`) — or reuse an existing one via
`GET /customer` (`find-all-customers`), which accepts `email`, `firstname`,
`lastname`, `company`, `reference` and `keywords[]` filters and returns a
paginated envelope (`total`, `per_page`, `current_page`, `data`).

## 2. Store a payment method

`POST /payment-method` (`create-a-payment-method`) against the customer id.
**Do not collect raw card data server-side.** Tokenize in the browser with
Stax.js (`https://staxjs.staxpayments.com/staxjs-captcha.js`) using the public
Web Payments token, never the secret API key.

## 3. Charge — always send an idempotency key

`POST /charge` (`charge-a-payment-method`) with `payment_method_id`, `total`
and `meta`. Include `idempotency_id`: a unique string of up to 255 characters,
UUID recommended.

- Replay with the same `idempotency_id` returns the **original** transaction
  instead of creating a second charge.
- Replay with the same key but different parameters returns an error naming the
  differing parameters.
- `meta.tax` between 0.1% and 30% of the total qualifies the transaction for
  Level 2 interchange rates.

`idempotency_id` is documented in the guides but is **not** declared as a
parameter on `POST /charge` in the published OpenAPI — send it anyway.

## 4. Reverse it

The line is **settlement**, not time:

| Situation | Operation | Path |
|---|---|---|
| Not yet settled | `void-transaction` | `POST /transaction/{id}/void` |
| Already settled | `refund-transaction` | `POST /transaction/{id}/refund` |
| You don't know | `void-or-refund-transaction` | `POST /transaction/{id}/void-or-refund` |

Prefer `void-or-refund-transaction` when the settlement state is unknown — Stax
picks the right one. Voids are full-amount only; refunds can be partial.
Voids and refunds appear as child transactions of the original in
`child_transactions`.

Card-present transactions taken through `/terminal/charge` reverse through
`/terminal/void`, `/terminal/refund` or `/terminal/void-or-refund`.

## 5. Errors and retries

- `422` returns a field-keyed object of message arrays, e.g.
  `{"total": ["The total field is required."]}` — there is no error code to
  branch on, only the status and the field keys.
- `401` may return a bare string array, e.g. `["not team admin"]`.
- On `429`, honour `Retry-After`. **Do not retry a 4xx in a loop:** Stax blocks
  an IP for one hour after 10 failed requests in a minute, on top of the
  200 requests/minute cap (the reference Overview page states 100/minute — plan
  for the lower number).

## Testing

Use a sandbox key. On legacy sandbox accounts (created before 2026-07-17) the
success card is `4111111111111111` and the failure card is `4012888888881881`;
ACH succeeds only with routing `021000021` / account `9876543210`. Failure cards
tokenize successfully and fail at charge time with the generic message
"Unable to process the purchase transaction" — the real issuer reason only
appears on a live gateway.
