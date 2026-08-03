# HeloPay Merchant API — Developer Documentation

Markdown version of the interactive developer docs served at `/docs`.
Everything here mirrors that page: the same endpoints, the same groups, the same
example bodies and callback payloads.

## Base URL

```text
https://connect.cradlevoices.com/api/v1
```

All examples below use this base URL. The endpoint paths and request payloads
are identical in sandbox and production — only your account's environment
decides whether real money moves.

## Endpoint index

| Group | Endpoint | Method | Path |
| --- | --- | --- | --- |
| Get started | Generate access token | `GET` | `/` |
| Get started | Get account status | `GET` | `/account/status` |
| Card payments | Create payment | `POST` | `/card/s2s/initiate` |
| Card payments | Trigger 3DS test | `POST` | `/card/s2s/initiate` |
| Transactions | Get transaction status | `GET` | `/transaction` |
| Webhooks | Callback examples | `POST` | Your endpoint |

---

# 1. Get started

## 1.1 Generate access token

```http
GET /
```

Exchange your API username and password for a short-lived bearer token.

Generate the API username and password in the merchant dashboard under
**My Account → API Credentials**. The password is shown only once, when you
generate it.

```bash
curl --location 'https://connect.cradlevoices.com/api/v1/' \
  --user 'YOUR_API_USERNAME:YOUR_API_PASSWORD'
```

Response:

```json
{
  "token": "eyJhbGciOi...",
  "expires_at": "2026-07-31T14:16:48Z"
}
```

Pass the token on every protected request:

```http
Authorization: Bearer YOUR_TOKEN
```

Never expose the API password or the bearer token in browser code. Both belong
on your server only.

## 1.2 Get account status

```http
GET /account/status
```

Check whether your account is active and currently operating in sandbox or
production.

```bash
curl --location 'https://connect.cradlevoices.com/api/v1/account/status' \
  --header 'Authorization: Bearer YOUR_TOKEN'
```

Response:

```json
{
  "merchant_id": "example",
  "account_status": "active",
  "environment": "sandbox"
}
```

`environment` is `sandbox` while you integrate and `production` once your
account has been upgraded to live.

---

# 2. Card payments

Both card calls use the same endpoint and the same payload shape. The card
details you send decide the outcome.

**Security:** this endpoint receives raw card details. Only collect, transmit,
or store card data if you meet your PCI obligations. Always use HTTPS, and never
log full card numbers or CVV values.

## 2.1 Create payment

```http
POST /card/s2s/initiate
```

Create a sandbox or production card payment using the same endpoint and payload.

```bash
curl --location 'https://connect.cradlevoices.com/api/v1/card/s2s/initiate' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "impalaMerchantId": "YOUR_MERCHANT_ID",
  "currency": "USD",
  "amount": 10,
  "externalId": "ORDER-10001",
  "callbackUrl": "https://merchant.example/api/payment/webhook",
  "redirectUrl": "https://merchant.example/payment/return",
  "ip_address": "192.168.1.100",
  "customer_first_name": "John",
  "customer_last_name": "Doe",
  "customer_phone_code": "+44",
  "customer_phone": "7700900123",
  "customer_email": "john.doe@example.com",
  "billing_first_name": "John",
  "billing_last_name": "Doe",
  "billing_street": "10 Downing Street",
  "billing_city": "London",
  "billing_country": "GB",
  "billing_post_code": "SW1A 2AA",
  "billing_state": "London",
  "card_holder": "John Doe",
  "card_number": "4242424242424242",
  "card_exp_month": "01",
  "card_exp_year": "2028",
  "card_security_code": "123",
  "description": "Sandbox success test",
  "channel": "3D"
}'
```

### Request fields

| Field | Required | Description |
| --- | --- | --- |
| `impalaMerchantId` | Yes | Merchant ID issued by HeloPay. |
| `currency` | Yes | Transaction currency, for example `USD`, `KES`, or `RWF`. |
| `amount` | Yes | Amount to charge. |
| `externalId` | Yes | Your unique transaction reference. Must be unique for every request. |
| `callbackUrl` | Yes | Your webhook URL for the final transaction status. |
| `redirectUrl` | No | Page the customer returns to after 3DS/browser completion. |
| `ip_address` | Yes | Customer IP address. |
| `customer_first_name` | Yes | Customer first name. |
| `customer_last_name` | Yes | Customer last name. |
| `customer_phone_code` | No | Phone country code, for example `+44` or `254`. |
| `customer_phone` | Yes | Customer phone number. |
| `customer_email` | Yes | Customer email address. |
| `billing_first_name` | Yes | Billing first name. |
| `billing_last_name` | Yes | Billing last name. |
| `billing_street` | Yes | Billing street address. |
| `billing_city` | Yes | Billing city. |
| `billing_country` | Yes | Billing country code, for example `GB` or `KE`. |
| `billing_post_code` | Yes | Billing postal code. |
| `billing_state` | Yes | Billing state or region. |
| `card_holder` | Yes | Name printed on the card. |
| `card_number` | Yes | Card number. |
| `card_exp_month` | Yes | Expiry month in `MM` format. |
| `card_exp_year` | Yes | Expiry year in `YYYY` format. |
| `card_security_code` | Yes | Card CVV/CVC. |
| `description` | No | Payment description. |
| `channel` | No | Channel label, for example `3D`. |

### Successful initiation

HeloPay accepted the request for processing. If `payment_url` is returned,
redirect the customer there to complete authentication.

```json
{
  "status": "success",
  "message": "Payment initiated successfully",
  "payment_url": "https://connect.cradlevoices.com/payment/9af0b6c6d5a24f62a2f1cdb78c3a1001",
  "secure_id": "9af0b6c6d5a24f62a2f1cdb78c3a1001",
  "external_id": "ORDER-10001",
  "order_id": "ORDER-10001-1710000000",
  "amount": 10,
  "currency": "USD",
  "payment_status": "PENDING",
  "status_description": "PENDING",
  "description": "Sandbox success test"
}
```

The payment is **not final** until you receive a callback. Store `secure_id`,
`external_id`, `order_id`, amount, currency, and a pending status.

### Failed initiation

```json
{
  "status": "error",
  "error": "Payment initiation failed",
  "code": "INIT_FAIL",
  "message": "Payment initiation failed",
  "secure_id": "9af0b6c6d5a24f62a2f1cdb78c3a1001",
  "external_id": "ORDER-10001"
}
```

Never treat a failed initiation as a successful charge. Show a payment failure
message and let the customer retry with a **new** `externalId`.

## 2.2 Trigger 3DS test

```http
POST /card/s2s/initiate
```

Simulate a payment that requires customer authentication and returns a hosted
payment URL. Same endpoint as above — only the payload differs.

```bash
curl --location 'https://connect.cradlevoices.com/api/v1/card/s2s/initiate' \
  --header 'Authorization: Bearer YOUR_TOKEN' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "impalaMerchantId": "YOUR_MERCHANT_ID",
  "currency": "USD",
  "amount": 200,
  "externalId": "ORDER-3DS-10001",
  "callbackUrl": "https://merchant.example/api/payment/webhook",
  "redirectUrl": "https://merchant.example/payment/return",
  "ip_address": "192.168.1.100",
  "customer_first_name": "John",
  "customer_last_name": "Doe",
  "customer_phone_code": "+1",
  "customer_phone": "9876543210",
  "customer_email": "john.doe@example.com",
  "billing_first_name": "John",
  "billing_last_name": "Doe",
  "billing_street": "123 Main Street",
  "billing_city": "New York",
  "billing_country": "US",
  "billing_post_code": "10001",
  "billing_state": "NY",
  "card_holder": "John Doe",
  "card_number": "4000000000003220",
  "card_exp_month": "01",
  "card_exp_year": "2028",
  "card_security_code": "123",
  "description": "Sandbox 3DS test",
  "channel": "3D"
}'
```

If `payment_status` is `processing`, redirect the customer to the returned
`payment_url`. After authentication, the customer returns to your `redirectUrl`.

### Customer redirect flow

1. Initiate the payment using `/card/s2s/initiate`.
2. Store the returned `secure_id` and mark the order as pending.
3. If `payment_url` is present, redirect the customer to it.
4. HeloPay sends the customer to the processor's authentication page.
5. After completion, the customer returns to your `redirectUrl`.
6. Use the server-to-server callback as the source of truth for the final status.

---

# 3. Transactions

## 3.1 Get transaction status

```http
GET /transaction
```

Retrieve the latest result using your external order reference.

```bash
curl --location 'https://connect.cradlevoices.com/api/v1/transaction?merchant=YOUR_MERCHANT_ID&external_id=ORDER-10001' \
  --header 'Authorization: Bearer YOUR_TOKEN'
```

You can also look a transaction up by the `secureId` HeloPay returned:

```bash
curl --location 'https://connect.cradlevoices.com/api/v1/transaction?merchant=YOUR_MERCHANT_ID&secureId=9af0b6c6d5a24f62a2f1cdb78c3a1001' \
  --header 'Authorization: Bearer YOUR_TOKEN'
```

### Query parameters

| Field | Required | Description |
| --- | --- | --- |
| `merchant` | Yes | Your HeloPay merchant ID. |
| `secureId` | Required if `external_id` is not supplied | The `secureId` returned during initiation. `secure_id` also works. |
| `external_id` | Required if `secureId` is not supplied | Your own reference sent during initiation. `externalId` also works. |

Response:

```json
{
  "transaction": {
    "transaction_status": "COMPLETED",
    "transaction_report": "COMPLETE",
    "currency": "USD",
    "amount": 10.00,
    "secure_id": "9af0b6c6d5a24f62a2f1cdb78c3a1001",
    "external_id": "ORDER-10001",
    "direction": "payin",
    "callback_url": "https://merchant.example/api/payment/webhook",
    "date_added": 1782912789
  }
}
```

### Status values

| Status | Meaning |
| --- | --- |
| `PENDING` | Initiated, waiting for customer action or a provider callback. |
| `COMPLETED` | Completed successfully. |
| `COMPLETE` | Completed successfully. Some callbacks use this form. |
| `FAILED` | Failed, rejected, timed out, or cancelled. |
| `EXPIRED` | Expired before completion. |

Use the callback as the primary source of truth. Use this endpoint for
reconciliation, customer support, and delayed-callback checks. `direction` is
`payin` for collections and `payout` for withdrawals.

---

# 4. Webhooks

## 4.1 Callback examples

```http
POST  →  your callbackUrl
```

Use these examples to build and test your server-side webhook handler.

Return HTTP 2xx quickly, process each `secureId` once, and use the callback —
not the browser redirect — as the final payment result.

```json
{
  "success": {
    "transactionStatus": "COMPLETE",
    "secureId": "SECURE_ID",
    "externalId": "ORDER-10001",
    "amount": 10,
    "currency": "USD",
    "message": "Payment completed"
  },
  "declined": {
    "transactionStatus": "FAILED",
    "secureId": "SECURE_ID",
    "externalId": "ORDER-10002",
    "amount": 10,
    "currency": "USD",
    "message": "Payment failed"
  },
  "pending": {
    "transactionStatus": "PENDING",
    "secureId": "SECURE_ID",
    "externalId": "ORDER-10003",
    "amount": 200,
    "currency": "USD",
    "message": "Payment authentication required"
  }
}
```

### Webhook requirements

Your endpoint must:

- Accept `POST` with `Content-Type: application/json`.
- Return HTTP 200 after processing.
- Reconcile using `externalId` and `secureId`.
- Treat duplicate callbacks as idempotent — the same `secureId` may arrive more
  than once.
- Never mark an order paid from browser query parameters alone.

---

# Sandbox and live

Your account processes in one environment at a time. Check it with
`/account/status`, or read the badge in the dashboard header.

- **Sandbox** — requests go to provider test environments. Transactions are
  recorded and appear in the dashboard under **Test**, but no real money moves.
  Sandbox activity never credits or debits your wallet balance and is excluded
  from live reporting.
- **Live** — real payments settle to your wallet.

Your integration code does not change between the two: same base URL, same
endpoints, same payloads. When your sandbox testing is complete, ask your
HeloPay administrator to upgrade the account to live.

# Errors

| Status | Meaning |
| --- | --- |
| 400 | Invalid or missing fields in the request body. |
| 401 | Missing, invalid, or expired bearer token. |
| 403 | Account not permitted to perform the request. |
| 404 | Transaction not found for the supplied reference. |
| 500 | Processing error on the HeloPay side — safe to retry with the same `externalId`. |

Error responses carry a human-readable `error` message, and often a `details`
field with the underlying cause:

```json
{
  "error": "Invalid or expired token"
}
```

# Integration checklist

1. Generate API credentials in the dashboard and store them server-side.
2. Fetch a bearer token and refresh it before `expires_at`.
3. Confirm your environment with `/account/status`.
4. Initiate a payment and store `secure_id` plus your `externalId`.
5. Handle the redirect when `payment_url` is returned.
6. Implement the callback endpoint and make it idempotent.
7. Reconcile with `/transaction` for anything that does not call back.
8. Test the success, declined, and 3DS paths in sandbox before going live.
