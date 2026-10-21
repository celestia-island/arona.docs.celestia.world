---
title: "認證與安全性"
description: "JWT session、API keys、三道管理閘門、密碼政策、雙軌速率限制與安全模型。"
---

# 認證與安全性

Arona 在兩條軌道認證呼叫者：互動式客戶端（聊天 + 管理介面、RPC 呼叫）使用
**JWT session token**，程式化的 OpenAI 相容流量使用 **API keys**
（`arona-…`）。另有一個獨立的管理 token 守護管理面。此頁面記錄其機制、
安全模型，以及安全稽核中已知的低風險遺留事項。

## 公網模式與轉接登入

Arona 提供兩種認證姿態，由 [TOML 設定](./configuration.md)中的 `auth.mode` 選擇：

- **`local`**（預設）—— 傳統行為。`auth.login` / `auth.register` 在
  JSON-RPC 面上接受郵箱+密碼。適合可信網路或嚴格反向代理之後。
- **`public`** —— 面向公網的姿態。密碼通道整體關閉（回傳指向轉接入口的
  錯誤）；互動式身份**只能**經下方轉接通道進入，程式化呼叫繼續用 API key，
  行為不變。`public` 模式下未設定任何轉接入口時伺服器拒絕啟動——兩個入口
  都忘了配的操作者等於把所有人鎖在門外。

存在兩扇門，而 arona 永遠不做自己的身份提供方：

### OAuth2 轉接（`[auth.oauth]`）

面向**外部面板**的標準 authorization-code + PKCE（S256）客戶端（提供方是
設定出來的，不是自託管的）：

1. `GET /auth/oauth/begin` —— arona 產生一次性 `state` 與 PKCE verifier
   （保存在伺服器端、有 TTL），重定向到面板的 authorize 端點。
2. 面板完成使用者認證後重定向回 `GET /auth/oauth/callback?code=…&state=…`。
3. arona 在 token 端點換取權杖、讀取 userinfo 端點、把面板身份對映到本地
   帳戶。回應與 `auth.login` 同構（`access_token`、`refresh_token`、
   `user`），客戶端無需新的權杖處理邏輯。

身份宣告取 `email`（OIDC 標準），其次取形似郵箱的 `preferred_username`；
裸的不透明 `sub` 一律不接受。未知身份在 `allow_signup`（預設 `true`）允許
時自動開號；**空實例上的第一次轉接登入直接引導為 administrators 群組成員**
——與 local 模式的首次註冊完全一致。

### 可信簽發者轉接（`[[auth.trusted_issuers]]`）

API 側的等價物（evernight 閘道模式）：其 **kirino ACCESS 權杖**可以被
arona 在所有接受 JWT 的持有者面（`/v1/*` 與 JSON-RPC 面）直接接受的面板。
每行宣告簽發者（`iss`，強制校驗）、共享驗證密鑰與可選 `audience`；校驗按
簽發者 fail-closed——來自其他簽發者（或無簽發者）的權杖一律拒絕，refresh
權杖永遠不能當會話持有者憑證。面板使用者的 `username` 宣告必須是郵箱
（arona 的使用者面以郵箱為主鍵）；它對映到**本地**帳戶，未知身份按該行的
`allow_signup` 決定是否開號。

### 發現與姿態上報

未認證的 `auth.methods` JSON-RPC 方法上報存在哪些門（`password`、
`oauth {begin_url, provider}`、`api_keys`、`trusted_issuers` —— 僅布林與
名字），`system.status` 攜帶同樣事實的增量 `auth` 區塊供運維查看。

## JWT session

Session 使用 `kirino_session` token 管理員簽發的 JWT access/refresh token 配對：

- **Access token TTL：900 秒（15 分鐘）。**
- **Refresh token TTL：604,800 秒（7 天）。**

Access tokens 認證 JSON-RPC 平面（`/api/rpc`）與 `GET /v1/models`；
SSE sidecar（`/api/rpc/events`）以 session id 為鍵，這是在已認證的 RPC
呼叫期間鑄造的能力，而非 bearer 憑證。`/v1/chat/completions`、
`/v1/embeddings` 與 `/v1/video/*` endpoint 需要 **API key**
（該處不接受 JWT）。Access tokens 生命週期短，因此被竊取的 token 只能短暫
使用。Refresh tokens 透過 `auth.refresh` 換成新的配對。

Refresh 使用 **token-family 輪換**：使用 refresh token 會使其失效並簽發
新的；重複使用已消費的 refresh token 會撤銷整個 family——
`auth.refresh` 以 `AUTH_ERROR` 回應，訊息為 `Refresh token reused`
（底層錯誤是 `TokenReused`，「refresh token has been reused — token family
revoked」），帳號必須重新登入。Family 撤銷是**記憶體內**的（一個
`revoked_families` 集合）：伺服器重啟會清除它，因此此保護在跨重啟時是
盡力而為（per-user session 狀態不會存活於重啟）。

簽署密鑰來自 `JWT_SECRET` 環境變數。在 `MOCK_MODE=1` 之外，若 `JWT_SECRET`
未設定或仍等於內建開發密鑰，伺服器**拒絕啟動**，因此正式環境執行個體
永遠不會意外以公開常數簽署 token。請使用強大、隨機的密鑰，且絕不提交它。

## API keys

API keys 是 OpenAI 相容介面的機器憑證：

- **格式：** `arona-{uuid}`。
- **儲存：** `api_keys` 資料表只儲存 key 的 **SHA-256 雜湊**——明文只回傳
  一次（在 `keys.create` 回應中），之後永遠無法找回。
- **Key 前綴：** 前 8 個字元（`key_prefix`）以明文儲存，供顯示與用量歸因；
  介面顯示遮罩形式，例如 `arona-XXXX…abcd`。
- **撤銷：** key 查詢會 join `api_keys.is_active = TRUE`，因此被撤銷的 key
  會立即停止驗證——沒有需要等待的 cache TTL。

## 管理權限分級

有三道不同的管理閘門，各有自己的憑證：

1. **`/api/admin/*` 路由** — backend 與 alias 管理
   （`POST/GET/DELETE /api/admin/backends`、`POST/GET/DELETE /api/admin/aliases`）
   需要 `Authorization: Bearer ARONA_ADMIN_TOKEN` header。當
   `ARONA_ADMIN_TOKEN` 未設定時，`check_admin` 一律失敗，每個管理路由
   都回傳 **401 "Admin access required"**——整個管理面是停用而非開放。

2. **`agents.*` 與 `engine.invoke` RPC 方法** — agent 叢集與
   engine 控制平面需要帳號具備 `users.is_admin =
   true` 的 JWT。已認證的非管理員會被拒絕，回傳實作定義的 code **-32007
   （`ADMIN_REQUIRED`）** 加上方法特定的提示
   （例如 `agents.deploy starts model deployments on GPU nodes`）；
   **未認證**的呼叫者會得到標準的 **-32005（`AUTH_ERROR`）**，
   因此伺服器不會透露該方法是特權方法。

3. **`billing.plan.set` 與 `billing.video.pricing.set` RPC 方法** —
   計費變更需要與管理 HTTP 路由相同的 Bearer `ARONA_ADMIN_TOKEN`；
   沒有它會回傳 `AUTH_ERROR`「Admin access required」。


4. **管理面身分（`group.*`、`group.credits.*`、`ledger.self`、`keys.*`、`billing.plan` RPC 方法）** —— 這些方法接受使用者 JWT，此外也接受與 admin 路由相同的 Bearer `ARONA_ADMIN_TOKEN`：此時權杖充當**營運者服務身分**（合成實例管理員 claims），供控制台工具驅動多租戶面。權杖刻意不授權推理/聊天面（`chat.send`、`/v1/*`）——它是管理憑證，不是模型憑證。

**第一個註冊的使用者成為管理員**（`users.is_admin = true`）。
之後的每次註冊都是一般使用者，且只有當 `ARONA_REGISTRATION_OPEN`
設為 truthy 值時才開放註冊。

## 密碼政策

密碼必須同時滿足**兩條**規則（在註冊與任何改密碼路徑上強制執行）：

- 至少 **8 個字元**，且
- **4 個字元類別**中至少 **3 類**：大寫、小寫、數字、
  特殊字元。

## 速率限制

速率限制在兩條獨立軌道上運行；任何一條都可能以 **429** 拒絕請求：

### 1. 記憶體內滑動視窗（依身分）

每個已認證的 `/v1` 請求都會通過一個以呼叫者身分為鍵的記憶體內滑動視窗
限制器：

- **API-key 呼叫**以 key 的 **SHA-256 雜湊**為鍵；
- **JWT 呼叫**以 `u:<email>` 為鍵——JWT 每 15 分鐘輪換，因此若以 token
  實例為視窗的鍵，每次 refresh 都會悄悄重置它。

預設額度是**每分鐘 60 個請求**，可用 `ARONA_API_RATE_LIMIT_RPM` 覆寫
（對扇出大量平行 LLM 呼叫的 agent pipeline 調高）。設為 **0 會阻擋每個請求**。

### 2. Tier 速率限制（per key，來自資料庫）

Billing tiers 攜帶 per-key 的 `rate_limit_rpm`。檢查會計算該 key 前綴在
**最近 60 秒**內的 `usage_records` 列數（usage 在每個回應後持久化，因此
視窗最多落後一個進行中的請求；資料庫失敗時 fail open）。預設的 **free tier
是 10 RPM**；pro／enterprise tiers 提高上限。每月 quota 強制共用同一條
拒絕路徑。

### 登入速率限制

憑證猜測在登入 endpoint 被節流：**每個 email 每 5 分鐘 5 次失敗嘗試**
與**每個 IP 每 5 分鐘 20 次**，每次之後都有 15 分鐘鎖定期。

### `Retry-After`

每個 429 回應都攜帶 `Retry-After` header，讓 OpenAI 相容的
客戶端退避而非猛打 endpoint：quota 拒絕將其設為**距月底的秒數**；
速率限制拒絕將其設為 **60**。quota 模型見[計費與用量](billing-usage.md)。

## 安全模型備註

- **CORS 允許任何來源**（`allow_origin(Any)`）——Arona 是被許多第一方與
  第三方客戶端使用的 backend；若你的部署必須限制來源，請在前面加上
  強制 CORS 的反向 proxy。
- **請求內文限制為 1 MB**（`RequestBodyLimitLayer`），限制了 gateway 的
  記憶體使用。
- **gateway 不終止 TLS**——它監聽純 HTTP。請放在終止 HTTPS 的反向 proxy
  後方（見[部署](deployment.md)）。
- **密鑰只來自環境**：`ARONA_ADMIN_TOKEN` 與
  `JWT_SECRET` 從環境變數讀取，且必須是強大隨機值，絕不可提交到儲存庫。
- 預設伺服器綁定位址是 `0.0.0.0`；請在網路層限制暴露範圍。

## 已知低風險遺留事項（來自稽核）

以下按現狀記錄；它們是刻意如此或目前接受，但當你把執行個體暴露到受信任
網路之外時值得知道：

- **`providers.list` 是公開的**，而 `providers.add`／`providers.update`／
  `providers.remove`／`providers.test` 受 admin 閘控（JWT + `users.is_admin`）。
  公開讀取路徑會揭露
  provider 目錄，但沒有機密內容。
- **`/ws/agent` 使用共享的 operator token，而不是每個 agent 的獨立身分**：只有攜帶與 `ARONA_ADMIN_TOKEN` 相符的 `Authorization: Bearer <operator token>` 時，panel 才會接受控制平面升級；若 panel 未設定該 token，控制平面保持關閉。GPU agent 會送出 `ARONA_AGENT_TOKEN`（未設定時回退到 `ARONA_ADMIN_TOKEN`）。由於憑證在所有 agent 間共享，令牌一旦外洩仍可冒充 agent；每個 agent 的獨立註冊憑證屬於後續強化項。任何跨越網路邊界的部署都應把控制平面放在 TLS 之後。
- **`memory.delete` 僅需 JWT、無所有權檢查**：任何已認證的使用者都可以
  依 `node_id` 刪除記憶節點。刪除記憶需要登入，但不需要擁有該節點。

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
