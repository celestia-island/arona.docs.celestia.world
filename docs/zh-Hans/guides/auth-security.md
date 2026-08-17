---
title: "认证与安全"
description: "JWT 会话、API key、三道管理员门禁、密码策略、双轨限流与安全模型。"
---

# 认证与安全

Arona 在两条轨道上认证调用方：面向交互客户端（聊天 + 管理 UI、RPC 调用）的
**JWT 会话 token**，以及面向程序化 OpenAI 兼容流量的 **API key**
（`arona-…`）。一个独立的 admin token 守护管理面。本页记录机制、安全模型，以及
安全审计中已知的低风险遗留项。

## JWT 会话

会话使用 `kirino_session` token 管理器签发的 JWT access/refresh token 对：

- **Access token TTL：900 秒（15 分钟）。**
- **Refresh token TTL：604,800 秒（7 天）。**

Access token 认证 JSON-RPC 平面（`/api/rpc`）和 `GET /v1/models`；SSE 旁路
（`/api/rpc/events`）以会话 id 为键，这是认证 RPC 调用期间铸造的能力，而不是
bearer 凭据。`/v1/chat/completions`、`/v1/embeddings` 和 `/v1/video/*`
端点要求 **API key**（那里不接受 JWT）。Access token 生命周期很短，因此被盗的
token 只能用一小段时间。Refresh token 通过 `auth.refresh` 兑换成新的token 对。

刷新使用 **token 家族轮换**：消费一个 refresh token 会使它失效并签发新的，
而重放已消费的 refresh token 会撤销整个家族——`auth.refresh` 以 `AUTH_ERROR`
应答，消息为 `Refresh token reused`（底层错误为 `TokenReused`，"refresh token
has been reused — token family revoked"），账号必须重新登录。家族撤销是
**内存态**的（一个 `revoked_families` 集合）：服务器重启会清空它，因此该保护
跨重启只是尽力而为（按用户的会话状态在重启后不保留）。

签名密钥来自 `JWT_SECRET` 环境变量。在 `MOCK_MODE=1` 之外，如果 `JWT_SECRET`
未设置或仍等于内置开发密钥，服务器**拒绝启动**，因此生产实例永远不会意外地用
公开常量签发 token。请使用强随机密钥，并且绝不提交它。

## API key

API key 是 OpenAI 兼容面的机器凭据：

- **格式：** `arona-{uuid}`。
- **存储：** `api_keys` 表只存储 key 的 **SHA-256 哈希**——明文只在
  `keys.create` 响应中返回一次，之后永远无法恢复。
- **Key 前缀：** 前 8 个字符（`key_prefix`）以明文存储，用于展示与用量归属；
  UI 显示如 `arona-XXXX…abcd` 的掩码形式。
- **撤销：** key 查找会 join `api_keys.is_active = TRUE`，因此被撤销的 key
  会立即停止通过校验——没有需要等待的缓存 TTL。

## 管理员层级

有三道不同的管理员门禁，各有自己的凭据：

1. **`/api/admin/*` 路由** —— backend 与别名管理
   （`POST/GET/DELETE /api/admin/backends`、`POST/GET/DELETE /api/admin/aliases`）
   要求 `Authorization: Bearer ARONA_ADMIN_TOKEN` 头。当 `ARONA_ADMIN_TOKEN`
   未设置时，`check_admin` 总是失败，每个 admin 路由都返回
   **401 "Admin access required"**——整个管理面是被禁用而非开放。

2. **`agents.*` 和 `engine.invoke` RPC 方法** —— agent 集群与引擎控制平面要求
   一个账号 `users.is_admin = true` 的 JWT。已认证的非管理员以实现定义的错误码
   **-32007（`ADMIN_REQUIRED`）** 加方法专属提示被拒绝
   （例如 `agents.deploy starts model deployments on GPU nodes`）；**未认证**
   的调用方得到标准的 **-32005（`AUTH_ERROR`）**，服务器不会暴露该方法是有
   特权的。

3. **`billing.plan.set` 和 `billing.video.pricing.set` RPC 方法** —— 计费变更
   要求与 admin HTTP 路由相同的 Bearer `ARONA_ADMIN_TOKEN`；没有它则返回
   `AUTH_ERROR` "Admin access required"。

**第一个注册用户成为管理员**（`users.is_admin = true`）。此后的每次注册都是
普通用户，并且只有在 `ARONA_REGISTRATION_OPEN` 设置为 truthy 值时注册才开放。

## 密码策略

密码必须同时满足**两条**规则（在注册和任何改密路径上强制）：

- 至少 **8 个字符**，且
- **4 个字符类别中的至少 3 类**：大写、小写、数字、特殊字符。

## 限流

限流在两条独立轨道上运行；任一条都可以用 **429** 拒绝请求：

### 1. 内存滑动窗口（按身份）

每个已认证的 `/v1` 请求都经过一个以调用方身份为键的内存滑动窗口限流器：

- **API key 调用** 以 key 的 **SHA-256 哈希**为键；
- **JWT 调用** 以 `u:<email>` 为键——JWT 每 15 分钟轮换一次，因此按 token
  实例给窗口设键会在每次刷新时静默重置。

默认预算为**每分钟 60 个请求**，可用 `ARONA_API_RATE_LIMIT_RPM` 覆盖
（对需要扇出大量并行 LLM 调用的 agent 管道可以调高）。设为 **0 会阻断每个
请求**。

### 2. Tier 限流（每 key，来自数据库）

Billing tier 携带每 key 的 `rate_limit_rpm`。检查会统计该 key 前缀在
**最近 60 秒**内的 `usage_records` 行数（usage 在每次响应后持久化，因此窗口
最多滞后一个进行中的请求；数据库故障时 fail open）。预置的 **free tier 为 10
RPM**；pro/enterprise tier 提高上限。月度配额执行共用同一条拒绝路径。

### 登录限流

登录端点会抑制凭据猜测：**每个 email 每 5 分钟窗口 5 次失败尝试**、**每个 IP
每 5 分钟窗口 20 次**，每次之后有 15 分钟的锁定期。

### `Retry-After`

每个 429 响应都携带 `Retry-After` 头，让 OpenAI 兼容客户端退避而不是猛打
端点：配额拒绝将其设为**距月末的秒数**；限流拒绝将其设为 **60**。配额模型见
[计费与用量](billing-usage.md)。

## 安全模型说明

- **CORS 允许任意来源**（`allow_origin(Any)`）——Arona 是被许多第一方和第三方
  客户端消费的 backend；如果你的部署必须限制来源，请在前面加一个执行 CORS 的
  反向代理。
- **请求体限制为 1 MB**（`RequestBodyLimitLayer`），约束 gateway 上的内存使用。
- **gateway 不终结 TLS**——它监听纯 HTTP。请把它放在终结 HTTPS 的反向代理
  后面（见 [部署](deployment.md)）。
- **密钥只来自环境**：`ARONA_ADMIN_TOKEN` 和 `JWT_SECRET` 从环境变量读取，
  必须是永不提交到仓库的强随机值。
- 默认服务器绑定地址为 `0.0.0.0`；在网络层限制暴露面。

## 已知低风险遗留项（来自审计）

以下内容按原样记录；它们是有意为之或暂时接受的，但在把实例暴露到可信网络之外
时值得了解：

- **`providers.list` 是公开的**，而 `providers.add` / `providers.update` /
  `providers.remove` / `providers.test` 需要 JWT。公开读路径会暴露 provider
  目录，但没有任何秘密。
- **`/ws/agent` 是一个未认证的控制平面**：GPU agent 不带凭据连接并自行注册
  （`register` / `heartbeat` / 命令结果帧）。任何能到达 WebSocket 端口的人
  都能注册一个假 agent。运维权衡见 [Agent 集群](agent-cluster.md)。
- **`memory.delete` 仅需 JWT 且无所有权检查**：任何已认证用户都能按
  `node_id` 删除记忆节点。删除记忆需要登录，但不需要拥有该节点。

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
