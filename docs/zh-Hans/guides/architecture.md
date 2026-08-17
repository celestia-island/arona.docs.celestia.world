---
title: "架构"
description: "Arona 的构成——工作区布局、请求在 gateway 中的路径、路由、健康探测、记忆、会话与有意的设计权衡。"
---

# 架构

本页介绍 Arona 的结构以及请求如何流经它：工作区布局、请求路径、gateway 与
路由器、健康检查、记忆客户端、会话与通知，最后是设计有意接受的限制与权衡。
运行示例见 [快速上手](quickstart.md)，日常运行时关注点见 [运维](operations.md)。

## 工作区布局

仓库是一个包含三个包的 Cargo 工作区：

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC,
│            # backends, migration, entities, providers, models, registry,
│            # orchestration
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

- `packages/core` 是库 crate（`_core`）。它包含服务器所需的一切：axum gateway
  （`gateway/`）、模型路由器（`routing/`）、backend 适配器（`backends/`）、
  计费（`billing/`）、认证（`auth.rs`）、记忆客户端（`memory/`）、JSON-RPC
  平面（`gateway/rpc.rs`）、schema（`migration/`、`entity/`）、模型元数据
  （`models/`、`providers/`、`registry/`）和模型编排（`orchestration/`）。
- `packages/agent` 构建在 GPU 节点上运行并通过 `/ws/agent` 回连的 `_agent`
  二进制（见 [agent-cluster](agent-cluster.md)）。
- `packages/cli` 构建用于安装、部署、serve、迁移和下载操作的 `_cli` 二进制。

本仓库不再有 Web 控制台：Vue 控制台已被移除，现位于
[shittim-chest](https://github.com/celestia-island/shittim-chest)（chest #291），
它通过 JSON-RPC 接口与 Arona 通信。Arona 本身是纯后端
（见 [概述](./README.md)）。

## 请求路径

入口是 `GatewayServer::app` 中组装的 axum 路由器
（`packages/core/src/gateway/server.rs`）。其路由表（server.rs:128-163）覆盖
OpenAI 兼容 REST 面（`/v1/chat/completions`、`/v1/embeddings`、`/v1/models`、
`/v1/health`）、视频生成、`/api/rpc` JSON-RPC 端点（POST + WebSocket 升级）、
SSE 旁路 `/api/rpc/events`、agent 控制平面 `/ws/agent`、位于 `/docs` 的
Swagger UI，以及 admin backend/别名管理端点。

路由器被包在一小栈 layer 中（server.rs:158-162）：

1. 作为 `Extension` 注入的认证管理器，使各 handler 的提取器可以访问它们。
2. 一个请求 id layer，复用入站的 `X-Request-ID` 头或生成一个，暴露给
   handler 和日志（`gateway/request_id.rs`）。
3. 一个 1 MB 请求体限制（`RequestBodyLimitLayer`）。
4. 一个宽松的 CORS layer（`*` 来源、`*` 头）。

由于 axum 自底向上应用 layer，CORS layer 在最外层。

每个 `/v1/*` handler 随后都经过同一个骨架：

1. **认证提取** —— 仅 key 的端点（`/v1/chat/completions`、`/v1/embeddings`、
   视频）用 `ApiKeyAuth`，`GET /v1/models` 用 `ApiKeyOrJwt`——它必须同时接受
   API key 和会话 JWT（`gateway/middleware.rs`）。提取器把 key/JWT 解析为
   用户 email、key 前缀、一个限流键（API key 的 SHA-256 哈希，或 JWT 的
   `u:<email>` 标签，这样轮换的 token 不会重置窗口）和一个可选的项目范围。
2. **计费门禁** —— 当用户的 tier 月度配额或每分钟限流超限时，
   `enforce_billing_gates`（server.rs:492-539）以 HTTP 429 + `Retry-After`
   拒绝请求。数据库故障 fail open：计费是尽力而为的，绝不是服务聊天的硬依赖。
3. **记忆召回**（聊天路径）—— 如果记忆客户端已配置且请求要求它，相关长期
   记忆作为 system 段落注入（见下文 [记忆客户端](#memory-client)）。故障从不
   阻断聊天；结果状态在 `X-Arona-Memory` 头中回显。
4. **会话持久化** —— 可选的 `conversation_id` 经过所有权检查，用户消息在
   发送时持久化。
5. **Gateway 分发** —— 请求交给 `Gateway`，它解析一个 backend、裁剪上下文、
   调用 backend trait。
6. **Usage 记录** —— 返回的（或流终止的）usage 通过 `UsageTracker` 在 key
   前缀下记录并持久化。

`Gateway` 本身以 `Arc<Gateway>` 存在于 `AppState` 中——没有外部互斥锁；内部
可变性保证并发的聊天/embeddings/流调用永远不会在一次上游 HTTP 往返中持有锁
（`gateway/mod.rs:29-53`）。

## Gateway 与路由器

`Gateway`（`packages/core/src/gateway/mod.rs`）拥有：

- **路由器状态** —— backend 列表与别名，由 `tokio::sync::RwLock` 保护。
  读侧解析在 await 之间借用；变更（register/remove/alias）持有短暂写锁，绝不
  在上游调用期间持有它。
- **一个请求计数器**（`AtomicU64`）和一个 `start_time`，供 `system.status`
  和健康端点使用。
- **一个部署映射**（`model_id → backend name`）用于 agent 部署的模型。
  `register_agent_backend` 构建一个名为 `agent-{model_id}` 的
  `ExternalApiBackend` 并插入路由器；重新注册同一模型会替换之前的 backend，
  `unregister_agent_backend` 在 `stop_result` 帧时移除它
  （见 [agent-cluster](agent-cluster.md)）。

Backend 解析发生在 `Router` 中（`packages/core/src/routing/mod.rs`）：

1. **别名解析** —— 配置的别名被改写为其目标。
2. **会话亲和性** —— 存在 `conversation_id` 时，路由器查询一个弱引用映射，
   把会话固定到它首次被服务的 backend。弱引用保证映射只在 backend 已注册或
   在途时存活，因此被移除的 backend 会消失而不会产生索引漂移。熔断器跳闸或
   固定的 backend 变 unhealthy 会降级为一次新的选择，从而重新绑定会话。
3. **候选过滤** —— 可选的 `provider` 提示按 backend 名称/kind 过滤；候选
   必须 healthy **且**熔断器未跳闸，并且必须列出所请求的模型。模型 id 精确
   匹配或通过 `:latest` 后缀约定匹配（裸 `nomic-embed-text` 请求匹配列出的
   `nomic-embed-text:latest`）。
4. **最少负载选择** —— 存活候选按命中计数器排序，选择负载最少的那个；会话
   固定（如有）同时记录。

在调用 backend 之前，`RequestPipeline::transform`
（`packages/core/src/pipeline.rs:422+`）把消息列表裁剪到 backend 的
`max_context_length`：system 消息完整保留，非 system 消息在放得下的前提下按
最新优先保留，单条超大消息按字符硬截断（启发式 token 计数器无法按 token 精确
截断）。随后调用走 `InferenceBackend` trait；成功与失败都记录回路由器的每
backend 熔断器（3 次失败、30 秒恢复、1 次半开调用——routing/mod.rs:57-64）。

## 健康检查器与探测

`run_health_checks`（`packages/core/src/gateway/health_checker.rs`）作为启动时
派生的后台任务运行（run.rs:135-150），每 60 秒间隔 probe 一次所有已注册的
backend。两个细节很重要：

- backend 列表**每轮都通过异步 fetcher 闭包重新获取**，因此启动后注册的
  backend（例如通过 admin API）无需重启即可被纳入。
- 第一轮立即运行，先于第一个间隔，因此进程一启动健康状态就建立起来。

`probe_backend` 是唯一的 probe 代码路径。它被一次性**注册时 probe**复用：
管理员注册 backend 后（server.rs:688-693）或持久化 backend 在启动时恢复后
（run.rs:122-127），即发即忘的 probe 会在约 1-2 秒内把 backend 转为 healthy，
而不是保持 fail-closed 直到下一轮 60 秒。这就是新注册的 external backend 的
模型列表几乎立即出现在 `GET /v1/models` 中的原因。

probe 本身是一个轻量 backend 调用（例如 external backend 以 2 秒 probe 超时
访问 `/v1/models`）；结果缓存在 backend 中，路由只选择缓存健康状态为
`Healthy` 的 backend（外加熔断器未跳闸）。

## 记忆客户端

记忆客户端（`packages/core/src/memory/mod.rs`）在服务器启动时由环境配置构建
（server.rs:95-97）：当设置了 `ARONA_MEMORY_URL` 和 `ARONA_MEMORY_TOKEN`
时，聊天请求通过 JSON-RPC WebSocket 查询 entelecheia Philia 记忆服务，
`recall_and_inject` 把相关记忆作为 system 段落（`## Relevant Long-Term
Memories`）前置到出站上下文。完成的轮次通过 `writeback_dialogue` 写回为
episode——这是 assistant 回复持久化后派生的即发即忘工作，因此记忆故障从不
阻塞或拖慢聊天响应路径。`ARONA_MEMORY_WRITEBACK`（默认开启）切换写回。
完整图景见 [memory-gateway](memory-gateway.md)。

每个聊天响应都携带一个 `X-Arona-Memory` 头，取三种状态之一：`enabled`
（召回运行并注入了内容）、`disabled`（未配置或请求传了 `memory: false`）或
`offline`（已配置但服务不可达）。

## 会话与通知

`AppState` 持有一个 `plana` `SessionManager`（`state.sessions`）。`chat.send`
等流式 RPC 会创建一个会话 id（`gateway/rpc.rs:1701`），并把通知——`chat.stream`
token、`models.progress` 部署进度、`realtime.event`——推送到该会话的通道。
客户端从 SSE 旁路 `GET /api/rpc/events?session=<id>`（server.rs:191-200）消费
它们；通知格式与订阅前窗口注意事项见 [events](../api/events.md)。

会话通道也用于请求/响应 RPC 调用：当客户端在 `POST /api/rpc` 上发送
`x-session-id` 头时，服务器把结果也推送到该会话通道（server.rs:184-188、
rpc.rs:128-144），因此客户端可以把 RPC 响应复用到一条已打开的 SSE 流上。

## 限制与设计权衡

设计有意接受许多限制；生产使用前请了解它们：

- **1 MB 请求体限制** —— 更大的请求体被该 layer 拒绝；如果需要大上下文调用，
  这是第一个要调优的地方。
- **CORS `*`** —— gateway 应答来自任何地方的跨源调用。对 API 没问题，但如果
  暴露给不受信任的客户端之外，请在前面加一个执行你自己 CORS 策略的代理。
- **计费 fail-open** —— 数据库不可用时，配额/限流执行降级为放行请求。计费是
  计量，不是访问控制。
- **SSE 流没有整体超时** —— 流式调用没有总截止时间（长时间生成是合法的）；
  挂起检测依赖 120 秒的每次读取空闲超时（`backends/external.rs:24-31`）。
  非流式调用有 600 秒的整体截止时间。
- **分词器估算 usage** —— 从不报告 usage 的 backend（ollama、ws_engine）
  按本地 CJK 感知分词器估算计费，原样记录（见 [billing-usage](billing-usage.md)）。
- **内存中的限流窗口与撤销** —— 每 key 滑动窗口和被撤销 key 集合存在于进程
  内存（`auth.rs`），因此重启会重置它们。认证级限流器按每 key 每窗口限制
  请求；计费 tier 限流器由数据库支撑（见 [auth-security](auth-security.md)
  和 [billing-usage](billing-usage.md)）。
- **`/ws/agent` 未认证** —— agent 控制平面接受任意说 register/heartbeat
  协议的 WebSocket。只有在你控制的网络上才是安全的。
- **gateway 无 TLS** —— 服务器绑定纯 HTTP；任何跨越网络边界的部署都应在前面
  终结 TLS（反向代理）。见 [deployment](deployment.md)。

在优雅一侧，服务器执行优雅停机：它安装 Ctrl+C 和 SIGTERM 处理器，记录
"draining connections"，并让进行中的请求完成后再退出（`gateway/run.rs:14-38`，
以及 run.rs:154-159 的 `with_graceful_shutdown` 接线）。

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table); gateway/mod.rs:29-53; routing/mod.rs:102-200; health_checker.rs:14-37; run.rs:135-159; memory/mod.rs:207-241; rpc.rs:128-144,1701 -->
