---
title: "测试"
description: "Arona 测试金字塔——单元测试、封闭式集成、PostgreSQL 门控集成、活服务器冒烟测试、mock 服务器以及真实凭据冒烟纪律。"
---

# 测试

Arona 的测试分层排列，使默认的 `cargo test` 运行快速、封闭，既不需要数据库也
不需要网络，而更重的测试套件是显式选择加入的，会真实地演练线上接口和一个真实
PostgreSQL。本页介绍各层、运行它们的命令，以及围绕真实凭据冒烟运行的工作区
纪律。

## 单元测试

大部分覆盖率是 `packages/core/src` 内的普通单元测试：217 个 `#[test]` /
`#[tokio::test]` 函数，加上 `packages/agent` 和 `packages/cli` 中约 23 个。
运行方式：

```bash
cargo test --workspace
```

无网络、无数据库。关键套件：

- **auth.rs** —— 密码策略（≥8 字符且 4 个字符类别中 ≥3 类）、原始
  INSERT/REVOKE SQL 中的 `::uuid` 转换、请求默认值，以及回退到 `false` 的
  admin 标志读取。
- **billing/mod.rs** —— 成本*或* token 维度上的配额计算、月度窗口
  （`month_start`、`seconds_until_month_end`）、限流上限（只在 *达到* RPM 时
  触发，`None` = 无限制）、月度用量 / tier / key 窗口查询的 SQL 形状守卫，
  以及优先采用上游报告数字的 `estimate_usage`。
- **routing/mod.rs** —— 别名解析、`:latest` 后缀匹配、provider 提示、
  最少负载选择与会话固定。
- **gateway/mod.rs** —— agent backend 注册：注册 `agent-{model_id}`、重新
  注册替换（而非重复）、注销恢复路由器。

## 封闭式集成（始终运行、无需数据库）

`packages/core/tests/gateway_integration.rs` 包含三个始终运行的测试，在不接触
数据库的情况下演练真实的序列化/契约逻辑：

- **A1** —— JSON-RPC id 回显序列化契约：数字、字符串和 null 请求 id 通过
  plana 的 `Id` 枚举往返，保持类型保真。
- **A2** —— admin 门禁错误码契约：`AUTH_ERROR`（-32005，匿名）和
  `ADMIN_REQUIRED`（-32007，已认证非管理员）保持区分、位于实现定义的区间，
  并且永不与 plana 的错误码或计费的 `QUOTA_ERROR`（-32006）冲突。
- **A3** —— `estimate_usage`：上游报告的 usage 原样优先；没有时，本地
  分词器估算产生非零的 prompt/completion 计数，其总和为 total。

`packages/core/tests/smoke.rs` 另外添加三个始终运行的测试：硬件检测、模型
注册表根路径，以及 `MOCK_MODE=1` 下的配置默认值。

## PG 门控集成

完整的进程内 gateway 套件——`packages/core/tests/gateway_integration.rs`——
在随机回环端口上启动完整的 axum 路由器，通过真实 admin API 注册一次性的
OpenAI 兼容 mock 上游，并用 reqwest 驱动线上接口。由于 `AuthManager` 在每条
路径上都与 PostgreSQL 通信（即使 `MOCK_MODE=1` 也只是把账号播种*进数据库*），
该套件由 `ARONA_TEST_PG=1` 门控，默认跳过。这 10 个测试：

- **T1** 注册 + 登录 + `keys.create`/`keys.list`（列表中原样掩码 key 的
  `arona-` 前缀）。
- **T2** REST 聊天并持久化 usage 记录到 PostgreSQL。
- **T3** 线上 JSON-RPC id 回显（成功与错误路径）。
- **T4** `agents.list` 上的 admin 门禁：匿名 → `AUTH_ERROR`，非管理员 →
  `ADMIN_REQUIRED`。
- **T5** 上游 401 → HTTP 502 `bad_gateway` 及上游详情。
- **T6** 注册时 probe 发布模型（无需静态模型列表，模型在 10 秒内出现在
  `GET /v1/models`）。
- **T7** 通过 `chat.send` 的会话持久化（两轮都落入 `conversations.get`）。
- **T8** free-tier 计费门禁：每 key 10 RPM，窗口内第 11 个请求是 429
  `rate_limit_exceeded`。
- **T9** SSE 流，终止 usage 从上游记录。
- **T10** 畸形 JSON → 400；未知模型 → 404 `model_not_found`。

用模块文档中的一次性 PostgreSQL 一行命令运行它（gateway_integration.rs:18-26）：

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

这些只是一次性测试容器的示例凭据——绝不要把它指向真实数据库。

## 活服务器冒烟

`packages/core/tests/auth_flow.rs` 针对**正在运行**的 Arona 服务器走完
`register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` 完整链路，镜像已部署的认证循环。它默认 `#[ignore]`——普通的
`cargo test` 运行从不接触网络。显式运行：

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

旋钮：

- `ARONA_TEST_URL` —— 基础 URL（默认 `http://127.0.0.1:8420`）。
- `ARONA_TEST_EXPECT_CHAT=1` —— 硬断言 `POST /v1/chat/completions` 返回
  200。没有它测试只断言认证通过（不是 401/403），因为目标环境可能没有配置
  推理提供商。

该套件还包含负面测试：未认证的聊天补全和未认证的 `GET /v1/models` 都必须以
401 被拒绝。

## Mock 服务器

`scripts/mock/server.py` 是一个基于 aiohttp 的 OpenAI 兼容假服务器，用于
快速上手和冒烟运行。它提供 `POST /v1/chat/completions`（非流式与 SSE）、
`GET /v1/models`、`GET /api/health`、位于 `/api/rpc` 的 JSON-RPC
WebSocket/HTTP 面、位于 `/api/rpc/events` 的 SSE 旁路，以及返回 mock API key
的 `GET /api/test-key`，方便其他服务发现它。默认监听 8429 端口（可用
`ARONA_MOCK_PORT` 覆盖端口，`ARONA_MOCK_HOST` 覆盖主机）。
[快速上手](quickstart.md) 用它搭建无需真实模型提供商的端到端环境。

## 真实凭据冒烟纪律

针对真实提供商（DeepSeek / GLM）的冒烟运行刻意**不是**仓库测试——它们需要
真实凭据和真实资金，因此不能进入 CI 或 git 树。记录在 gateway_integration
模块文档（gateway_integration.rs:54-55）中的工作区约定是：

- 证据文件位于 `/mnt/work/arona-pr*-smoke.md` —— 工作区本地文件，永不提交
  到 git。
- 凭据只来自环境；预算保持小额。
- 每个触及推理路径的 PR 都要有一份书面证据记录。

mock 服务器在 CI 和本地开发中充当这些运行的替身；真实凭据冒烟是发布时的人工
步骤。

## CI

`.github/workflows/ci.yml` 在组织自托管 runner
（`[self-hosted, linux, x64, local]`）上运行 `cargo fmt`、`cargo clippy`、
`cargo test --workspace` 和 `cargo-deny`；`ci-hosted.yml` 在 GitHub 托管
runner 上镜像同样的检查。`.github/workflows/docs.yml` 用 lagrange 构建本站点，
并在触碰 `docs/**` 的 push 时部署到 GitHub Pages。

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
