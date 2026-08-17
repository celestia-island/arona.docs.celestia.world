---
title: "配置"
description: "Arona 服务器读取的全部环境变量参考，含默认值与语义。"
---

# 配置

Arona **完全通过环境变量**配置，在进程启动时读取一次（`Config::load` 位于
`packages/core/src/config.rs`，另有少量在首次使用时读取）。没有配置文件：修改
变量后重启服务器即可生效。

本页是服务器代码读取的所有配置的参考，按关注点分组。为完整性也包含了仅 mock
使用和运行时的变量。

## 参考表

| 变量 | 默认值 | 用途 |
| --- | --- | --- |
| `DATABASE_URL` | *(必填)* | PostgreSQL 连接 URL。 |
| `JWT_SECRET` | *(mock 模式外必填)* | 用于签名 JWT 的密钥。 |
| `ARONA_HOST` | `0.0.0.0` | 绑定地址（回退到 `SHITTIM_CHEST_HOST`）。 |
| `ARONA_PORT` | `8420` | 绑定端口（回退到 `SHITTIM_CHEST_PORT`）。 |
| `ARONA_DATA_DIR` | 未设置 | 本地数据目录。 |
| `ARONA_ADMIN_TOKEN` | 未设置 | `/api/admin/*` 与 admin RPC 方法的 Bearer token。 |
| `ARONA_REGISTRATION_OPEN` | `0` | truthy（`1`/`true`/`yes`/`on`）时开放注册。 |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | 每 key 每分钟的内存限流；`0` 会阻断所有请求。 |
| `MOCK_MODE` | 未设置 | 只要存在（任意值）即启用开发 mock 模式。 |
| `MOCK_SEED_PATH` | 未设置 | mock 模式下执行的原始 SQL 种子文件。 |
| `ARONA_MEMORY_URL` | 未设置 | Philia 记忆网关 WebSocket URL。 |
| `ARONA_MEMORY_TOKEN` | 未设置 | 记忆网关的 token。 |
| `ARONA_MEMORY_WRITEBACK` | `true` | 是否将完成的聊天写回记忆；接受 `true`/`false`（其他值回退到默认）。 |
| `ARONA_AGENT_NAME` | `arona-agent` | GPU 节点 agent 身份。 |
| `ARONA_PANEL_URL` | `localhost:8080` | agent 连接的地址（`ws://<panel_url>/ws/agent`）。 |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | 用于 `evernight://` backend URL 的本地 evernight agent。 |
| `ARONA_MISTRALRS` | 未设置 | 只要存在即强制 Gguf 模型方案使用 Mistral.rs 引擎。 |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | 派生的 llama.cpp 模型服务器绑定的接口。 |
| `HF_ENDPOINT` | `https://huggingface.co` | 模型下载使用的 Hugging Face 基础 URL。 |
| `GITHUB_TOKEN` | 未设置 | GitHub 模型注册表的访问 token。 |
| `RUST_LOG` | 未设置 | 追踪过滤器，例如 `info` 或 `arona=debug,info`。 |

## 必填变量

### `DATABASE_URL`

PostgreSQL 连接 URL。**必填**：为空时服务器以
`FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.` 退出，
`migrate` CLI 子命令也会拒绝运行。`serve` 在启动时自动创建/更新 schema。

### `JWT_SECRET`

用于签名 `auth.login` 和 `auth.register` 签发的 access/refresh JWT 对的密钥。
**生产环境必填**：代码内置了一个开发回退值
（`dev-secret-not-for-production-use-only-32chars`），但除非 `MOCK_MODE=1`，
否则服务器拒绝使用它启动：

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

请使用长随机值（例如 `openssl rand -hex 32`）。

## 服务器

### `ARONA_HOST` / `ARONA_PORT`

gateway 的绑定地址与端口。为兼容旧版会回退到 `SHITTIM_CHEST_HOST` /
`SHITTIM_CHEST_PORT`；最终默认值为 `0.0.0.0:8420`。

### `ARONA_DATA_DIR`

可选的本地数据目录，随应用状态传递给需要临时位置的组件。默认未设置。

## 安全与访问控制

### `ARONA_ADMIN_TOKEN`

保护 `/api/admin/*` HTTP 路由（`POST/GET/DELETE /api/admin/backends`、
`/api/admin/aliases`）以及 `billing.plan.set` / `billing.video.pricing.set`
RPC 方法的 Bearer token。当它**未设置**时，上述每个路由都以 `Admin access
required` (401) 拒绝——没有默认值。请在启动服务器前设置为强随机值。

### `ARONA_REGISTRATION_OPEN`

通过 `auth.register` 开放自助注册。truthy 值恰好是 `1`、`true`、`yes`、`on`
（不区分大小写）；其他任何值——包括 `0`、`false`、`off` 或未设置/空变量——
保持关闭。该标志在启动时读取一次。**第一个注册用户始终被允许**（即使注册关闭）
并成为管理员。

### `ARONA_API_RATE_LIMIT_RPM`

每 key 的内存滑动窗口限流（每分钟请求数），应用于每个已认证的 `/v1/*` 请求
（聊天、embeddings、视频、models），以 API key 哈希（或接受 JWT 的
`GET /v1/models` 的 `u:<email>` 标签）为键。RPC 平面不受此限流器限制——
`/v1/*` 的 auth 提取器是唯一的调用方。默认 `60`。该值解析一次存入进程级
`OnceLock`。**值为 `0` 会阻断每个请求**——检查逻辑是 `entry.len() >= rpm`，
因此为 `0` 时没有任何请求能通过。这是 gateway 级限制；billing tier 还会在此
之上施加各自的每 key 限制。

## 开发

### `MOCK_MODE`

开发 mock 模式，通过**存在性**启用——检查逻辑是
`std::env::var("MOCK_MODE").is_ok()`，因此*任意*值（包括 `0` 或设置为空）
都会启用它。它：

- 解除 `JWT_SECRET` 要求（内置开发密钥变为可接受）；
- 播种四个演示账号（`demiurge@celestia.world`、`momoi@celestia.world`、
  `midori@celestia.world`、`yuzu@celestia.world`，密码 `33550336`）；
- 在绑定监听器之前等待种子完成。

切勿在生产环境使用。

### `MOCK_SEED_PATH`

仅 mock 模式下有效，指向一个原始 SQL 文件，代替内置账号种子执行（语句按 `;`
分割，跳过 `--` 注释）。如果文件无法读取，则回退使用内置种子。

## 记忆网关

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

长期记忆网关（entelecheia Philia）的配置。除非 `ARONA_MEMORY_URL` 和
`ARONA_MEMORY_TOKEN` 都设置且非空，否则记忆**完全禁用**：召回与写回都被跳过。
启用后：

- 已完成的聊天会被召回并作为上下文注入，且
- `ARONA_MEMORY_WRITEBACK`（默认 `true`）控制是否将完成的聊天写回记忆服务；
  `0` 或 `false` 禁用写回。

记忆故障从不阻断聊天；结果状态通过 `X-Arona-Memory` 响应头回显
（`enabled` / `disabled` / `offline`）。

## Agent 身份与集群

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

GPU 节点 agent 二进制（`_agent`）的身份：`ARONA_AGENT_NAME`
（默认 `arona-agent`）作为 agent 的名称/id 上报给 panel，
`ARONA_PANEL_URL`（默认 `localhost:8080`）是 agent 连接的地址
（`ws://<panel_url>/ws/agent`）。

agent 自身的 HTTP API **硬编码**绑定 `0.0.0.0:5790`——没有对应的绑定地址
环境变量。

### `ARONA_AGENT_BIND_ADDR`

当 agent 部署 Gguf 模型时，**派生的 llama.cpp 模型服务器**绑定的接口，以便其他
机器可以访问该引擎（例如 `0.0.0.0`）。默认 `127.0.0.1`。注意这不是 agent
HTTP API 的绑定（后者固定为 `0.0.0.0:5790`）。

## Evernight 桥接

### `ARONA_EVERNIGHT_URL`

用于将 `evernight://` backend URL 解析为本地 TCP 转发的本地 evernight agent
的 WebSocket URL。默认 `ws://127.0.0.1:3001/ws`。

## 模型运行时与下载

### `ARONA_MISTRALRS`

只要存在（任意值）即强制 Gguf 模型方案使用 Mistral.rs 引擎，否则默认使用
llama.cpp。存在性语义同 `MOCK_MODE`。

### `HF_ENDPOINT`

Hugging Face 模型下载（`hf:` 源）的基础 URL，默认 `https://huggingface.co`。
当 huggingface.co 不可达时，可设为镜像如 `https://hf-mirror.com`。由模型下载器
读取；尾部斜杠会被去除。

### `GITHUB_TOKEN`

GitHub 模型注册表（`gh:` 源）用于 API 访问的访问 token。默认未设置；没有它
将受 GitHub API 限流约束。

## 日志

### `RUST_LOG`

`tracing_subscriber` 在启动时应用的标准追踪过滤器，例如 `info` 或
`arona=debug,info`。遵循常规 `RUST_LOG` 语义（`error`/`warn`/`info`/`debug`/
`trace`，支持按目标覆盖）。

## 默认值速览

| 设置 | 默认值 |
| --- | --- |
| 绑定地址 / 端口 | `0.0.0.0:8420` |
| 每 key API 限流 | 60 RPM |
| Agent 名称 | `arona-agent` |
| Panel URL | `localhost:8080` |
| 记忆写回 | 开启 |
| 注册 | 关闭 |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
