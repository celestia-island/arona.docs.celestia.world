---
title: "設定"
description: "Arona 伺服器讀取的每個環境變數參考，包含預設值與語意。"
---

# 設定

Arona **完全透過環境變數**設定，於程序啟動時讀取一次（`Config::load` 位於
`packages/core/src/config.rs`，另有少數變數在首次使用時讀取）。沒有設定檔：
變更變數後重新啟動伺服器即可生效。

此頁面是伺服器程式碼讀取的一切設定的參考，依關注領域分組。mock 專用與
執行期變數也一併納入以求完整。

## 參考表

| 變數 | 預設值 | 用途 |
| --- | --- | --- |
| `DATABASE_URL` | *(必填)* | PostgreSQL 連線 URL。 |
| `JWT_SECRET` | *(mock 模式以外必填)* | 用於簽署 JWT 的密鑰。 |
| `ARONA_HOST` | `0.0.0.0` | 綁定位址（退回 `SHITTIM_CHEST_HOST`）。 |
| `ARONA_PORT` | `8420` | 綁定連接埠（退回 `SHITTIM_CHEST_PORT`）。 |
| `ARONA_DATA_DIR` | 未設定 | 本機資料目錄。 |
| `ARONA_ADMIN_TOKEN` | 未設定 | `/api/admin/*` 與管理 RPC 方法的 Bearer token。 |
| `ARONA_REGISTRATION_OPEN` | `0` | Truthy（`1`／`true`／`yes`／`on`）時開放註冊。 |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | 每個 key 的記憶體內每分鐘請求限制；`0` 會阻擋一切。 |
| `MOCK_MODE` | 未設定 | 存在（任意值）即啟用開發用 mock 模式。 |
| `MOCK_SEED_PATH` | 未設定 | mock 模式中執行的原始 SQL seed 檔案。 |
| `ARONA_MEMORY_URL` | 未設定 | Philia 記憶 gateway 的 WebSocket URL。 |
| `ARONA_MEMORY_TOKEN` | 未設定 | 記憶 gateway 的 token。 |
| `ARONA_MEMORY_WRITEBACK` | `true` | 是否將完成的聊天回合寫回記憶；接受 `true`／`false`（其他任何值退回預設值）。 |
| `ARONA_AGENT_NAME` | `arona-agent` | GPU 節點 agent 的身分識別。 |
| `ARONA_PANEL_URL` | `localhost:8080` | agent 連線的目標（`ws://<panel_url>/ws/agent`）。 |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | 用於解析 `evernight://` backend URL 的本機 evernight agent。 |
| `ARONA_MISTRALRS` | 未設定 | 存在時強制 Gguf 模型方案使用 Mistral.rs engine。 |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | 產生的 llama.cpp 模型伺服器所綁定的介面。 |
| `HF_ENDPOINT` | `https://huggingface.co` | 模型下載用的 Hugging Face 基礎 URL。 |
| `GITHUB_TOKEN` | 未設定 | GitHub 模型登錄檔的存取 token。 |
| `RUST_LOG` | 未設定 | 追蹤過濾器，例如 `info` 或 `arona=debug,info`。 |

## 必填變數

### `DATABASE_URL`

PostgreSQL 連線 URL。**必填**：當其為空時，伺服器會以
`FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.` 結束，
且 `migrate` CLI 子命令拒絕執行。schema 會由 `serve` 在啟動時自動建立／更新。

### `JWT_SECRET`

用於簽署 `auth.login` 與 `auth.register` 簽發的 access/refresh JWT 配對的
密鑰。**正式環境必填**：程式碼內嵌了開發用後備值
（`dev-secret-not-for-production-use-only-32chars`），但除非 `MOCK_MODE=1`，
否則伺服器拒絕以該值啟動：

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

請使用長且隨機的值（例如 `openssl rand -hex 32`）。

## 伺服器

### `ARONA_HOST`／`ARONA_PORT`

gateway 的綁定位址與連接埠。為相容舊版，它們會退回
`SHITTIM_CHEST_HOST`／`SHITTIM_CHEST_PORT`；最終預設值為 `0.0.0.0:8420`。

### `ARONA_DATA_DIR`

選用的本機資料目錄，隨 app state 攜帶，供需要暫存位置的元件使用。
預設未設定。

## 安全性與存取控制

### `ARONA_ADMIN_TOKEN`

守護 `/api/admin/*` HTTP 路由（`POST/GET/DELETE
/api/admin/backends`、`/api/admin/aliases`）與
`billing.plan.set`／`billing.video.pricing.set` RPC 方法的 Bearer token。
當它**未設定**時，這些路由每一個都會以 `Admin access required`（401）拒絕——
沒有預設值。請在啟動伺服器前將其設為強大的隨機值。

### `ARONA_REGISTRATION_OPEN`

透過 `auth.register` 開放自行註冊。Truthy 值嚴格限定為 `1`、`true`、`yes`、
`on`（不分大小寫）；其他任何值——包括 `0`、`false`、`off`，或未設定／空變數——
保持關閉。此旗標在啟動時讀取一次。**第一個註冊的使用者永遠允許**（即使註冊
關閉）並成為管理員。

### `ARONA_API_RATE_LIMIT_RPM`

每個 key 的記憶體內滑動視窗速率限制（每分鐘請求數），套用於每個已認證的
`/v1/*` 請求（chat、embeddings、video、models），以 API-key 雜湊為鍵
（JWT 可用的 `GET /v1/models` 則以 `u:<email>` 標籤為鍵）。RPC 平面不受此
限制器約束——只有 `/v1/*` 的 auth extractor 會呼叫它。預設 `60`。此值會被
解析一次存入程序級 `OnceLock`。**值為 `0` 會阻擋每個請求**——檢查邏輯是
`entry.len() >= rpm`，所以 `0` 時沒有任何請求能通過。這是 gateway 全域限制；
billing tiers 會在此之上施加各自的 per-key 限制。

## 開發

### `MOCK_MODE`

開發用 mock 模式，以**存在與否**啟用——檢查邏輯是
`std::env::var("MOCK_MODE").is_ok()`，所以*任何*值（包括 `0` 或「設定了但為空」）
都會啟用。它會：

- 解除 `JWT_SECRET` 的必要性（內建開發密鑰變成可接受）；
- seed 四個示範帳號（`demiurge@celestia.world`、`momoi@celestia.world`、
  `midori@celestia.world`、`yuzu@celestia.world`，密碼 `33550336`）；
- 在綁定 listener 之前等待 seed 完成。

切勿在正式環境使用。

### `MOCK_SEED_PATH`

僅限 mock 模式，指向一個原始 SQL 檔案，執行它取代內建帳號 seed
（語句以 `;` 分割，`--` 註解略過）。若檔案無法讀取，則退回內建 seed。

## 記憶 gateway

### `ARONA_MEMORY_URL`／`ARONA_MEMORY_TOKEN`／`ARONA_MEMORY_WRITEBACK`

長期記憶 gateway（entelecheia Philia）的設定。除非 `ARONA_MEMORY_URL` 與
`ARONA_MEMORY_TOKEN` 都已設定且非空，否則記憶**完全停用**。啟用時：

- 完成的聊天回合會被 recall 並注入為 context，且
- `ARONA_MEMORY_WRITEBACK`（預設 `true`）控制是否將完成的回合
  寫回記憶服務；`0` 或 `false` 停用寫回。

記憶失敗絕不阻擋聊天；結果狀態會反映在 `X-Arona-Memory` 回應 header
（`enabled`／`disabled`／`offline`）。

## Agent 身分與叢集

### `ARONA_AGENT_NAME`／`ARONA_PANEL_URL`

GPU 節點 agent 二進位檔（`_agent`）的身分識別：`ARONA_AGENT_NAME`
（預設 `arona-agent`）以 agent 名稱／id 回報給 panel，而
`ARONA_PANEL_URL`（預設 `localhost:8080`）是 agent 連線的目標
（`ws://<panel_url>/ws/agent`）。

agent 自身的 HTTP API **硬編碼**綁定 `0.0.0.0:5790`——沒有對應的
綁定位址環境變數。

### `ARONA_AGENT_BIND_ADDR`

當 agent 部署 Gguf 模型時，**產生的 llama.cpp 模型伺服器**所綁定的介面，
以便其他機器可以連到該 engine（例如 `0.0.0.0`）。預設 `127.0.0.1`。
請注意這*不是* agent HTTP API 的綁定（後者固定為 `0.0.0.0:5790`）。

## Evernight bridge

### `ARONA_EVERNIGHT_URL`

本機 evernight agent 的 WebSocket URL，用於將 `evernight://` backend URL
解析為本機 TCP 轉發。預設 `ws://127.0.0.1:3001/ws`。

## 模型執行環境與下載

### `ARONA_MISTRALRS`

存在（任意值）時，強制 Gguf 模型方案使用 Mistral.rs engine，否則預設為
llama.cpp。存在語意與 `MOCK_MODE` 相同。

### `HF_ENDPOINT`

Hugging Face 模型下載（`hf:` 來源）的基礎 URL，預設
`https://huggingface.co`。當 huggingface.co 無法連線時，可設為
`https://hf-mirror.com` 之類的鏡像。由模型下載器讀取；結尾斜線會被去除。

### `GITHUB_TOKEN`

GitHub 模型登錄檔（`gh:` 來源）用於 API 存取的 token。預設未設定；
沒有它時會套用 GitHub API 速率限制。

## 日誌

### `RUST_LOG`

`tracing_subscriber` 在啟動時套用的標準追蹤過濾器，例如
`info` 或 `arona=debug,info`。遵循一般的 `RUST_LOG` 語意
（`error`／`warn`／`info`／`debug`／`trace`、依目標覆寫）。

## 預設值一覽

| 設定 | 預設值 |
| --- | --- |
| 綁定位址／連接埠 | `0.0.0.0:8420` |
| 每個 key 的 API 速率限制 | 60 RPM |
| Agent 名稱 | `arona-agent` |
| Panel URL | `localhost:8080` |
| 記憶寫回 | 開啟 |
| 註冊 | 關閉 |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
