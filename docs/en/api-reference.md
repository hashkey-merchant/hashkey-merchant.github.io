# API reference

Merchant API base path: `https://{host}/api/v1`

All `/merchant/*` routes require [HMAC authentication](authentication.md).

## Base URLs

QA (testnet + mainnet tokens): `https://merchant-qa.hashkeymerchant.com`

Staging (mainnet tokens only): `https://merchant-stg.hashkeymerchant.com`

Production (mainnet tokens only): `https://merchant.hashkey.com`

## Overview

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| **POST** | `/merchant/orders` | Create one-time payment order | HMAC |
| **POST** | `/merchant/orders/reusable` | Create reusable payment order | HMAC |
| **GET** | `/merchant/payments` | Query one-time payments | HMAC |
| **GET** | `/merchant/payments/reusable` | Query reusable payments | HMAC |

---

## Create one-time order

`POST /merchant/orders`

> Creates a one-time order and returns a checkout URL. The same `cart_mandate_id` cannot be used twice.

**Auth**: HMAC (`X-App-Key`, `X-Signature`, `X-Timestamp`, `X-Nonce`)

### Request body

```json
{
  "cart_mandate": {
    "contents": {
      "id": "ORDER-20240301-001",
      "user_cart_confirmation_required": true,
      "payment_request": {
        "method_data": [
          {
            "supported_methods": "https://www.x402.org/",
            "data": {
              "x402Version": 2,
              "network": "sepolia",
              "chain_id": 11155111,
              "contract_address": "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
              "pay_to": "0x99c12865b7e4e0cb8a708b448909b76aa1729afd",
              "coin": "USDC"
            }
          }
        ],
        "details": {
          "id": "PAY-REQ-20240301-001",
          "display_items": [
            {"label": "Item A", "amount": {"currency": "USD", "value": "10.00"}},
            {"label": "Item B", "amount": {"currency": "USD", "value": "5.00"}}
          ],
          "total": {"label": "Total", "amount": {"currency": "USD", "value": "15.00"}}
        }
      },
      "cart_expiry": "2024-03-01T12:00:00Z",
      "merchant_name": "My Store"
    },
    "merchant_authorization": "eyJhbGciOiJFUzI1NksiLCJ0eXAiOiJKV1QifQ..."
  },
  "redirect_url": "https://yoursite.com/payment/callback"
}
```

### Field reference

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `cart_mandate.contents.id` | string | Yes | Order id (`cart_mandate_id`, ID1) |
| `cart_mandate.contents.user_cart_confirmation_required` | bool | Yes | Require user confirmation |
| `cart_mandate.contents.payment_request.method_data` | array | Yes | Payment methods |
| `method_data[].supported_methods` | string | Yes | Currently `"https://www.x402.org/"` |
| `method_data[].data.x402Version` | int | Yes | Must be `2` |
| `method_data[].data.network` | string | Yes | Network name (e.g. `sepolia`) |
| `method_data[].data.chain_id` | int | Yes | Chain id |
| `method_data[].data.contract_address` | string | Yes | Token contract |
| `method_data[].data.pay_to` | string | Yes | Payee address |
| `method_data[].data.coin` | string | Yes | Symbol (e.g. `USDC`) |
| `details.id` | string | Yes | `payment_request_id` (ID2) |
| `details.display_items` | array | No | Line items |
| `details.total` | object | Yes | Total (`label` + `amount`) |
| `contents.cart_expiry` | string | Yes | RFC 3339 expiry (~2h typical) |
| `contents.merchant_name` | string | Yes | Merchant display name |
| `merchant_authorization` | string | Yes | ES256K JWT — [Authentication](authentication.md) |
| `redirect_url` | string | No | Post-payment redirect |

### Success response

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "payment_request_id": "PAY-REQ-20240301-001",
    "payment_url": "https://pay.hashkey.com/flow/xxx",
    "multi_pay": false
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `payment_request_id` | string | ID2 |
| `payment_url` | string | Checkout URL for the payer |
| `multi_pay` | bool | Always `false` for one-time orders |

> [!NOTE]
> Duplicate `cart_mandate_id` returns `40001`. For multiple charges on the same device/sku, use the reusable order API.

---

## Create reusable order

`POST /merchant/orders/reusable`

> Same `cart_mandate_id` can be paid multiple times over its lifetime.

**Auth**: HMAC

**Body / response**: Same shape as one-time, except `multi_pay` is `true`.

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "payment_request_id": "PAY-REQ-001",
    "payment_url": "https://pay.hashkey.com/flow/xxx",
    "multi_pay": true
  }
}
```

> [!IMPORTANT]
> Set `cart_expiry` to cover the full lifecycle (e.g. rental period). Too short blocks future payments.

---

## Query one-time payments

`GET /merchant/payments`

> Exactly **one** of the query parameters below.

**Auth**: HMAC

### Query parameters

| Name | In | Type | Description |
|------|-----|------|-------------|
| `cart_mandate_id` | query | string | By ID1 — returns an array |
| `payment_request_id` | query | string | By ID2 — single object |
| `flow_id` | query | string | By ID3 — single object |

### Examples

```bash
GET /api/v1/merchant/payments?cart_mandate_id=ORDER-20240301-001
GET /api/v1/merchant/payments?payment_request_id=PAY-REQ-20240301-001
GET /api/v1/merchant/payments?flow_id=b660fdc3-ac04-437f-921f-efbfb8d089f7
```

### Sample (`cart_mandate_id`)

```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "payment_request_id": "PAY-REQ-20240301-001",
      "request_id": "req_20240301_abc123",
      "token_address": "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
      "flow_id": "b660fdc3-ac04-437f-921f-efbfb8d089f7",
      "app_key": "ak_xxx",
      "amount": "15000000",
      "usd_amount": "15.00",
      "token": "USDC",
      "chain": "eip155:11155111",
      "network": "sepolia",
      "extra_protocol": "eip3009",
      "status": "payment-finalized",
      "payer_address": "0x1234...",
      "to_pay_address": "0x99c1...",
      "risk_level": "Low",
      "tx_signature": "0xabcd...",
      "broadcast_at": "2024-03-01T10:00:15Z",
      "gas_limit": 150000,
      "gas_fee": "0.001",
      "service_fee_rate": "0.0000",
      "service_fee_type": "free",
      "deadline_time": "2024-03-01T12:00:00Z",
      "created_at": "2024-03-01T10:00:00Z",
      "updated_at": "2024-03-01T10:01:30Z",
      "completed_at": "2024-03-01T10:01:30Z"
    }
  ]
}
```

---

## Query reusable payments

`GET /merchant/payments/reusable`

> Paginated list or single row. **One** filter at a time.

**Auth**: HMAC

### Query parameters

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `cart_mandate_id` | string | — | Page by ID1 |
| `flow_id` | string | — | Page by ID3 |
| `request_id` | string | — | Single row by ID5 |
| `page` | int | `1` | Page number |
| `page_size` | int | `20` | Page size |

### Examples

```bash
GET /api/v1/merchant/payments/reusable?cart_mandate_id=DEVICE-001&page=1&page_size=20
GET /api/v1/merchant/payments/reusable?flow_id=b660fdc3-ac04-437f-921f-efbfb8d089f7&page=1&page_size=20
GET /api/v1/merchant/payments/reusable?request_id=req_20240301_abc123
```

### Paged response

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "list": [
      {
        "payment_request_id": "pay_req_002",
        "request_id": "pay_002",
        "token_address": "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
        "flow_id": "b660fdc3-ac04-437f-921f-efbfb8d089f7",
        "app_key": "ak_xxx",
        "amount": "1000000",
        "usd_amount": "1.00",
        "token": "USDC",
        "chain": "eip155:11155111",
        "network": "sepolia",
        "status": "payment-finalized",
        "payer_address": "0x1234...",
        "to_pay_address": "0x99c1...",
        "tx_signature": "0xabcd...",
        "created_at": "2024-03-01T11:00:00Z",
        "completed_at": "2024-03-01T11:01:30Z"
      }
    ],
    "total": 15,
    "page": 1,
    "page_size": 20
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `list` | array | Rows, newest `created_at` first |
| `total` | int | Total count |
| `page` | int | Current page |
| `page_size` | int | Page size |

---

## Payment record fields

Returned on all payment queries (`PaymentItemResponse`):

| Field | Type | Description |
|-------|------|-------------|
| `payment_request_id` | string | ID2 |
| `request_id` | string | ID5 |
| `token_address` | string | Token contract |
| `flow_id` | string | ID3 |
| `app_key` | string | Application id |
| `amount` | string | Amount in smallest units (e.g. USDC 6 decimals) |
| `usd_amount` | string | USD notionals |
| `token` | string | Symbol |
| `chain` | string | CAIP-2 (e.g. `eip155:11155111`) |
| `network` | string | Network name |
| `extra_protocol` | string | `eip3009` / `permit2` |
| `status` | string | See [state machine](cart-mandate.md#payment-state-machine) |
| `status_reason` | string? | Failure reason |
| `payer_address` | string | Payer |
| `to_pay_address` | string | Payee |
| `risk_level` | string | AML risk |
| `tx_signature` | string | Tx hash once included on chain |
| `broadcast_at` | string? | First broadcast time |
| `gas_limit` | int | Gas limit |
| `gas_fee` | string | Gas fee |
| `gas_fee_amount` | string | Gas fee amount |
| `service_fee_rate` | string | Service fee rate |
| `service_fee_type` | string | `free` / `price_include` / `price_extra` |
| `deadline_time` | string | Payment deadline |
| `created_at` | string | Created |
| `updated_at` | string | Updated |
| `completed_at` | string? | Set when status is `payment-finalized` |

---

## Envelope

```json
{
  "code": 0,
  "msg": "success",
  "data": { }
}
```

- `code = 0` → success
- `code != 0` → error; `msg` is English
- See [Appendix — Error codes](appendix.md#error-codes)
