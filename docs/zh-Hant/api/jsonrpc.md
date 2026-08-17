---
title: "JSON-RPC API 參考"
description: "位於 /api/rpc 的 Arona 管理平面 JSON-RPC 2.0 API——經由 HTTP 與 WebSocket 的 chat、realtime、engine、auth、keys、providers、agents、memory、conversations、usage、billing、video 與 system 方法。"
---

# JSON-RPC API 參考

Arona 在 `/api/rpc` 暴露 JSON-RPC 2.0 介面作為管理平面：
auth、keys、providers、agents、memory、conversations、usage、billing、
video、realtime 與串流聊天。它補足 OpenAI 相容的 REST 介面
（`/v1/*`，見[OpenAI 相容 REST API](./openai-rest.md)）；key 認證的
推理工作負載用 REST，session／帳號管理與串流控制用 JSON-RPC。
[快速入門](../guides/quickstart.md) 走過第一個端到端回合。

此介面分派 **39 個請求方法**加一個匿名的 WebSocket-only liveness
方法 `system.probe`（共 40 個方法）。每個請求都是帶 `jsonrpc: "2.0"`、
`method` 字串、選用 `params` 物件與選用 `id` 的 JSON-RPC 2.0 物件。

## 傳輸

- **HTTP POST `/api/rpc`** — 請求／回應。送出 `Content-Type:
  application/json`。JWT 放在 `Authorization: Bearer <jwt>` header。
  請求內文上限 1 MiB。
- **WebSocket `GET /api/rpc`** — 長效連線。瀏覽器無法在 WebSocket
  upgrade 上設定自訂 headers，所以 JWT 以 `?token=<jwt>` 查詢參數
  傳送；伺服器在內部將其折入 `Authorization: Bearer` header
  （見 `packages/core/src/gateway/server.rs`）。已認證的 socket 可以
  無限期保持連線。
- **批次請求** — 內文是 JSON 陣列的 POST 會逐元素執行，並以相同順序的
  JSON 回應陣列回答。
- **匿名存取** — 沒有 JWT 的 WebSocket 上，公開方法
  （`auth.register`／`auth.login`／`auth.refresh`、`providers.list`、
  `system.status`）仍可呼叫，`system.probe` 在 socket 關閉前以單一
  ack 回答。其他每個方法都需要有效的 JWT；admin 閘控的方法額外需要
  admin 帳號（見下方圖例）。匿名 socket 也受 10 秒閒置逾時約束。
- **Session 附掛** — `POST /api/rpc` 上的 `x-session-id` header
  額外將 RPC 回應本身推上該 session 通道，與串流通知並列。

## Ids

請求 `id` 值以類型保真度回聲：`null` → `null`、字串 → 字串、整數 →
數字，其他任何東西（floats、物件、超出 i64 範圍的整數）→ JSON 字串
呈現。省略的 `id` 以 `null` 回答。

## 伺服器 → 客戶端通知（SSE sidecar）

Tokens、部署進度與 realtime 事件**不會**在 WebSocket socket 上送達。
每個串流 RPC 建立一個 session id，並將通知以 server-sent events 推送到
`GET /api/rpc/events?session=<session_id>`。在 RPC 呼叫回傳 session id
**之前或緊接其後**訂閱 SSE endpoint——呼叫回傳與 SSE 訂閱建立之間發出的
通知會被丟棄（預訂閱視窗）。建議模式是先開啟 SSE 串流，再觸發 RPC。

通知方法：`chat.stream`（`chat.send` 的每 token 一則）、
`models.progress`（`agents.deploy` 的 agent 模型下載進度）、
`realtime.event`（開啟的 realtime session 的伺服器事件），與
`video.progress`／`video.done`／`video.failed`（非同步視訊任務）。
完整目錄見[事件與通知](./events.md)。

## 錯誤碼

| Code | 名稱 | 意義 |
| --- | --- | --- |
| `-32700` | Parse error | 請求內文不是有效的 JSON。 |
| `-32600` | Invalid request | 請求物件格式錯誤，例如缺少 `method`。 |
| `-32601` | Method not found | 未知的 `method` 字串；訊息會回聲它。 |
| `-32602` | Invalid params | `params` 對該方法反序列化失敗。 |
| `-32603` | Internal error | 未預期的伺服器失敗。 |
| `-32000` | `APP_ERROR` | 一般應用程式錯誤——例如找不到 conversation／provider／agent、沒有可部署的在線 agent。 |
| `-32005` | `AUTH_ERROR` | `"Authentication required"`——缺少或無效的 JWT。admin-token 方法在 bearer token 與 `ARONA_ADMIN_TOKEN` 不符時也用它（`"Admin access required"`）。 |
| `-32006` | `QUOTA_ERROR` | JWT 閘控的 RPC 方法（`chat.send`）每月計費 quota 超限。 |
| `-32007` | `ADMIN_REQUIRED` | 已認證的**非管理員**呼叫 admin 閘控方法（`agents.*`、`engine.invoke`）；訊息包含方法特定的提示。 |

> `agents.*` 與 `engine.invoke` 方法僅限管理員：它們需要帳號具備
> `users.is_admin = true` 的 JWT。已認證的非管理員以 `-32007`
> （`ADMIN_REQUIRED`）被拒絕；未認證的呼叫者得到標準的 `AUTH_ERROR`，
> 因此伺服器不會透露該方法是特權方法。

## Auth 圖例

| 圖例 | 憑證 |
| --- | --- |
| **public** | 不需要憑證。 |
| **JWT** | HTTP 上 `Authorization: Bearer <jwt>`，或 WebSocket 上 `?token=<jwt>`。 |
| **admin（JWT + is_admin）** | `users.is_admin = true` 帳號的 Bearer JWT。 |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN`（環境設定；未設定時方法一律被拒，default-deny）。 |

此頁面中的所有範例憑證與位址都是佔位符（RFC 5737 範例用 IP、`sk-xxx`
keys）。此圖例背後的完整 auth 模型見
[認證與安全性](../guides/auth-security.md)。

## Chat

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model`（string）、`messages`（`{ role, content, images?, tool_calls? }` 的陣列）、`temperature?`（number）、`max_tokens?`（integer）、`conversation_id?`（string）、`memory?`（bool）、`extra?`（object）、`tools?`（OpenAI 風格函式定義的陣列）、`provider?`（string） | 送出串流聊天回合。回傳 `{ "stream_id", "memory" }`——`memory` 是 recall 狀態（`enabled`／`disabled`／`offline`）；tokens 以 `chat.stream` 通知在 SSE sidecar 上送達。帶 `conversation_id` 時，伺服器端組裝完成的持久化歷史並持久化該回合。計費閘控（每月 quota → `-32006`）；usage 在 `jwt-<user-uuid>` 下記錄。 |

## Realtime（全雙工音訊／視訊 session）

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model`（string）、`config?`（session 設定物件）、`conversation_id?`（string） | 對提供 `model` 的 backend 開啟全雙工 session。回傳 `{ "session_id", "stream_session" }`：用 `session_id` 呼叫 `realtime.event`／`realtime.stop`，並在 SSE sidecar 訂閱 `stream_session` 以接收 `realtime.event` 通知。 |
| `realtime.event` | JWT | `session_id`（string）、`event`（客戶端事件——audio append/commit/clear、image frame、response create/cancel、session stop） | 將一個客戶端事件送入開啟的 session；它會被轉發到 upstream backend。回傳 `{ "ok": true }`。 |
| `realtime.stop` | JWT | `session_id`（string） | 關閉並移除 session。回傳 `{ "removed": bool }`。 |

## Engine（通用感知／控制通道）

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `engine.invoke` | admin（JWT + is_admin） | `model`（string）、`method`（string）、`params?`（object） | 在提供 `model` 的 backend 上同步請求／回應呼叫任意 engine 方法——`sensor.ingest`／`control.setpoint` 型呼叫的高頻通道（20–30 Hz 迴圈）。結果是 backend 的原始回應。 |

## Auth

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `auth.register` | public | `email`、`password`、`name?` | 註冊帳號。只在註冊開放時允許（`ARONA_REGISTRATION_OPEN`）；第一個註冊的使用者成為管理員。回傳與 `auth.login` 相同的 token 回應（`access_token`、`refresh_token`、`token_type`、`expires_in`、`user`）。 |
| `auth.login` | public | `email`、`password` | 登入。回傳 `access_token`、`refresh_token`、`token_type`、`expires_in`、`user`（`{ id, email, name, is_admin }`）。依 IP 與帳號速率限制。 |
| `auth.refresh` | public | `refresh_token` | 將 refresh token 換成新的 access token（與新的 refresh token）。被重用或過期的 refresh tokens 以 `AUTH_ERROR` 被拒絕。 |
| `auth.me` | JWT | — | 目前使用者設定檔：`{ "id", "email", "name" }`。 |

## Keys

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | 列出呼叫者的 API keys（id、name、`key_prefix`、project、時間戳、active 旗標）。 |
| `keys.create` | JWT | `name`、`project?` | 建立 API key。回傳 `{ id, name, key, key_prefix, project, created_at }`——`key` 中的完整 `arona-<uuid>` 密鑰只顯示**一次**；請立即儲存。 |
| `keys.revoke` | JWT | `key_id` | 撤銷 API key。回傳 `{ "ok": true }`。 |

## Providers

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | 列出已知 providers：內建官方條目加自訂條目，作為顯示 metadata（`id`、`name`、`description`、`website_domain`、`is_official`、`is_operator`）。刻意公開——清單不攜帶憑證；只有下面的變更操作是 JWT 閘控。 |
| `providers.add` | JWT | `id`、`name`、`description?`、`website_domain?` | 新增自訂 provider 條目。回傳 `{ "ok": true }`。 |
| `providers.update` | JWT | `provider_id`、`name?`、`description?`、`website_domain?` | 更新自訂 provider 的欄位（只更新提供的那些）。回傳 `{ "ok": true }`。 |
| `providers.remove` | JWT | `provider_id` | 移除自訂 provider。回傳 `{ "ok": true }`。 |
| `providers.test` | JWT | — | 測試 provider 連線。Stub：回傳 `{ "ok": true, "message": "Provider connection test not yet implemented" }`。 |

## Agents

所有 `agents.*` 方法僅限管理員（JWT + `is_admin`）。Agent 節點經由
`GET /ws/agent` 外送連線；此 RPC 群控制登錄檔（見
[Agent 叢集](../guides/agent-cluster.md)）。

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `agents.list` | admin（JWT + is_admin） | — | 列出已註冊的 agent 節點：id、name、host、`online`／`offline` 狀態（以 heartbeat 為基礎）、GPU 摘要、已部署模型、version、時間戳。 |
| `agents.register` | admin（JWT + is_admin） | `machine_name`、`version` | 向 tunnel manager 註冊 agent 節點。回傳 `{ "agent_id", "token" }`（token 是 agent 的控制平面憑證）。 |
| `agents.deregister` | admin（JWT + is_admin） | `agent_id` | 取消註冊（斷開）agent。回傳 `{ "ok": true }`。 |
| `agents.status` | admin（JWT + is_admin） | `agent_id` | 每個 agent 的狀態：online 旗標、host、GPU 摘要、已載入模型、GPU 使用率、heartbeat／連線時間戳。 |
| `agents.deploy` | admin（JWT + is_admin） | `model_id`、`agent_id?`（空／缺失 = 最少負載節點；沒有在線則報錯） | 在 agent 上部署模型。回傳 `{ "ok": true, "stream_id" }`——在 SSE sidecar 訂閱 `stream_id` 以接收 `models.progress` 下載通知。 |
| `agents.stop` | admin（JWT + is_admin） | `agent_id`、`model_id` | 停止已部署的模型。回傳 `{ "ok": true, "stream_id": null }`（無進度串流）。 |

## Memory

長期記憶由 entelecheia Philia 服務經由 WebSocket 提供；失敗絕不阻擋
聊天（見[記憶 Gateway](../guides/memory-gateway.md)）。

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | 記憶 gateway 狀態：`{ "enabled", "writeback", "events" }`——旗標加上最多 50 筆最近活動事件（最新在前）。 |
| `memory.delete` | JWT | `node_id` | 刪除已儲存的記憶節點。回傳 `{ "deleted": bool }`。 |

## Conversations

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | 列出呼叫者的對話，帶相對年齡時間戳。 |
| `conversations.create` | JWT | `title?`（預設 `New Conversation`） | 建立對話。回傳新的對話物件。 |
| `conversations.get` | JWT | `conversation_id`（legacy alias：`id`） | 帶訊息取得對話。所有權檢查；跨使用者存取被拒絕。 |
| `conversations.delete` | JWT | `conversation_id`（legacy alias：`id`） | 刪除對話（僅擁有者）。回傳 `{ "ok": true }`。 |

> `conversations.get`／`conversations.delete` 也接受舊版儀表板客戶端的
> legacy `id` key；兩者都存在時 `conversation_id` 優先。

## Usage

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?`（integer，預設 50，限制在 1–200）、`offset?`（integer，預設 0）、`project?`（string） | 呼叫者的分頁 usage records，最新在前，涵蓋 API-key 列（`arona-XX` 前綴）與 JWT 歸因列（`jwt-<user-uuid>`）。回傳 `{ "records", "total", "limit", "offset", "project" }`；`project` 過濾器只收窄到帶 key 標籤的列。 |

## Billing

Tiers、quotas 與用量會計說明於
[計費與用量](../guides/billing-usage.md)。

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | 目前的計費狀態：`{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }`——每月用量（`cost_usd`、tokens、請求數）與剩餘 quota。 |
| `billing.plan.set` | admin token | `user_email`、`tier` | 設定使用者的 billing tier。回傳 `{ "ok": true }`。bearer 與 `ARONA_ADMIN_TOKEN` 不符時以 `AUTH_ERROR` 拒絕。 |
| `billing.video.pricing.get` | JWT | — | 視訊定價表。回傳 `{ "pricing": [...] }`。 |
| `billing.video.pricing.set` | admin token | `model`、`mode?`（預設 `per_second_resolution`）、`base_price?`（number，預設 0）、`price_per_second?`（number，預設 0）、`price_per_frame?`（number，預設 0）、`resolution_coeff?`（object）、`currency?`（預設 `USD`）、`enabled?`（bool，預設 `true`） | 為模型 upsert 視訊定價。回傳 `{ "ok": true }`。bearer 與 `ARONA_ADMIN_TOKEN` 不符時以 `AUTH_ERROR` 拒絕。 |

## Video

非同步視訊生成任務（見[Realtime 與視訊](../guides/realtime-video.md)）。
任務進度以 `video.progress`／`video.done`／`video.failed` 通知推送到
session 通道。

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`、`prompt`、`negative_prompt?`、`images?`（`{ data_base64, mime_type }` 的陣列）、`duration_seconds?`（integer）、`width?`（integer）、`height?`（integer）、`provider?`（string）、`extra?`（object） | 送出非同步視訊生成任務。回傳 `{ "job_id", "stream_id" }`——訂閱 `stream_id` 以接收進度通知。 |
| `video.get` | JWT | `job_id`（UUID） | 輪詢任務的狀態／結果（status、progress、result、error、cost）。 |
| `video.list` | JWT | `limit?`（integer，預設 20） | 列出呼叫者的任務。回傳 `{ "jobs": [...] }`。 |
| `video.cancel` | JWT | `job_id`（UUID） | 取消運行中的任務。回傳 `{ "ok": true }`。 |

## System

| 方法 | Auth | 參數 | 說明 |
| --- | --- | --- | --- |
| `system.status` | public | — | 彙總的 gateway 狀態：`{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`。 |
| `system.probe` | anonymous（僅 WS） | — | 經由 WebSocket 傳輸的一次性 liveness 探測。伺服器 ack `{ "ok": true, "status": "ok" }` 然後關閉 socket——匿名訪客永遠不會持有開啟的連線。未認證 socket 上任何其他方法都以 `AUTH_ERROR` 被拒絕。 |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
