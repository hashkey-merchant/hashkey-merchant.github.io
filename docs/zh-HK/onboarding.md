# 商戶對接流程

## 對接步驟

1. **產生商戶金鑰對** — 產生 secp256k1（ES256K）金鑰對
2. **提交註冊資料** — 將機構資訊、公鑰與支援的鏈／幣種提交予營運團隊
3. **完成電郵驗證** — 收到邀請連結後完成註冊與機構綁定
4. **建立應用程式並取得憑證** — 登入商戶後台建立應用程式，取得 `app_key` 與 `app_secret`

---

## 1. 產生商戶金鑰對

商戶需要產生一對 **secp256k1（ES256K）** 金鑰，用於簽署 `merchant_authorization` JWT。

```bash
# 產生 EC 私鑰（secp256k1 曲線）
openssl ecparam -name secp256k1 -genkey -noout -out merchant_private_key.pem

# 匯出對應公鑰
openssl ec -in merchant_private_key.pem -pubout -out merchant_public_key.pem
```

> [!IMPORTANT]
> 私鑰請妥善存放於伺服器端，切勿暴露予前端或上載至程式碼庫。

---

## 2. 提交註冊資料

將以下資訊提供予營運人員，由營運透過後台建立機構邀請：

| 欄位 | 必填 | 說明 | 範例 |
|------|------|------|------|
| `organization_name` | 是 | 機構名稱 | `Acme Pay` |
| `email` | 是 | 管理員電郵 | `admin@acme.com` |
| `description` | 否 | 機構描述 | `Acme 支付業務團隊` |
| `default_language` | 是 | 預設語言 | `zh-HK` / `en` |
| `public_key` | 是 | 商戶公鑰（PEM 格式） | `-----BEGIN PUBLIC KEY-----...` |
| `supported_chain_tokens` | 是 | 支援的鏈／幣種清單（至少 1 項） | 見下方 |

**主網（Mainnet）**

| 網絡 | Chain ID | 代幣 | 合約地址 | 精度 | 穩定幣 | 支付協議 |
|------|----------|------|----------|------|--------|----------|
| ethereum | 1 | USDC | `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` | 6 | 是 | EIP-3009 |
| ethereum | 1 | USDT | `0xdac17f958d2ee523a2206206994597c13d831ec7` | 6 | 是 | Permit2 |
| ethereum | 1 | HSK | `0xe7c6bf469e97eeb0bfb74c8dbff5bd47d4c1c98a` | 18 | 否 | Permit2 |
| hashkey | 177 | USDC | `0x054ed45810DbBAb8B27668922D110669c9D88D0a` | 6 | 是 | EIP-3009 |
| hashkey | 177 | USDT | `0xF1B50eD67A9e2CC94Ad3c477779E2d4cBfFf9029` | 6 | 是 | Permit2 |

**測試網（Testnet）**

| 網絡 | Chain ID | 代幣 | 合約地址 | 精度 | 穩定幣 | 支付協議 |
|------|----------|------|----------|------|--------|----------|
| sepolia | 11155111 | USDC | `0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238` | 6 | 是 | EIP-3009 |
| sepolia | 11155111 | USDT | `0xff5588b3b38dff1b4b49bfdcbf985e84d8751a0e` | 6 | 是 | Permit2 |
| sepolia | 11155111 | HSK | `0x31bdac8e4b897e470b70ebe286f94245baa793c2` | 18 | 否 | Permit2 |
| hashkey-testnet | 133 | USDC | `0x79AEc4EeA31D50792F61D1Ca0733C18c89524C9e` | 6 | 是 | EIP-3009 |
| hashkey-testnet | 133 | USDT | `0x372325443233fEbaC1F6998aC750276468c83CC6` | 6 | 是 | Permit2 |


營運建立邀請後，管理員電郵會收到邀請連結，點擊完成註冊與機構綁定。

---

## 3. 取得應用程式憑證

登入商戶後台，建立應用程式後系統返回以下憑證：

| 欄位 | 說明 | 用途 |
|------|------|------|
| `app_key` | 應用程式識別 | 放入請求標頭 `X-App-Key` |
| `app_secret` | 應用程式密鑰（**嚴禁外洩**） | 用於 HMAC-SHA256 簽章計算 |

取得憑證後即可開始呼叫 Merchant API，詳見 [認證與簽章](authentication.md)。
