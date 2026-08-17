---
title: "计费与用量"
description: "usage 记录、按模型成本、billing tier、配额与限流执行、按项目的 key、视频定价与 usage.list RPC。"
---

# 计费与用量

Arona 计量每个模型请求，并在 gateway 上执行分层配额与限流。按模型价格来自共享
的 plana 定价表（绝不在 arona 中重复实现），usage 以 `usage_records` 行持久化，
整个月度全景通过 `usage.list` RPC 暴露。

## Usage 记录

每个被计量的请求最终都是 `usage_records` 表中的一行
（`m20250101_000006_create_usage_records`）：

| 列 | 类型 | 含义 |
| --- | --- | --- |
| `id` | `UUID` | 主键，自动生成。 |
| `api_key_id` | `VARCHAR(64)` | **key 前缀**——API key 的前 8 个字符（key 形如 `arona-{uuid}`）——或 JWT 归属的 RPC 通道的合成 `jwt-<user-uuid>` id。 |
| `model` | `VARCHAR(128)` | 请求路由到的模型 id。 |
| `backend` | `VARCHAR(64)` | Backend kind：`gateway`、`rpc`、`realtime` 或 backend 能力名称。 |
| `prompt_tokens` | `INTEGER` | 输入 token，上游报告或估算。 |
| `completion_tokens` | `INTEGER` | 输出 token，上游报告或估算。 |
| `total_tokens` | `INTEGER` | 两者之和。 |
| `cost` | `DOUBLE PRECISION` | 计算出的 USD 成本；模型没有定价行时为 `NULL`。 |
| `created_at` | `TIMESTAMPTZ` | 请求完成的时间。 |

`api_key_id`、`model` 和 `created_at` 上有索引（月度聚合和限流窗口扫描的列）。

## 记录通道

usage 在每个被计量的通道上记录：

1. **REST 非流式** —— `POST /v1/chat/completions` 和
   `POST /v1/embeddings` 在响应生成后记录上游报告的确切 usage。
2. **REST 流式（SSE）** —— 当流携带上游报告的 usage 时以其为准（OpenAI
   兼容的终止块 `usage` 字段）；否则记录本地 CJK 感知分词器估算
   （`estimate_usage`）的原值。既没有文本也没有 usage 的流**完全不被**记录。
3. **RPC `chat.send`** —— 同样的估算与上游对比逻辑；行以合成的
   `jwt-<user-uuid>` id 归属，从而能 join 回用户。
4. **实时会话** —— 每个完成的 `response_done` 转录记录其 token usage
   （非零时），归属 `jwt-<user-uuid>`，backend 为 `realtime`。
5. **视频任务** —— 完成的任务记录一个明确的美元成本（见
   [视频定价](#video-pricing)）；token 计数为零。

记录是尽力而为的：失败的插入只记日志，绝不会让请求失败。

## 成本计算

成本根据规范的每 1M-token 定价表计算
（`plana_llm_provider::metering::lookup_pricing`，所有 celestia-island
服务共享）：

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

表中的模型匹配基于小写化模型 id 的子串（更具体的家族优先）。当模型没有定价行
时，`cost` 为 `NULL`。**不要在 arona 中重复实现定价——更新 plana 的表。**

## 层级

Tier 位于 `billing_tiers` 表中，首次迁移时播种
（`m20250101_000007_create_billing_tiers`）。`NULL` 配额列表示该维度「无限制」。
没有 `tier_id` 的用户回退到播种的 `free` tier。

| Tier | 月度 USD 配额 | 月度 token 配额 | 每 key RPM |
| --- | --- | --- | --- |
| `free` | $1.00 | 100,000 | 10 |
| `pro` | $20.00 | 5,000,000 | 120 |
| `enterprise` | 无限制（`NULL`） | 无限制（`NULL`） | 1000 |

Tier 分配是管理员操作（`billing.plan.set` RPC）；当前 tier 与用量通过
`billing.plan` 呈现。

## 配额与限流执行

### REST（`/v1/*`）

在每个**被计量**的 REST 端点——`/v1/chat/completions`、
`/v1/embeddings` 和 `/v1/video/generations`——之前，gateway 对 key 认证的
请求执行两道门禁：

- **月度配额** —— 对照自当前自然月第一刻以来累计的用量检查 tier 的
  `monthly_quota_usd` 和 `quota_tokens` 限制。任一维度达到上限即触发门禁。
- **每 key 限流** —— 对照该 key 前缀在最近 60 秒窗口内记录的请求数检查 tier
  的 `rate_limit_rpm`。（`/v1/models` 和健康端点不被计量、不受门禁。）

拒绝是 HTTP **429 Too Many Requests**，带 `Retry-After` 头和 OpenAI 风格
错误体：

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| 拒绝 | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| 月度配额耗尽 | `quota_error` / `quota_exceeded` | 距**下个自然月**开始的秒数 |
| Tier 限流超限 | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

JWT 认证的 `chat.send` 经过同样的月度配额门禁，但针对的是**整个用户**窗口
（该调用不携带 API key）。拒绝是 JSON-RPC 错误，错误码为实现定义的 `-32006`
（`QUOTA_ERROR`），消息与 REST 配额拒绝相同。RPC 路径上没有每 key 限流——
限流是 key 范围的，而 RPC 调用没有 key。实时和视频 **RPC** 方法不受配额门禁。

## Fail-open 权衡

计费**设计上就是尽力而为**。如果配额或限流检查背后的数据库查询失败，检查返回
`Unknown`，请求被**放行**（仅记录日志）而不是阻断聊天。运维人员可以依赖 429
来保护容量，但当数据库不健康时不能将其视为硬性保证——记录在案的权衡是优先
保证聊天路径的可用性，而非严格计量。

## 按项目的 key

API key 可以带 `project` 标签创建（`api_keys.project`，未设置时为
`default`）。配额执行会尊重它：

- 带非 `default` 项目标签的 key 对照**该项目自己的桶**归属的用量检查配额
  （`PROJECT_MONTHLY_USAGE_SQL`）。
- `default` / 无标签的 key 保持**整个用户**窗口，与项目功能之前的行为一致。

JWT 归属的 RPC 行（`jwt-<user-uuid>`）不携带项目标签，并且**有意排除**在项目
窗口之外——它们仍然计入整个用户窗口，因此无法通过 RPC 通道发送流量来「隐藏」
某个项目。

## 视频定价

视频生成使用模型专属、任务式的定价（按 token 定价对视频没有意义）。定价行位于
`video_pricing` 表；没有配置行时 `compute_cost` 回退到占位默认值。

| 模式 | 成本 |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution`（默认） | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` 是一个以短边像素值为键的 JSON 对象（例如 `"768"`）；`"*"`
键是回退值。默认定价行为模式 `per_second_resolution`、`base_price` 0.0、
`price_per_second` 0.005、`resolution_coeff {"*": 1.0}`。行通过
`billing.video.pricing.get`（任意 JWT）和 `billing.video.pricing.set`
（Bearer `ARONA_ADMIN_TOKEN`）管理；任务完成时计算出的成本记录到该任务的
key 名下。

## usage.list

`usage.list`（JWT）返回调用方分页的 usage 记录，涵盖**两种**行：API key 行
（通过 key 前缀 join）和 JWT 归属行（通过合成的 `jwt-<user-uuid>` id join），
最新的在前：

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `limit` | `50` | 限制在 `1..=200`。 |
| `offset` | `0` | 分页偏移。 |
| `project` | 未设置 | 设置时只返回归属到带该项目标签的 key 的记录（JWT 行被排除）。 |

响应为 `{ "records": [...], "total", "limit", "offset", "project" }`，其中每条
记录携带 `id`、`model`、`backend`、`prompt_tokens`、`completion_tokens`、
`total_tokens`、`cost` 和 `created_at`。月度配额聚合使用相同的 join 形状，
因此配额计算与 usage 视图在范围上始终一致。

## 相关

- [快速上手](quickstart.md) — 获取 key 并发出你的第一次被计量请求。
- [配置](configuration.md) — gateway 的环境变量。
- [认证与安全](auth-security.md) — API key 创建与 `project` 标签。
- [实时与视频](realtime-video.md) — 视频定价背后的视频任务生命周期。
- [运维](operations.md) — 健康探测与可观测性。
- [OpenAI 兼容 REST API](../api/openai-rest.md) — `/v1/*` 接口。
- [JSON-RPC API](../api/jsonrpc.md) — `usage.list`、`billing.plan`、`billing.video.pricing.*`。
- [概述](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
