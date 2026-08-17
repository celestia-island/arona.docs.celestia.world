---
title: "运维"
description: "运行中的 arona-server 的健康端点、RUST_LOG 追踪、上游超时、错误映射与故障排查。"
---

# 运维

本页面向运行 `arona-server serve` 的运维人员。涵盖你要 probe 的健康端点、
值得 grep 的日志行、应用于上游 backend 的超时模型、backend 故障如何映射为
HTTP 错误，以及常见的运维陷阱。部署本身见[部署指南](./deployment.md)。

## 健康矩阵

三个健康端点都不需要认证，只要进程在服务就返回 `200 OK`——没有 liveness /
readiness 之分：

| 端点 | 响应 |
| --- | --- |
| `/healthz`、`/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | 与上面相同的详细响应体 |
| `/api/health` | plana `HealthResponse`：`status`、`version`（`CARGO_PKG_VERSION`）、`kind`（`Dev`）、`uptime`（秒）、`network`（transport / region / asn）、`build_hash`（`BUILD_HASH`）、`engine_version`（`"0.1.0"`） |

`/healthz` 和 `/readyz` 是同一处理器的别名，`/v1/health` 与之共享，因此
Kubernetes 风格的 probe 与 OpenAI 兼容的健康路由可以互换。`/api/health` 额外
提供 uptime、network 与引擎版本。负载均衡器和守护进程用 `/readyz`；需要更丰富
的负载时用 `/api/health`。

## 日志

服务器通过 `tracing` 记录日志，用标准的 `RUST_LOG` 变量过滤（`RUST_LOG=info`
是常见设置；`RUST_LOG=debug` 会暴露 probe 流量）。值得了解的事件，按大致出现
频率排序：

| 日志行 | 级别 | 含义 |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | 每个聊天请求一条，带 `key_prefix`、`model`、`stream` 和 `request_id` —— 最简单的按请求审计线索。 |
| `request completed` | info | 由 `logging_middleware` 助手在每个**非流式** `/v1/chat/completions` 和 `/v1/embeddings` 响应后记录：`method`、`path`、`status`、`latency_ms`、`trace_id`。（流式聊天改为在开始时记录 `chat completions SSE request`。） |
| `usage recorded` / `usage persisted` | info | 一条 usage 行被记录（内存态，带 token/成本），然后写入 `usage_records` 表。 |
| `external probe: sending` / `external probe: returned` | debug | 对 external backend 的 `/v1/models` 的健康 probe；`matched` 说明 probe 是否在 2 秒 probe 超时内完成。 |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | 被计费门禁拒绝的 `/v1/*` 请求——客户端收到 429 加 `Retry-After`。 |
| `rpc billing gate rejected: monthly quota exceeded` | warn | JWT 认证方法的 RPC 侧配额门禁（整个用户窗口；JSON-RPC 错误响应）。 |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | 启动恢复：从数据库加载管理员注册的 backend 和 agent 节点并重新使其可路由。 |
| `Shutdown signal received, draining connections…` | info | 优雅停机开始（SIGINT/SIGTERM）。 |

## 超时模型

超时在用于 external backend 的上游客户端上强制（`packages/core/src/backends/external.rs`）：

| 超时 | 值 | 适用范围 |
| --- | --- | --- |
| 连接 | 10 秒 | 建立上游 TCP/TLS 连接。 |
| 读空闲 | 每次读取 120 秒 | 每次上游调用；每收到一个字节都会重置计时，因此健康但缓慢的流永远不会被切断。 |
| 非流式整体 | 600 秒 | 非流式聊天/embeddings 调用——缓慢但存活的上游不能永远占用一个请求。 |
| 流式（SSE） | 无 | 流式调用**没有整体截止时间**；长时间生成是合法的，挂起检测依赖读空闲超时。 |
| 健康 probe | 2 秒 | `/v1/models` probe。 |

## 错误映射

backend 故障在聊天/embeddings 处理器中映射为 HTTP 状态
（`packages/core/src/gateway/server.rs`）：

| 条件 | HTTP | `type` / `code` | 消息 |
| --- | --- | --- | --- |
| 上游非 2xx 状态（`UpstreamStatus`） | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| 上游传输失败（`RequestFailed`） | **502 Bad Gateway** | `server_error` / `bad_gateway` | 传输错误字符串 |
| 其他任意 backend 错误 | **500** | `server_error` / `backend_error` | 错误字符串 |
| 没有该模型的 backend（`NoBackend`） | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| 无效 API key（`Unauthorized`） | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| 限流（`RateLimited`） | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

设计意图：调用方可以区分「你的提供商拒绝或失败了」（502）与「gateway 本身
坏了」（500）。每个错误体都是相同的 OpenAI 风格形状——
`{"error":{"message":...,"type":...,"code":...}}`（`json_error_response`）。
计费门禁的 429 还额外携带 `Retry-After` 头，并分别使用
`quota_error`/`quota_exceeded`（配额）与 `rate_limit_error`/`rate_limit_exceeded`
（tier 限流）。

## 故障排查

### 新注册的 backend 在 probe 之前保持 fail-closed

External backend 从不明的健康状态开始，报告 `"<url> not probed yet"`。当
(a) 健康检查器的第一轮运行——启动时立即，之后每 60 秒——或 (b) 注册或恢复时
发起的即发即忘 probe 成功（通常约 1-2 秒内）时，它们转为 healthy。在此之前，
路由到该 backend 的请求按设计 fail closed。

### 对 probe 的 `/models` 返回 404 对某些 backend 是正常的

External probe 访问 `GET {base}/v1/models`（带路径前缀的基础 URL 则为
`{base}/models`）。有些 OpenAI 兼容服务器实现了聊天但不暴露模型列表——Zhipu
GLM 编程方案端点就是其中之一。**404 被容忍**：backend 被标记为 healthy，
管理员配置的 models 列表保持路由权威。只有真正失败的 probe（超时、网络错误、
其他非 2xx）才会把 backend 标记为 unhealthy。

### 未产生任何内容的 SSE 流不计费

流式响应只有在流产生了文本**或**携带了终止 usage 时才记录 usage；两者都没有
的流完全不被记录。如果你看到一个请求没有对应的 `usage recorded` 行，检查该流
是否真的产生了内容。

### 版本上报

健康响应体中的 `version` 是 `CARGO_PKG_VERSION`；`build_hash` 是
`packages/core/build.rs` 在构建时输出的 `BUILD_HASH` 值。跨节点比较
`build_hash` 可确认它们运行的是同一产物。

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
