---
title: "事件與通知"
description: "Server-sent event（SSE）sidecar——chat.stream、models.progress、realtime.event 與視訊通知。"
---

# 事件與通知

串流 tokens、部署進度與 realtime 事件**不會**在 JSON-RPC WebSocket socket
上送達。每個串流 RPC 建立一個 **session id**，並以 server-sent events 將
通知推送到 SSE endpoint：

```
GET /api/rpc/events?session=<session_id>
```

## 先訂閱再送出的配方

RPC 呼叫回傳 session id 與 SSE 訂閱建立之間發出的通知**會被丟棄**
（預訂閱視窗）。可靠的模式是：

1. 先開啟 SSE 串流（它會阻塞直到 session id 附掛）。
2. 觸發回傳 session id 的 RPC（例如 `chat.send`、
   `agents.deploy`、`realtime.start`、`video.create`）。
3. 通知到達時從 SSE 串流讀取。

每個通知都是 JSON-RPC 2.0 型訊息，帶 `"jsonrpc": "2.0"`、一個 `method`
與一個 `params` 物件。

## 通知目錄

### `chat.stream`

每個 token 一則通知，由 `chat.send` 產生（以及任何使用 session 通道的
串流聊天路徑）：

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — 一個內容 delta。
- `is_complete` — `false` 直到最終 chunk（upstream 附掛 finish reason
  時，最終內容 chunk 可能已攜帶非空 token 的 `is_complete:
  true`）；**終端**通知永遠緊隨其後，帶空的 `token` 與 `is_complete: true`。
- 串流錯誤以終端通知送達，帶 `token: "Stream error: ..."` 與
  `is_complete: true`（見 `packages/core/src/gateway/rpc.rs`）。

### `models.progress`

`agents.deploy` 的模型下載進度，從 agent 轉發。`stream_id` 來自
`agents.deploy` 回應。

### `realtime.event`

開啟的全雙工 realtime session 的伺服器事件，推送到 session 通道
（`packages/core/src/gateway/realtime.rs`）。經由 `realtime.event` RPC
送出的客戶端事件會被轉發到 upstream；伺服器事件在此送達。

### 視訊任務通知

`video.create` 任務在 session 通道上推送進度
（`packages/core/src/gateway/video.rs`）：

| 方法 | Payload（params） | 意義 |
| --- | --- | --- |
| `video.progress` | `job_id`、`stream_id`、`status: "running"`、`progress`（0–90） | 任務運行中。 |
| `video.done` | `job_id`、`stream_id`、`result`、`cost` | 任務完成；`result` 攜帶 artifact URL。 |
| `video.failed` | `job_id`、`stream_id`、`error` | 任務失敗或被取消。 |

## 排序備註

- SSE 串流依 session 排序；tokens 依生成順序送達。
- 單一 session id 只能被一個 SSE 訂閱者消費；在回傳該 id 的 RPC 之前
  （或緊接其後）開啟串流。
- `POST /api/rpc` 上的 `x-session-id` header 也會把 RPC **回應**本身
  附掛到 session 通道——供想要在同一串流上回聲回應的客戶端使用。

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
