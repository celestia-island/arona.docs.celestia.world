---
title: "快速上手"
description: "使用内置 mock 上游完成 Arona 端到端体验：迁移、serve、注册 backend、创建 API key 并聊天。"
---

# 快速上手

本指南带你在一台机器上完成一次完整的 Arona 端到端部署，使用**内置 mock 上游**
——不需要真实模型权重、GPU 或外部 API 账号。完成后你将拥有：

- 一个正在运行的 Arona gateway（`/v1/*` OpenAI 兼容 REST API，外加
  `/api/rpc` JSON-RPC 管理平面），
- 注册为 `external` backend 的 mock 上游，
- 一个用户账号和一个 API key，
- 一次针对 mock 的非流式**和**流式聊天，
- 通过 `usage.list` 可见的 usage 记录。

## 前置条件

- **Rust 工具链**（参见仓库根目录的 `rust-toolchain.toml`）。
- 带 `aiohttp` 的 **Python 3**——仅 mock 上游需要
  （`pip install aiohttp`）。
- 一个**正在运行的 PostgreSQL** 实例及其连接 URL。

## 1. 设置环境变量

Arona 在**进程启动时**从环境变量读取配置。有两个是必填的：`DATABASE_URL` 和
`JWT_SECRET`——没有它们服务器拒绝启动（除非 `MOCK_MODE=1`）。强烈建议设置
`ARONA_ADMIN_TOKEN`：没有它，每个 `/api/admin/*` 路由都会返回 401。

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

这些变量在进程启动时只读取一次——如果修改它们，请重启服务器。完整变量参考见
[配置](configuration.md)。

## 2. 迁移并启动服务器

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

对于全新的数据库，仅 `serve` 就足够了：它在启动时自动迁移。服务器默认绑定
`0.0.0.0:8420`（可用 `ARONA_HOST` / `ARONA_PORT` 覆盖）。

## 3. 启动 mock 上游

在第二个终端中：

```bash
python3 scripts/mock/server.py
```

mock 是一个 aiohttp 服务器，默认监听 `127.0.0.1:8429`
（`ARONA_MOCK_PORT` 可覆盖端口）。它在启动时打印自己的 API key，同时提供
`GET /api/test-key`，返回 `{"api_key": ..., "base_url": ...}`。它暴露了少量
模型 id——包括下文用到的 `gpt-5.5`——并同时应答普通与流式聊天补全。

记下打印的 key：

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. 将 mock 注册为 external backend

Backend 通过 admin HTTP API 注册：

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

backend 在注册时会被立即 probe，约 1-2 秒内转为 healthy；在该 probe 完成之前，
它保持在 fail-closed 的「尚未 probe」状态（见下方故障排查框）。配置会被持久化，
因此 backend 在重启后依然存在。

## 5. 注册账号并登录

账号位于 JSON-RPC 平面，`POST /api/rpc`。由于设置了
`ARONA_REGISTRATION_OPEN=1`，`auth.register` 是开放的；第一个注册的用户将成为
管理员。

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

密码必须至少 8 个字符**且**包含 4 个字符类别（大写、小写、数字、特殊字符）中的
至少 3 类。然后登录以获取 JWT 对：

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

导出响应中的 `access_token`：

```bash
export JWT="<access_token from the login response>"
```

## 6. 创建 API key

`keys.create` 使用 JWT 认证，并且**只返回一次**完整的 `arona-{uuid}`
密钥——数据库只存储它的 SHA-256 哈希，所以请立即复制：

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. 聊天（非流式）

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

你会得到一个 OpenAI 风格的补全对象，包含 `choices[0].message` 和
`usage` 块。

## 8. 聊天（流式）

同一端点加上 `"stream": true` 后以 server-sent events 应答：每个 token 一个
`data:` 块，以最终的 `data: [DONE]` 块结束：

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. 验证 usage

每次聊天都会在该 key 的前缀下记录一行 usage。用 JWT 查询：

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

你应该能看到上面 `gpt-5.5` 请求对应的一条或多条记录。

## 故障排查

- **`No backend available for model: gpt-5.5`（HTTP 404，`code:
  model_not_found`）** —— 没有注册的 backend 提供该模型 id。要么 backend
  从未注册（或其 `models` 列表不包含该 id），要么注册调用失败。用
  `GET /api/admin/backends`（admin token）检查。
- **`all backends unhealthy`（HTTP 500，`backend_error`）** —— 确实有
  backend 注册了该模型，但没有一个候选是 healthy 的。新注册的 external
  backend 从 fail-closed 的「尚未 probe」状态开始，一旦注册时的 probe 完成
  （约 1-2 秒后）即转为 healthy；如果你在这个窗口内聊天就会遇到此错误。
  稍等片刻重试，或确认 mock 确实在 `127.0.0.1:8429` 上运行。
- **`/v1/*` 返回 HTTP 401** —— 缺少 `Authorization` 头会产生
  `Missing Authorization header. Use: Bearer <api_key>`；未知 key 会产生
  `Invalid API key`。请仔细检查 `$AR_KEY`（完整密钥，不是前缀）。
- **`/api/admin/*` 返回 HTTP 401 `Admin access required`** —— bearer token
  与 `ARONA_ADMIN_TOKEN` 不匹配，或该变量未设置（此时路由总是拒绝）。
  设置后重启服务器。
- **`auth.register` 失败并提示 "Registration is closed"** —— 当
  `ARONA_REGISTRATION_OPEN` 不为 truthy 时注册被禁用。请在启动服务器
  **之前**设置 `ARONA_REGISTRATION_OPEN=1`（它在启动时读取），或成为第一个
  用户——第一个注册的用户总是被允许并成为管理员。
- **HTTP 429 限流** —— 三个独立的限制可能触发：
  - 每 key 的内存限制，默认 60 RPM
    （`ARONA_API_RATE_LIMIT_RPM`）→ `Rate limit exceeded. Try again later.`；
  - free billing tier 的每 key 10 RPM 限制 → 429 并带
    `Retry-After: 60` 头；
  - free tier 的每月 $1 / 100k-token 配额 → 429，`Retry-After`
    指向下一个计费周期。

## 下一步

- [配置](configuration.md) — 全部环境变量。
- [Backend](backends.md) — backend 类型、URL 语义与探测。
- [部署](deployment.md) — 裸机、systemd、Docker。
- [OpenAI 兼容 REST API](../api/openai-rest.md) — 完整的 `/v1/*` 接口。
- [JSON-RPC API](../api/jsonrpc.md) — 上面用到的管理平面。

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
