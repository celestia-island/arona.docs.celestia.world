---
title: "測試"
description: "Arona 測試金字塔——單元測試、封閉式整合、PostgreSQL 閘控整合、live-server 冒煙測試、mock 伺服器，與真實憑證冒煙紀律。"
---

# 測試

Arona 的測試分層安排，讓預設的 `cargo test` 執行快速、封閉且不需要資料庫
或網路，而較重的套件是明確的 opt-in，會實際操練真實 wire 介面與真實
PostgreSQL。此頁面說明各層、執行它們的命令，以及 workspace 圍繞真實憑證
冒煙運行的紀律。

## 單元測試

覆蓋率的大宗是 `packages/core/src` 內的純單元測試：
217 個 `#[test]`／`#[tokio::test]` 函式，外加 `packages/agent` 與
`packages/cli` 中約 23 個。它們以：

```bash
cargo test --workspace
```

運行。無網路、無資料庫。重點套件：

- **auth.rs** — 密碼政策（≥8 字元且 4 個字元類別中 ≥3 類）、原始
  INSERT/REVOKE SQL 中的 `::uuid` 轉換、請求預設值，以及退回 `false`
  的 admin 旗標讀取。
- **billing/mod.rs** — cost *或* token 維度的 quota 數學、每月視窗
  （`month_start`、`seconds_until_month_end`）、速率限制上限
  （只在*達到* RPM 時觸發、`None` = 無限制）、每月用量／tier／key 視窗
  查詢的 SQL 形狀守衛，以及偏好 upstream 回報數字的 `estimate_usage`。
- **routing/mod.rs** — alias 解析、`:latest` 尾綴比對、provider
  提示、最少負載選擇與對話釘選。
- **gateway/mod.rs** — agent-backend 註冊：註冊 `agent-{model_id}`、
  重新註冊取代（而非重複）、取消註冊還原路由器。

## 封閉式整合（always-run、免 DB）

`packages/core/tests/gateway_integration.rs` 包含三個 always-run 測試，
操練真實的序列化／契約邏輯而不碰資料庫：

- **A1** — JSON-RPC id 回聲序列化契約：數字、字串與 null 請求 id
  以類型保真度往返 plana 的 `Id` enum。
- **A2** — admin 閘門錯誤碼契約：`AUTH_ERROR`（-32005、匿名）
  與 `ADMIN_REQUIRED`（-32007、已認證的非管理員）保持不同、位於
  實作定義的範圍內，且絕不與 plana 的 codes 或計費的 `QUOTA_ERROR`
  （-32006）衝突。
- **A3** — `estimate_usage`：upstream 回報的 usage 原樣優先；沒有它時
  本機 tokenizer 估算產生非零的 prompt/completion 計數，其總和為
  兩者之和。

`packages/core/tests/smoke.rs` 額外加入三個 always-run 測試：硬體
偵測、模型登錄檔根路徑，與 `MOCK_MODE=1` 下的設定預設值。

## PG 閘控整合

完整的 in-process gateway 套件——`packages/core/tests/gateway_integration.rs`——
在隨機 loopback 連接埠上轉起完整的 axum 路由器，透過真實管理 API 註冊
可拋棄的 OpenAI 相容 mock upstreams，並以 reqwest 驅動 wire 介面。
因為 `AuthManager` 在每個路徑都與 PostgreSQL 對話（即使 `MOCK_MODE=1`
也只是把帳號 seed *進資料庫*），此套件以 `ARONA_TEST_PG=1` 閘控、預設
略過。10 個測試：

- **T1** register + login + `keys.create`／`keys.list`（列表中原 key
  遮罩、`arona-` 前綴）。
- **T2** REST chat 與 usage-record 持久化進 PostgreSQL。
- **T3** 跨 wire 的 JSON-RPC id 回聲（成功與錯誤路徑）。
- **T4** `agents.list` 上的 admin 閘門：匿名 → `AUTH_ERROR`、非管理員 →
  `ADMIN_REQUIRED`。
- **T5** upstream 401 → HTTP 502 `bad_gateway` 並帶 upstream detail。
- **T6** 註冊時探測發佈模型（無靜態模型清單時，模型 10s 內出現在
  `GET /v1/models`）。
- **T7** 透過 `chat.send` 的對話持久化（兩個回合都落在
  `conversations.get`）。
- **T8** free-tier 計費閘門：每 key 10 RPM，視窗內第 11 個請求是
  429 `rate_limit_exceeded`。
- **T9** 帶終端 usage（自 upstream 記錄）的 SSE 串流。
- **T10** 格式錯誤 JSON → 400；未知模型 → 404 `model_not_found`。

用 module docs 中的可拋棄 Postgres 一行命令運行它
（gateway_integration.rs:18-26）：

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

這些只是可拋棄測試容器的範例憑證——切勿指向真實資料庫。

## Live-server 冒煙

`packages/core/tests/auth_flow.rs` 對**live** Arona 伺服器走完整的
`register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` 鏈，映照已部署的認證迴圈。它預設 `#[ignore]`——一般的
`cargo test` 執行絕不碰網路。明確運行它：

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

旋鈕：

- `ARONA_TEST_URL` — 基礎 URL（預設 `http://127.0.0.1:8420`）。
- `ARONA_TEST_EXPECT_CHAT=1` — 硬性斷言 `POST /v1/chat/completions` 回傳
  200。沒有它，測試只斷言認證通過（非 401/403），因為目標環境可能沒有
  設定推理 provider。

套件也包含負面測試：未認證的聊天補全與未認證的 `GET /v1/models` 都必須
以 401 被拒絕。

## Mock 伺服器

`scripts/mock/server.py` 是以 aiohttp 為基礎的 OpenAI 相容假伺服器，
供快速入門與冒煙執行使用。它提供 `POST /v1/chat/completions`
（非串流與 SSE）、`GET /v1/models`、`GET /api/health`、位於 `/api/rpc`
的 JSON-RPC WebSocket/HTTP 介面、位於 `/api/rpc/events` 的 SSE sidecar，
以及回傳 mock API key 的 `GET /api/test-key`，讓其他服務可以發現它。
預設監聽連接埠 8429（`ARONA_MOCK_PORT` 覆寫連接埠、`ARONA_MOCK_HOST`
覆寫 host）。[快速入門](quickstart.md) 用它搭起一個無真實模型 provider
的端到端環境。

## 真實憑證冒煙紀律

對真實 providers（DeepSeek／GLM）的冒煙運行刻意**不是**儲存庫測試——
它們需要真實憑證與真實花費，所以不能存在於 CI 或 git 樹中。workspace
慣例（記錄於 gateway_integration module docs，gateway_integration.rs:54-55）
是：

- 證據檔案位於 `/mnt/work/arona-pr*-smoke.md`——workspace 本機，
  絕不提交進 git。
- 憑證只來自環境；預算保持小額。
- 每個觸及推理路徑的 PR 都會有一份書面證據記錄。

mock 伺服器是這些運行在 CI 與本機開發中的替代品；真實憑證冒煙是
release 時的人工步驟。

## CI

`.github/workflows/ci.yml` 在組織的自託管 runners
（`[self-hosted, linux, x64, local]`）上運行 `cargo fmt`、`cargo clippy`、`cargo test
--workspace` 與 `cargo-deny`；`ci-hosted.yml` 在 GitHub-hosted
runners 上鏡像相同的檢查。`.github/workflows/docs.yml` 以 lagrange 建置
此 docs 站，並在推送到 `docs/**` 時部署到 GitHub Pages。

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
