# 認證與簽章

Hashkey Merchant API 採用兩層認證機制：

1. **HMAC-SHA256 請求簽章** — 所有 Merchant API 請求必須附上，驗證請求來源與完整性
2. **ES256K JWT 簽章** — 建立支付訂單時的 `merchant_authorization` 欄位，驗證 Cart Mandate 內容的真實性

---

## HMAC-SHA256 請求簽章

### 必要的請求標頭

所有 Merchant API（`/api/v1/merchant/*`）每個請求必須附上以下 4 個 Header：

| 請求標頭 | 說明 | 範例 |
|--------|------|------|
| `X-App-Key` | 應用程式識別 | `ak_xxxxxxxx` |
| `X-Signature` | HMAC-SHA256 簽章（十六進制） | `a1b2c3d4e5f6...` |
| `X-Timestamp` | Unix 時間戳（秒） | `1709123456` |
| `X-Nonce` | 防重放隨機數（建議 UUID 或 12–32 位隨機 hex） | `a3f1b2c4d5e6` |

### 簽章演算法

**第 1 步：計算請求本文雜湊**

- 有請求本文時：先將 JSON 本文按 **Canonical JSON** 規則序列化為字串，再對該字串（UTF-8 位元組）計算 **SHA-256**，將摘要編成**小寫十六進制**字串，即為 `bodyHash`（亦即 `bodyHash = hex(SHA256(canonicalJSON(requestBody)))`）。
- 無請求本文（一般為 GET）時：`bodyHash` 為空字串 `""`（簽章訊息中仍佔一行，見下方範例）。

**第 2 步：拼接簽章訊息**

使用換行符號 `\n` 分隔 6 個部分：

```
message = "{METHOD}\n{PATH}\n{QUERY}\n{bodyHash}\n{timestamp}\n{nonce}"
```

| 部分 | 說明 | 範例 |
|------|------|------|
| `METHOD` | HTTP 方法（大寫） | `POST`、`GET` |
| `PATH` | 完整請求路徑 | `/api/v1/merchant/orders` |
| `QUERY` | 不含 `?` 的查詢字串，無 query 時為空字串 | `cart_mandate_id=ORDER-123` |
| `bodyHash` | 請求本文 SHA256 雜湊，無 body 時為空字串 | `abc123def456...` |
| `timestamp` | Unix 時間戳（秒） | `1709123456` |
| `nonce` | 隨機數 | `a3f1b2c4d5e6` |

**第 3 步：HMAC-SHA256 簽章**

```
signature = hex(HMAC-SHA256(app_secret, message))
```

### 簽章訊息範例

**POST 請求（有 body）：**

```
POST\n/api/v1/merchant/orders\n\nabc123def456...\n1709123456\na3f1b2c4d5e6
```

**GET 請求（無 body）：**

```
GET\n/api/v1/merchant/payments\ncart_mandate_id=ORDER-123\n\n1709123456\na3f1b2c4d5e6
```

### 安全限制

- **時間戳有效期**：±300 秒（5 分鐘），超出範圍將被拒絕
- **Nonce 唯一性**：同一 `app_key` 下，nonce 在 5 分鐘內不可重複使用（防重放攻擊）

### Bash 完整範例

```bash
#!/bin/bash
APP_KEY="your_app_key"
APP_SECRET="your_app_secret"
TS=$(date +%s)
NONCE=$(openssl rand -hex 16)
PATH_ONLY="/api/v1/merchant/orders"
BODY='{"cart_mandate":{...}}'

# 1. 計算 body SHA256
BODY_HASH=$(printf "%s" "$BODY" | openssl dgst -sha256 -hex | awk '{print $NF}')

# 2. 拼接簽章訊息
SIGN_STR="POST\n${PATH_ONLY}\n\n${BODY_HASH}\n${TS}\n${NONCE}"

# 3. 計算 HMAC-SHA256
SIG=$(printf "%b" "$SIGN_STR" | openssl dgst -sha256 -hmac "$APP_SECRET" -hex | awk '{print $NF}')

# 4. 發送請求
curl "https://merchant-qa.hashkeymerchant.com${PATH_ONLY}" \
  -X POST \
  -H "Content-Type: application/json" \
  -H "X-App-Key: ${APP_KEY}" \
  -H "X-Timestamp: ${TS}" \
  -H "X-Nonce: ${NONCE}" \
  -H "X-Signature: ${SIG}" \
  -d "$BODY"
```

### GET 請求簽章範例

```bash
TS=$(date +%s)
NONCE=$(openssl rand -hex 16)
PATH_ONLY="/api/v1/merchant/payments"
QUERY="cart_mandate_id=ORDER-20240301-001"

# GET 無 body，bodyHash 為空
SIGN_STR="GET\n${PATH_ONLY}\n${QUERY}\n\n${TS}\n${NONCE}"
SIG=$(printf "%b" "$SIGN_STR" | openssl dgst -sha256 -hmac "$APP_SECRET" -hex | awk '{print $NF}')

curl "https://merchant-qa.hashkeymerchant.com${PATH_ONLY}?${QUERY}" \
  -H "X-App-Key: ${APP_KEY}" \
  -H "X-Timestamp: ${TS}" \
  -H "X-Nonce: ${NONCE}" \
  -H "X-Signature: ${SIG}"
```

---

## ES256K JWT 簽章（merchant_authorization）

建立支付訂單時，`cart_mandate.merchant_authorization` 欄位需要商戶使用私鑰簽署 JWT，以證明 Cart Mandate 內容的完整性與真實性。

### 演算法規格

| 項目 | 規格 |
|------|------|
| **演算法** | ES256K（ECDSA + secp256k1 + SHA-256） |
| **橢圓曲線** | secp256k1（與比特幣、以太坊相同） |
| **JWT Header** | `{"alg":"ES256K","typ":"JWT"}` |
| **私鑰格式** | PKCS8（`BEGIN PRIVATE KEY`）或 SEC1（`BEGIN EC PRIVATE KEY`） |

> [!IMPORTANT]
> 非 ES256K 演算法的 JWT 會被拒絕。

### JWT Claims

| 宣告 | 類型 | 說明 |
|------|------|------|
| `iss` | string | Issuer — 商戶名稱 |
| `sub` | string | Subject — 商戶名稱 |
| `aud` | string | Audience — 固定為 `"HashkeyMerchant"` |
| `iat` | int64 | Issued At — JWT 簽發時間戳 |
| `exp` | int64 | Expiration — JWT 過期時間（建議簽發後 1 小時） |
| `jti` | string | JWT ID — 唯一識別符，格式 `JWT-{timestamp}-{random}` |
| `cart_hash` | string | Cart Contents 的 SHA-256 雜湊值（64 位十六進制） |

### 簽章流程

1. **組裝 Cart Contents** — 組裝支付資訊的 `contents` 物件
2. **計算 Cart Hash** — 對 `contents` 進行 Canonical JSON 序列化後計算 SHA-256 雜湊（詳見 [Cart Mandate 組裝](cart-mandate.md)）
3. **簽署 JWT** — 使用商戶私鑰（ES256K）簽署包含上述 Claims 的 JWT
4. **填入欄位** — 將 JWT 字串填入 `cart_mandate.merchant_authorization`

### 金鑰產生

```bash
# 產生 EC 私鑰（secp256k1）
openssl ecparam -name secp256k1 -genkey -noout -out merchant_private_key.pem

# 匯出對應公鑰（提交予營運）
openssl ec -in merchant_private_key.pem -pubout -out merchant_public_key.pem
```
