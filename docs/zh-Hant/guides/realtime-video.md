---
title: "Realtime 與視訊"
description: "全雙工 realtime session（realtime.start/event/stop）、engine.invoke 感知／控制通道，與非同步視訊生成任務。"
---

# Realtime 與視訊

Arona 在純文字聊天之外暴露兩個多模態通道：**全雙工 realtime session**
（經由單一雙向通道進出的語音／視訊）與**非同步視訊生成**（task 型任務，
在背景運行並回報進度）。兩者都路由到提供所要求模型的 backend，且都透過
計費層計量。

## Realtime session

realtime session 是**一個客戶端**與**一個 upstream** 之間的雙向通道：
雲端 realtime API（OpenAI-Realtime／Qwen-Omni-Realtime WebSocket
詞彙）或本機 CEP engine。客戶端事件經由 JSON-RPC 送達並轉發到 upstream；
伺服器事件以 `realtime.event` 通知經由 session 的 SSE 通道推回。音訊以
base64 PCM16 傳輸（16 kHz 客戶端→模型、24 kHz 模型→客戶端），與雲端
廠商的 wire 格式一致，因此 gateway 原封不動地傳遞位元組
（`packages/core/src/backends/realtime.rs:1-19`）。

### `realtime.start`

對提供 `model` 的 backend 開啟 session（JWT；參數 `model`、
`config?`、`conversation_id?`——`packages/core/src/gateway/rpc.rs:1890-1898,
1914-1984`）。backend **必須**宣告 `realtime` 能力（音訊／視訊
modalities）；否則呼叫明確失敗，錯誤為
`model {model} does not support realtime sessions (no audio/video modality)`——
沒有靜默退回文字聊天的機制
（`packages/core/src/gateway/realtime.rs:138-142`）。

支援兩種 upstream 類型（`packages/core/src/gateway/realtime.rs:143-167`）：

- **CEP engine upstream** — 透過 Celestia Engine Protocol 的
  `Engine.InvokeStart` 串流通道路由事件，因此本機部署的 omni engine
  以無新 wire 格式的方式加入相同的客戶端介面。
- **Cloud upstream** — 固定 `wss://` URL，使用雲端 realtime 事件詞彙
  （`session.update`、`input_audio_buffer.*`、`response.audio.delta`、
  ……）。cloud 實作在重連時重新發出 `session.update`。

回應是 `{ "session_id": ..., "stream_session": ... }`——在呼叫**之前**
（或緊接其後）訂閱 `/api/rpc/events?session=<stream_session>` 以接收
伺服器事件。選用的 `conversation_id` 將語音轉錄以 assistant 訊息持久化，
並為計費記錄 token usage（`packages/core/src/gateway/realtime.rs:32-85`）。

### `realtime.event`

將一個客戶端事件送入 session（JWT；參數 `session_id`、`event`——
`packages/core/src/gateway/rpc.rs:1989-2013`）。支援的事件包括
`session.update`、`input_audio_buffer.append`／`.commit`／`.clear`、
`input_image_buffer.append`、`response.create`、`response.cancel` 與
`session.stop`。`send_event` 是**非阻塞**的：事件排入 mpsc 通道，
forwarder task 將其寫到 upstream（`packages/core/src/gateway/realtime.rs:254-280`）。
只有 session 的擁有者可以送事件。

### `realtime.stop`

關閉並移除 session（JWT；參數 `session_id`——
`packages/core/src/gateway/rpc.rs:2016-2034`）。每個 session 恰好擁有
一個**forwarder task**，它持有 upstream 並多工兩個方向：客戶端事件從佇列
消費，upstream 事件在同一個迴圈輪詢。當 upstream 關閉或 session 被停止時，
forwarder 結束並移除 registry 條目
（`packages/core/src/gateway/realtime.rs:195-250`）。

伺服器事件以 `realtime.event` 通知推回，參數為 `{ session_id, event }`，
經由 session 通道——見[事件與通知](../api/events.md)。

## `engine.invoke`

`engine.invoke` 是通用的**同步** engine-method 通道
（ADMIN：JWT + `is_admin`；參數 `model`、`method`、`params?`——
`packages/core/src/gateway/rpc.rs:261-264,2049-2079`）。它在提供 `model`
的 backend 上呼叫任意方法並直接回傳結果，使其成為高頻感知／控制通道：
` sensor.ingest`、`control.setpoint` 型呼叫以 20-30 Hz 迴圈執行。沒有
通用呼叫通道的 backends（所有 OpenAI 相容 HTTP backends）明確拒絕，
錯誤為 `backend does not support generic invocation`
（`packages/core/src/backends/mod.rs:573-586`）。

## 視訊生成（REST）

視訊任務是 REST 介面上的 OpenAI 型非同步任務（API key 認證——
`packages/core/src/gateway/server.rs:876-993`；見
[OpenAI 相容 REST API](../api/openai-rest.md)）：

**`POST /v1/video/generations`**

| 欄位 | 類型 | 備註 |
| --- | --- | --- |
| `model` | string | 必填——選取支援視訊的 backend。 |
| `prompt` | string | 必填。 |
| `negative_prompt` | string? | |
| `images` | array? | Base64 data URLs（`data:image/png;base64,...`），以 `{ data_base64, mime_type }` 承載。 |
| `duration_seconds` | int? | |
| `width`／`height` | int? | |
| `provider` | string? | Backend 選取提示（與 backend 名稱比對）。 |
| `extra` | object? | Backend 特定的覆寫（seed、steps、cfg、……）。 |

回應：

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`** 輪詢任務並回傳 `id`、`object`、
`model`、`status`、`progress`、`result`、`error`、`cost`、`created_at`。
任務以呼叫者為範圍：屬於其他使用者的任務回傳 404。REST 介面強制與
聊天路徑相同的計費閘門（每月 quota、每分鐘速率限制）。

## 視訊生成（RPC）

相同的能力也可經由 JSON-RPC 使用（JWT——
`packages/core/src/gateway/rpc.rs:371-386,1296-1431`）：

| 方法 | 參數 | 回傳 |
| --- | --- | --- |
| `video.create` | 與 REST 呼叫相同的欄位 | `{ job_id, stream_id }`。 |
| `video.get` | `job_id` | 任務檢視（status、progress、result、cost、……）。 |
| `video.list` | `limit?`（預設 20，限制在 1-100） | `{ jobs: [...] }`，最新在前。 |
| `video.cancel` | `job_id` | `{ "ok": true }` — 只有擁有者可以取消。 |

`video.create` 回傳 `stream_id`；訂閱
`/api/rpc/events?session=<stream_id>` 以接收任務通知
（`video.progress`／`video.done`／`video.failed`——見
[事件與通知](../api/events.md)）。

## Backend

視訊生成**僅限雲端**：MiniMax H3 Open Platform API，backend 類型
`minimax-cloud`（`BackendKind::CloudVideo`——
`packages/core/src/backends/mod.rs:502-504,720-727,759-761`）。流程是
task 型：

1. `POST /v1/video_generation_v2` → `task_id`
2. 輪詢 `GET /v1/query/video_generation_v2?task_id=...` 直到 `Success`／
   `Fail`／仍為 `Pending`
3. 成功時，經由
   `GET /v1/files/{file_id}/content` → `{ file: { download_url } }` 解析 artifact

（`packages/core/src/backends/minimax_cloud.rs:66-210`）。MiniMax backend
不提供 chat／embeddings；其能力宣告 `supports_video_generation` 與
`realtime: false`（能力模型見[Backends](./backends.md)）。路由只會對
`supports_video_generation` 的 backends 解析視訊請求，並尊重選用的
`provider` 提示（`packages/core/src/routing/mod.rs:205-263`）。

**ComfyUI backend 已被移除**，在模型平台收斂期間：設定 backend 類型
`"comfyui"` 會失敗，錯誤為 `comfyui backend removed`
（`packages/core/src/backends/mod.rs:756-757`）。沒有自架視訊路徑；
視訊一律經由 `minimax-cloud` backend。

## 任務生命週期與定價

視訊任務依 `queued → running → done | failed | cancelled` 移動
（`packages/core/src/gateway/video.rs`）：

- **create** — 任務列被持久化（`queued`、progress 0），並產生一個
  poller task（`video.rs:109-176`）。
- **running** — poller 送出任務（progress 5），然後每 1.5 秒輪詢，
  每數次疊代將 progress 增加 5，最高 **90**（`video.rs:178-275`）。
  輪詢錯誤會記錄並重試。
- **done** — progress 100、結果 URL 與計算出的成本被持久化、
  usage 被記錄，並扇出 `video.done` 通知（`video.rs:332-368`）。
- **failed** — submit 或 poll 失敗 → `video.failed`；900 次輪詢疊代
  （約 22.5 分鐘）後任務以 `generation timed out` 失敗。
- **cancelled** — `video.cancel` 設定一個旗標，poller 在下一輪觀察到它；
  任務被標記為 `cancelled`，並以錯誤 `cancelled` 觸發 `video.failed`
  （`video.rs:389-400`）。

Usage 以視訊特定的成本記錄：`record_video` 寫入一筆每請求的 usage record，
token 為零、帶明確的美元成本（`packages/core/src/billing/mod.rs:496-531`）。

**定價**是模型特定的，位於 `video_pricing` 資料表
（`packages/core/src/billing/video.rs`）：

| 模式 | 公式 |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution`（預設） | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` 將短邊像素鍵（例如 `"768"`）對映到倍數，以 `"*"`
作為後備值。沒有設定列的模型退回：模式 `per_second_resolution`、
`base_price` 0.0、`price_per_second` 0.005、`price_per_frame` 0.0、
`resolution_coeff {"*": 1.0}`、貨幣 USD（`billing/video.rs:20-32`）。
用 `billing.video.pricing.get`（JWT）查詢列、用 `billing.video.pricing.set`
（admin token）upsert——見[JSON-RPC API](../api/jsonrpc.md)。
usage records 如何彙總進每月計費見[計費與用量](./billing-usage.md)。

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
