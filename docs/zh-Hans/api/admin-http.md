---
title: "Admin HTTP API"
description: "Bearer token 管理面——通过 /api/admin/* 注册/列出/移除 backend 并管理模型别名。"
---

# Admin HTTP API

`/api/admin/*` 接口管理 gateway 的 **backend**（上游模型提供商）、**别名**
（模型名称 → 模型 id 重定向）和**用量查询**（聚合计量）。它是 RPC 管理平面的
HTTP 对应物（见 [JSON-RPC API](./jsonrpc.md)），主要由运维人员和 admin UI
使用。

## 认证

每个 `/api/admin/*` 路由都要求：

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN` 在进程启动时从环境读取（`GatewayServer::new`）。如果该
变量**未设置**，或出示的 token 不匹配，请求以 `401` 被拒绝：

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

Bearer 前缀不区分大小写匹配（`Bearer` 或 `bearer`）。

> 与 `/v1/*` 接口不同，admin 认证从不回退到 API key 或 JWT，并且用精确 token
> 比较执行——用新值重启进程即可轮换 token。

## Backend

Backend 是 gateway 背后可路由的上游。注册使 backend 立即可路由、持久化其
配置供重启恢复、probe 它（约 1-2 秒内转为 healthy），并且对桥接 URL 保持隧道
存活。Backend 类型与 URL 语义详见 [Backend](../guides/backends.md)。

### POST /api/admin/backends —— 注册 backend

请求体（除注明外所有字段可选）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `type` | string | Backend kind。取值为 `external`（任意 OpenAI 兼容 HTTP API）、`ollama`（本地或远程 ollama 服务器）、`engine`（通过 `ws://`/`wss://` 的 CEP 引擎）、`minimax-cloud`（云视频 API）。MDD 引擎名称（`llama_cpp`、`vllm`、`ollama`、`cloud`、`external_api`、`candle`、`native`、...）通过 planner 解析。`comfyui` **被拒绝**（`comfyui backend removed`）；其他任何值 → `400` `unknown_type`。缺失时默认为 `ollama`。 |
| `url` | string | Backend 基础 URL。`evernight://<node>/<service>` 桥接 URL 通过本地 evernight agent 解析为本地 TCP 转发（解析失败 → `502` `evernight_unreachable`）。默认为 `http://localhost:11434`。 |
| `api_key` | string | 可选的上游 API key，在上游调用时作为 `Authorization: Bearer` 发送。 |
| `name` | string | Backend 名称。缺失时默认为 `type` 值。用作路由 `provider` 提示和配置行身份。 |
| `models` | string[] | 静态模型列表。probe 未发现任何模型时作为路由来源。对 `external` backend，发现的模型在静态列表之后合并（静态 id 保持优先级）；`engine` backend 先返回其发现的模型缓存，静态 id 追加其后；`minimax-cloud` 不做模型发现（其 probe 只对 `/v1/query/available_models` 做健康 ping），只服务静态列表。`ollama` 忽略它，从 `/api/tags` 发现模型。 |
| `workflow` | object | 可选。遗留——历史上被已移除的 ComfyUI backend 使用；当前没有 backend 读取它（为 `backend_configs` 列兼容性保留）。 |

示例：

```bash
curl -X POST http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "name": "mock-upstream",
    "url": "http://192.0.2.20:11434",
    "api_key": "sk-xxx",
    "models": ["Qwen/Qwen3-1.7B"]
  }'
```

成功 → `200`：

```json
{ "status": "ok", "message": "backend registered" }
```

注册的副作用：

- backend **立即注册并可路由**（无需重启）。
- 配置**持久化**到 `backend_configs` 表并在启动时恢复（数据库失败只记日志，
  绝不阻塞响应）。
- 立即运行一次即发即忘的 **probe**，使 backend 在约 1-2 秒内转为 healthy，
  而不是保持 fail-closed 直到下一轮 60 秒健康检查。
- 对 `evernight://` URL，一个 **keepalive 任务**监视隧道：以新本地端口重连
  时，它会透明地重建并以同名重新注册 backend。

### GET /api/admin/backends —— 列出 backend

```json
{
  "backends": {
    "count": 2,
    "health": [
      ["backend_0:ExternalApi", "Healthy"],
      ["backend_1:Ollama", { "Unhealthy": "connection refused" }]
    ]
  },
  "models": [
    { "id": "Qwen/Qwen3-1.7B", "object": "model", "owned_by": "mock-upstream" }
  ]
}
```

- `backends.count` —— **healthy** backend 的数量。
- `backends.health` —— 每 backend 的 `backend_<index>:<kind>` 标签与健康
  状态（`Healthy` / `Degraded` / `Unhealthy`）。`<index>` 是 `DELETE
  /api/admin/backends` 使用的路由器注册索引。
- `models` —— 当前可路由的每个模型 id（与 `GET /v1/models` 相同的列表，
  不做快速上手合并；见 [OpenAI 兼容 REST](./openai-rest.md#get-v1models)）。

### DELETE /api/admin/backends —— 移除 backend

通过 JSON body 中的路由器**索引**标识——而不是名称：

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `index` | integer | 是 | 路由器注册索引，匹配 `GET /api/admin/backends` 健康报告中的 `backend_<index>` 标签。 |

- 缺少 `index` → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`。
- 索引越界 → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`。
- 成功 → `200` `{ "status": "ok", "message": "backend removed" }`。
- 持久化的 `backend_configs` 行尽力而为地删除：backend 名称从其模型列表的
  `owned_by` 恢复；不匹配则把行留在存储中（数据库失败只记日志，绝不致命）。

## 别名

别名把一个模型名称映射到另一个（`alias` → `target`），使针对一个模型 id 的
请求路由到不同的 backend 模型。别名在路由之前解析，因此统一适用于聊天、
embeddings 和视频查询。

> 别名**只是内存中的路由器状态**——它们不持久化，重启即丢失。请在启动后注册
> 它们，或从你自己的供应状态重建。

### POST /api/admin/aliases —— 添加别名

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `alias` | string | 是 | 客户端将请求的模型名称。 |
| `target` | string | 是 | 请求路由到的模型 id。 |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- 缺少 `alias` → `400` `missing_alias`；缺少 `target` → `400` `missing_target`。
- 成功 → `200` `{ "status": "ok", "message": "alias added" }`。
- 添加已存在的别名会替换其 target。

### GET /api/admin/aliases —— 列出别名

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

对按别名排序返回。

### DELETE /api/admin/aliases —— 移除别名

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `alias` | string | 是 | 要移除的别名。 |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- 缺少 `alias` → `400` `missing_alias`。
- 移除未知别名是无操作成功 → `200` `{ "status": "ok", "message": "alias removed" }`。

## 用量

Arona 是计量的唯一事实来源：每个产生 usage 行的代理请求都记录在
`usage_records` 中。请求可以携带**归因引用**——`POST /v1/chat/completions`
（流式与非流式）和 `POST /v1/embeddings` 上的 `x-celestia-ref` 请求头，或
JSON-RPC `realtime.start` 方法的 `ref_id` 参数（通常是调用方服务选定的会话
UUID）——存入该行的 `ref_id` 列。`POST /api/admin/usage/query` 对这些引用做
聚合，让下游服务（如 shittim-chest）能对账它们归因的用量，而无需自建本地
计量。

### POST /api/admin/usage/query —— 聚合用量查询

请求体（所有字段可选）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `ref_ids` | string[] | 要聚合的归因引用（`WHERE ref_id = ANY(...)`）。缺失 / 为空 / 全为空白 → **全局**（所有行）。条目会被 trim；每条截断到 64 字符。 |
| `since` | string | `created_at` 下界（含）。`YYYY-MM-DD`（该日零点，UTC）或 RFC3339 时间戳。 |
| `until` | string | `created_at` 上界（不含）。`YYYY-MM-DD` 覆盖**整整天**（不含次日零点，UTC）；RFC3339 时间戳则是不含的那个时刻。 |
| `group_by` | string | 额外按键聚合：`model` \| `backend` \| `ref` \| `day`（`day` = `created_at` 的 UTC 日桶，键格式 `YYYY-MM-DD`）。其他值 → `400` `bad_group_by`。 |
| `limit` | integer | `include_records` 的页大小，限制 1–500，默认 100。 |
| `offset` | integer | `include_records` 的页偏移，默认 0。 |
| `include_records` | bool | 同时返回分页的原始行及 `records_total`。默认 `false`。 |

所有过滤条件按 AND 组合。聚合完全在 SQL 内完成（`GROUP BY`，无内存全表扫
描）；分组按 `total_tokens` 降序（同分按组键升序打破平局），原始行最新的在
前（同刻按行 id）——两个排序都是确定性的，保证翻页稳定。

```bash
curl -X POST http://192.0.2.10:8080/api/admin/usage/query \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "ref_ids": ["0b914cb6-2d71-4c55-9f1a-3f2b1c9d8e7f"],
    "since": "2026-08-01",
    "until": "2026-08-21",
    "group_by": "model",
    "include_records": false
  }'
```

成功 → `200` —— `totals` 恒在；仅当设置了 `group_by` 才有 `groups`；仅当
`include_records` 为 `true` 才有 `records` / `records_total` / `limit` /
`offset`：

```json
{
  "totals": {
    "requests": 12,
    "prompt_tokens": 3400,
    "completion_tokens": 810,
    "total_tokens": 4210,
    "cost_usd": 0.0121
  },
  "groups": [
    {
      "key": "Qwen/Qwen3-1.7B",
      "requests": 12,
      "prompt_tokens": 3400,
      "completion_tokens": 810,
      "total_tokens": 4210,
      "cost_usd": 0.0121
    }
  ]
}
```

`include_records: true` 时每行携带 `id`、`ref_id`、`api_key_id`、`model`、
`backend`、`prompt_tokens`、`completion_tokens`、`total_tokens`、`cost` 和
`created_at`（RFC3339），最新的在前：

```json
{
  "totals": { "requests": 1, "prompt_tokens": 120, "completion_tokens": 30,
              "total_tokens": 150, "cost_usd": 0.0004 },
  "records_total": 1,
  "limit": 100,
  "offset": 0,
  "records": [
    {
      "id": "8f14e45f-ceea-467a-9eaa-3728a80d62a1",
      "ref_id": "0b914cb6-2d71-4c55-9f1a-3f2b1c9d8e7f",
      "api_key_id": "arona-Xy",
      "model": "Qwen/Qwen3-1.7B",
      "backend": "gateway",
      "prompt_tokens": 120,
      "completion_tokens": 30,
      "total_tokens": 150,
      "cost": 0.0004,
      "created_at": "2026-08-20T09:30:12+00:00"
    }
  ]
}
```

说明：

- `cost_usd` 是 `SUM(cost)`，忽略 `NULL` cost（未定价模型的行和零 token 的
  realtime 行贡献 `0`）。
- 全局查询按 `ref` 分组时，无引用行的桶为 `"key": null`。
- 带 `ref_id` 启动的 realtime 会话即使引擎上报零 token 也写 usage 行（本地
  CEP 语音引擎）——这些行显示 `0` token 和 `null` cost。
- 畸形的 `since`/`until` → `400` `bad_since` / `bad_until`；未知
  `group_by` → `400` `bad_group_by`；数据库故障 → `500` `internal_error`
  （详情在服务端日志）。

## 持久化总结

| 资源 | 持久化？ | 重启恢复 |
| --- | --- | --- |
| Backend | 是 —— `backend_configs` 表（`name` 键，注册时 upsert，移除时删除）。 | 是：启动时恢复；external backend 以 fail-closed 启动，首轮 probe 后转为 healthy。`evernight://` URL 在启动时通过桥接重新解析。 |
| 别名 | 否 —— 仅内存 `Router.aliases`。 | 否。 |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler), admin_usage_query (POST /api/admin/usage/query); packages/core/src/billing/usage_query.rs (query parsing, SQL builders, executor); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
