# Webhook payment notifications

When a payment reaches `payment-successful`, `payment-included`, or `payment-failed`, the gateway POSTs to your configured `webhook_url` so you do not need to poll.

---

## Configuration

`webhook_url` is stored per application (AppCredential) in the merchant console:

1. Sign in → **Applications**
2. Edit the app
3. Set **Payment webhook URL** (`webhook_url`) — **HTTPS only**

**Requirements**

- Respond with HTTP **2xx within 10 seconds** or the delivery is retried
- If `webhook_url` is empty, no callbacks are sent—poll the APIs instead

---

## Signature (HMAC-SHA256)

Each callback includes:

```
X-Signature: t=<unix_timestamp>,v1=<hmac_hex>
```

| Part | Meaning |
|------|---------|
| `t` | Unix seconds when signed |
| `v1` | HMAC-SHA256 hex digest |

### Payload

```
message   = timestamp + "." + raw_request_body
signature = hex(HMAC-SHA256(app_secret, message))
```

The key is your application **`app_secret`**.

### Verification checklist

1. Parse `t` and `v1` from `X-Signature`
2. Ensure `|now - t| ≤ 300` seconds
3. Recompute HMAC with the raw body
4. Compare with `crypto/subtle` / `hmac.Equal` (constant time)

### Go example

```go
func verifyWebhookSignature(r *http.Request, rawBody []byte, appSecret string) error {
    sig := r.Header.Get("X-Signature")
    if sig == "" {
        return fmt.Errorf("missing X-Signature header")
    }

    var ts int64
    var received string
    for _, part := range strings.Split(sig, ",") {
        if strings.HasPrefix(part, "t=") {
            ts, _ = strconv.ParseInt(strings.TrimPrefix(part, "t="), 10, 64)
        } else if strings.HasPrefix(part, "v1=") {
            received = strings.TrimPrefix(part, "v1=")
        }
    }

    if math.Abs(float64(time.Now().Unix()-ts)) > 300 {
        return fmt.Errorf("timestamp out of tolerance")
    }

    msg := fmt.Sprintf("%d.%s", ts, rawBody)
    mac := hmac.New(sha256.New, []byte(appSecret))
    mac.Write([]byte(msg))
    expected := hex.EncodeToString(mac.Sum(nil))

    if !hmac.Equal([]byte(expected), []byte(received)) {
        return fmt.Errorf("signature mismatch")
    }
    return nil
}
```

---

## HTTP request

| Header | Value |
|--------|-------|
| `Content-Type` | `application/json` |
| `X-Signature` | `t=...,v1=...` |

---

## Payload fields

### Common

| Field | Type | Description |
|-------|------|-------------|
| `event_type` | string | Always `payment` |
| `payment_request_id` | string | ID2 |
| `request_id` | string | ID5 |
| `cart_mandate_id` | string | ID1 |
| `payer_address` | string | Payer wallet |
| `amount` | string | Smallest units |
| `token` | string | Symbol |
| `token_address` | string | Contract |
| `chain` | string | CAIP-2 |
| `network` | string | Network name |
| `status` | string | `payment-successful` / `payment-failed` / `payment-included` |
| `created_at` | string | RFC 3339 |

### Success extras

| Field | Type |
|-------|------|
| `tx_signature` | string |
| `completed_at` | string |

### Failure extras

| Field | Type |
|-------|------|
| `status_reason` | string |

---

## Samples

### Success

```json
{
  "event_type": "payment",
  "payment_request_id": "PAY-REQ-20240301-001",
  "request_id": "req_20240301_abc123",
  "cart_mandate_id": "ORDER-20240301-001",
  "payer_address": "0x1234567890abcdef1234567890abcdef12345678",
  "amount": "15000000",
  "token": "USDC",
  "token_address": "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
  "chain": "eip155:11155111",
  "network": "sepolia",
  "status": "payment-successful",
  "created_at": "2024-03-01T10:00:00Z",
  "tx_signature": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab",
  "completed_at": "2024-03-01T10:01:30Z"
}
```

### Failure

```json
{
  "event_type": "payment",
  "payment_request_id": "PAY-REQ-20240301-001",
  "request_id": "req_20240301_def456",
  "cart_mandate_id": "ORDER-20240301-001",
  "payer_address": "0x1234567890abcdef1234567890abcdef12345678",
  "amount": "15000000",
  "token": "USDC",
  "token_address": "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
  "chain": "eip155:11155111",
  "network": "sepolia",
  "status": "payment-failed",
  "created_at": "2024-03-01T10:00:00Z",
  "status_reason": "Transaction reverted on chain"
}
```

---

## Your HTTP response

Return **2xx within 10 seconds**:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"code": 0}
```

Only the status code is inspected.

**Best practices**

- Validate business fields (`amount`, `token`, `cart_mandate_id`) before side effects
- Handlers must be **idempotent**—the same `request_id` may arrive more than once due to retries

---

## Retries

Failed deliveries retry up to **6** times:

| Failure # | Delay |
|-----------|-------|
| 1 | 1 minute |
| 2 | 5 minutes |
| 3 | 15 minutes |
| 4 | 1 hour |
| 5 | 6 hours |
| 6 | 24 hours |

After the final failure the notification is marked `FAILED` and will not retry—query the API for the ground truth.

> [!NOTE]
> Each `request_id` is delivered at most once at the application layer thanks to internal idempotency.
