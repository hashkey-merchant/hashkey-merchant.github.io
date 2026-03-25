# Webhook 支付結果通知

支付進入特定狀態（`payment-successful`／`payment-included`／`payment-failed`）後，網關會主動向商戶設定的 `webhook_url` 發送 HTTP POST 回呼，商戶毋須輪詢即可即時得知支付結果。

---

## 設定方式

`webhook_url` 綁定於應用程式憑證（AppCredential）層級，透過商戶後台設定：

1. 登入商戶後台，進入「應用程式管理」
2. 選擇對應應用程式，點擊「編輯設定」
3. 填寫「支付結果回呼地址」（`webhook_url`）並儲存，必須使用 HTTPS

**要求：**

- 服務須於 **10 秒**內返回 HTTP 2xx，否則視為失敗並觸發重試
- 若未設定 `webhook_url`，支付完成後不會發送任何通知，商戶須主動輪詢查詢支付狀態

---

## Webhook 簽章驗證（HMAC-SHA256）

為防止偽造請求與重放攻擊，網關於每次回呼時均會在請求標頭附上簽章。

### 簽章格式

```
X-Signature: t=<unix_timestamp>,v1=<hmac_hex>
```

| 部分 | 說明 |
|------|------|
| `t` | 簽章時刻的 Unix 時間戳（秒） |
| `v1` | HMAC-SHA256 簽章的十六進制字串 |

### 簽章計算方式

```
message   = timestamp + "." + raw_request_body
signature = hex(HMAC-SHA256(app_secret, message))
```

簽章金鑰即為建立應用程式時取得的 **`app_secret`**，毋須另外申請。

### 商戶驗證步驟

1. 從請求標頭解析 `X-Signature`，取得 `t`（時間戳）與 `v1`（簽章）
2. 驗證時間戳與目前時間差不超過 **5 分鐘**（防重放攻擊）
3. 使用 `app_secret` 與 `t`、原始請求本文重新計算簽章
4. 使用**常數時間比較**（`hmac.Equal`）比對兩個簽章，一致則接受請求

### Go 驗證範例

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

    // 防重放：時間戳偏差不超過 5 分鐘
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

## 回呼請求格式

網關向 `webhook_url` 發送 HTTP POST 請求：

**請求標頭：**

| 請求標頭 | 說明 | 範例 |
|--------|------|------|
| `Content-Type` | 固定為 `application/json` | `application/json` |
| `X-Signature` | HMAC-SHA256 簽章 | `t=1709123456,v1=a1b2c3d4...` |

---

## 回呼參數

### 公共欄位

| 欄位 | 類型 | 說明 | 範例 |
|------|------|------|------|
| `event_type` | string | 事件類型，固定為 `payment` | `payment` |
| `payment_request_id` | string | 支付請求 ID（ID2） | `PAY-REQ-20240301-001` |
| `request_id` | string | 支付唯一識別（ID5） | `req_20240301_abc123` |
| `cart_mandate_id` | string | 訂單 ID（ID1） | `ORDER-20240301-001` |
| `payer_address` | string | 付款人錢包地址 | `0x1234...abcd` |
| `amount` | string | 支付金額（最小單位） | `"15000000"` |
| `token` | string | 代幣符號 | `USDC` |
| `token_address` | string | 代幣合約地址 | `0x1c7D...` |
| `chain` | string | 鏈識別（CAIP-2 格式） | `eip155:11155111` |
| `network` | string | 所屬網絡 | `sepolia` |
| `status` | string | 支付狀態 | `payment-successful`／`payment-failed`/`payment-included` |
| `created_at` | string | 建立時間（RFC 3339） | `2024-03-01T10:00:00Z` |

### 成功附加欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `tx_signature` | string | 鏈上交易雜湊 |
| `completed_at` | string | 支付完成時間（RFC 3339） |

### 失敗附加欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `status_reason` | string | 失敗原因描述 |

---

## 回呼範例

### 支付成功

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

### 支付失敗

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

## 商戶回應規範

商戶伺服器**必須**於 10 秒內返回 HTTP 2xx（200–299）狀態碼：

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"code": 0}
```

> 回應本文內容不作要求，網關僅檢查 HTTP 狀態碼。

**注意事項：**

- 收到回呼後應先校驗業務資料（`amount`、`token`、`cart_mandate_id` 是否與本地記錄一致），再進行後續處理
- 回呼處理邏輯應保持**冪等**：同一 `request_id` 可能因重試多次送達，請勿重複出貨／扣款

---

## 重試機制

若商戶服務未於逾時時間內返回 2xx，或網絡連線失敗，網關會按以下間隔自動重試，最多 **6 次**：

| 第 N 次失敗 | 下次重試間隔 |
|------------|------------|
| 第 1 次 | 1 分鐘後 |
| 第 2 次 | 5 分鐘後 |
| 第 3 次 | 15 分鐘後 |
| 第 4 次 | 1 小時後 |
| 第 5 次 | 6 小時後 |
| 第 6 次 | 24 小時後 |

超過 6 次重試仍失敗後，通知狀態標記為 `FAILED`，**不再重試**。此時商戶須透過主動查詢 API 取得最終支付狀態。

> [!NOTE]
> 每個 `request_id` 保證最多投遞一次，網關內部有冪等保護，不會重複建立通知記錄。
