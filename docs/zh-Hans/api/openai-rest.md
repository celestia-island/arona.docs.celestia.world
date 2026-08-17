---
title: "OpenAI 兼容 REST API"
description: "OpenAI 风格 /v1/* 参考——聊天补全、embeddings、模型列表、异步视频生成、错误结构与限流。"
---

# OpenAI 兼容 REST API

Arona 在 `/v1/*` 下暴露 OpenAI 兼容的 REST 接口，用于 LLM 聊天、embeddings、
模型列表、健康探测和异步视频生成。把任意 OpenAI SDK 指向基础 URL 即可用于
聊天和 embeddings；视频端点遵循 OpenAI 的任务式提交/轮询约定。

所有请求与响应体都是 JSON。错误使用统一结构（见 [错误](#errors)）；中间件层
的认证失败是唯一的例外，以纯文本返回（见 [认证](#authentication)）。

## 端点一览

| 方法 | 路径 | 描述 |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | 聊天轮次，流式或非流式。 |
| `POST` | `/v1/embeddings` | 一个或多个输入的嵌入向量。 |
| `GET` | `/v1/models` | 路由器模型与快速上手模型合并。 |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`。 |
| `POST` | `/v1/video/generations` | 提交一个异步视频生成任务。 |
| `GET` | `/v1/video/generations/{id}` | 轮询视频任务的状态 / 结果。 |

`/api/health`、`/healthz` 和 `/readyz` 是额外的就绪探测（`/v1/health` 的
Kubernetes 风格别名）。

## 认证

聊天、embeddings 和视频端点使用 `Authorization: Bearer` 头中的 **API key**
认证。API key 通过管理平面创建（`keys.create`，见
[JSON-RPC API](./jsonrpc.md#keys)），形如 `arona-<uuid>`。它们以 SHA-256
哈希形式存储在服务器端。

```
Authorization: Bearer arona-CHANGE_ME
```

- **缺少头** → `401` 纯文本：`Missing Authorization header. Use: Bearer <api_key>`。
- **无效或已撤销的 key** → `401` 纯文本：`Invalid API key`。
- `GET /v1/models` 额外接受 **JWT** access token（由 `auth.login` /
  `auth.register` 签发），使 Web 控制台可以用与 RPC 平面相同的 token 列出
  模型。对该端点，消息为 `Missing Authorization header. Use: Bearer <api_key_or_jwt>`
  和 `Invalid API key or JWT`。

中间件层的拒绝是纯文本响应体，而不是 [错误](#errors) 中描述的 JSON 错误结构
——JSON 结构只有在请求到达 handler 后才会产生。

每个已认证的 `/v1` 请求还会经过一个**内存每 key 限流器**（默认 60 RPM，
60 秒窗口，可用 `ARONA_API_RATE_LIMIT_RPM` 配置）。超出时返回 `429` 纯文本：
`Rate limit exceeded. Try again later.` Tier 级配额与限流单独执行，返回带
`Retry-After` 头的 JSON 429（见 [429 与 Retry-After](#429-and-retry-after)）。

> API key、项目及其范围的完整管理见
> [认证与安全](../guides/auth-security.md)。

## POST /v1/chat/completions

核心 OpenAI 兼容聊天端点，支持流式以及 arona 专属扩展（`conversation_id`、
`memory`、`extra`、`provider`）。

### 请求体

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `model` | string | 是 | `GET /v1/models` 列出的模型 id。 |
| `messages` | array | 是 | 聊天轮次，见下文。 |
| `stream` | boolean | 否 | 默认 `false`。为 `true` 时响应是 SSE 流（见 [流式](#streaming)）。 |
| `temperature` | number | 否 | 采样温度，透传上游。 |
| `max_tokens` | integer | 否 | 补全 token 上限，透传上游。 |
| `conversation_id` | string | 否 | 会话亲和性 + 持久化。会话必须存在且属于该 API key 用户（否则 `403` `conversation_forbidden`，不存在则 `404` `conversation_not_found`）。用户消息在发送时持久化，assistant 回复在轮次完成时持久化；路由把会话固定到首次服务它的 backend。 |
| `memory` | boolean | 否 | 记忆网关覆盖。默认 `true`（记忆网关启用时注入记忆召回）；`false` 对本请求禁用召回注入。 |
| `extra` | object | 否 | 自由形式透传，合并进上游负载的顶层（见下文）。 |
| `tools` | array | 否 | OpenAI 风格函数调用定义，原样透传上游。 |
| `provider` | string | 否 | 显式 backend 选择提示，不区分大小写地匹配 backend **名称**（或 kind）。设置后，只有匹配提示的 backend 是候选。 |

**`messages` 条目**是 `{ "role": "user" | "assistant" | "system", "content": "..." }`。
两个扩展会为多模态 / agent 负载透传上游：

- `images` —— 视觉请求的附加图片（`{ "media_type", "data", "position" }`
  对象数组；external backend 将其渲染为 OpenAI `image_url` 内容片段）。
- `tool_calls` —— 上游模型产生的函数调用负载，在后续轮次中回显。external
  backend **必须**转发这些，否则 agent 管道（如 scepter 技能链）会丢失每次
  工具调用。

**`extra` 合并规则**：每个 `extra` 键都会在顶层合并进上游请求负载，有两个硬性
保证——保留键 `model`、`messages`、`stream`、`temperature`、`max_tokens`
和 `options` **永不**被覆盖，gateway 自身已设置的任何键也不会被覆盖。非对象
的 `extra` 值被忽略。

**`tools` 条目**是 `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }`，原样转发。

### 非流式响应

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

- `choices[].message` 在函数调用轮次中可能携带 `tool_calls`。
- 请求的记忆状态在 **`X-Arona-Memory`** 响应头中回显：`enabled` |
  `disabled` | `offline`。

### 流式

设置 `"stream": true`。响应是 `text/event-stream` SSE 流——每个块一行
`data:`，每行携带一个 JSON `ChatChunk`：

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta` 携带 `content`（函数调用流中还有带
  `index`/`id`/`type`/`function` 的 `tool_calls` delta）。
- 在 OpenAI 兼容上游上，**终止块**携带带真实 token 计数的 `usage` 字段；
  gateway 记录它（对从不报告 usage 的上游，例如 ollama / ws_engine，回退到
  本地分词器估算）。
- 流以 `data: [DONE]` 结束。
- 流错误以 `data:` 事件传递，携带
  `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}`；
  `[DONE]` 事件仍然跟随，失败流的 usage 记录与 assistant 持久化被跳过。
- SSE 响应同样带有 `X-Arona-Memory` 头。

### 示例

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

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `model` | string | 是 | 嵌入模型 id（例如 `nomic-embed-text` —— 裸名称也匹配 `:latest` tag）。 |
| `input` | string 或 string[] | 是 | 单个输入，或多个。 |

响应：`{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`。

## GET /v1/models

列出当前可路由的模型：每个 healthy 已注册 backend 的模型列表，与内置**快速上手
模型**合并（始终通告，即使尚未注册 backend）：`Qwen/Qwen3-0.6B`、
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

快速上手模型以 `owned_by` 设置为其提供商出现；路由器模型携带所属 backend 的
名称。

## 视频生成

面向支持视频的 backend（例如 `minimax-cloud`，见
[Backend](../guides/backends.md)）的任务式视频端点。任务异步推进；轮询状态
端点直到 `done`。

### POST /v1/video/generations

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `model` | string | 是 | 注册在支持视频的 backend 上的视频模型 id。 |
| `prompt` | string | 是 | 生成提示词。 |
| `negative_prompt` | string | 否 | 负面提示词。 |
| `images` | array | 否 | 条件/参考图片，为 `{ "data_base64": "...", "mime_type": "image/png" }` 对象数组。 |
| `duration_seconds` | integer | 否 | 请求的时长。 |
| `width` / `height` | integer | 否 | 输出分辨率。 |
| `provider` | string | 否 | 显式 backend 选择提示（backend 名称）。 |
| `extra` | object | 否 | Backend 专属工作流覆盖（seed、steps、cfg、...）。 |

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

错误：`model` 或 `prompt` 缺失时为 `400` `missing_fields`；没有 healthy 支持
视频的 backend 提供该模型时为 `503` `video_backend_error` / `no_backend`；
月度配额耗尽时为 `429` `quota_error` / `quota_exceeded`。

### GET /v1/video/generations/{id}

返回任务状态：

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

- `status`：`queued` | `running` | `done` | `failed` | `cancelled`；`progress`
  在运行中从 0 推进到 90，`done` 时达到 100。
- `result`（`done` 时）：`{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }` —— `url` 指向 backend 提供的生成产物。
- `error`（`failed` / `cancelled` 时）和 `cost` 在适用时填充。
- 错误：非 UUID id 为 `400` `bad_id`；任务不存在或属于另一个 API key 为
  `404` `no_job`。

视频任务还会通过 RPC SSE 旁路扇出进度（`video.progress` / `video.done` /
`video.failed`，见 [事件与通知](./events.md#video-job-notifications)）。

## 错误

Gateway 级错误使用一种结构（`json_error_response`）：

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| 状态 | `type` / `code` | 何时 |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`、`missing_index`、`bad_id`、... | 请求字段畸形或缺失。 |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id` 属于另一个用户。 |
| `404` | `invalid_request_error` / `model_not_found` | 没有 backend 提供所请求的模型。消息：`No backend available for model: <model>`。 |
| `404` | `invalid_request` / `conversation_not_found` | 会话不存在。 |
| `404` | `not_found` / `no_job` | 视频任务不存在。 |
| `502` | `server_error` / `bad_gateway` | 上游非 2xx：消息 `upstream <status>: <detail>`（detail 来自上游错误体，限制 4 KB）。传输失败（连接/读取/超时）也映射为 502，带错误字符串。 |
| `500` | `server_error` / `backend_error` | 其他 backend 失败（例如 backend 不支持该操作）。 |
| `500` | `server_error` / `internal_error` | 任何剩余的 gateway 内部错误。 |
| `429` | 见下文 | 带 `Retry-After` 的配额 / 限流拒绝。 |

## 429 与 Retry-After

429 响应包含 `Retry-After` 头（秒），使 OpenAI 兼容客户端退避：

| 触发 | 状态体 | `Retry-After` |
| --- | --- | --- |
| 月度配额超限 | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | 距下个月的秒数。 |
| Tier 每分钟限流 | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`。 |
| 内存每 key 限流器（默认 60 RPM） | 纯文本 `Rate limit exceeded. Try again later.` | 无（中间件拒绝）。 |

Tier、配额范围与用量核算见 [计费与用量](../guides/billing-usage.md)。

## Usage 记录

每个 `/v1` 请求在完成时都会在该 API key 前缀（`arona-XX`）下记录一条 usage
行（非流式聊天、流式聊天的终止块、embeddings，以及完成时带计算成本的视频
任务）。记录模型与配额执行见 [计费与用量](../guides/billing-usage.md)。

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
