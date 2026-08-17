---
title: "架構"
description: "Arona 如何組成——workspace 佈局、請求穿過 gateway 的路徑、路由、健康探測、記憶、session，以及刻意的設計取捨。"
---

# 架構

此頁面說明 Arona 如何結構化、請求如何穿流而過：workspace 佈局、請求路徑、
gateway 與路由器、健康檢查、記憶客戶端、session 與通知，最後是設計接受的
刻意限制與取捨。運行中的範例見[快速入門](quickstart.md)，日常執行期關注
事項見[營運](operations.md)。

## Workspace 佈局

儲存庫是包含三個套件的 Cargo workspace：

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC,
│            # backends, migration, entities, providers, models, registry,
│            # orchestration
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

- `packages/core` 是函式庫 crate（`_core`）。它包含伺服器需要的一切：
  axum gateway（`gateway/`）、模型路由器（`routing/`）、backend
  adapters（`backends/`）、計費（`billing/`）、認證（`auth.rs`）、
  記憶客戶端（`memory/`）、JSON-RPC 平面（`gateway/rpc.rs`）、
  schema（`migration/`、`entity/`）、模型 metadata（`models/`、`providers/`、
  `registry/`）與模型編排（`orchestration/`）。
- `packages/agent` 建置運行在 GPU 節點上、連回 `/ws/agent` 的 `_agent`
  二進位檔（見[agent-cluster](agent-cluster.md)）。
- `packages/cli` 建置用於 install、deploy、serve、migrate 與 download
  操作的 `_cli` 二進位檔。

此儲存庫已不再有網頁儀表板：Vue 儀表板已被移除，現位於
[shittim-chest](https://github.com/celestia-island/shittim-chest)（chest #291），
它透過 JSON-RPC 介面與 Arona 溝通。Arona 本身是純 backend
（見[總覽](./README.md)）。

## 請求路徑

進入點是在 `GatewayServer::app` 組裝的 axum 路由器
（`packages/core/src/gateway/server.rs`）。其路由表（server.rs:128-163）
涵蓋 OpenAI 相容 REST 介面（`/v1/chat/completions`、
`/v1/embeddings`、`/v1/models`、`/v1/health`）、視訊生成、
`/api/rpc` JSON-RPC endpoint（POST + WebSocket upgrade）、SSE sidecar
`/api/rpc/events`、agent 控制平面 `/ws/agent`、位於 `/docs` 的 Swagger UI，
以及管理 backend／alias 管理 endpoint。

路由器包覆在一小疊 layers 中（server.rs:158-162）：

1. 以 `Extension`s 提供的 Auth managers，讓各 handler 的 extractors 能觸及它們。
2. 一個 request-id layer，重複使用入站的 `X-Request-ID` header 或產生
   一個，並暴露給 handlers 與日誌（`gateway/request_id.rs`）。
3. 1 MB 請求內文限制（`RequestBodyLimitLayer`）。
4. 一個寬鬆的 CORS layer（`*` origin、`*` headers）。

因為 axum 自底向上套用 layers，CORS layer 在最外層。

每個 `/v1/*` handler 接著透過相同的骨架運行：

1. **Auth 抽取** — key-only endpoints（`/v1/chat/completions`、
   `/v1/embeddings`、視訊）用 `ApiKeyAuth`，`GET /v1/models` 用
   `ApiKeyOrJwt`（後者必須同時接受 API keys 與 session JWTs，
   `gateway/middleware.rs`）。extractor 將 key／JWT 解析為使用者 email、
   key 前綴、速率限制鍵（API key 的 SHA-256 雜湊，或 JWT 的
   `u:<email>` 標籤，使輪換的 token 不會重置視窗）與選用的專案範圍。
2. **計費閘門** — `enforce_billing_gates`（server.rs:492-539）在使用者的
   tier 每月 quota 或每分鐘速率限制超限時，以 HTTP 429 + `Retry-After`
   拒絕請求。資料庫失敗時 fail open：計費是盡力而為，絕不是提供聊天的
   硬依賴。
3. **記憶 recall**（聊天路徑）— 若記憶客戶端已設定且請求要求它，
   相關的長期記憶會以 system 區段注入（見下方[記憶客戶端](#memory-client)）。
   失敗絕不阻擋聊天；結果狀態反映在 `X-Arona-Memory` header。
4. **對話持久化** — 選用的 `conversation_id` 會做所有權檢查，使用者回合
   在送出時持久化。
5. **Gateway 分派** — 請求交給 `Gateway`，它解析 backend、修剪 context、
   呼叫 backend trait。
6. **Usage 記錄** — 回傳（或串流終端）的 usage 透過 `UsageTracker`
   在 key 前綴下記錄並持久化。

`Gateway` 本身以 `Arc<Gateway>` 存在於 `AppState`——沒有外部 mutex；
內部可變性讓並發的 chat/embeddings/stream 呼叫永遠不會在 upstream HTTP
往返期間持有鎖（`gateway/mod.rs:29-53`）。

## Gateway 與路由器

`Gateway`（`packages/core/src/gateway/mod.rs`）掌管：

- **路由器狀態** — backend 清單與 aliases，以 `tokio::sync::RwLock`
  保護。讀端的解析會跨 await 借用；變更（register/remove/alias）取得
  短暫的寫鎖，且永遠不會跨 upstream 呼叫持有它。
- **一個請求計數器**（`AtomicU64`）與 `system.status` 及健康 endpoint
  使用的 `start_time`。
- **一個 deployments 對映**（`model_id → backend name`），供 agent 部署的
  模型使用。`register_agent_backend` 建置名為 `agent-{model_id}` 的
  `ExternalApiBackend` 並插入路由器；重新註冊同一個模型會取代先前的
  backend，而 `unregister_agent_backend` 在收到 `stop_result` frame 時
  將其移除（見[agent-cluster](agent-cluster.md)）。

Backend 解析發生在 `Router`（`packages/core/src/routing/mod.rs`）：

1. **Alias 解析** — 設定的 alias 被改寫為其目標。
2. **Session affinity** — 有 `conversation_id` 時，路由器檢查一個弱參考
   對映，將對話釘在它首次被服務的 backend。弱參考讓對映只在 backend
   已註冊或進行中時存活，因此被移除的 backend 消失而不會有索引漂移。
   觸發的 circuit breaker 或不健康的被釘選 backend 會降級為全新選擇，
   並重新綁定對話。
3. **候選過濾** — 選用的 `provider` 提示依 backend 名稱／類型過濾；
   候選者必須健康*且* circuit breaker 開啟，且必須列出所要求的模型。
   模型 id 精確比對，或透過 `:latest` 尾綴慣例（裸 `nomic-embed-text`
   請求匹配列出的 `nomic-embed-text:latest`）。
4. **最少負載選擇** — 存活的候選者依其命中計數器排序，選取負載最低者；
   對話釘選（若有的話）同時被記錄。

在呼叫 backend 之前，`RequestPipeline::transform`
（`packages/core/src/pipeline.rs:422+`）將訊息清單修剪到 backend 的
`max_context_length`：system 訊息完整保留，非 system 訊息在塞得下時
最新優先保留，單一過大的訊息以字元硬截斷（啟發式 token 計數器無法
逐 token 精確截斷）。呼叫接著走 `InferenceBackend` trait；成功與失敗
都會記錄回路由器每個 backend 的 circuit breaker（3 次失敗、30s 恢復、
1 次 half-open 呼叫——routing/mod.rs:57-64）。

## 健康檢查器與探測

`run_health_checks`（`packages/core/src/gateway/health_checker.rs`）以
啟動時產生的背景任務運行（run.rs:135-150），每 60 秒間隔探測每個已註冊
backend 一次。兩個細節值得注意：

- backend 清單**每一輪都透過 async fetcher closure 重新取得**，
  因此啟動後才註冊的 backends（例如經由管理 API）不需要重啟就被納入。
- 第一輪在間隔屆滿前立即運行，因此程序一啟動健康狀態就建立起來。

`probe_backend` 是單一探測碼路徑。它被一次性**註冊時探測**重用：管理員
註冊 backend 後（server.rs:688-693）或已持久化的 backend 在開機時還原後
（run.rs:122-127），fire-and-forget 探測會在約 1–2s 內使 backend 轉為
健康，不必維持 fail-closed 直到下一輪 60s。這就是新註冊的 external
backend 的模型清單幾乎立即出現在 `GET /v1/models` 的原因。

探測本身是輕量的 backend 呼叫（例如 external backend 以 2s 探測逾時打
`/v1/models`）；結果快取在 backend 中，路由只會選取快取健康狀態為
`Healthy` 的 backend（加上開啟的 circuit breaker）。

## 記憶客戶端

記憶客戶端（`packages/core/src/memory/mod.rs`）在伺服器啟動時從環境設定
建置（server.rs:95-97）：當 `ARONA_MEMORY_URL` 與 `ARONA_MEMORY_TOKEN`
已設定時，聊天請求會經由 JSON-RPC WebSocket 查詢 entelecheia Philia
記憶服務，`recall_and_inject` 將相關記憶以 system 區段
（`## Relevant Long-Term Memories`）前置到外送 context。完成的回合經由
`writeback_dialogue` 以 episodes 寫回——在 assistant 回覆持久化後產生的
fire-and-forget 工作，因此記憶失敗絕不阻擋或拖慢聊天回應路徑。
`ARONA_MEMORY_WRITEBACK`（預設開啟）切換寫回。全貌見
[memory-gateway](memory-gateway.md)。

每個聊天回應都攜帶 `X-Arona-Memory` header，狀態三選一：`enabled`
（recall 執行並注入）、`disabled`（未設定或請求傳入 `memory: false`），
或 `offline`（已設定但服務無法觸及）。

## Session 與通知

`AppState` 持有 `plana` 的 `SessionManager`（`state.sessions`）。諸如
`chat.send` 的串流 RPC 會建立 session id（`gateway/rpc.rs:1701`）並將
通知——`chat.stream` tokens、`models.progress` 部署進度、
`realtime.event`——推上該 session 的通道。客戶端從 SSE sidecar
`GET /api/rpc/events?session=<id>`（server.rs:191-200）消費它們；
通知格式與預訂閱視窗注意事項見[events](../api/events.md)。

session 通道也用於請求／回應 RPC 呼叫：當客戶端在 `POST /api/rpc`
上送出 `x-session-id` header 時，伺服器也會將結果推上該 session 通道
（server.rs:184-188, rpc.rs:128-144），因此客戶端可以把 RPC 回應多工
到已開啟的 SSE 串流上。

## 限制與設計取捨

設計刻意接受若干限制；正式環境使用前請先了解：

- **1 MB 請求內文限制** — 更大的內文會被 layer 拒絕；若你需要大 context
  呼叫，這是第一個要調整的地方。
- **CORS `*`** — gateway 回應任何地方的跨來源呼叫。對 API 沒問題，
  但若你要暴露給受信任客戶端以外，請在前面加上強制你自己 CORS 政策的
  proxy。
- **Fail-open 計費** — 資料庫不可用時，quota／速率限制強制降級為允許
  請求。計費是計量，不是存取控制。
- **SSE 串流沒有總體逾時** — 串流呼叫沒有總截止時間（長時間生成是合法
  的）；掛死偵測依賴 120s 每次讀取的閒置逾時（`backends/external.rs:24-31`）。
  非串流呼叫有 600s 總截止時間。
- **Tokenizer 估算 usage** — 從不回報 usage 的 backends（ollama、
  ws_engine）以本機 CJK 感知 tokenizer 估算計費，原樣記錄
  （見[billing-usage](billing-usage.md)）。
- **記憶體內速率限制視窗與撤銷** — per-key 滑動視窗與被撤銷 key 集合
  存在程序記憶體（`auth.rs`），因此重啟會重置它們。auth 層限制器限制
  每個視窗每個 key 的請求數；billing tier 限制器以資料庫為後盾
  （見[auth-security](auth-security.md)與[billing-usage](billing-usage.md)）。
- **`/ws/agent` 不需認證** — agent 控制平面接受任何使用
  register/heartbeat 協定的 WebSocket。它只在你能控制的網路上才安全。
- **gateway 沒有 TLS** — 伺服器綁定純 HTTP；任何跨越網路邊界的部署都要
  在前面終止 TLS（反向 proxy）。見[deployment](deployment.md)。

在優雅的一面，伺服器會執行優雅關閉：安裝 Ctrl+C 與 SIGTERM handlers、
記錄「draining connections」，並在程序結束前讓進行中的請求完成
（`gateway/run.rs:14-38`，以及 run.rs:154-159 的 `with_graceful_shutdown`
接線）。

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table); gateway/mod.rs:29-53; routing/mod.rs:102-200; health_checker.rs:14-37; run.rs:135-159; memory/mod.rs:207-241; rpc.rs:128-144,1701 -->
