---
title: "Arona"
description: "AI模型的自我部署与远程管理平台——gateway、backend、计费、agent、记忆。"
---

# Arona

**AI模型的自我部署与远程管理平台。**

Arona 是使用 Rust (axum) 编写的**纯后端**平台：它既是 OpenAI 兼容的模型
gateway，也是你所自托管模型的管理平面。它提供 `/v1/*` OpenAI 兼容 REST API、
JSON-RPC 2.0 管理平面（`/api/rpc`）、agent 控制平面（`/ws/agent`）以及位于
`/docs` 的 Swagger UI。

**不附带任何 Web 控制台和 CLI 聊天工具**——聊天与管理界面位于
[shittim-chest](https://github.com/celestia-island/shittim-chest)，
它通过 RPC 接口与 Arona 通信。Arona 专注于服务端：路由、计费、认证、模型部署、
agent 与记忆。

## 功能矩阵

| 领域 | Arona 提供的能力 |
| --- | --- |
| **对话核心** | OpenAI 兼容的 `chat.completions`（流式与非流式）、`embeddings`、`models` 列表；流式响应以 `[DONE]` 块结尾，并在最终块携带真实 usage。 |
| **Backend** | 管理员注册的上游：`external`（任意 OpenAI 兼容 HTTP API）、`ollama`、CEP `engine`（WebSocket）、`minimax-cloud` 视频，以及指向工业/边缘服务的 `evernight://` 桥接 URL。 |
| **认证** | JWT access/refresh token 对（15 分钟 / 7 天）、以 SHA-256 哈希存储的 API key（`arona-{uuid}`）、三级管理员、密码策略、双轨限流。 |
| **计费与用量** | 预置层级（free / pro / enterprise）、所有通道的每次请求 usage 记录、plana 定价表、按项目的配额范围、429 + `Retry-After`。 |
| **模型管理** | 模型下载（`hf:` / `ms:` / `gh:` 源）、`_agent` GPU 节点部署、已部署模型自动注册为可路由 backend。 |
| **实时与多模态** | 全双工 `realtime.*` 会话、`engine.invoke` 感知/控制通道、异步视频生成任务（MiniMax 云）。 |
| **Agent 集群** | GPU 节点通过 `/ws/agent` 连接、最少负载放置、会话亲和性、节点跨重启持久化。 |
| **记忆网关** | 通过 entelecheia Philia 实现长期记忆：召回注入、episode 写回、显式降级。 |
| **运维** | 健康探测、`RUST_LOG` 追踪、上游错误映射（502 与 500）、优雅停机、启动时自动迁移。 |

## 定位

Arona 是一个 **gateway + 平台**：它将模型流量路由到你的 backend，把模型部署到
你的 GPU agent，并对一切进行计量。

- 对比 **pi** —— pi 是面向模型的 CLI 助手；arona 没有 CLI 聊天。Arona 是
  pi（及其他工具）所对接的平台。
- 对比 **one-api / new-api** —— 它们是模型提供商前面的 API key 网关；
  arona 额外提供**模型部署**（下载权重并在你的 agent 上运行）、完整的管理
  RPC 平面、计费层级与记忆。
- 对比 **LiteLLM** —— 它是同级网关；arona 还拥有 gateway 背后模型的部署
  生命周期。

## 从这里开始

- [快速上手](quickstart.md) — 使用内置 mock 上游完成端到端体验。
- [配置](configuration.md) — 全部环境变量。
- [部署](deployment.md) — 裸机、systemd、Docker、进程守护。
- [Backend](backends.md) — backend 类型、URL 语义与探测。
- [OpenAI 兼容 REST API](../api/openai-rest.md) — `/v1/*`。
- [JSON-RPC API](../api/jsonrpc.md) — 完整的管理平面。

## 仓库结构

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

Web 控制台已从本仓库移除，现位于
[shittim-chest](https://github.com/celestia-island/shittim-chest)（chest #291）。

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
