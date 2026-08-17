---
title: "OpenAI 相容 REST API"
description: "OpenAI 風格 /v1/* 參考——聊天補全、embeddings、模型列表、非同步視訊生成、錯誤形狀與速率限制。"
---

# OpenAI 相容 REST API

Arona 在 `/v1/*` 下暴露 OpenAI 相容的 REST 介面，供 LLM 聊天、
embeddings、模型列表、健康探測與非同步視訊生成使用。任何指向基礎 URL 的
OpenAI SDK 都能用於聊天與 embeddings；視訊 endpoint 遵循 OpenAI 的
task 型 submit/poll 慣例。

所有請求和回應內文都是 JSON。錯誤使用統一的形狀（見
[錯誤](#errors)）；middleware 層的認證失敗是唯一的例外，以純文字回傳
（見[認證](#authentication)）。

## Endpoints 一覽

| 方法 | 路徑 | 說明 |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | 聊天回合，串流或非串流。 |
| `POST` | `/v1/embeddings` | 一個或多個輸入的 embedding 向量。 |
| `GET` | `/v1/models` | 路由器模型合併 quick-start 模型。 |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`。 |
| `POST` | `/v1/video/generations` | 送出非同步視訊生成任務。 |
| `GET` | `/v1/video/generations/{id}` | 輪詢視訊任務的狀態／結果。 |

`/api/health`、`/healthz` 與 `/readyz` 是額外的就緒探測
（`/v1/health` 的 Kubernetes 型同義詞）。

## 認證

聊天、embeddings 與視訊 endpoint 以 `Authorization: Bearer` header 中的
**API key** 認證。API keys 透過管理平面建立（`keys.create`，見
[JSON-RPC API](./jsonrpc.md#keys)），形式為 `arona-<uuid>`。伺服器端以
SHA-256 雜湊儲存。

```
Authorization: Bearer arona-CHANGE_ME
```

- **缺少 header** → `401` 純文字：`Missing Authorization header. Use: Bearer <api_key>`。
- **無效或被撤銷的 key** → `401` 純文字：`Invalid API key`。
- `GET /v1/models` 額外接受 **JWT** access token（由 `auth.login`／
  `auth.register` 簽發），因此網頁儀表板可以用它與 RPC 平面相同的 token
  列出模型。對該 endpoint，訊息是
  `Missing Authorization header. Use: Bearer <api_key_or_jwt>` 與
  `Invalid API key or JWT`。

Middleware 層的拒絕是純文字內文，不是[錯誤](#errors)所述的 JSON 錯誤
形狀——JSON 形狀只在請求到達 handler 時產生。

每個已認證的 `/v1` 請求也會通過**記憶體內 per-key 速率限制器**（預設
60 RPM、60 秒視窗，可用 `ARONA_API_RATE_LIMIT_RPM` 設定）。超過它回傳
`429` 純文字：`Rate limit exceeded. Try again later.` Tier 層級的 quota
與速率限制分別強制執行，回傳帶 `Retry-After` header 的 JSON 429（見
[429 與 Retry-After](#429-and-retry-after)）。

> 管理 API keys、專案與其範圍涵蓋於
> [認證與安全性](../guides/auth-security.md)。

## POST /v1/chat/completions

核心的 OpenAI 相容聊天 endpoint，支援串流與 arona 特定的擴充
（`conversation_id`、`memory`、`extra`、`provider`）。

### 請求內文

| 欄位 | 類型 | 必填 | 備註 |
| --- | --- | --- | --- |
| `model` | string | 是 | `GET /v1/models` 列出的模型 id。 |
| `messages` | array | 是 | 聊天回合，見下文。 |
| `stream` | boolean | 否 | 預設 `false`。為 `true` 時回應是 SSE 串流（見[串流](#streaming)）。 |
| `temperature` | number | 否 | 取樣溫度，轉發到 upstream。 |
| `max_tokens` | integer | 否 | 補全 token 上限，轉發到 upstream。 |
| `conversation_id` | string | 否 | Session affinity + 持久化。對話必須存在且屬於 API-key 使用者（否則 `403` `conversation_forbidden`；不存在則 `404` `conversation_not_found`）。使用者回合在送出時持久化，assistant 回覆在回合完成時持久化；路由將對話釘在首次服務它的 backend。 |
| `memory` | boolean | 否 | 記憶 gateway 覆寫。預設 `true`（記憶 gateway 啟用時注入記憶 recall）；`false` 停用此請求的 recall 注入。 |
| `extra` | object | 否 | 自由形式的透傳，合併進 upstream payload 頂層（見下文）。 |
| `tools` | array | 否 | OpenAI 風格的函式呼叫定義，原樣透傳給 upstream。 |
| `provider` | string | 否 | 明確的 backend 選取提示，比對 backend **名稱**（或類型，不分大小寫）。設定時，只有符合提示的 backends 是候選者。 |

**`messages` 條目**是 `{ "role": "user" | "assistant" | "system", "content": "..." }`。
兩個擴充為多模態／agent 工作負載轉發到 upstream：

- `images` — 視覺請求的附加圖片（`{ "media_type", "data", "position" }`
  物件的陣列；external backend 將它們呈現為 OpenAI `image_url` content parts）。
- `tool_calls` — upstream 模型產生的函式呼叫 payload，要在後續回合中
  回聲。External backend 必須轉發這些，否則 agent pipeline（例如
  scepter skill chains）會失去每一次工具呼叫。

**`extra` 合併規則**：每個 `extra` key 都合併進 upstream 請求 payload
的頂層，有兩個硬保證——保留 key `model`、`messages`、`stream`、
`temperature`、`max_tokens` 與 `options` **永遠**不會被覆寫，gateway
自己已設定的任何 key 也不會。非物件的 `extra` 值會被忽略。

**`tools` 條目**是 `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }`，原樣轉發。

### 非串流回應

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1720000000,
  "model": "Qwen/Qwen3-1.7B",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 12, "completion_tokens": 2, "total_tokens": 14 }
}
```

- `choices[].message` 對函式呼叫回合可能攜帶 `tool_calls`。
- 請求的記憶狀態在 **`X-Arona-Memory`** 回應 header 回聲：
  `enabled`｜`disabled`｜`offline`。

### 串流

設定 `"stream": true`。回應是 `text/event-stream` SSE 串流——每個 chunk
一行 `data:`，各攜帶單一 JSON `ChatChunk`：

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta` 攜帶 `content`（函式呼叫串流則帶 `tool_calls` delta，
  含 `index`／`id`／`type`／`function`）。
- 在 OpenAI 相容的 upstreams 上，**終端 chunk** 攜帶真實 token 計數的
  `usage` 欄位；gateway 記錄它（對從不回報 usage 的 upstreams，例如
  ollama／ws_engine，退回本機 tokenizer 估算）。
- 串流以 `data: [DONE]` 終止。
- 串流錯誤以攜帶
  `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}`
  的 `data:` 事件送達；`[DONE]` 事件仍會跟隨，而失敗串流的 usage 記錄與
  assistant 持久化會被略過。
- `X-Arona-Memory` header 也出現在 SSE 回應上。

### 範例

```bash
curl http://192.0.2.10:8080/v1/chat/completions \
  -H "Authorization: Bearer arona-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-1.7B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

## POST /v1/embeddings

| 欄位 | 類型 | 必填 | 備註 |
| --- | --- | --- | --- |
| `model` | string | 是 | Embedding 模型 id（例如 `nomic-embed-text`——裸名稱也匹配 `:latest` tag）。 |
| `input` | string 或 string[] | 是 | 一個輸入，或多個。 |

回應：`{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`。

## GET /v1/models

列出今天可路由的模型：每個健康已註冊 backend 的模型列表，合併內建的
**quick-start 模型**（永遠宣告，即使在 backend 註冊前）：`Qwen/Qwen3-0.6B`、
`Qwen/Qwen3-1.7B`、`HuggingFaceTB/SmolLM2-1.7B-Instruct`、
`google/gemma-3-1b-it`、`microsoft/Phi-4-mini-instruct`、
`deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`。

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

Quick-start 模型以 `owned_by` 設為其 provider 出現；路由器模型攜帶擁有
backend 的名稱。

## 視訊生成

供支援視訊的 backends（例如 `minimax-cloud`，見
[Backends](../guides/backends.md)）使用的 task 型視訊 endpoint。任務
非同步進展；輪詢狀態 endpoint 直到 `done`。

### POST /v1/video/generations

| 欄位 | 類型 | 必填 | 備註 |
| --- | --- | --- | --- |
| `model` | string | 是 | 註冊在支援視訊 backend 上的視訊模型 id。 |
| `prompt` | string | 是 | 生成提示。 |
| `negative_prompt` | string | 否 | 負面提示。 |
| `images` | array | 否 | 條件／參考圖片，為 `{ "data_base64": "...", "mime_type": "image/png" }` 物件的陣列。 |
| `duration_seconds` | integer | 否 | 要求的時長。 |
| `width`／`height` | integer | 否 | 輸出解析度。 |
| `provider` | string | 否 | 明確的 backend 選取提示（backend 名稱）。 |
| `extra` | object | 否 | Backend 特定的 workflow 覆寫（seed、steps、cfg、……）。 |

成功 → `200`：

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

錯誤：`model` 或 `prompt` 缺席時 `400` `missing_fields`；沒有健康且支援
視訊的 backend 提供該模型時 `503` `video_backend_error`／`no_backend`；
每月 quota 用盡時 `429` `quota_error`／`quota_exceeded`。

### GET /v1/video/generations/{id}

回傳任務狀態：

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "running",
  "progress": 40,
  "result": null,
  "error": null,
  "cost": 0.0,
  "created_at": 1720000000
}
```

- `status`：`queued`｜`running`｜`done`｜`failed`｜`cancelled`；`progress`
  運行時在 0–90 之間推進，`done` 時達到 100。
- `result`（`done` 時）：`{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }`——`url` 指向 backend 提供的已生成 artifact。
- `error`（`failed`／`cancelled` 時）與 `cost` 在適用時填入。
- 錯誤：非 UUID id 時 `400` `bad_id`；任務不存在或屬於其他 API key 時
  `404` `no_job`。

視訊任務也會將進度扇出到 RPC SSE sidecar
（`video.progress`／`video.done`／`video.failed`，見
[事件與通知](./events.md#video-job-notifications)）。

## 錯誤

Gateway 層級的錯誤使用單一形狀（`json_error_response`）：

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| 狀態 | `type`／`code` | 何時 |
| --- | --- | --- |
| `400` | `invalid_request`／`missing_fields`、`missing_index`、`bad_id`、…… | 格式錯誤或缺少請求欄位。 |
| `403` | `auth_error`／`conversation_forbidden` | `conversation_id` 屬於另一位使用者。 |
| `404` | `invalid_request_error`／`model_not_found` | 沒有 backend 提供所要求的模型。訊息：`No backend available for model: <model>`。 |
| `404` | `invalid_request`／`conversation_not_found` | 找不到對話。 |
| `404` | `not_found`／`no_job` | 找不到視訊任務。 |
| `502` | `server_error`／`bad_gateway` | Upstream 非 2xx：訊息 `upstream <status>: <detail>`（detail 來自 upstream 錯誤內文，限制 4 KB）。傳輸失敗（connect/read/timeout）也對映為 502 並帶錯誤字串。 |
| `500` | `server_error`／`backend_error` | 其他 backend 失敗（例如 backend 不支援該操作）。 |
| `500` | `server_error`／`internal_error` | 任何其餘的 gateway 內部錯誤。 |
| `429` | 見下文 | Quota／速率限制拒絕，帶 `Retry-After`。 |

## 429 與 Retry-After

429 回應包含 `Retry-After` header（秒），讓 OpenAI 相容
客戶端退避：

| 觸發 | 狀態內文 | `Retry-After` |
| --- | --- | --- |
| 每月 quota 超限 | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | 距下個月的秒數。 |
| Tier 每分鐘速率限制 | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`。 |
| 記憶體內 per-key 限制器（預設 60 RPM） | 純文字 `Rate limit exceeded. Try again later.` | 無（middleware 拒絕）。 |

Tiers、quota 範圍與用量會計說明於
[計費與用量](../guides/billing-usage.md)。

## Usage 記錄

每個 `/v1` 請求在完成時（非串流聊天、串流聊天在終端 chunk、embeddings，
與完成時帶計算成本的視訊任務）於 API-key 前綴（`arona-XX`）下記錄一筆
usage 列。記錄模型與 quota 如何強制執行見
[計費與用量](../guides/billing-usage.md)。

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
