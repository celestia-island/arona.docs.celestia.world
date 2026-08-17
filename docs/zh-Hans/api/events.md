---
title: "事件与通知"
description: "Server-sent event (SSE) 旁路——chat.stream、models.progress、realtime.event 与视频通知。"
---

# 事件与通知

流式 token、部署进度和实时事件**不会**在 JSON-RPC WebSocket socket 上投递。
每个流式 RPC 创建一个**会话 id**，并把通知作为 server-sent events 推送到
SSE 端点：

```
GET /api/rpc/events?session=<session_id>
```

## 先订阅再发送配方

RPC 调用返回会话 id 与 SSE 订阅建立之间发出的通知会被**丢弃**（订阅前窗口）。
可靠的模式是：

1. 先打开 SSE 流（它会阻塞直到会话 id 被挂接）。
2. 发起返回会话 id 的 RPC（例如 `chat.send`、`agents.deploy`、`realtime.start`、
   `video.create`）。
3. 从 SSE 流读取陆续到达的通知。

每个通知都是 JSON-RPC 2.0 风格的消息，带 `"jsonrpc": "2.0"`、一个 `method`
和一个 `params` 对象。

## 通知目录

### `chat.stream`

每个 token 一条通知，由 `chat.send`（以及任何使用会话通道的流式聊天路径）
产生：

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` —— 一个内容增量。
- `is_complete` —— 在最终块之前为 `false`（上游附加 finish reason 时，最终
  内容块可能已经携带非空 token 的 `is_complete: true`）；**终止**通知总是紧随
  其后，带空的 `token` 和 `is_complete: true`。
- 流错误以终止通知投递，带 `token: "Stream error: ..."` 和 `is_complete: true`
  （见 `packages/core/src/gateway/rpc.rs`）。

### `models.progress`

`agents.deploy` 的模型下载进度，从 agent 转发。`stream_id` 来自
`agents.deploy` 响应。

### `realtime.event`

打开的实时全双工会话的服务器事件，推送到会话通道
（`packages/core/src/gateway/realtime.rs`）。通过 `realtime.event` RPC 发送的
客户端事件被转发到上游；服务器事件在这里到达。

### 视频任务通知

`video.create` 任务通过会话通道推送进度（`packages/core/src/gateway/video.rs`）：

| 方法 | 负载（params） | 含义 |
| --- | --- | --- |
| `video.progress` | `job_id`、`stream_id`、`status: "running"`、`progress`（0–90） | 任务正在运行。 |
| `video.done` | `job_id`、`stream_id`、`result`、`cost` | 任务完成；`result` 携带产物 URL。 |
| `video.failed` | `job_id`、`stream_id`、`error` | 任务失败或被取消。 |

## 排序说明

- SSE 流按会话有序；token 按生成顺序到达。
- 一个会话 id 只能被一个 SSE 订阅者消费；请在返回该 id 的 RPC 之前（或紧随
  其后）打开流。
- `POST /api/rpc` 上的 `x-session-id` 头会把 RPC **响应**本身也挂接到会话
  通道——供希望在同一流上回显响应的客户端使用。

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
