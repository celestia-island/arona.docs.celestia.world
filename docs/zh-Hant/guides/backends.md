---
title: "Backends"
description: "Backend 類型（external、ollama、engine、minimax-cloud、evernight bridge）、URL 語意、健康探測、模型探索、aliases 與路由。"
---

# Backends

**backend** 是提供模型流量的 upstream。Arona 將 OpenAI 相容的請求
（`/v1/chat/completions`、`/v1/embeddings`、模型列表、視訊任務）路由到其中
一個已註冊的 backend，為每個請求計量，並持續更新每個 backend 的健康狀態與
模型庫存。

Backends 由管理員透過 `POST /api/admin/backends` 註冊
（見[管理 HTTP API](../api/admin-http.md)），持久化到 `backend_configs` 資料表，
並在啟動時自動還原。每次註冊攜帶 `name`、`type`、`url`、選用的 `api_key`
與選用的靜態 `models` 清單。已持久化的 backends 能存活於重啟；還原的
backends 以 fail-closed 啟動並立即被探測。

## Backend 類型

| `type` | 傳輸 | 協定 | 用途 |
| --- | --- | --- | --- |
| `external` | HTTP(S) | OpenAI 相容 REST | 任何 chat/embeddings API（雲端或自架） |
| `ollama` | HTTP(S) | Ollama 原生 API（`/api/chat`、`/api/embed`、`/api/tags`） | 本機或遠端 Ollama 伺服器；僅憑 URL 即可建置 |
| `engine` | `ws://`／`wss://` | CEP（Celestia Engine Protocol）、WebSocket + JSON-RPC | 任何使用 CEP 互通標準溝通的 engine（`plana::engine`） |
| `minimax-cloud` | HTTPS | MiniMax H3 task 型 API（submit + poll） | 雲端視訊生成 |
| `evernight://<node>/<service>` | bridge URL | 透過本機 evernight agent 解析為本機 TCP 轉發 | 只能經由 evernight mesh 觸及的工業／邊緣服務 |
| `agent-{model}` | HTTP | OpenAI 相容（external） | GPU agent 部署模型時自動註冊 |

### `external` — 任何 OpenAI 相容 HTTP API

通用 backend：對任何使用 OpenAI REST 形狀溝通的伺服器提供聊天補全（串流與
非串流）與 embeddings。以基礎 `url`、`api_key`（選用）與選用的靜態 `models`
清單設定：

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

靜態 `models` 清單具有權威性：它會合併在探測時發現的任何模型之前
（見[模型探索](#model-discovery)）。

### `ollama` — 僅憑 URL 建置

Ollama backend 僅憑 URL 即可建置——不需要 API key、不需要模型清單。它使用
Ollama 的原生協定：聊天用 `/api/chat`、embeddings 用 `/api/embed`、
健康探測與模型探索用 `/api/tags`。

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — 透過 WebSocket 的 CEP

`engine` backend 連到暴露 `ws://`（或 `wss://`）的 engine，並使用
**Celestia Engine Protocol**（CEP）溝通：這是定義於 `plana::engine` 的
WebSocket + JSON-RPC 2.0 互通標準。任何語言撰寫、實作 handshake → methods →
串流通知流程的 engine，都能以第一級 backend 註冊，Arona 端零 adapter 程式碼。
Wire methods：`Engine.Handshake`（第一則訊息；身分 + 能力）、
`Engine.Chat`、`Engine.ChatStart`（串流；chunks 以 `Engine.ChatChunk` 通知
送達）、`Engine.Embeddings` 與 `Engine.Models`。連線在首次使用時惰性建立，
任何錯誤時斷開；下一次呼叫會重新連線並重新 handshake。

### `minimax-cloud` — task 型視訊生成

雲端視訊 backend 驅動 MiniMax H3 Open Platform API：送出生成任務、輪詢
完成，然後從結果讀取 artifact URL。這取代了已移除的 ComfyUI backend
（見下文）；視訊任務透過 `/v1/video/generations` 或 `video.*` RPC 方法送出，
進度透過 `video.progress`／`video.done`／`video.failed` 通知傳遞
（見[Realtime 與視訊](realtime-video.md)）。

### `evernight://` bridge URL

形式為 `evernight://<node>/<service>` 的 backend URL **不會**被直接連線。
本機 host 上的 evernight agent 將其解析（透過 agent 的 WebSocket endpoint
的 `Bridge.Connect` JSON-RPC 呼叫）為本機 TCP 轉發，backend 便對
`http://127.0.0.1:<local_port>` 運行，而非硬編碼的遠端位址。這就是
單一 panel 架構：Arona panel 透過 mesh 觸及其他節點上的服務（CEP engines、
scepter、……），設定中永遠不嵌入遠端位址。

keepalive 任務每 15 秒探測一次隧道；當遠端重新啟動、隧道在新的本機連接埠
重建時，受影響的 backend 會以新 URL **透明重建**——持久化的設定保留
`evernight://` URL，因此重啟後會重新解析。對 `engine` backend，解析出的
`http://127.0.0.1:<port>` 轉發會轉換為 `ws://` 以配合 WebSocket 傳輸。

### Agent 部署的模型自動註冊

當 GPU agent 完成部署模型時，gateway 會註冊一個名為
`agent-{model_id}` 的 backend（`http://{agent host}:{port}` 上的
`ExternalApiBackend`），使模型立即可路由；停止部署時會再次取消註冊。
完整的部署生命週期見[Agent 叢集](agent-cluster.md)。

### `comfyui` 會被拒絕

`comfyui` backend 類型會被明確拒絕，錯誤訊息為
`comfyui backend removed`：ComfyUI backend 在模型平台收斂期間被移除，
視訊生成現已改由 `minimax-cloud` 進行。註冊 `comfyui` backend 會回傳
HTTP 400。

## URL 語意

設定的基礎 URL 如何對應到實際 endpoint，取決於 URL 是否帶有路徑元件：

- **Root 型基礎** — 路徑為空或 `/` 的 URL 會被視為 host root，
  並沿用 OpenAI 的 `/v1` 慣例：`{base}/v1/chat/completions`、
  `{base}/v1/models`。範例：`http://192.0.2.20:8429`、
  `https://api.deepseek.com`。
- **Path 型基礎** — 帶非空路徑的 URL 會被視為伺服器實際提供服務的完整
  API 前綴，endpoint 直接接在其後：`{base}/chat/completions`、
  `{base}/models`。這是 `/v1` 慣例以外的 OpenAI 相容伺服器所需的。
  Zhipu GLM coding plan 是典型範例：其 API 位於
  `https://open.bigmodel.cn/api/coding/paas/v4`，chat 直接在
  `{base}/chat/completions`，而且**完全沒有 `/models` endpoint**——
  標準 `/api/paas/v4` root 對 coding-plan keys 會回傳 balance 錯誤。
- 設定的基礎 URL 上的**結尾斜線**會被正規化移除，因此拼接永遠不會產生
  雙斜線。

## 探測與健康

背景健康檢查器每 **60 秒**探測一次每個已註冊的 backend；每一輪都會重新
取得 backend 清單，因此啟動後才註冊的 backends 不需要重啟就會被納入。
每次管理端註冊也會立即觸發一次探測，使 backend 在約 1–2 秒內轉為健康，
不必等待下一輪檢查器。

- **External backends** 以 **2 秒逾時**探測 `GET {base}/models`
  （root 型基礎則為 `{base}/v1/models`）。**404 是可接受的**：有些
  伺服器實作了 chat 卻沒有模型列表（GLM coding plan 沒有 `/models`
  endpoint），所以 404 會將 backend 標記為健康，管理端設定的 `models`
  清單便成為路由來源。逾時、網路失敗與其他非 2xx 回應會將 backend
  標記為不健康。
- **Ollama backends** 以相同的 2 秒逾時探測 `/api/tags`。
- Backends 以 **fail-closed** 啟動——回報為 `not probed yet`——直到
  第一次成功探測為止，因此新註冊（或還原）的 backend 在驗證前絕不會
  收到流量。

健康狀態會逐 backend 快取，路由器的每個請求都會查詢它；不健康的 backends
會被排除在候選選擇之外（見[路由](#routing)）。

## 模型探索

backend 會宣告它提供的模型 id，路由器將請求與該宣告比對：

- **External** backends 宣告從探測回應解析出的模型（`data` 與 `models`
  陣列皆可接受），並與管理端設定的靜態清單合併——靜態 id 保持順序與
  優先權，動態 id 去重後附加。當伺服器沒有 models endpoint 時，靜態清單
  單獨作為路由來源。
- **Ollama** backends 宣告 `/api/tags` 回傳的 tags。
- **Agent 部署**的模型只宣告實際部署的 `model_id`。

對外介面是 `GET /v1/models`（需認證），列出每個健康 backend 的可路由模型
（見[OpenAI 相容 REST API](../api/openai-rest.md)）。

## Aliases 與名稱正規化

Aliases 將請求的模型 id 對映到目標 id。路由時會先解析 alias，因此對
alias 的請求會由宣告目標的 backend 提供服務：

```json
{ "alias": "fast-chat", "target": "deepseek-v4-flash" }
```

Aliases 透過 `/api/admin/aliases` 管理端點管理，立即生效。

名稱比對也會正規化 Ollama 型 tags：backend 列出
`nomic-embed-text:latest` 時，對裸 `nomic-embed-text` 的請求也能匹配，
因此 embedding/chat 請求不需要維護 `:latest` 尾綴。明確的 tag
（`qwen3:0.6b`）只匹配該確切 tag。

## 路由

每個請求都會透過路由器解析，選出一個 backend：

1. **Alias 解析** — 請求的模型 id 先經由 alias 表對映（若有的話）。
2. **Provider hint** — 選用的 `provider` 欄位依 backend 名稱
   （或 kind 名稱，例如視訊 backend 的 `cloud`）過濾候選者。
3. **僅限健康候選者** — backend 必須回報 `Healthy` *且*通過
   其 circuit breaker（連續 3 次失敗會開啟 breaker 30 秒，含一次
   half-open 測試呼叫）才可被選取。
4. **最少計數選擇** — 提供該模型的候選者依各自的 per-backend 請求計數器
   排序，選取負載最低者。這會將負載分散到提供相同模型的健康 backends。
5. **Session affinity** — 當請求攜帶 `conversation_id` 時，該對話會被釘在
   它首次使用的 backend。釘選存放在 `Weak` 參考對映中，因此被移除的
   backend 會從對映消失，不會有索引漂移。Affinity 是盡力而為：在對話的
   各回合間重複使用同一個 backend，可讓 upstream 重複使用每對話的執行期
   狀態（熱 context、KV 快取）。若被釘選的 backend 已變得不健康
   （或釘選因 backend 被移除而失效），路由器會退回全新的最少計數選擇並
   **重新綁定**該對話。

若沒有健康 backend 提供該模型，路由失敗：未知模型產生 `model not found`
（HTTP 404），已知但無法觸及的模型產生 `all
backends unhealthy`，以 500 internal server error 呈現。HTTP 502 保留給
*可觸及的* upstream 回報的失敗（選取後的非 2xx upstream 回應與傳輸失敗）。
完整的錯誤對映見
[營運](operations.md)。

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
