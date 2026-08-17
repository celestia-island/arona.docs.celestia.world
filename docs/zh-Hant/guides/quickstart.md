---
title: "快速入門"
description: "使用內建 mock upstream 的完整端到端 Arona 流程：migrate、serve、註冊 backend、建立 API key 並開始聊天。"
---

# 快速入門

本指南帶你在單一機器上使用**內建 mock upstream** 完成一次完整的端到端
Arona 設定——不需要真實模型權重、GPU 或外部 API 帳號。完成後你將擁有：

- 一個運行中的 Arona gateway（`/v1/*` OpenAI 相容 REST API 加上
  `/api/rpc` JSON-RPC 管理平面），
- 以 `external` backend 註冊的 mock upstream，
- 一個使用者帳號與一支 API key，
- 對 mock 的一次可用非串流**與**串流聊天回合，
- 可透過 `usage.list` 查看的 usage records。

## 前置需求

- **Rust 工具鏈**（參閱儲存庫根目錄的 `rust-toolchain.toml`）。
- **Python 3** 並安裝 `aiohttp`——僅 mock upstream 需要
  （`pip install aiohttp`）。
- 一個**運行中的 PostgreSQL** 執行個體及其連線 URL。

## 1. 設定環境

Arona 在**程序啟動時**從環境變數讀取設定。其中有兩個是必填的：`DATABASE_URL`
與 `JWT_SECRET`——沒有它們伺服器會拒絕啟動（除非設定 `MOCK_MODE=1`）。
強烈建議設定 `ARONA_ADMIN_TOKEN`：沒有它，每個 `/api/admin/*` 路由都會回傳 401。

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

這些變數只在程序啟動時讀取一次——若你變更它們，請重新啟動伺服器。
完整的變數參考請見[設定](configuration.md)。

## 2. 執行 migration 並啟動伺服器

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

對於全新的資料庫，單獨執行 `serve` 就夠了：它會在啟動時自動 migration。
伺服器預設綁定 `0.0.0.0:8420`（可用 `ARONA_HOST`／`ARONA_PORT` 覆寫）。

## 3. 啟動 mock upstream

在第二個終端機中：

```bash
python3 scripts/mock/server.py
```

mock 是一個 aiohttp 伺服器，預設監聽 `127.0.0.1:8429`
（`ARONA_MOCK_PORT` 可覆寫連接埠）。它會在啟動時印出 API key，也提供
`GET /api/test-key`，回傳 `{"api_key": ..., "base_url": ...}`。它暴露
少數幾個模型 id——包括下方會用到的 `gpt-5.5`——並同時回應一般與串流
聊天補全。

擷取印出的 key：

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. 將 mock 註冊為 external backend

Backends 透過管理 HTTP API 註冊：

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

backend 在註冊當下立即被探測，約 1-2 秒內轉為健康；在探測完成前，它會維持在
fail-closed 的「尚未探測」（not probed yet）狀態（見下方疑難排解框）。設定會
持久化，因此 backend 能存活於重啟。

## 5. 註冊帳號並登入

帳號位於 JSON-RPC 平面，`POST /api/rpc`。因為已設定 `ARONA_REGISTRATION_OPEN=1`，
`auth.register` 為開放狀態；第一個註冊的使用者會成為管理員。

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

密碼必須至少 8 個字元**且**包含 4 個字元類別（大寫、小寫、數字、特殊字元）中
至少 3 類。接著登入以取得 JWT 配對：

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

匯出回應中的 `access_token`：

```bash
export JWT="<access_token from the login response>"
```

## 6. 建立 API key

`keys.create` 需要 JWT 認證，且只會回傳一次**完整**的 `arona-{uuid}`
密鑰——資料庫只儲存其 SHA-256 雜湊，所以現在就複製它：

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. 聊天（非串流）

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

你會得到一個 OpenAI 風格的補全物件，帶有 `choices[0].message`
與 `usage` 區塊。

## 8. 聊天（串流）

同一個 endpoint 加上 `"stream": true` 會以 server-sent events 回應：
每個 token 一個 `data:` chunk，最後以 `data: [DONE]` chunk 收尾：

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. 驗證用量

每次聊天回合都會在該 key 的前綴下記錄一筆 usage 列。使用 JWT 查詢：

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

你應該會看到上述 `gpt-5.5` 請求的一筆或多筆記錄。

## 疑難排解

- **`No backend available for model: gpt-5.5`（HTTP 404，`code:
  model_not_found`）** — 沒有已註冊的 backend 提供該模型 id。可能是 backend
  從未註冊（或其 `models` 清單不含該 id），或註冊呼叫失敗。用
  `GET /api/admin/backends`（admin token）檢查。
- **`all backends unhealthy`（HTTP 500，`backend_error`）** — 該模型*有*
  註冊的 backend，但沒有任何候選者健康。新註冊的 external backend 從
  fail-closed 的「尚未探測」狀態開始，約 1-2 秒後註冊時的探測完成即轉為
  健康；若你在那個時間窗內聊天就會碰到這個錯誤。稍等片刻再試，或確認
  mock 確實運行在 `127.0.0.1:8429`。
- **`/v1/*` 回傳 HTTP 401** — 缺少 `Authorization` header 會得到
  `Missing Authorization header. Use: Bearer <api_key>`；未知的 key 會得到
  `Invalid API key`。請再次確認 `$AR_KEY`（完整密鑰，不是前綴）。
- **`/api/admin/*` 回傳 HTTP 401 `Admin access required`** — bearer token
  與 `ARONA_ADMIN_TOKEN` 不符，或該變數未設定（此時該路由一律拒絕）。
  設定後請重新啟動伺服器。
- **`auth.register` 失敗並回報「Registration is closed」** — 當
  `ARONA_REGISTRATION_OPEN` 不是 truthy 時，註冊會被停用。請在**啟動**
  伺服器**之前**設定 `ARONA_REGISTRATION_OPEN=1`（它在啟動時讀取），
  或成為第一個使用者——第一個註冊的使用者永遠允許註冊並成為管理員。
- **HTTP 429 速率限制** — 有三個獨立的限制可能觸發：
  - 每個 key 的記憶體內限制，預設 60 RPM
    （`ARONA_API_RATE_LIMIT_RPM`）→ `Rate limit exceeded. Try again later.`；
  - free billing tier 每個 key 10 RPM 的限制 → 429 並帶
    `Retry-After: 60` header；
  - free tier 每月 $1／100k token 的 quota → 429 並帶 `Retry-After`
    指向下一個計費週期。

## 後續步驟

- [設定](configuration.md) — 所有環境變數。
- [Backends](backends.md) — backend 類型、URL 語意與探測。
- [部署](deployment.md) — bare metal、systemd、Docker。
- [OpenAI 相容 REST API](../api/openai-rest.md) — 完整的 `/v1/*` 介面。
- [JSON-RPC API](../api/jsonrpc.md) — 上面用到的管理平面。

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
