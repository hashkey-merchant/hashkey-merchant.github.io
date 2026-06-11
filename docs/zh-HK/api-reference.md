# API 文件

Merchant API 基礎路徑：`https://{host}/api/v1`

所有 `/merchant/*` 端點均需 [HMAC 簽章認證](authentication.md)。

## API 伺服器地址

測試（支援測試網／主網 token）：`https://merchant-qa.hashkeymerchant.com`

Staging 環境（僅支援主網 token）：`https://merchant-stg.hashkeymerchant.com`

生產環境（僅支援主網 token）：`https://merchant.hashkey.com`


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
          "total": {"label": "總計", "amount": {"currency": "USD", "value": "15.00"}}
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
| `method_data[].data.coin` | string | 是 | 代幣符號（如 `USDC`） |
| `details.id` | string | 是 | 支付請求 ID（`payment_request_id`，ID2） |
| `details.display_items` | array | 否 | 商品明細清單 |
| `details.total` | object | 是 | 總金額（`label` + `amount`） |
| `contents.cart_expiry` | string | 是 | 授權過期時間（RFC 3339 格式），建議 2 小時 |
| `contents.merchant_name` | string | 是 | 商戶名稱 |
| `merchant_authorization` | string | 是 | 商戶 JWT 簽章（ES256K），詳見 [認證與簽章](authentication.md) |
| `redirect_url` | string | 否 | 支付完成後跳轉 URL |

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

### 回應範例（按 cart_mandate_id）

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
| `payment_request_id` | string | 支付請求 ID（ID2） |
| `request_id` | string | 支付記錄唯一識別（ID5） |
| `token_address` | string | 代幣合約地址 |
| `flow_id` | string | 支付流程 ID（ID3） |
| `app_key` | string | 應用程式識別 |
| `amount` | string | 支付金額（最小單位，如 USDC 為 6 位精度，`"15000000"` = 15 USDC） |
| `usd_amount` | string | 支付對應的 USD 金額 |
| `token` | string | 代幣符號 |
| `chain` | string | 鏈識別（CAIP-2 格式，如 `eip155:11155111`） |
| `network` | string | 區塊鏈網絡名稱 |
| `extra_protocol` | string | x402 協議類型（`eip3009` / `permit2`） |
| `status` | string | 支付狀態（見[支付狀態機](cart-mandate.md#支付狀態機)） |
| `status_reason` | string? | 狀態原因（失敗時返回） |
| `payer_address` | string | 付款人錢包地址 |
| `to_pay_address` | string | 收款地址 |
| `risk_level` | string | AML 風險等級 |
| `tx_signature` | string | 鏈上交易雜湊（交易打包後返回） |
| `broadcast_at` | string? | 首次廣播時間（RFC 3339） |
| `gas_limit` | int | Gas 限制 |
| `gas_fee` | string | Gas 費用 |
| `gas_fee_amount` | string | Gas 費用金額 |
| `service_fee_rate` | string | 服務費率 |
| `service_fee_type` | string | 手續費類型（`free` / `price_include` / `price_extra`） |
| `deadline_time` | string | 支付截止時間（RFC 3339） |
| `created_at` | string | 建立時間（RFC 3339） |
| `updated_at` | string | 更新時間（RFC 3339） |
| `completed_at` | string? | 完成時間（RFC 3339，`payment-finalized` 時返回） |

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
