---
name: stax-onboard-a-merchant
description: Onboard a sub-merchant onto a Stax Connect partner brand — create, enroll, supply registration data and documents, then watch underwriting to ACTIVE.
api: Stax API
operations:
  - create-a-merchant
  - enroll-a-merchant
  - get-merchant-registration-data
  - update-merchant-registration-data
  - add-file-to-merchant-registration
  - delete-file-from-merchant-registration
  - get-registration-documents
  - retrieve-terms-and-conditions
  - create-a-new-api-key-for-merchant
  - list-api-keys-for-merchant
  - get-merchant-by-id
  - get-all-merchants
  - create-partner-level-webhook-for-your-brand
---

# Onboard a sub-merchant (Partner API)

This flow needs a **partner** API key (`PartnerApiKey`), not a merchant key.
Base URL `https://apiprod.fattlabs.com`.

## 1. Subscribe to the outcome first

Before you create anything, register partner-brand webhooks with
`POST /webhookadmin/webhook/brand`
(`create-partner-level-webhook-for-your-brand`) for:

- `create_merchant`
- `update_underwriting` — keep listening until
  `registration.underwriting_status` is `APPROVED`
- `update_merchant_status` — when `merchant.status` is `ACTIVE` the merchant can
  process
- `update_electronic_signature`

Partner webhooks fire for every sub-merchant under your brand.

## 2. Create or enroll

- `POST /merchant` (`create-a-merchant`) creates the merchant record. A `422`
  lists the missing fields, typically `company_name` and `contact_email`.
- `POST /admin/enroll` (`enroll-a-merchant`) does merchant + registration + user
  in one request — the hybrid path.

## 3. Registration data and documents

- `GET /merchant/{id}/registration` (`get-merchant-registration-data`)
- `PUT /merchant/{id}/registration` (`update-merchant-registration-data`) —
  upsert
- `GET /underwriting/merchant/{id}/registration/documents`
  (`get-registration-documents`) tells you what underwriting needs
- `POST /merchant/{id}/registration/file` (`add-file-to-merchant-registration`)
  uploads one; `DELETE .../file/{file_id}` removes it
- `GET /underwriting/terms-and-conditions/{id}`
  (`retrieve-terms-and-conditions`) for the signer

## 4. Issue the merchant its key

Once `merchant.status` is `ACTIVE`:
`POST /merchant/{id}/apikey` (`create-a-new-api-key-for-merchant`), audited with
`GET /merchant/{id}/apikey` (`list-api-keys-for-merchant`). That key is
merchant-scoped and cannot reach partner operations.

For single sign-on into Stax from your own UI, mint a 24-hour token with
`GET /ephemeral` (`ephemeral-authentication-tokens-1`).

## 5. Test it first

Partner-sandbox access is gated: you need a valid partner account with signed
documents, and you request the API sandbox first at
`https://staxpayments.com/request-sandbox/`. In Stax Connect turn on **Test
Mode**, then add a test merchant. Note that in sandbox the `create_deposit`
webhook's `external_id` does not match `batch_id` — treat it as a notification
and call the deposit APIs for detail.

## Reversibility

Merchant creation has no documented undo. Registration **files** can be deleted
(`delete-file-from-merchant-registration`); registration data is upserted, not
versioned. Plan enrollment as a forward-only flow.
