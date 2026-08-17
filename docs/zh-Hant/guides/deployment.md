---
title: "部署"
description: "從 release asset、安裝腳本或原始碼安裝 arona-server；以 bare metal 搭配 systemd、Docker Compose 或 malkuth 監督運行；並安全地升級。"
---

# 部署

Arona 以**單一自包含 Rust 二進位檔**交付，由 `_cli` crate 建置。release
workflow 將其發佈為 `arona-server-<tag>-linux-x86_64`
（`.github/workflows/release.yml`），同一個二進位檔身兼兩職：

- **API 伺服器** — `serve`（gateway；`migrate` 明確套用 schema
  migration）；
- **模型工具** — `install`（硬體偵測）、`status`、`deploy
  <model>`、`download <repo>`、`connect <panel-url>`。

伺服器需要 PostgreSQL。schema 會在啟動時自動建立並 migration，所以部署
大致就是「取得二進位檔、指向資料庫、執行它」。先從[快速入門](./quickstart.md)
看完整的端到端流程，再回來看正式環境的配置。

## 需求

- **Linux x86_64** 以使用預先建置的 release asset；任何 Rust 1.91+ 支援的
  平台都可以從原始碼建置。
- **PostgreSQL** — Docker Compose 範例使用 `postgres:16-alpine`；
  任何近期版本都可用。沒有 `DATABASE_URL` 時伺服器拒絕啟動。
- **`curl`、`python3`、`ca-certificates`**（若你使用安裝腳本；
  在 Debian/RedHat 上腳本會自己透過 apt/dnf 安裝它們）。
- 一個可寫入的二進位檔位置（例如 `/usr/local/bin`），以及——若你執行
  模型下載——`ARONA_DATA_DIR` 下模型快取的磁碟空間。

## 安裝

### 從 release asset

下載你要的 tag 對應的 asset 並使其可執行：

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### 從安裝腳本

儲存庫附有 `scripts/install.sh`：它會從 GitHub API 解析最新的 release tag
（或尊重 `ARONA_VERSION` 覆寫），下載相符的 asset，預設安裝為
`~/.local/bin` 中的 `arona-server`（用 `ARONA_BIN_DIR` 覆寫目錄），
並印出接下來的步驟：

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### 從原始碼

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` 正是 release workflow 與
Dockerfile 所建置的內容。

## 設定

所有設定都透過環境變數進行；[設定參考](./configuration.md) 記錄了每個
變數，而儲存庫根目錄的 `.env.example` 是可用的起點。最低需求組合：

| 變數 | 必填？ | 用途 |
| --- | --- | --- |
| `DATABASE_URL` | 是 | PostgreSQL 連線字串；未設定時伺服器結束。 |
| `JWT_SECRET` | 是 | 簽署 token 的密鑰；除非 `MOCK_MODE=1`，否則伺服器拒絕使用內建開發密鑰運行。 |
| `ARONA_ADMIN_TOKEN` | 強烈建議 | `/api/admin/*` 路由的共用 bearer token；沒有它這些路由一律回傳 401。 |

選用：`ARONA_HOST`（預設 `0.0.0.0`）、`ARONA_PORT`（預設 `8420`）、
`ARONA_REGISTRATION_OPEN`（truthy `1`／`true`／`yes`／`on`——開放註冊；
第一個註冊的使用者成為管理員）、`ARONA_DATA_DIR`（模型快取
根目錄）、`ARONA_MEMORY_URL`／`ARONA_MEMORY_TOKEN`／`ARONA_MEMORY_WRITEBACK`
（記憶 gateway）、`ARONA_API_RATE_LIMIT_RPM`（per-key 速率限制）與
`RUST_LOG`（追蹤過濾器）。

## 資料庫 migration

`serve` 會連上資料庫並在啟動時套用所有待處理的 schema migration
（`init_database` → `Migrator::up`），所以沒有獨立的部署步驟。
若要明確套用 migration——例如在首次啟動前驗證——執行：

```bash
arona-server migrate
```

資料庫使用者需要目標 schema 的 **`CREATE` 權限**，因為啟動 migration
會建立資料表。除了 schema 之外沒有資料 migration：資料庫*就是*狀態，
所以升級前請備份它（見[升級與備份](#upgrade-and-backup)）。

## Bare metal 搭配 systemd

範例 unit 檔（`/etc/systemd/system/arona.service`）。**以下所有密鑰值都是
佔位符——使用前請替換 `CHANGE_ME`：**

```ini
[Unit]
Description=Arona API server
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=arona
Environment=DATABASE_URL=postgres://arona:CHANGE_ME@127.0.0.1:5432/arona
Environment=JWT_SECRET=CHANGE_ME
Environment=ARONA_ADMIN_TOKEN=CHANGE_ME
Environment=ARONA_HOST=0.0.0.0
Environment=ARONA_PORT=8420
Environment=RUST_LOG=info
# Optional:
# Environment=ARONA_REGISTRATION_OPEN=1
# Environment=ARONA_MEMORY_URL=ws://127.0.0.1:8424/ws
# Environment=ARONA_MEMORY_TOKEN=CHANGE_ME
# Environment=ARONA_MEMORY_WRITEBACK=1
# Environment=ARONA_DATA_DIR=/var/lib/arona
ExecStart=/usr/local/bin/arona-server serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

接著啟用並啟動它：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

儲存庫根目錄附有 `docker-compose.yml`，含兩個服務：

- **`arona`** — 從 `Dockerfile` 建置映像（tag `arona:local`），
  發佈 `${ARONA_PORT:-8420}:8420`，且需要 `DATABASE_URL`、
  `JWT_SECRET` 與 `ARONA_ADMIN_TOKEN`——若 `.env` 缺少任何一個，
  Compose 會以 `:?` 型錯誤快速失敗。其 healthcheck 執行
  `curl -fsS http://127.0.0.1:8420/readyz`。
- **`postgres`** — `postgres:16-alpine`，使用僅佔位符的憑證
  （`POSTGRES_USER`／`POSTGRES_PASSWORD`／`POSTGRES_DB`；預設
  密碼是 `change-me`——請覆寫它）與一個具名 volume
  `arona-pgdata`。

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

內建的 postgres 服務在 Compose 網路內可透過 host `postgres` 連到。
`Dockerfile` 在 `rust:1.91-slim-bookworm` builder 中建置 `_cli` 與
`_agent`，在 `debian:bookworm-slim` runtime 中安裝 `ca-certificates`、
`curl` 與 `python3`，將兩個二進位檔複製進去，暴露連接埠 8420
（伺服器）與 5790（內建 `_agent` 的本機 API），並以 `arona-server serve`
作為 entrypoint 運行。

## 以 malkuth 監督部署（workspace 慣例）

在此 workspace 中，arona 以 **malkuth** 監督下的
`arona-malkuth.service` 運行。此模式適用於部署在此的任何服務：

- malkuth 監督 `arona-server serve` 程序——它會產生、探測、失敗時重啟，
  並在關閉前排空連線。
- 每個受監督的服務只透過每個服務的 **info port** 與每個服務的 **proxy
  port** 對外暴露；服務本身綁定 `127.0.0.1`，永遠無法從網路直接連到。
- 監督器以 `--watch <deployment-path>` 啟動：當該路徑上的二進位檔
  變更時——例如升級複製了新檔案覆蓋上去——malkuth 會執行**滾動重啟**
  （rolling restart），一次一個執行個體，先排空連線。

營運後果：

- 在監督器 proxy 後方運行時，將 `ARONA_HOST=127.0.0.1` 綁定；
  proxy 是唯一對外暴露的進入點。
- 升級就是「將新二進位檔複製到部署路徑」：watcher 會觸發滾動重啟。
  你也可以明確重啟受監督的 unit。
- 將監督器的健康檢查指向 `/readyz`（或 `/api/health`）；見
  [健康探測](#health-probes)。

## 健康探測

伺服器暴露兩組不需認證的健康端點（兩者也涵蓋於
[營運指南](./operations.md)）：

- `GET /healthz`、`GET /readyz`（同義）與 `GET /v1/health` 回傳
  `200` 並帶 `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`。
- `GET /api/health` 回傳 plana 的 `HealthResponse` 形狀：`status`、
  `version`、`kind`、`uptime`（秒）、`network`、`build_hash` 與
  `engine_version`。

將負載平衡器、監督器與容器 healthcheck 指向 `/readyz`；需要 uptime 與
network 詳細資料時使用 `/api/health`。

## 升級與備份

- **保留前一個二進位檔。** 覆寫 `arona-server` 前，先複製一份到旁邊——
  `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak`——
  這樣若新版本無法啟動，你可以立刻回滾二進位檔。
- **備份 PostgreSQL。** 資料庫保存所有狀態——backends、agent
  節點、使用者、keys、對話與用量——而唯一自動的 schema 變更就是啟動
  migration。每次升級前 `pg_dump` 一次 `arona` 資料庫。
- **資料庫使用者需要 `CREATE` 權限**，因為 migration 在啟動時執行；
  唯讀角色無法啟動伺服器。
- **優雅關閉。** 伺服器會在 SIGINT/SIGTERM 時排空進行中的連線，
  所以請優先使用 `systemctl restart arona` 或監督器重啟，而非 `kill -9`。

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
