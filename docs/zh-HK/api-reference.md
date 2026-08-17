# API 文件

Merchant API 基礎路徑：`https://{host}/api/v1`

所有 `/merchant/*` 端點均需 [HMAC 簽章認證](authentication.md)。

## API 伺服器地址

測試（支援測試網）：`https://merchant-qa.hashkeymerchant.com`

Staging 環境（僅支援主網 token）：`https://merchant-stg.hashkeymerchant.com`

生產環境（僅支援主網 token）：`https://hsp.hashkey.com`


## API 總覽

| 方法 | 路徑 | 說明 | 認證 |
|------|------|------|------|
| **POST** | `/merchant/orders` | 建立單次支付訂單 | HMAC |
| **POST** | `/merchant/orders/reusable` | 建立可重複支付訂單 | HMAC |
| **GET** | `/merchant/payments` | 查詢單次支付記錄 | HMAC |
| **GET** | `/merchant/payments/reusable` | 查詢可重複支付記錄 | HMAC |

---

## 建立單次支付訂單

`POST /merchant/orders`

> 建立單次支付訂單，返回支付網關連結。同一 `cart_mandate_id` 不可重複建立。

**認證**：HMAC 簽章（`X-App-Key` + `X-Signature` + `X-Timestamp` + `X-Nonce`）

### 請求本文

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
            {"label": "商品 A", "amount": {"currency": "USD", "value": "10.00"}},
            {"label": "商品 B", "amount": {"currency": "USD", "value": "5.00"}}
          ],
          "total": {"label": "總計", "amount": {"currency": "USD", "value": "15.00"}},
          "modifiers": [
            {
              "total": {"currency": "USD", "value": "16.00"},
              "additional_display_items": [
                {
                  "label": "服務費",
                  "amount": {"currency": "USD", "value": "1.00"},
                  "pending": true,
                  "refund_period": 0
                }
              ],
              "data": {"method_data_indexes": [0]}
            }
          ]
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

### 請求參數說明

| 路徑 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `cart_mandate.contents.id` | string | 是 | 訂單 ID（`cart_mandate_id`，ID1），商戶自訂 |
| `cart_mandate.contents.user_cart_confirmation_required` | bool | 是 | 是否需要用戶確認 |
| `cart_mandate.contents.payment_request.method_data` | array | 是 | 支付方式清單 |
| `method_data[].supported_methods` | string | 是 | 現時支援 `"https://www.x402.org/"`，日後將有更多擴展 |
| `method_data[].data.x402Version` | int | 是 | 固定為 `2` |
| `method_data[].data.network` | string | 是 | 網絡名稱（如 `sepolia`） |
| `method_data[].data.chain_id` | int | 是 | 鏈 ID（如 `11155111`） |
| `method_data[].data.contract_address` | string | 是 | 代幣合約地址 |
| `method_data[].data.pay_to` | string | 是 | 收款地址 |
| `method_data[].data.coin` | string | 是 | 代幣（如 `USDC`） |
| `details.id` | string | 是 | 支付請求 ID（`payment_request_id`，ID2） |
| `details.display_items` | array | 否 | 商品明細清單 |
| `details.total` | object | 是 | 商品總金額（`label` + `amount`）；幣種固定為 `USD` |
| `details.modifiers` | array | 否 | 按所選支付方式套用的額外費用 |
| `details.modifiers[].total` | object | 是 | 最終支付金額（`currency` + `value`）；幣種固定為 `USD` |
| `details.modifiers[].additional_display_items` | array | 是 | 額外費用清單；Checkout 會匯總顯示多個項目 |
| `details.modifiers[].additional_display_items[].label` | string | 是 | 額外費用說明 |
| `details.modifiers[].additional_display_items[].amount` | object | 是 | 額外費用金額；幣種固定為 `USD` |
| `details.modifiers[].additional_display_items[].pending` | bool | 是 | 固定為 `true` |
| `details.modifiers[].additional_display_items[].refund_period` | int | 是 | 暫不支援；請設為 `0` |
| `details.modifiers[].data.method_data_indexes` | int array | 是 | `payment_request.method_data` 的有效零起始索引 |
| `contents.cart_expiry` | string | 是 | 授權過期時間（RFC 3339 格式），建議 2 小時 |
| `contents.merchant_name` | string | 是 | 商戶名稱 |
| `merchant_authorization` | string | 是 | 商戶 JWT 簽章（ES256K），詳見 [認證與簽章](authentication.md) |
| `redirect_url` | string | 否 | 支付完成後跳轉 URL |

> [!IMPORTANT]
> 每個修改器的 `modifier.total.value` 必須等於 `details.total.amount.value` 加上 `additional_display_items[].amount.value` 的總和。如有傳入 `display_items`，其金額總和必須等於 `details.total.amount.value`。修改器只會套用於 `method_data_indexes` 指定的支付方式。詳情請參閱 [Cart Mandate 組裝](cart-mandate.md#modifiers支付修改器)。

### 成功回應

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

| 欄位 | 類型 | 說明 |
|------|------|------|
| `payment_request_id` | string | 支付請求 ID（ID2） |
| `payment_url` | string | 支付網關頁面連結，將此連結傳送予用戶 |
| `multi_pay` | bool | 單次支付訂單固定返回 `false` |

> [!NOTE]
> 同一個 `cart_mandate_id` 不可重複建立單次支付訂單，重複提交會返回 `40001` 錯誤。若需同一裝置／商品下多次支付，請使用可重複支付訂單 API。

---

## 建立可重複支付訂單

`POST /merchant/orders/reusable`

> 建立可重複支付訂單，同一 `cart_mandate_id` 可於其後多次發起支付。

**認證**：HMAC 簽章

**請求本文**：與單次支付訂單相同（見上文）。

**回應**：與單次支付訂單相同，但 `multi_pay` 固定返回 `true`。

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
> 可重複支付訂單場景的 `cart_expiry` 應涵蓋整個業務生命週期（如裝置租賃期），建議設定為 365 天或更長。設定過短會導致其後無法發起新支付。

---

## 查詢單次支付訂單

`GET /merchant/payments`

> 查詢單次支付訂單的支付記錄。以下三個參數**擇一**。

**認證**：HMAC 簽章

### 查詢參數

| 參數 | 位置 | 類型 | 說明 |
|------|------|------|------|
| `cart_mandate_id` | query | string | 按訂單 ID（ID1）查詢，返回陣列 |
| `payment_request_id` | query | string | 按支付請求 ID（ID2）查詢，返回單筆 |
| `flow_id` | query | string | 按支付流程 ID（ID3）查詢，返回單筆 |

### 請求範例

```bash
# 按 cart_mandate_id 查詢
GET /api/v1/merchant/payments?cart_mandate_id=ORDER-20240301-001

# 按 payment_request_id 查詢
GET /api/v1/merchant/payments?payment_request_id=PAY-REQ-20240301-001

# 按 flow_id 查詢
GET /api/v1/merchant/payments?flow_id=b660fdc3-ac04-437f-921f-efbfb8d089f7
```

### 回應範例（按 payment_request_id 或 flow_id）

```json
{
  "code": 0,
  "data": {
    "app_key": "ABCD1234",
    "base_fee": "0.0010",
    "broadcast_at": "2026-01-06T15:06:30Z",
    "chain": "eip155:11155111",
    "completed_at": "2026-01-06T15:10:30Z",
    "content_id": "cart_123456",
    "created_at": "2026-01-06T15:04:05Z",
    "deadline_time": "2026-01-06T16:04:05Z",
    "extra_protocol": "eip3009",
    "flow_id": "FLOW123456",
    "gas_fee": "0.05",
    "gas_fee_advanced": true,
    "gas_fee_amount": "0.000045",
    "gas_limit": 100000,
    "included_at": "2026-01-06T15:08:00Z",
    "network": "sepolia",
    "network_fee": "0.05",
    "pay_amount": "100.30",
    "payer_address": "0x1234567890abcdef",
    "payment_request_id": "7glTuOSMKPZwmXZV4hbS",
    "order_amount": "100.00",
    "product_amount": "99.00",
    "request_id": "20260106150405123456",
    "status": "payment-finalized",
    "status_reason": "验证失败",
    "to_pay_address": "0xabcdef1234567890",
    "token": "USDC",
    "token_address": "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
    "tx_signature": "0xabcd1234...",
    "updated_at": "2026-01-06T15:05:30Z",
    "usd_additional_amount": "1.00",
    "usd_amount": "100.25",
    "usd_amount_rate": "1",
    "usd_pay_amount": "100.30",
    "usd_product_amount": "99.25"
  },
  "msg": "string"
}
```

按 `cart_mandate_id` 查詢時，`data` 會返回由相同支付記錄物件組成的陣列。

---

## 查詢可重複支付訂單

`GET /merchant/payments/reusable`

> 查詢可重複支付訂單的支付記錄，支援分頁。以下查詢方式**擇一**。

**認證**：HMAC 簽章

### 查詢參數

| 參數 | 位置 | 類型 | 預設值 | 說明 |
|------|------|------|--------|------|
| `cart_mandate_id` | query | string | — | 按訂單 ID（ID1）分頁查詢 |
| `flow_id` | query | string | — | 按支付流程 ID（ID3）分頁查詢 |
| `request_id` | query | string | — | 按支付明細 ID（ID5）查詢單筆 |
| `page` | query | int | `1` | 頁碼 |
| `page_size` | query | int | `20` | 每頁數量 |

### 請求範例

```bash
# 按 cart_mandate_id 分頁查詢
GET /api/v1/merchant/payments/reusable?cart_mandate_id=DEVICE-001&page=1&page_size=20

# 按 flow_id 分頁查詢
GET /api/v1/merchant/payments/reusable?flow_id=b660fdc3-ac04-437f-921f-efbfb8d089f7&page=1&page_size=20

# 按 request_id 查詢單筆
GET /api/v1/merchant/payments/reusable?request_id=req_20240301_abc123
```

### 分頁回應範例

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "list": [
      {
        "app_key": "ABCD1234",
        "base_fee": "0.0010",
        "broadcast_at": "2026-01-06T15:06:30Z",
        "chain": "eip155:11155111",
        "completed_at": "2026-01-06T15:10:30Z",
        "content_id": "cart_123456",
        "created_at": "2026-01-06T15:04:05Z",
        "deadline_time": "2026-01-06T16:04:05Z",
        "extra_protocol": "eip3009",
        "flow_id": "FLOW123456",
        "gas_fee": "0.05",
        "gas_fee_advanced": true,
        "gas_fee_amount": "0.000045",
        "gas_limit": 100000,
        "included_at": "2026-01-06T15:08:00Z",
        "network": "sepolia",
        "network_fee": "0.05",
        "pay_amount": "100.30",
        "payer_address": "0x1234567890abcdef",
        "payment_request_id": "7glTuOSMKPZwmXZV4hbS",
        "order_amount": "100.00",
        "product_amount": "99.00",
        "request_id": "20260106150405123456",
        "status": "payment-finalized",
        "status_reason": "验证失败",
        "to_pay_address": "0xabcdef1234567890",
        "token": "USDC",
        "token_address": "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
        "tx_signature": "0xabcd1234...",
        "updated_at": "2026-01-06T15:05:30Z",
        "usd_additional_amount": "1.00",
        "usd_amount": "100.25",
        "usd_amount_rate": "1",
        "usd_pay_amount": "100.30",
        "usd_product_amount": "99.25"
      }
    ],
    "total": 15,
    "page": 1,
    "page_size": 20
  }
}
```

| 欄位 | 類型 | 說明 |
|------|------|------|
| `list` | array | 支付記錄清單，按 `created_at` 倒序排列 |
| `total` | int | 支付記錄總數 |
| `page` | int | 目前頁碼 |
| `page_size` | int | 每頁數量 |

---

## 支付記錄欄位說明

以下欄位在所有查詢 API 的 `PaymentItemResponse` 中返回：

| 欄位 | 類型 | 說明 |
|------|------|------|
| `content_id` | string | `cart_mandate.contents.id` |
| `payment_request_id` | string | 支付請求 ID（ID2） |
| `request_id` | string | 支付記錄唯一識別（ID5） |
| `flow_id` | string | 支付流程 ID（ID3） |
| `app_key` | string | 應用程式識別 |
| `product_amount` | string | 商品金額（幣本位） |
| `order_amount` | string | 訂單金額（幣本位），即商品金額加額外費用 |
| `pay_amount` | string | 支付數量（幣本位） |
| `usd_product_amount` | string | 商品金額（USD） |
| `usd_additional_amount` | string | 額外費用（USD） |
| `usd_amount` | string | 金額（USD） |
| `usd_amount_rate` | string | USD 與幣種的匯率 |
| `usd_pay_amount` | string | 最終支付金額（USD） |
| `token` | string | 代幣 |
| `token_address` | string | 代幣合約地址 |
| `payer_address` | string | 付款人錢包地址 |
| `to_pay_address` | string | 收款地址 |
| `chain` | string | 鏈識別（CAIP-2 格式，如 `eip155:11155111`） |
| `network` | string | 區塊鏈網絡名稱 |
| `extra_protocol` | string | 鏈上支付協議（`eip3009` / `permit2`） |
| `base_fee` | string | 基礎費用 |
| `network_fee` | string | 網絡費用 |
| `gas_fee` | string | Gas 費用 |
| `gas_fee_amount` | string | Gas 費用金額 |
| `gas_fee_advanced` | bool | 是否由商戶墊付 Gas 費用 |
| `gas_limit` | int | Gas 上限 |
| `status` | string | 支付狀態（見[支付狀態機](cart-mandate.md#支付狀態機)） |
| `status_reason` | string? | 狀態原因 |
| `tx_signature` | string | 鏈上交易Hash（交易打包後返回） |
| `deadline_time` | string | 支付截止時間（RFC 3339） |
| `created_at` | string | 建立時間（RFC 3339） |
| `updated_at` | string | 更新時間（RFC 3339） |
| `broadcast_at` | string? | 首次廣播時間（RFC 3339） |
| `included_at` | string? | 交易被打包進區塊的時間（RFC 3339） |
| `completed_at` | string? | 完成時間（RFC 3339） |

---

## 統一回應格式

所有 API 回應統一使用以下結構：

```json
{
  "code": 0,
  "msg": "success",
  "data": { ... }
}
```

- `code = 0` 表示成功
- `code != 0` 表示錯誤，`msg` 欄位為英文錯誤描述
- 錯誤碼詳見 [附錄 — 錯誤碼](appendix.md#錯誤碼)
