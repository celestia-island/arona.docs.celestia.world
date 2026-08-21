---
title: "Backend"
description: "Backend 类型（external、ollama、engine、minimax-cloud、evernight 桥接）、URL 语义、健康探测、模型发现、别名与路由。"
---

# Backend

**backend** 是提供模型流量的上游。Arona 将 OpenAI 兼容请求
（`/v1/chat/completions`、`/v1/embeddings`、模型列表、视频任务）路由到某个已
注册的 backend，对每个请求计量，并持续维护每个 backend 的健康状态与模型清单。

Backend 由管理员通过 `POST /api/admin/backends` 注册（见
[Admin HTTP API](../api/admin-http.md)），持久化到 `backend_configs` 表，并在
启动时自动恢复。每条注册携带一个 `name`、一个 `type`、一个 `url`、一个可选的
`api_key` 和一个可选的静态 `models` 列表。持久化的 backend 在重启后依然存在；
恢复的 backend 以 fail-closed 状态启动并立即被 probe。

## Backend 类型

| `type` | 传输 | 协议 | 用途 |
| --- | --- | --- | --- |
| `external` | HTTP(S) | OpenAI 兼容 REST | 任意聊天/embeddings API（云或自托管） |
| `ollama` | HTTP(S) | Ollama 原生 API（`/api/chat`、`/api/embed`、`/api/tags`） | 本地或远程 Ollama 服务器；仅凭 URL 即可构建 |
| `engine` | `ws://` / `wss://` | CEP（Celestia Engine Protocol），WebSocket + JSON-RPC | 任意使用 CEP 交换标准的引擎（`plana::engine`） |
| `minimax-cloud` | HTTPS | MiniMax H3 任务式 API（提交 + 轮询） | 云视频生成 |
| `evernight://<node>/<service>` | 桥接 URL | 通过本地 evernight agent 解析为本地 TCP 转发 | 只能通过 evernight 网格访问的工业/边缘服务 |
| `agent-{model}` | HTTP | OpenAI 兼容（external） | GPU agent 部署模型时自动注册 |

### `external` —— 任意 OpenAI 兼容 HTTP API

通用 backend：针对任意遵循 OpenAI REST 形态的服务器提供聊天补全（流式与非
流式）和 embeddings。使用基础 `url`、可选的 `api_key` 和可选的静态 `models`
列表配置：

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

静态 `models` 列表具有权威性：它排在 probe 时发现的任何模型之前（见
[模型发现](#model-discovery)）。

### `ollama` —— 仅凭 URL 构建

Ollama backend 仅凭 URL 构建——不需要 API key，不需要模型列表。它使用 Ollama
的原生协议：聊天走 `/api/chat`、embeddings 走 `/api/embed`、健康探测与模型
发现走 `/api/tags`。

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` —— 基于 WebSocket 的 CEP

`engine` backend 连接到暴露 `ws://`（或 `wss://`）的引擎，使用 **Celestia
Engine Protocol**（CEP）通信：这是定义在 `plana::engine` 中的 WebSocket +
JSON-RPC 2.0 交换标准。任何语言编写的、实现了 handshake → 方法 →
流式通知流程的引擎，都能零适配代码注册为一等 backend。线上方法：
`Engine.Handshake`（第一条消息；身份 + 能力）、`Engine.Chat`、
`Engine.ChatStart`（流式；块以 `Engine.ChatChunk` 通知到达）、
`Engine.Embeddings` 和 `Engine.Models`。连接在首次使用时惰性建立，任何错误时
断开；下一次调用会重新连接并重新握手。

### `minimax-cloud` —— 任务式视频生成

云视频 backend 驱动 MiniMax H3 开放平台 API：提交生成任务、轮询完成、再从
结果中读取产物 URL。这就是取代已移除的 ComfyUI backend 的方案（见下文）；
视频任务通过 `/v1/video/generations` 或 `video.*` RPC 方法提交，并通过
`video.progress` / `video.done` / `video.failed` 通知汇报进度
（见 [实时与视频](realtime-video.md)）。

### `evernight://` 桥接 URL

形如 `evernight://<node>/<service>` 的 backend URL **不会**被直接访问。本地
主机上的 evernight agent 将其解析（通过 agent 的 WebSocket 端点发起
`Bridge.Connect` JSON-RPC 调用）为本地 TCP 转发，backend 针对
`http://127.0.0.1:<local_port>` 运行，而不是硬编码的远程地址。这就是单 panel
架构：Arona panel 通过网格访问其他节点上的服务（CEP 引擎、scepter 等），而
无需在配置中嵌入任何远程地址。

一个 keepalive 任务每 15 秒 probe 一次隧道；当远端重启且隧道在新的本地端口上
重建时，受影响的 backend 会用新 URL **透明地重建**——持久化的配置保留
`evernight://` URL，因此重启会重新解析它。对于 `engine` backend，解析出的
`http://127.0.0.1:<port>` 转发会适配为 `ws://` 用于 WebSocket 传输。

### Agent 部署的模型自动注册

当 GPU agent 完成模型部署时，gateway 会注册一个名为 `agent-{model_id}`
的 backend（基于 `http://{agent host}:{port}` 的 `ExternalApiBackend`），使
模型立即可路由；停止部署则再次注销。完整部署生命周期见
[Agent 集群](agent-cluster.md)。

### `comfyui` 被拒绝

`comfyui` backend 类型被显式拒绝，错误为 `comfyui backend removed`：ComfyUI
backend 在模型平台收敛期间被移除，视频生成现在通过 `minimax-cloud` 运行。
注册 `comfyui` backend 会返回 HTTP 400。

## URL 语义

配置的基础 URL 如何映射到实际端点，取决于 URL 是否带有路径组件：

- **根式基础** —— 路径为空或 `/` 的 URL 被视为主机根，保留 OpenAI 的 `/v1`
  约定：`{base}/v1/chat/completions`、`{base}/v1/models`。示例：
  `http://192.0.2.20:8429`、`https://api.deepseek.com`。
- **路径式基础** —— 带非空路径的 URL 被视为服务器实际提供的完整 API 前缀，
  端点直接追加：`{base}/chat/completions`、`{base}/models`。这是 `/v1`
  约定之外的 OpenAI 兼容服务器所需要的。Zhipu GLM 编程方案是典型例子：其 API
  位于 `https://open.bigmodel.cn/api/coding/paas/v4`，聊天直接在
  `{base}/chat/completions`，并且**根本没有 `/models` 端点**——标准的
  `/api/paas/v4` 根路径对编程方案 key 会返回余额错误。
- 配置基础 URL 上的**尾部斜杠**会被归一化去除，拼接时不会产生双斜杠。

## 探测与健康

后台健康检查器每 **60 秒** probe 一次所有已注册的 backend；backend 列表在每轮
都会重新获取，因此启动后注册的 backend 无需重启即可被纳入。每次 admin 注册也
会触发一次立即 probe，使 backend 在约 1-2 秒内转为 healthy，而不是等待下一轮
检查。

- **External backend** probe `GET {base}/models`（根式基础则为
  `{base}/v1/models`），**2 秒超时**。**404 被容忍**：有些服务器实现了聊天但
  不暴露模型列表（GLM 编程方案没有 `/models` 端点），因此 404 会将该 backend
  标记为 healthy，管理员配置的 `models` 列表成为路由来源。超时、网络故障及其他
  非 2xx 响应会把 backend 标记为 unhealthy。
- **Ollama backend** 以相同的 2 秒超时 probe `/api/tags`。
- Backend 以 **fail-closed** 启动——报告为 `not probed yet`——直到首次成功
  probe，因此新注册（或恢复）的 backend 在验证之前永远不会接收流量。

健康状态按 backend 缓存，路由在每次请求时都会查询；unhealthy 的 backend 被
排除在候选选择之外（见 [路由](#routing)）。

## 模型发现

backend 通告它提供的模型 id，路由按此通告匹配请求：

- **External** backend 通告从 probe 响应解析出的模型（`data` 和 `models`
  数组都接受），并与管理员配置的静态列表合并——静态 id 保持顺序与优先级，
  动态 id 去重后追加。当服务器没有 models 端点时，仅静态列表作为路由来源。
- **Ollama** backend 通告 `/api/tags` 返回的 tags。
- **Agent 部署**的模型只通告实际部署的 `model_id`。

公开面是 `GET /v1/models`（需认证），它列出每个 healthy backend 的可路由模型
（见 [OpenAI 兼容 REST API](../api/openai-rest.md)）。

## 别名与名称归一化

别名把请求的模型 id 映射到目标 id。路由时先解析别名，因此针对别名的请求由
通告目标的任意 backend 服务：

```json
{ "alias": "fast-chat", "target": "deepseek-v4-flash" }
```

别名通过 `/api/admin/aliases` admin 端点管理，立即生效。

名称匹配还会归一化 Ollama 风格 tags：backend 列出 `nomic-embed-text:latest`
时能匹配裸请求 `nomic-embed-text`，因此 embedding/聊天请求无需维护 `:latest`
后缀。显式 tag（`qwen3:0.6b`）只匹配该确切 tag。

## 路由

每个请求都通过路由器解析，路由器选择一个 backend：

1. **别名解析** —— 请求的模型 id 通过别名表映射（如有）。
2. **Provider 提示** —— 可选的 `provider` 字段按 backend 名称（或 kind 名称，
   例如视频 backend 的 `cloud`）过滤候选。
3. **仅 healthy 候选** —— backend 必须报告 `Healthy` **且**通过其熔断器
   （连续 3 次失败会打开熔断器 30 秒，期间允许一次半开测试调用）才可被选中。
4. **最少计数选择** —— 提供该模型的候选按其每 backend 请求计数器排序，选择
   负载最少的那个。这会在提供同一模型的 healthy backend 之间分散负载。
5. **会话亲和性** —— 当请求携带 `conversation_id` 时，会话被固定到它首次
   使用的 backend。该固定记录在 `Weak` 引用映射中，因此被移除的 backend 会从
   映射中消失而不会产生索引漂移。亲和性是尽力而为的：在同一会话的多个轮次中
   复用同一 backend，让上游能复用按会话的运行时状态（热上下文、KV 缓存）。
   如果固定的 backend 已变 unhealthy（或固定引用随 backend 移除而失效），路由
   会回退到全新的一次最少计数选择并**重新绑定**该会话。

如果没有 healthy backend 提供该模型，路由失败：未知模型产生 `model not found`
（HTTP 404），已知但不可达的模型产生 `all backends unhealthy`，以 500 内部
服务器错误呈现。HTTP 502 保留给*可达*上游报告的失败（选中后的非 2xx 上游响应
和传输失败）。完整错误映射见 [运维](operations.md)。

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
