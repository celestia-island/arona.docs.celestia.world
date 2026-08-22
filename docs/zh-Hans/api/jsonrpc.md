---
title: "JSON-RPC API 参考"
description: "位于 /api/rpc 的 Arona 管理平面 JSON-RPC 2.0 API——通过 HTTP 和 WebSocket 提供 chat、realtime、engine、auth、keys、providers、agents、memory、conversations、usage、billing、video 和 system 方法。"
---

# JSON-RPC API 参考

Arona 在 `/api/rpc` 暴露 JSON-RPC 2.0 接口，用于管理平面：auth、keys、
providers、agents、memory、conversations、usage、billing、video、realtime
和流式聊天。它与 OpenAI 兼容 REST 面（`/v1/*`，见
[OpenAI 兼容 REST API](./openai-rest.md)）互补；key 认证的推理负载用 REST，
会话/账号管理与流控用 JSON-RPC。[快速上手](../guides/quickstart.md) 走通了
第一次端到端轮次。

该接口分发 **39 个请求方法**，外加一个匿名的仅 WebSocket 存活方法
`system.probe`（共 40 个方法）。每个请求都是 JSON-RPC 2.0 对象，含
`jsonrpc: "2.0"`、一个 `method` 字符串、一个可选的 `params` 对象和一个可选的
`id`。

## 传输

- **HTTP POST `/api/rpc`** —— 请求/响应。发送 `Content-Type: application/json`。
  JWT 放在 `Authorization: Bearer <jwt>` 头中。请求体上限为 1 MiB。
- **WebSocket `GET /api/rpc`** —— 长连接。浏览器无法在 WebSocket 升级时设置
  自定义头，因此 JWT 以 `?token=<jwt>` 查询参数传输；服务器内部将其折入
  `Authorization: Bearer` 头（见 `packages/core/src/gateway/server.rs`）。
  已认证的 socket 可以无限期保持连接。
- **批量请求** —— body 为 JSON 数组的 POST 会逐元素执行，并以相同顺序的响应
  JSON 数组应答。
- **匿名访问** —— 在没有 JWT 的 WebSocket 上，公开方法（`auth.register`/
  `auth.login`/`auth.refresh`、`providers.list`、`system.status`）仍可调用，
  `system.probe` 在 socket 关闭前以单个 ack 应答。其他所有方法都需要有效
  JWT；admin 门控方法还额外需要 admin 账号（见下方图例）。匿名 socket 还受
  10 秒空闲超时约束。
- **会话挂接** —— `POST /api/rpc` 上的 `x-session-id` 头会把 RPC 响应本身
  也推送到该会话通道，与流式通知并列。

## Id

请求 `id` 值按类型保真回显：`null` → `null`、字符串 → 字符串、整数 → 数字，
其他任何值（浮点数、对象、超出 i64 范围的整数）→ JSON 字符串渲染。省略 `id`
以 `null` 应答。

## 服务器 → 客户端通知（SSE 旁路）

Token、部署进度和实时事件**不会**在 WebSocket socket 上投递。每个流式 RPC
创建一个会话 id，并把通知作为 server-sent events 推送到
`GET /api/rpc/events?session=<session_id>`。请在 RPC 调用返回会话 id **之前
或紧随其后**订阅 SSE 端点——调用返回与 SSE 订阅建立之间发出的通知会被丢弃
（订阅前窗口）。推荐模式是先打开 SSE 流，再发起 RPC。

通知方法：`chat.stream`（`chat.send` 的每个 token 一条）、`models.progress`
（`agents.deploy` 的 agent 模型下载进度）、`realtime.event`（打开的实时会话
的服务器事件），以及 `video.progress` / `video.done` / `video.failed`
（异步视频任务）。完整目录见 [事件与通知](./events.md)。

## 错误码

| 代码 | 名称 | 含义 |
| --- | --- | --- |
| `-32700` | Parse error | 请求体不是有效 JSON。 |
| `-32600` | Invalid request | 请求对象畸形，例如缺少 `method`。 |
| `-32601` | Method not found | 未知的 `method` 字符串；消息会回显它。 |
| `-32602` | Invalid params | `params` 对该方法反序列化失败。 |
| `-32603` | Internal error | 意外的服务器故障。 |
| `-32000` | `APP_ERROR` | 通用应用错误——例如会话/provider/agent 不存在、没有在线 agent 可供部署。 |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` —— JWT 缺失或无效。admin token 方法的 bearer token 与 `ARONA_ADMIN_TOKEN` 不匹配时也使用（`"Admin access required"`）。 |
| `-32006` | `QUOTA_ERROR` | JWT 门控 RPC 方法（`chat.send`）的月度计费配额超限。 |
| `-32007` | `ADMIN_REQUIRED` | 已认证**非管理员**调用 admin 门控方法（`agents.*`、`engine.invoke`）；消息包含方法专属提示。 |

> `agents.*` 和 `engine.invoke` 方法仅限 admin：它们要求账号
> `users.is_admin = true` 的 JWT。已认证的非管理员以 `-32007`
> （`ADMIN_REQUIRED`）被拒绝；未认证的调用方得到标准的 `AUTH_ERROR`，
> 服务器不会暴露该方法是有特权的。

## 认证图例

| 图例 | 凭据 |
| --- | --- |
| **public** | 不需要凭据。 |
| **JWT** | HTTP 上 `Authorization: Bearer <jwt>`，或 WebSocket 上 `?token=<jwt>`。 |
| **admin（JWT + is_admin）** | 账号 `users.is_admin = true` 的 Bearer JWT。 |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN`（环境配置；未设置时方法总是被拒绝，默认拒绝）。 |

本文档中的所有示例凭据和地址都是占位符（RFC 5737 文档 IP、`sk-xxx` key）。
该图例背后的完整认证模型见 [认证与安全](../guides/auth-security.md)。

## Chat

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model`（string）、`messages`（`{ role, content, images?, tool_calls? }` 数组）、`temperature?`（number）、`max_tokens?`（integer）、`conversation_id?`（string）、`memory?`（bool）、`extra?`（object）、`tools?`（OpenAI 风格函数定义数组）、`provider?`（string） | 发送一轮流式聊天。返回 `{ "stream_id", "memory" }` —— `memory` 是召回状态（`enabled` / `disabled` / `offline`）；token 以 `chat.stream` 通知到达 SSE 旁路。带 `conversation_id` 时，服务端组装已完成的持久化历史并持久化本轮。计费门控（月度配额 → `-32006`）；usage 记录在 `jwt-<user-uuid>` 下。 |

## Realtime（全双工音频/视频会话）

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model`（string）、`config?`（会话配置对象）、`conversation_id?`（string） | 针对提供 `model` 的 backend 打开全双工会话。返回 `{ "session_id", "stream_session" }`：`session_id` 用于 `realtime.event` / `realtime.stop`，并订阅 SSE 旁路的 `stream_session` 以接收 `realtime.event` 通知。 |
| `realtime.event` | JWT | `session_id`（string）、`event`（客户端事件——音频 append/commit/clear、图像帧、response create/cancel、会话停止） | 向打开的会话发送一个客户端事件；它被转发到上游 backend。返回 `{ "ok": true }`。 |
| `realtime.stop` | JWT | `session_id`（string） | 关闭并移除会话。返回 `{ "removed": bool }`。 |

## Engine（通用感知/控制通道）

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `engine.invoke` | admin（JWT + is_admin） | `model`（string）、`method`（string）、`params?`（object） | 在提供 `model` 的 backend 上同步请求/响应调用任意引擎方法——`sensor.ingest` / `control.setpoint` 风格调用的高频通道（20–30 Hz 循环）。结果是 backend 的原始响应。 |

## Auth

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `auth.register` | public | `email`、`password`、`name?` | 注册账号。只在注册开放时允许（`ARONA_REGISTRATION_OPEN`）；第一个注册用户成为管理员。返回与 `auth.login` 相同的 token 响应（`access_token`、`refresh_token`、`token_type`、`expires_in`、`user`）。 |
| `auth.login` | public | `email`、`password` | 登录。返回 `access_token`、`refresh_token`、`token_type`、`expires_in`、`user`（`{ id, email, name, is_admin }`）。按 IP 和账号限流。 |
| `auth.refresh` | public | `refresh_token` | 用 refresh token 兑换新的 access token（和一个新的 refresh token）。重放或过期的 refresh token 以 `AUTH_ERROR` 拒绝。 |
| `auth.me` | JWT | — | 当前用户资料：`{ "id", "email", "name" }`。 |

## Keys

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | 列出调用方的 API key（id、name、`key_prefix`、project、时间戳、active 标志）。 |
| `keys.create` | JWT | `name`、`project?` | 创建 API key。返回 `{ id, name, key, key_prefix, project, created_at }` —— `key` 中的完整 `arona-<uuid>` 密钥**只显示一次**；请立即保存。 |
| `keys.revoke` | JWT | `key_id` | 撤销 API key。返回 `{ "ok": true }`。 |

## Providers

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | 列出已知 provider：内置官方条目加自定义条目，作为展示元数据（`id`、`name`、`description`、`website_domain`、`is_official`、`is_operator`）。按设计公开——列表不携带凭据；只有下面的变更操作受 JWT 门控。 |
| `providers.add` | JWT | `id`、`name`、`description?`、`website_domain?` | 添加自定义 provider 条目。返回 `{ "ok": true }`。 |
| `providers.update` | JWT | `provider_id`、`name?`、`description?`、`website_domain?` | 更新自定义 provider 的字段（只更新提供的）。返回 `{ "ok": true }`。 |
| `providers.remove` | JWT | `provider_id` | 移除自定义 provider。返回 `{ "ok": true }`。 |
| `providers.test` | JWT | — | 测试 provider 连接。Stub：返回 `{ "ok": true, "message": "Provider connection test not yet implemented" }`。 |

## Agents

所有 `agents.*` 方法仅限 admin（JWT + `is_admin`）。Agent 节点通过
`GET /ws/agent` 出站连接；该 RPC 组控制注册表（见
[Agent 集群](../guides/agent-cluster.md)）。

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `agents.list` | admin（JWT + is_admin） | — | 列出已注册的 agent 节点：id、name、host、`online`/`offline` 状态（基于心跳）、GPU 摘要、已部署模型、version、时间戳。 |
| `agents.register` | admin（JWT + is_admin） | `machine_name`、`version` | 向 tunnel 管理器注册 agent 节点。返回 `{ "agent_id", "token" }`（token 是 agent 的控制平面凭据）。 |
| `agents.deregister` | admin（JWT + is_admin） | `agent_id` | 注销（断开）一个 agent。返回 `{ "ok": true }`。 |
| `agents.status` | admin（JWT + is_admin） | `agent_id` | 单 agent 状态：online 标志、host、GPU 摘要、已加载模型、GPU 利用率、心跳/连接时间戳。 |
| `agents.deploy` | admin（JWT + is_admin） | `model_id`、`agent_id?`（空/缺失 = 最少负载节点；无在线节点则报错） | 在 agent 上部署模型。返回 `{ "ok": true, "stream_id" }` —— 订阅 SSE 旁路的 `stream_id` 以接收 `models.progress` 下载通知。 |
| `agents.stop` | admin（JWT + is_admin） | `agent_id`、`model_id` | 停止已部署的模型。返回 `{ "ok": true, "stream_id": null }`（无进度流）。 |

## Memory

长期记忆由 entelecheia Philia 服务通过 WebSocket 提供；故障从不阻断聊天
（见 [记忆网关](../guides/memory-gateway.md)）。

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | 记忆网关状态：`{ "enabled", "writeback", "events" }` —— 标志加上最多 50 条近期活动事件（最新的在前）。 |
| `memory.delete` | JWT | `node_id` | 删除已存储的记忆节点。返回 `{ "deleted": bool }`。 |

## Conversations

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | 列出调用方的会话，带相对时间戳。 |
| `conversations.create` | JWT | `title?`（默认 `New Conversation`） | 创建会话。返回新的会话对象。 |
| `conversations.get` | JWT | `conversation_id`（遗留别名：`id`） | 获取带消息的会话。所有权检查；跨用户访问被拒绝。 |
| `conversations.delete` | JWT | `conversation_id`（遗留别名：`id`） | 删除会话（仅所有者）。返回 `{ "ok": true }`。 |

> `conversations.get` / `conversations.delete` 也接受旧版 dashboard 客户端的
> 遗留 `id` 键；两者都出现时 `conversation_id` 优先。

## Usage

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?`（integer，默认 50，限制 1–200）、`offset?`（integer，默认 0）、`project?`（string） | 调用方分页的 usage 记录，最新的在前，涵盖 API key 行（`arona-XX` 前缀）和 JWT 归属行（`jwt-<user-uuid>`）。返回 `{ "records", "total", "limit", "offset", "project" }`；`project` 过滤器只收窄到 key 标记的行。 |

## Billing

Tier、配额与用量核算见 [计费与用量](../guides/billing-usage.md)。

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | 当前计费状态：`{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` —— 月度 usage（`cost_usd`、token、请求数）与剩余配额。 |
| `billing.plan.set` | admin token | `user_email`、`tier` | 设置用户的 billing tier。返回 `{ "ok": true }`。bearer 与 `ARONA_ADMIN_TOKEN` 不匹配时以 `AUTH_ERROR` 拒绝。 |
| `billing.video.pricing.get` | JWT | — | 视频定价表。返回 `{ "pricing": [...] }`。 |
| `billing.video.pricing.set` | admin token | `model`、`mode?`（默认 `per_second_resolution`）、`base_price?`（number，默认 0）、`price_per_second?`（number，默认 0）、`price_per_frame?`（number，默认 0）、`resolution_coeff?`（object）、`currency?`（默认 `USD`）、`enabled?`（bool，默认 `true`） | upsert 某模型的视频定价。返回 `{ "ok": true }`。bearer 与 `ARONA_ADMIN_TOKEN` 不匹配时以 `AUTH_ERROR` 拒绝。 |

## Video

异步视频生成任务（见 [实时与视频](../guides/realtime-video.md)）。任务进度以
`video.progress` / `video.done` / `video.failed` 通知推送到会话通道。

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`、`prompt`、`negative_prompt?`、`images?`（`{ data_base64, mime_type }` 数组）、`duration_seconds?`（integer）、`width?`（integer）、`height?`（integer）、`provider?`（string）、`extra?`（object） | 提交一个异步视频生成任务。返回 `{ "job_id", "stream_id" }` —— 订阅 `stream_id` 以接收进度通知。 |
| `video.get` | JWT | `job_id`（UUID） | 轮询任务状态/结果（status、progress、result、error、cost）。 |
| `video.list` | JWT | `limit?`（integer，默认 20） | 列出调用方的任务。返回 `{ "jobs": [...] }`。 |
| `video.cancel` | JWT | `job_id`（UUID） | 取消运行中的任务。返回 `{ "ok": true }`。 |

## System

| 方法 | 认证 | 参数 | 描述 |
| --- | --- | --- | --- |
| `system.status` | public | — | 聚合 gateway 状态：`{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`。 |
| `system.probe` | anonymous（仅 WS） | — | 通过 WebSocket 传输的一次性存活 probe。服务器 ack `{ "ok": true, "status": "ok" }` 然后关闭 socket——匿名访客永远不会持有打开的连接。未认证 socket 上的任何其他方法都以 `AUTH_ERROR` 拒绝。 |

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
