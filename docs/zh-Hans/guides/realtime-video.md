---
title: "实时与视频"
description: "全双工实时会话（realtime.start/event/stop）、engine.invoke 感知/控制通道、异步视频生成任务。"
---

# 实时与视频

除了普通文本聊天，Arona 还暴露两个多模态通道：**全双工实时会话**（通过一个
双向通道收发语音/视频）和**异步视频生成**（在后台运行并汇报进度的任务式任务）。
两者都路由到提供所请求模型的 backend，并且都经过计费层计量。

## 实时会话

实时会话是**一个客户端**与**一个上游**之间的双向通道：云实时 API
（OpenAI-Realtime / Qwen-Omni-Realtime WebSocket 词汇）或本地 CEP 引擎。
客户端事件通过 JSON-RPC 到达并被转发到上游；服务器事件以 `realtime.event`
通知经会话的 SSE 通道推回。音频以 base64 PCM16 传输（客户端→模型 16 kHz，
模型→客户端 24 kHz），与云厂商的线上格式一致，因此 gateway 原样透传字节
（`packages/core/src/backends/realtime.rs:1-19`）。

### `realtime.start`

针对提供 `model` 的 backend 开启一个会话（JWT；参数 `model`、`config?`、
`conversation_id?` —— `packages/core/src/gateway/rpc.rs:1890-1898, 1914-1984`）。
backend **必须**声明 `realtime` 能力（音频/视频模态）；否则调用显式失败，报
`model {model} does not support realtime sessions (no audio/video modality)`——
不会静默回退到文本聊天
（`packages/core/src/gateway/realtime.rs:138-142`）。

支持两种上游类型（`packages/core/src/gateway/realtime.rs:143-167`）：

- **CEP 引擎上游** —— 通过 Celestia Engine Protocol 的 `Engine.InvokeStart`
  流式通道路由事件，因此本地部署的 omni 引擎无需新线上格式即可接入同一个
  客户端界面。
- **云上游** —— 一个使用云实时事件词汇的固定 `wss://` URL
  （`session.update`、`input_audio_buffer.*`、`response.audio.delta`、...）。
  云实现在重连时重新发送 `session.update`。

响应为 `{ "session_id": ..., "stream_session": ... }`——在调用之前（或紧随
其后）订阅 `/api/rpc/events?session=<stream_session>` 以接收服务器事件。
可选的 `conversation_id` 会把语音转录持久化为 assistant 消息，并记录 token
usage 用于计费（`packages/core/src/gateway/realtime.rs:32-85`）。

### `realtime.event`

向会话发送一个客户端事件（JWT；参数 `session_id`、`event` ——
`packages/core/src/gateway/rpc.rs:1989-2013`）。支持的事件包括
`session.update`、`input_audio_buffer.append` / `.commit` / `.clear`、
`input_image_buffer.append`、`response.create`、`response.cancel` 和
`session.stop`。`send_event` 是**非阻塞**的：事件在 mpsc 通道上排队，转发任务
将其写入上游（`packages/core/src/gateway/realtime.rs:254-280`）。只有会话
所有者可以发送事件。

### `realtime.stop`

关闭并移除会话（JWT；参数 `session_id` ——
`packages/core/src/gateway/rpc.rs:2016-2034`）。每个会话恰好拥有一个**转发
任务**，它持有上游并复用两个方向：客户端事件从队列消费，上游事件在同一循环中
轮询。转发任务在上游关闭或会话被停止时退出，移除注册表条目
（`packages/core/src/gateway/realtime.rs:195-250`）。

服务器事件以 `realtime.event` 通知推送，参数为 `{ session_id, event }`，
经会话通道发送——见 [事件与通知](../api/events.md)。

## `engine.invoke`

`engine.invoke` 是通用的**同步**引擎方法通道
（ADMIN：JWT + `is_admin`；参数 `model`、`method`、`params?` ——
`packages/core/src/gateway/rpc.rs:261-264,2049-2079`）。它在提供 `model` 的
backend 上调用任意方法并直接返回结果，使其成为高频感知/控制通道：
`sensor.ingest`、`control.setpoint` 风格调用，20-30 Hz 循环。没有通用调用通道
的 backend（所有 OpenAI 兼容 HTTP backend）以 `backend does not support
generic invocation` 显式拒绝（`packages/core/src/backends/mod.rs:573-586`）。

## 视频生成（REST）

视频任务是 REST 接口上的 OpenAI 风格异步任务（API key 认证 ——
`packages/core/src/gateway/server.rs:876-993`；见
[OpenAI 兼容 REST API](../api/openai-rest.md)）：

**`POST /v1/video/generations`**

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `model` | string | 必填 —— 选择支持视频的 backend。 |
| `prompt` | string | 必填。 |
| `negative_prompt` | string? | |
| `images` | array? | Base64 data URL（`data:image/png;base64,...`），以 `{ data_base64, mime_type }` 携带。 |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | Backend 选择提示（按 backend 名称匹配）。 |
| `extra` | object? | Backend 专属覆盖（seed、steps、cfg、...）。 |

响应：

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`** 轮询任务并返回 `id`、`object`、`model`、
`status`、`progress`、`result`、`error`、`cost`、`created_at`。任务按调用方
隔离：属于其他用户的任务返回 404。REST 接口执行与聊天路径相同的计费门禁
（月度配额、每分钟限流）。

## 视频生成（RPC）

同样的能力也可以通过 JSON-RPC 使用（JWT ——
`packages/core/src/gateway/rpc.rs:371-386,1296-1431`）：

| 方法 | 参数 | 返回 |
| --- | --- | --- |
| `video.create` | 与 REST 调用相同的字段 | `{ job_id, stream_id }`。 |
| `video.get` | `job_id` | 任务视图（status、progress、result、cost、...）。 |
| `video.list` | `limit?`（默认 20，限制 1-100） | `{ jobs: [...] }`，最新的在前。 |
| `video.cancel` | `job_id` | `{ "ok": true }` —— 只有所有者可以取消。 |

`video.create` 返回一个 `stream_id`；订阅
`/api/rpc/events?session=<stream_id>` 以接收任务通知
（`video.progress` / `video.done` / `video.failed` —— 见
[事件与通知](../api/events.md)）。

## Backend

视频生成**仅限云**：MiniMax H3 开放平台 API，backend kind 为
`minimax-cloud`（`BackendKind::CloudVideo` ——
`packages/core/src/backends/mod.rs:502-504,720-727,759-761`）。流程是任务式
的：

1. `POST /v1/video_generation_v2` → `task_id`
2. 轮询 `GET /v1/query/video_generation_v2?task_id=...` 直到 `Success` /
   `Fail` / 仍为 `Pending`
3. 成功后，通过 `GET /v1/files/{file_id}/content` → `{ file: { download_url } }`
   解析产物

（`packages/core/src/backends/minimax_cloud.rs:66-210`）。MiniMax backend
不提供聊天/embeddings；其能力声明 `supports_video_generation` 和
`realtime: false`（能力模型见 [Backend](./backends.md)）。路由只针对具有
`supports_video_generation` 的 backend 解析视频请求，并遵循可选的 `provider`
提示（`packages/core/src/routing/mod.rs:205-263`）。

**ComfyUI backend 已被移除**，发生在模型平台收敛期间：配置 backend kind
`"comfyui"` 会以 `comfyui backend removed` 失败
（`packages/core/src/backends/mod.rs:756-757`）。没有自托管视频路径；视频总是
通过 `minimax-cloud` backend 进行。

## 任务生命周期与定价

视频任务依次经过 `queued → running → done | failed | cancelled`
（`packages/core/src/gateway/video.rs`）：

- **create** —— 持久化任务行（`queued`，进度 0）并派生一个 poller 任务
  （`video.rs:109-176`）。
- **running** —— poller 提交任务（进度 5），然后每 1.5 秒轮询一次，每几轮把
  进度提升 5，最高到 **90**（`video.rs:178-275`）。轮询错误记录日志并重试。
- **done** —— 进度 100，持久化结果 URL 与计算出的成本，记录 usage，并扇出
  `video.done` 通知（`video.rs:332-368`）。
- **failed** —— 提交或轮询失败 → `video.failed`；经过 900 次轮询迭代
  （约 22.5 分钟）后任务以 `generation timed out` 失败。
- **cancelled** —— `video.cancel` 设置一个标志，poller 在下一轮观察到它；
  任务被标记为 `cancelled`，并触发带错误 `cancelled` 的 `video.failed`
  （`video.rs:389-400`）。

usage 以视频专属成本记录：`record_video` 写入一条每请求的 usage 记录，token
为零、美元成本明确（`packages/core/src/billing/mod.rs:496-531`）。

**定价**是模型专属的，位于 `video_pricing` 表（`packages/core/src/billing/video.rs`）：

| 模式 | 公式 |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution`（默认） | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` 将短边像素键（例如 `"768"`）映射为倍数，`"*"` 为回退值。
没有配置行的模型回退到：模式 `per_second_resolution`、`base_price` 0.0、
`price_per_second` 0.005、`price_per_frame` 0.0、`resolution_coeff {"*": 1.0}`、
货币 USD（`billing/video.rs:20-32`）。用 `billing.video.pricing.get`（JWT）
查询行，用 `billing.video.pricing.set`（admin token）upsert——见
[JSON-RPC API](../api/jsonrpc.md)。usage 记录如何聚合进月度计费见
[计费与用量](./billing-usage.md)。

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
