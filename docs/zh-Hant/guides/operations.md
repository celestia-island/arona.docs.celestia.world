---
title: "營運"
description: "健康 endpoint、RUST_LOG 追蹤、upstream 逾時、錯誤對映，與運行中 arona-server 的疑難排解。"
---

# 營運

此頁面供運行 `arona-server serve` 的操作員使用。內容涵蓋你要探測的健康
endpoint、值得 grep 的日誌行、套用於 upstream backends 的逾時模型、
backend 失敗如何對映為 HTTP 錯誤，以及常絆倒人的營運陷阱。部署本身涵蓋
於[部署指南](./deployment.md)。

## 健康矩陣

三個健康 endpoint 都不需認證，只要程序在提供服務就回傳 `200 OK`——
沒有 liveness／readiness 之分：

| Endpoint | 回應 |
| --- | --- |
| `/healthz`、`/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | 與上方相同的詳細內文 |
| `/api/health` | plana `HealthResponse`：`status`、`version`（`CARGO_PKG_VERSION`）、`kind`（`Dev`）、`uptime`（秒）、`network`（transport／region／asn）、`build_hash`（`BUILD_HASH`）、`engine_version`（`"0.1.0"`） |

`/healthz` 與 `/readyz` 是相同 handler 的同義詞，`/v1/health` 與其共用，
因此 Kubernetes 型 probe 與 OpenAI 相容的健康路由可以互換。`/api/health`
額外提供 uptime、network 與 engine 版本。負載平衡器與監督器用 `/readyz`；
需要更豐富的 payload 時用 `/api/health`。

## 日誌

伺服器透過 `tracing` 記錄，以標準 `RUST_LOG` 變數過濾
（`RUST_LOG=info` 是常見設定；`RUST_LOG=debug` 會揭露 probe 流量）。
值得知道的事件，依大致頻率排序：

| 日誌行 | 等級 | 它告訴你什麼 |
| --- | --- | --- |
| `chat completions request`／`chat completions SSE request` | info | 每個聊天請求一筆，帶 `key_prefix`、`model`、`stream` 與 `request_id`——最簡單的每請求稽核軌跡。 |
| `request completed` | info | 由 `logging_middleware` helper 在每個**非串流** `/v1/chat/completions` 與 `/v1/embeddings` 回應後記錄：`method`、`path`、`status`、`latency_ms`、`trace_id`。（串流聊天改在開始時記錄 `chat completions SSE request`。） |
| `usage recorded`／`usage persisted` | info | 一筆 usage 列被記錄（記憶體內，帶 tokens／cost），然後寫入 `usage_records` 資料表。 |
| `external probe: sending`／`external probe: returned` | debug | 對 external backend `/v1/models` 的健康探測；`matched` 表示探測是否在 2s 探測逾時內完成。 |
| `billing gate rejected: monthly quota exceeded`／`billing gate rejected: tier rate limit exceeded` | warn | 被計費閘門拒絕的 `/v1/*` 請求——客戶端收到 429 加 `Retry-After`。 |
| `rpc billing gate rejected: monthly quota exceeded` | warn | JWT 認證方法的 RPC 端 quota 閘門（整個使用者視窗；JSON-RPC 錯誤回應）。 |
| `restored persisted backends`／`restored backend`／`restored persisted agent nodes` | info | 啟動還原：管理端註冊的 backends 與 agent 節點從資料庫載入並再次可路由。 |
| `Shutdown signal received, draining connections…` | info | 優雅關閉開始（SIGINT／SIGTERM）。 |

## 逾時模型

逾時在 external backends 使用的 upstream 客戶端上強制執行
（`packages/core/src/backends/external.rs`）：

| 逾時 | 值 | 套用對象 |
| --- | --- | --- |
| 連線 | 10s | 建立 upstream TCP/TLS 連線。 |
| 讀取閒置 | 每次讀取 120s | 每個 upstream 呼叫；每收到一個位元組就重置時鐘，因此健康但慢的串流永不被切斷。 |
| 非串流總計 | 600s | 非串流 chat/embeddings 呼叫——慢但存活的 upstream 不能永遠占住請求。 |
| 串流（SSE） | 無 | 串流呼叫**沒有總體截止時間**；長時間生成是合法的，掛死偵測依賴讀取閒置逾時。 |
| 健康探測 | 2s | `/v1/models` 探測。 |

## 錯誤對映

Backend 失敗在 chat/embeddings handlers 對映為 HTTP 狀態
（`packages/core/src/gateway/server.rs`）：

| 條件 | HTTP | `type`／`code` | 訊息 |
| --- | --- | --- | --- |
| Upstream 非 2xx 狀態（`UpstreamStatus`） | **502 Bad Gateway** | `server_error`／`bad_gateway` | `upstream <status>: <detail>` |
| Upstream 傳輸失敗（`RequestFailed`） | **502 Bad Gateway** | `server_error`／`bad_gateway` | 傳輸錯誤字串 |
| 任何其他 backend 錯誤 | **500** | `server_error`／`backend_error` | 錯誤字串 |
| 沒有提供該模型的 backend（`NoBackend`） | **404** | `invalid_request_error`／`model_not_found` | `No backend available for model: <model>` |
| 無效的 API key（`Unauthorized`） | **401** | `authentication_error`／`invalid_api_key` | `Invalid API key` |
| 速率限制（`RateLimited`） | **429** | `rate_limit_error`／`rate_limit_exceeded` | `Rate limit exceeded` |

設計意圖：呼叫者能區分「你的 provider 拒絕或失敗」（502）與「gateway
本身壞了」（500）。每個錯誤內文都是相同的 OpenAI 型形狀——
`{"error":{"message":...,"type":...,"code":...}}`（`json_error_response`）。
計費閘門的 429 額外攜帶 `Retry-After` header，並分別使用
`quota_error`／`quota_exceeded`（quota）與
`rate_limit_error`／`rate_limit_exceeded`（tier 速率限制）。

## 疑難排解

### 新註冊的 backend 在探測前維持 fail-closed

External backends 以未知的健康狀態啟動，回報
`"<url> not probed yet"`。當 (a) 健康檢查器的第一輪運行——啟動時立即、
之後每 60s——或 (b) 註冊或還原時啟動的 fire-and-forget 探測成功時，它們
轉為健康，通常約 1-2 秒內。在那之前，路由到該 backend 的請求刻意
fail closed。

### 對某些 backends，探測的 `/models` 回傳 404 是正常的

External 探測會打 `GET {base}/v1/models`（帶路徑前綴的基礎 URL 則為
`{base}/models`）。有些 OpenAI 相容伺服器實作了 chat 卻不提供模型列表——
Zhipu GLM coding-plan endpoint 就是其一。**404 是可接受的**：backend 被
標記為健康，管理端設定的 models 清單繼續作為路由來源。只有真正失敗的
探測（逾時、網路錯誤、其他非 2xx）會把 backend 標記為不健康。

### 沒有產出任何內容的 SSE 串流不會被計費

串流回應只有在串流產生了文字**或**攜帶終端 usage 時才記錄 usage；
兩者皆無就結束的串流完全不記錄。若你看到請求沒有對應的 `usage recorded`
行，請檢查串流是否真的產生了內容。

### 版本回報

健康內文中的 `version` 是 `CARGO_PKG_VERSION`；`build_hash` 是
`packages/core/build.rs` 產出的建置期 `BUILD_HASH` 值。跨節點比較
`build_hash` 以確認它們都運行相同的 artifact。

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
