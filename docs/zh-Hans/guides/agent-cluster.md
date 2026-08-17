---
title: "Agent 集群"
description: "多节点 GPU 集群——用 CLI 下载模型权重、在 GPU 节点上运行 _agent 二进制、通过 agents.* RPC 接口驱动部署。"
---

# Agent 集群

Arona 的部署故事分为两半。**panel**（`arona` 服务器二进制）拥有路由、计费、
认证和管理平面。每个 GPU 节点运行一个 **`_agent` 进程**，它拥有模型权重和本地
服务进程。Agent 与 panel 的 agent 控制平面（`/ws/agent`）建立一条长连接
WebSocket；panel 通过该 socket 推送 `deploy` / `stop` 命令，agent 将下载进度、
心跳和命令结果流式回传。一旦模型在 agent 上运行，panel 就把它注册为可路由
backend，使 `/v1/*` 和 RPC 流量能到达它——控制平面走 WebSocket，数据平面是
到 agent 本地引擎端口的普通 HTTP。

## 下载模型权重（CLI）

`_cli` 二进制从 HuggingFace、ModelScope 或 GitHub releases 下载模型权重：

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **Repo 形式** —— `hf:owner/repo`（默认；裸 `owner/repo` 也解析为
  HuggingFace）、`ms:owner/repo`（ModelScope）、`gh:owner/repo[:tag]`
  （GitHub release，tag 可选）。长前缀 `huggingface:`、`modelscope:` 和
  `github:` 同样接受；不带斜杠的裸 id 解析到 Ollama 注册表
  （`packages/core/src/models/download.rs:21-28,55-86`）。
- **`--filter <glob|prefix>`** —— 可重复；只下载匹配 glob（或前缀）的清单文件。
  不带 filter 时选择**整个 repo**。
- **确认** —— 未过滤的下载在开始前总是询问 `Continue? [y/N]`，除非传了
  `--yes`。过滤后的下载从不提示；当选中的总量达到或超过 2 GiB 时，改为打印一条
  信息横幅（`NO_CONFIRM_THRESHOLD`、`packages/cli/src/main.rs:12-15, 439-464`）。
- **`--out <dir>`** —— 覆盖默认目标 `~/.arona/models/<repo-id>`。
- **`--revision <rev>`** —— 覆盖任何内联的 `:rev` 后缀
  （`hf:owner/repo:rev`）。
- **续传** —— 中断的下载自动续传：保留 `.part` 文件，通过 Range 请求从其当前
  长度继续；完整文件按大小跳过，清单携带摘要时还会做 SHA-256 校验
  （`packages/cli/src/main.rs` 的 `verify_sha256`/`summarize`）。
- **重试** —— 网络错误最多重试 3 次，间隔 5 秒；非网络错误立即失败
  （`packages/cli/src/main.rs:277-302`）。
- **`HF_ENDPOINT`** —— 切换 HuggingFace 基础 URL，例如镜像：

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

其他 CLI 命令（`packages/cli/src/main.rs:28-53`）：

| 命令 | 用途 |
| --- | --- |
| `install` | 一键环境设置：检测硬件配置并打印 backend / 量化建议。 |
| `status` | 打印硬件配置。 |
| `deploy <model>` | 本地解析模型并报告是否已缓存。 |
| `download` | 下载模型权重（上文）。 |
| `serve` | 启动 API 服务器（panel）。 |
| `connect <url>` | 连接到管理 panel。 |
| `migrate` | 运行数据库迁移。 |

## `_agent` 二进制

`_agent` 在每个 GPU 节点上运行，完全由环境变量配置
（`packages/core/src/config.rs:37-40`）：

| 变量 | 默认值 | 含义 |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | 唯一节点 id；panel 将其用作 `agent_id`。 |
| `ARONA_PANEL_URL` | `localhost:8080` | Panel 的 `host:port`；agent 连接到 `ws://{ARONA_PANEL_URL}/ws/agent`。 |

完整的环境变量参考见 [配置](./configuration.md)（panel 侧变量、数据库、密钥）。

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

行为：

- **控制连接** —— agent 回连到 `ws://{ARONA_PANEL_URL}/ws/agent`
  （`packages/agent/src/panel.rs:23`）。连接时发送一个携带 `agent_name`、
  `gpu_info` 和已部署模型列表的 `register` 帧；panel 将 agent 的 TCP 对端地址
  记录为其 `host`。
- **重连退避** —— 从 1 秒开始，成倍增长到 60 秒上限
  （`packages/agent/src/panel.rs:27,33-34,63`）。
- **心跳** —— 每 30 秒 agent 上报 GPU 利用率、已加载模型数和运行时长。当
  agent 最近一次心跳超过 30 秒时，panel 认为其离线。
- **本地 HTTP API** —— 绑定**固定**地址 `0.0.0.0:5790`；没有绑定地址环境变量
  （`packages/agent/src/main.rs:109`）。panel 将该端口与 agent 记录的 host
  组合，构建已部署模型的数据平面 URL。
- **命令** —— panel 通过 socket 排队 `deploy` / `stop` 命令。`deploy` 命令
  携带 `model_id` 和 `stream_id`；下载进度以 `deploy_progress` 帧在同一
  socket 上流式回传（panel 将其转发为 `models.progress` SSE 通知，见
  [事件与通知](../api/events.md)），最后的 `deploy_result` 帧报告本地引擎的
  `port` 和 `backend`。`stop` 以 `stop_result` 应答。

在服务守护程序（systemd、malkuth 等）下运行 `_agent`，使其能自动重连；panel
容忍任意一侧的重启（见下文 [节点持久化](#node-persistence)）。

## Agent 控制平面 RPC

整个 agent 接口都有 admin 门禁：每个方法都需要有效 JWT **且**是 admin 账号
（`validate_admin_jwt` 检查 `is_admin_email`；
`packages/core/src/gateway/rpc.rs:106-118,301-337`）。

| 方法 | 参数 | 返回 |
| --- | --- | --- |
| `agents.list` | — | 集群拓扑：`id`、`name`、`host`、`status`（`online`/`offline`）、GPU 摘要、`models`、`last_heartbeat`、`version`、`connected_at`。 |
| `agents.register` | `machine_name`、`version` | `agent_id`、`token`。 |
| `agents.deregister` | `agent_id` | `{ "ok": true }` —— 移除该节点。 |
| `agents.status` | `agent_id` | `online`、GPU 摘要、`gpu_utilization`、`models`、`host`、`connected_at`、`last_heartbeat`。 |
| `agents.deploy` | `model_id`、`agent_id?` | `{ "ok": true, "stream_id" }` —— 空的 `agent_id` 自动选择最少负载节点。 |
| `agents.stop` | `agent_id`、`model_id` | `{ "ok": true }` —— 停止该部署。 |

`agents.deploy` 返回一个 `stream_id`；在调用**之前**或紧随其后订阅
`/api/rpc/events?session=<stream_id>`，以接收 `models.progress` 下载通知
（见 [事件与通知](../api/events.md)）。传输与认证细节见
[JSON-RPC API](../api/jsonrpc.md)。

## 已部署模型的自动注册

当 `deploy_result` 帧报告成功时，panel 在 gateway 路由器中注册一个名为
**`agent-{model_id}`** 的 `ExternalApiBackend`，基础 URL 为
`http://{agent-host}:{port}`——agent 记录的 host 加上它报告的引擎端口
（`packages/core/src/gateway/server.rs:310-366`、
`packages/core/src/gateway/mod.rs:253-270`）。已部署的模型成为普通可路由
backend：`/v1/chat/completions`、embeddings 和 RPC 聊天都能到达它，别名适用，
健康检查器会 probe 它（backend 类型与探测语义见 [Backend](./backends.md)）。

- 重新部署同一模型（例如在另一个 agent 上）会替换之前的 backend。
- 成功的 `stop_result` 会再次注销它（`packages/core/src/gateway/mod.rs:274-287`）；
  该模型 id 停止解析。

## 放置

没有显式 `agent_id` 的部署走最少负载放置
（`packages/core/src/gateway/tunnel.rs:214-229`）：候选是最近心跳在 30 秒内的
agent，选择**平均 GPU 利用率最低**的那个。没有遥测的 agent 排在最后但仍可被
选中。如果没有 agent 在线，RPC 以 `No online agents available for deployment`
失败。

在路由侧，会话被**固定到一个 backend**（会话亲和性）：第一个服务该会话的
backend 被记录并在后续轮次复用，因此按会话的状态（如运行时的 KV 缓存）保持
温热（`packages/core/src/routing/mod.rs:31-34,110-138`）。如果固定的 backend
变 unhealthy，路由降级为一次新的选择并重新固定。

## 节点持久化

Agent 节点持久化在 `agent_nodes` 表中（`agent_id`、`machine_name`、
`version`、`host`、`gpu_info`、`models`、`connected_at`、`last_heartbeat`；
`packages/core/src/gateway/tunnel.rs:8-12`）。panel 启动时恢复已持久化的行，
使先前注册的节点在重启后依然可见；恢复的条目在各自 agent 通过 WebSocket
重新连接之前是**无发送方**的（`packages/core/src/gateway/run.rs:74-85`）。
因此，向已恢复但断开的节点部署会失败，直到其 `_agent` 重新连接。

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
