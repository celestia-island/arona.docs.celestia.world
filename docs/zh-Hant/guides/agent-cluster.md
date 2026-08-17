---
title: "Agent 叢集"
description: "多節點 GPU 叢集——用 CLI 下載模型權重、在 GPU 節點上運行 _agent 二進位檔，並透過 agents.* RPC 介面驅動部署。"
---

# Agent 叢集

Arona 的部署故事分為兩半。**panel**（`arona` 伺服器二進位檔）掌管路由、
計費、認證與管理平面。每個 GPU 節點運行一個 **`_agent` 程序**，掌管模型
權重與本機的服務程序。Agents 開啟一條長效 WebSocket 連回 panel 的 agent
控制平面（`/ws/agent`）；panel 沿該 socket 推送 `deploy`／`stop` 命令，
agent 則回傳下載進度、heartbeats 與命令結果。模型在 agent 上運行後，
panel 會將其註冊為可路由的 backend，使 `/v1/*` 與 RPC 流量可以觸及——
控制平面走 WebSocket，資料平面則是對 agent 本機 engine 連接埠的純 HTTP。

## 下載模型權重（CLI）

`_cli` 二進位檔從 HuggingFace、ModelScope 或 GitHub releases 下載模型權重：

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **Repo 形式** — `hf:owner/repo`（預設；裸 `owner/repo` 也解析為
  HuggingFace）、`ms:owner/repo`（ModelScope）、`gh:owner/repo[:tag]`
  （GitHub release，tag 選用）。長前綴 `huggingface:`、
  `modelscope:` 與 `github:` 也可接受；沒有斜線的裸 id 解析為
  Ollama 登錄檔（`packages/core/src/models/download.rs:21-28,55-86`）。
- **`--filter <glob|prefix>`** — 可重複；只下載符合 glob（或前綴）的
  manifest 檔案。沒有 filter 時選取**整個 repo**。
- **確認** — 未過濾的下載在開始前一律詢問 `Continue? [y/N]`，除非傳入
  `--yes`。過濾的下載永不提示；當選取的總量達到或超過 2 GiB 時，
  改印出資訊性橫幅（`NO_CONFIRM_THRESHOLD`、`packages/cli/src/main.rs:12-15,
  439-464`）。
- **`--out <dir>`** — 覆寫預設目的地
  `~/.arona/models/<repo-id>`。
- **`--revision <rev>`** — 覆寫任何行內的 `:rev` 尾綴
  （`hf:owner/repo:rev`）。
- **續傳** — 中斷的下載會自動續傳：保留 `.part` 檔案，下載從其目前長度
  以 Range 請求繼續；完整檔案依大小略過，且當 manifest 攜帶 digest 時，
  以 SHA-256 驗證（`packages/cli/src/main.rs` 的 `verify_sha256`／`summarize`）。
- **重試** — 網路錯誤最多重試 3 次、間隔 5 秒；非網路錯誤立即失敗
  （`packages/cli/src/main.rs:277-302`）。
- **`HF_ENDPOINT`** — 切換 HuggingFace 基礎 URL，例如鏡像：

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

其他 CLI 命令（`packages/cli/src/main.rs:28-53`）：

| 命令 | 用途 |
| --- | --- |
| `install` | 一鍵環境設定：偵測硬體設定檔並印出 backend／量化建議。 |
| `status` | 印出硬體設定檔。 |
| `deploy <model>` | 在本機解析模型並回報是否已快取。 |
| `download` | 下載模型權重（見上文）。 |
| `serve` | 啟動 API 伺服器（panel）。 |
| `connect <url>` | 連到管理 panel。 |
| `migrate` | 執行資料庫 migration。 |

## `_agent` 二進位檔

`_agent` 在每個 GPU 節點上運行，純粹以環境變數設定
（`packages/core/src/config.rs:37-40`）：

| 變數 | 預設值 | 意義 |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | 唯一的節點 id；panel 以它作為 `agent_id`。 |
| `ARONA_PANEL_URL` | `localhost:8080` | Panel 的 `host:port`；agent 連到 `ws://{ARONA_PANEL_URL}/ws/agent`。 |

完整的環境變數參考（panel 端變數、資料庫、密鑰）見
[設定](./configuration.md)。

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

行為：

- **控制連線** — agent 連回 `ws://{ARONA_PANEL_URL}/ws/agent`
  （`packages/agent/src/panel.rs:23`）。連線時送出 `register` frame，
  攜帶 `agent_name`、`gpu_info` 與已部署模型的清單；panel 將 agent 的
  TCP peer 位址記錄為其 `host`。
- **重連退避** — 從 1 秒開始、倍增直到 60 秒上限
  （`packages/agent/src/panel.rs:27,33-34,63`）。
- **Heartbeats** — 每 30 秒 agent 回報 GPU 使用率、已載入模型數與
  uptime。當 agent 最後一次 heartbeat 早於 30 秒時，panel 視其離線。
- **本機 HTTP API** — 綁定**固定**位址 `0.0.0.0:5790`；沒有
  綁定位址環境變數（`packages/agent/src/main.rs:109`）。panel 將此
  連接埠與 agent 記錄的 host 組合，建出已部署模型的資料平面 URL。
- **命令** — panel 沿 socket 排入 `deploy`／`stop` 命令。`deploy` 命令
  攜帶 `model_id` 與 `stream_id`；下載進度以 `deploy_progress` frames
  在同一 socket 上串流回傳（panel 將其轉發為 `models.progress` SSE
  通知，見[事件與通知](../api/events.md)），最後的 `deploy_result`
  frame 回報本機 engine 的 `port` 與 `backend`。`stop` 以
  `stop_result` 回應。

請在服務監督器（systemd、malkuth、……）下運行 `_agent`，使其自動重連；
panel 能容忍任一端重啟（見下方[節點持續性](#node-persistence)）。

## Agent 控制平面 RPC

整個 agent 介面都以管理員權限控管：每個方法都需要有效的 JWT **且**管理員
帳號（`validate_admin_jwt` 檢查 `is_admin_email`；
`packages/core/src/gateway/rpc.rs:106-118,301-337`）。

| 方法 | 參數 | 回傳 |
| --- | --- | --- |
| `agents.list` | — | 叢集拓撲：`id`、`name`、`host`、`status`（`online`／`offline`）、GPU 摘要、`models`、`last_heartbeat`、`version`、`connected_at`。 |
| `agents.register` | `machine_name`、`version` | `agent_id`、`token`。 |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — 移除節點。 |
| `agents.status` | `agent_id` | `online`、GPU 摘要、`gpu_utilization`、`models`、`host`、`connected_at`、`last_heartbeat`。 |
| `agents.deploy` | `model_id`、`agent_id?` | `{ "ok": true, "stream_id" }` — 空的 `agent_id` 自動鎖定最少負載的節點。 |
| `agents.stop` | `agent_id`、`model_id` | `{ "ok": true }` — 停止部署。 |

`agents.deploy` 回傳 `stream_id`；在呼叫**之前**或緊接其後訂閱
`/api/rpc/events?session=<stream_id>`，以接收 `models.progress` 下載通知
（見[事件與通知](../api/events.md)）。傳輸與認證細節見
[JSON-RPC API](../api/jsonrpc.md)。

## 已部署模型的自動註冊

當 `deploy_result` frame 回報成功時，panel 會將名為 **`agent-{model_id}`**
的 `ExternalApiBackend` 註冊進 gateway 路由器，基礎 URL 為
`http://{agent-host}:{port}`——agent 記錄的 host 加上它回報的 engine
連接埠（`packages/core/src/gateway/server.rs:310-366`、
`packages/core/src/gateway/mod.rs:253-270`）。已部署的模型變成一般可路由的
backend：`/v1/chat/completions`、embeddings 與 RPC chat 都能觸及它，
aliases 適用，健康檢查器會探測它（backend 類型與探測語意見
[Backends](./backends.md)）。

- 重新部署同一個模型（例如在不同 agent 上）會取代先前的 backend。
- 成功的 `stop_result` 會再次取消註冊
  （`packages/core/src/gateway/mod.rs:274-287`）；模型 id 停止解析。

## 放置

沒有明確 `agent_id` 的部署會走最少負載放置
（`packages/core/src/gateway/tunnel.rs:214-229`）：候選者是最近一次
heartbeat 在 30 秒內的 agents，選取**平均 GPU 使用率最低**者。沒有
telemetry 的 agents 排在最後但仍可被選取。若沒有 agent 在線，RPC 失敗，
錯誤為 `No online agents available for deployment`。

在路由端，對話**釘在單一 backend**（session affinity）：對話首次使用的
backend 會被記錄並在後續回合重複使用，因此 per-conversation 狀態例如
執行期 KV 快取能保持熱度（`packages/core/src/routing/mod.rs:31-34,110-138`）。
若被釘選的 backend 變得不健康，路由會降級為全新選擇並重新釘選。

## 節點持續性

Agent 節點存在 `agent_nodes` 資料表（`agent_id`、`machine_name`、
`version`、`host`、`gpu_info`、`models`、`connected_at`、`last_heartbeat`；
`packages/core/src/gateway/tunnel.rs:8-12`）。panel 啟動時會還原已持久化的
列，因此先前註冊的節點在重啟後仍然可見；還原的條目在每個 agent 透過
WebSocket 重連前是**無 sender**的（`packages/core/src/gateway/run.rs:74-85`）。
因此對已還原但未連線的節點部署會失敗，直到其 `_agent` 重連為止。

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
