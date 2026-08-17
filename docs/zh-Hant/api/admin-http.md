---
title: "管理 HTTP API"
description: "Bearer-token 管理介面——經由 /api/admin/* 註冊／列出／移除 backends 並管理模型 aliases。"
---

# 管理 HTTP API

`/api/admin/*` 介面管理 gateway 的 **backends**（upstream 模型
providers）與 **aliases**（模型名稱 → 模型 id 重定向）。它是 RPC 管理
平面的 HTTP 對應物（見[JSON-RPC API](./jsonrpc.md)），主要由操作員與
管理介面使用。

## 認證

每個 `/api/admin/*` 路由都需要：

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN` 在程序啟動時從環境讀取（`GatewayServer::new`）。
若變數**未設定**，或送出的 token 不符，請求以 `401` 被拒絕：

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

Bearer 前綴不分大小寫比對（`Bearer` 或 `bearer`）。

> 與 `/v1/*` 介面不同，admin 認證絕不退回 API keys 或 JWTs，
> 且以精確 token 比對強制執行——用新值重新啟動程序來輪換 token。

## Backends

Backends 是 gateway 後方可路由的 upstreams。註冊會讓 backend 立即可路由、
持久化其設定供重啟還原、探測它（約 1–2 s 內轉為健康），並對 bridge URLs
維持隧道存活。Backend 類型與 URL 語意詳述於
[Backends](../guides/backends.md)。

### POST /api/admin/backends — 註冊 backend

請求內文（除註明者外所有欄位選用）：

| 欄位 | 類型 | 備註 |
| --- | --- | --- |
| `type` | string | Backend 類型。`external`（任何 OpenAI 相容 HTTP API）、`ollama`（本機或遠端 ollama 伺服器）、`engine`（`ws://`／`wss://` 上的 CEP engine）、`minimax-cloud`（雲端視訊 API）之一。MDD engine 名稱（`llama_cpp`、`vllm`、`ollama`、`cloud`、`external_api`、`candle`、`native`、……）經由 planner 解析。`comfyui` **被拒絕**（`comfyui backend removed`）；其他任何值 → `400` `unknown_type`。缺失時預設 `ollama`。 |
| `url` | string | Backend 基礎 URL。`evernight://<node>/<service>` bridge URLs 透過本機 evernight agent 解析為本機 TCP 轉發（解析失敗 → `502` `evernight_unreachable`）。預設 `http://localhost:11434`。 |
| `api_key` | string | 選用的 upstream API key，在 upstream 呼叫上以 `Authorization: Bearer` 送出。 |
| `name` | string | Backend 名稱。缺失時預設為 `type` 值。用作路由的 `provider` 提示與設定列身分。 |
| `models` | string[] | 靜態模型清單。探測未發現任何模型時是路由來源。對 `external` backends，發現的模型合併在靜態清單之後（靜態 id 保持優先權）；`engine` backends 先回傳其發現的模型快取、再附加靜態 id；`minimax-cloud` 不做模型探索（其 probe 只對 `/v1/query/available_models` 做健康 ping）並單獨服務靜態清單。`ollama` 忽略它，它從 `/api/tags` 發現模型。 |
| `workflow` | object | 選用。Legacy——歷史上由已移除的 ComfyUI backend 消費；目前沒有 backend 讀取它（保留以相容 `backend_configs` 欄位）。 |

範例：

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

註冊的副作用：

- backend **立即註冊且可路由**（不需重啟）。
- 設定**持久化**到 `backend_configs` 資料表並在啟動時還原（資料庫失敗
  會被記錄但絕不阻擋回應）。
- 一個 fire-and-forget **probe** 立刻執行，使 backend 在約 1–2 s 內轉為
  健康，不必維持 fail-closed 直到下一輪 60 s 健康檢查器。
- 對 `evernight://` URLs，一個 **keepalive task** 監看隧道：以新本機
  連接埠重連時，它會透明重建並以相同名稱重新註冊 backend。

### GET /api/admin/backends — 列出 backends

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

- `backends.count` — **健康** backends 的數量。
- `backends.health` — 每個 backend 的 `backend_<index>:<kind>` 標籤與
  健康狀態（`Healthy`／`Degraded`／`Unhealthy`）。`<index>` 是
  `DELETE /api/admin/backends` 使用的路由器註冊索引。
- `models` — 今天可路由的每個模型 id（與 `GET /v1/models` 相同的
  列表，沒有 quick-start 合併；見
  [OpenAI 相容 REST](./openai-rest.md#get-v1models)）。

### DELETE /api/admin/backends — 移除 backend

以 JSON 內文中的路由器**索引**識別——不是名稱：

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| 欄位 | 類型 | 必填 | 備註 |
| --- | --- | --- | --- |
| `index` | integer | 是 | 路由器註冊索引，與 `GET /api/admin/backends` 健康報告中的 `backend_<index>` 標籤對應。 |

- 缺少 `index` → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`。
- 索引超出範圍 → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`。
- 成功 → `200` `{ "status": "ok", "message": "backend removed" }`。
- 持久化的 `backend_configs` 列會盡力刪除：backend 名稱從其模型列表的
  `owned_by` 恢復；不符時列留在 store 中（資料庫失敗被記錄，絕非致命）。

## Aliases

Aliases 將一個模型名稱對映到另一個（`alias` → `target`），因此對一個
模型 id 的請求會路由到不同的 backend 模型。Aliases 在路由前解析，
因此對 chat、embeddings 與視訊查詢一致適用。

> Aliases **僅是記憶體內的路由器狀態**——它們不持久化，重啟時遺失。
> 請在啟動後註冊它們，或從你自己的 provisioning 狀態重建。

### POST /api/admin/aliases — 新增 alias

| 欄位 | 類型 | 必填 | 備註 |
| --- | --- | --- | --- |
| `alias` | string | 是 | 客戶端會請求的模型名稱。 |
| `target` | string | 是 | 請求被路由到的模型 id。 |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- 缺少 `alias` → `400` `missing_alias`；缺少 `target` → `400`
  `missing_target`。
- 成功 → `200` `{ "status": "ok", "message": "alias added" }`。
- 新增已存在的 alias 會取代其 target。

### GET /api/admin/aliases — 列出 aliases

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

配對依 alias 排序回傳。

### DELETE /api/admin/aliases — 移除 alias

| 欄位 | 類型 | 必填 | 備註 |
| --- | --- | --- | --- |
| `alias` | string | 是 | 要移除的 alias。 |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- 缺少 `alias` → `400` `missing_alias`。
- 移除未知的 alias 是 no-op 成功 → `200`
  `{ "status": "ok", "message": "alias removed" }`。

## 持久化摘要

| 資源 | 持久化？ | 重啟時還原 |
| --- | --- | --- |
| Backends | 是——`backend_configs` 資料表（`name` key，註冊時 upsert、移除時刪除）。 | 是：啟動時還原；external backends 以 fail-closed 啟動，第一輪探測後轉為健康。`evernight://` URLs 在啟動時經由 bridge 重新解析。 |
| Aliases | 否——僅記憶體內 `Router.aliases`。 | 否。 |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
