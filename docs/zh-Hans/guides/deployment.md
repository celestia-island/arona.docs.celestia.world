---
title: "部署"
description: "从发布产物、脚本或源码安装 arona-server；在裸机上用 systemd、在 Docker Compose 中或在 malkuth 守护下运行；并安全升级。"
---

# 部署

Arona 以**单个自包含 Rust 二进制**形式发布，由 `_cli` crate 构建。发布工作流
将其发布为 `arona-server-<tag>-linux-x86_64`
（`.github/workflows/release.yml`），同一个二进制承担两种角色：

- **API 服务器** —— `serve`（gateway，`migrate` 显式应用 schema 迁移）；
- **模型工具** —— `install`（硬件检测）、`status`、`deploy <model>`、
  `download <repo>` 和 `connect <panel-url>`。

服务器需要 PostgreSQL。schema 在启动时自动创建并迁移，因此部署基本就是
「拿到二进制、指向数据库、运行它」。先从[快速上手](./quickstart.md)完成端到端
体验，再回到本页了解生产环境布局。

## 要求

- **Linux x86_64** 使用预构建发布产物；Rust 1.91+ 支持的任意平台都可以从源码
  构建。
- **PostgreSQL** —— Docker Compose 示例使用 `postgres:16-alpine`；任意近期
  版本均可。没有 `DATABASE_URL` 服务器拒绝启动。
- **`curl`、`python3`、`ca-certificates`** 如果使用安装脚本
  （在 Debian/RedHat 上脚本会通过 apt/dnf 自行安装）。
- 一个可写的二进制位置（例如 `/usr/local/bin`），以及——如果要运行模型下载
  ——`ARONA_DATA_DIR` 下模型缓存所需的磁盘空间。

## 安装

### 从发布产物安装

下载你所需 tag 的产物并赋予执行权限：

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### 从安装脚本安装

仓库附带 `scripts/install.sh`：它从 GitHub API 解析最新发布 tag（或遵循
`ARONA_VERSION` 覆盖），下载匹配的产物，默认安装为 `~/.local/bin` 下的
`arona-server`（可用 `ARONA_BIN_DIR` 覆盖目录），并打印后续步骤：

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### 从源码构建

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` 正是发布工作流和 Dockerfile 所构建的内容。

## 配置

所有配置都通过环境变量进行；[配置参考](./configuration.md) 记录了每个变量，
仓库根目录的 `.env.example` 是一个可用的起点。最小集合：

| 变量 | 必填？ | 用途 |
| --- | --- | --- |
| `DATABASE_URL` | 是 | PostgreSQL 连接串；未设置时服务器退出。 |
| `JWT_SECRET` | 是 | token 签名密钥；除非 `MOCK_MODE=1`，否则服务器拒绝使用内置开发密钥运行。 |
| `ARONA_ADMIN_TOKEN` | 强烈建议 | `/api/admin/*` 路由的共享 Bearer token；没有它这些路由总是返回 401。 |

可选：`ARONA_HOST`（默认 `0.0.0.0`）、`ARONA_PORT`（默认 `8420`）、
`ARONA_REGISTRATION_OPEN`（truthy `1`/`true`/`yes`/`on`——开放注册；第一个
注册用户成为管理员）、`ARONA_DATA_DIR`（模型缓存根目录）、
`ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`
（记忆网关）、`ARONA_API_RATE_LIMIT_RPM`（每 key 限流）以及 `RUST_LOG`
（追踪过滤器）。

## 数据库迁移

`serve` 在启动时连接数据库并应用所有待处理的 schema 迁移
（`init_database` → `Migrator::up`），因此没有单独的部署步骤。要显式应用
迁移——例如在首次启动前验证它们——运行：

```bash
arona-server migrate
```

数据库用户需要对目标 schema 具有 **`CREATE` 权限**，因为启动迁移会创建表。
除了 schema 之外没有数据迁移：数据库*就是*状态，因此在升级前请备份它
（见 [升级与备份](#upgrade-and-backup)）。

## 裸机 + systemd

示例 unit 文件（`/etc/systemd/system/arona.service`）。**以下所有密钥值都是
占位符——使用前请替换 `CHANGE_ME`：**

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

然后启用并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

仓库根目录附带一个 `docker-compose.yml`，包含两个服务：

- **`arona`** —— 从 `Dockerfile` 构建镜像（tag `arona:local`），发布
  `${ARONA_PORT:-8420}:8420`，并且要求 `DATABASE_URL`、`JWT_SECRET` 和
  `ARONA_ADMIN_TOKEN`——如果 `.env` 中缺少任何一个，Compose 会以 `:?` 风格
  的错误快速失败。其 healthcheck 运行 `curl -fsS http://127.0.0.1:8420/readyz`。
- **`postgres`** —— `postgres:16-alpine`，仅占位符凭据
  （`POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`；默认密码为
  `change-me`——请覆盖它）和一个命名卷 `arona-pgdata`。

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

捆绑的 postgres 服务在 Compose 网络内通过主机名 `postgres` 访问。`Dockerfile`
在 `rust:1.91-slim-bookworm` 构建器中构建 `_cli` 和 `_agent`，在
`debian:bookworm-slim` 运行时中安装 `ca-certificates`、`curl` 和 `python3`，
将两个二进制复制进去，暴露端口 8420（服务器）和 5790（捆绑的 `_agent` 的
本地 API），并以 `arona-server serve` 作为入口点运行。

## 使用 malkuth 守护部署（工作区约定）

在本工作区，arona 在 **malkuth** 守护下以 `arona-malkuth.service` 运行。该模式
适用于此处部署的任何服务：

- malkuth 守护 `arona-server serve` 进程——它负责派生、probe、失败重启，并在
  关闭前排空连接。
- 每个受守护的服务只通过每服务一个**info 端口**和一个**proxy 端口**暴露；
  服务本身绑定 `127.0.0.1`，网络永远无法直接到达。
- 守护进程以 `--watch <deployment-path>` 启动：当该路径下的二进制发生变化
  ——例如升级复制了新文件覆盖它——malkuth 执行**滚动重启**，一次一个实例，
  先排空连接。

运维影响：

- 在守护进程代理后面运行时绑定 `ARONA_HOST=127.0.0.1`；代理是唯一的网络
  入口点。
- 升级就是「把新二进制复制到部署路径」：watcher 会触发滚动重启。你也可以
  显式重启受守护的 unit。
- 将守护进程的 health check 指向 `/readyz`（或 `/api/health`）；见
  [健康探测](#health-probes)。

## 健康探测

服务器暴露两个无需认证的健康系列（[运维指南](./operations.md) 中也有介绍）：

- `GET /healthz`、`GET /readyz`（别名）和 `GET /v1/health` 返回
  `200` 及 `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`。
- `GET /api/health` 返回 plana 的 `HealthResponse` 结构：`status`、
  `version`、`kind`、`uptime`（秒）、`network`、`build_hash` 和
  `engine_version`。

将负载均衡器、守护进程和容器 healthcheck 指向 `/readyz`；需要 uptime 和网络
详情时使用 `/api/health`。

## 升级与备份

- **保留旧二进制。** 覆盖 `arona-server` 之前先复制一份——
  `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak`——
  这样如果新版本无法启动，可以立即回滚二进制。
- **备份 PostgreSQL。** 数据库保存所有状态——backend、agent 节点、用户、key、
  会话和用量——而唯一的自动 schema 变更就是启动迁移。每次升级前
  `pg_dump` 一次 `arona` 数据库。
- **数据库用户需要 `CREATE` 权限**，因为迁移在启动时运行；只读角色无法启动
  服务器。
- **优雅关闭。** 服务器在收到 SIGINT/SIGTERM 时排空进行中的连接，因此优先
  使用 `systemctl restart arona` 或守护进程重启，而不是 `kill -9`。

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
