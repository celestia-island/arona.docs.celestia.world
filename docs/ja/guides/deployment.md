---
title: "デプロイ"
description: "リリースアセット、スクリプト、またはソースから arona-server をインストールし、systemd でのベアメタル、Docker Compose、malkuth 監視下で実行し、安全にアップグレードする方法。"
---

# デプロイ

Arona は `_cli` クレートからビルドされる**単一の自己完結型 Rust バイナリ**として提供されます。リリースワークフローはこれを `arona-server-<tag>-linux-x86_64` として公開し（`.github/workflows/release.yml`）、同じバイナリが 2 つの役割を果たします:

- **API サーバー** — `serve`（gateway。`migrate` はスキーママイグレーションを明示的に適用）;
- **モデルツーリング** — `install`（ハードウェア検出）、`status`、`deploy <model>`、`download <repo>`、`connect <panel-url>`。

サーバーは PostgreSQL を必要とします。スキーマは起動時に自動的に作成・マイグレーションされるため、デプロイはほぼ「バイナリを取得してデータベースに向けて実行する」だけです。エンドツーエンドの手順は[クイックスタート](./quickstart.md)から始め、本番レイアウトはここに戻って確認してください。

## 要件

- プリビルドのリリースアセットには **Linux x86_64**。Rust 1.91+ がサポートする任意のプラットフォームでソースからビルドできます。
- **PostgreSQL** — Docker Compose の例は `postgres:16-alpine` を使用します。最近のバージョンならどれでも動作します。`DATABASE_URL` がないとサーバーは起動を拒否します。
- インストールスクリプトを使う場合は **`curl`、`python3`、`ca-certificates`**（Debian/RedHat ではスクリプトが apt/dnf 経由で自身でインストールします）。
- バイナリ用の書き込み可能な場所（例: `/usr/local/bin`）と、モデルダウンロードを実行する場合は `ARONA_DATA_DIR` 配下にモデルキャッシュ用のディスク容量。

## インストール

### リリースアセットから

希望するタグのアセットをダウンロードして実行可能にします:

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### インストールスクリプトから

リポジトリには `scripts/install.sh` が含まれています。GitHub API から最新のリリースタグを解決し（`ARONA_VERSION` による上書きも可能）、一致するアセットをダウンロードして、デフォルトでは `~/.local/bin` に `arona-server` としてインストールし（ディレクトリは `ARONA_BIN_DIR` で上書き）、次のステップを表示します:

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### ソースから

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` は、リリースワークフローと Dockerfile がビルドするものとまったく同じです。

## 設定

すべての設定は環境変数で行います。[設定リファレンス](./configuration.md)にすべての変数が記載されており、リポジトリルートの `.env.example` は動作する出発点です。最小セット:

| 変数 | 必須? | 用途 |
| --- | --- | --- |
| `DATABASE_URL` | はい | PostgreSQL 接続文字列。未設定だとサーバーは終了します。 |
| `JWT_SECRET` | はい | token署名用シークレット。`MOCK_MODE=1` でない限り、組み込みの開発シークレットでの実行を拒否します。 |
| `ARONA_ADMIN_TOKEN` | 強く推奨 | `/api/admin/*` ルート用の共有 Bearer token。これがないとそれらのルートは常に 401 を返します。 |

任意: `ARONA_HOST`（デフォルト `0.0.0.0`）、`ARONA_PORT`（デフォルト `8420`）、`ARONA_REGISTRATION_OPEN`（truthy な `1`/`true`/`yes`/`on` — サインアップを開放。最初に登録したユーザーが admin になります）、`ARONA_DATA_DIR`（モデルキャッシュのルート）、`ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`（memory gateway）、`ARONA_API_RATE_LIMIT_RPM`（キーごとのレート制限）、`RUST_LOG`（トレーシングフィルター）。

## データベースマイグレーション

`serve` は起動時にデータベースに接続し、未適用のスキーママイグレーションをすべて適用します（`init_database` → `Migrator::up`）。したがって、別途デプロイステップはありません。マイグレーションを明示的に適用するには — 例えば最初の起動前に検証するため — 次を実行します:

```bash
arona-server migrate
```

起動時マイグレーションがテーブルを作成するため、データベースユーザーには対象スキーマに対する **`CREATE` 権限**が必要です。スキーマ以外のデータマイグレーションはありません。データベース*自体が*状態なので、アップグレード前にはバックアップしてください（[Upgrade and backup](#upgrade-and-backup)を参照）。

## systemd でのベアメタル

ユニットファイルの例（`/etc/systemd/system/arona.service`）。**以下のシークレット値はすべてプレースホルダーです — 使用前に `CHANGE_ME` を置き換えてください:**

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

次に有効化して起動します:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

リポジトリルートには 2 つのサービスを持つ `docker-compose.yml` が含まれています:

- **`arona`** — `Dockerfile` からイメージをビルドし（タグ `arona:local`）、`${ARONA_PORT:-8420}:8420` を公開します。`DATABASE_URL`、`JWT_SECRET`、`ARONA_ADMIN_TOKEN` を必要とし、いずれかが `.env` にないと Compose は `:?` スタイルのエラーで即座に失敗します。ヘルスチェックは `curl -fsS http://127.0.0.1:8420/readyz` を実行します。
- **`postgres`** — プレースホルダーのみの認証情報を持つ `postgres:16-alpine`（`POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`。デフォルトのパスワードは `change-me` — 上書きしてください）と、名前付きボリューム `arona-pgdata`。

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

同梱の postgres サービスには Compose ネットワーク内のホスト `postgres` で到達できます。`Dockerfile` は `rust:1.91-slim-bookworm` ビルダーで `_cli` と `_agent` をビルドし、`debian:bookworm-slim` ランタイムに `ca-certificates`、`curl`、`python3` をインストールし、両バイナリをコピーして、ポート 8420（サーバー）と 5790（同梱の `_agent` のローカル API）を公開し、エントリポイントとして `arona-server serve` を実行します。

## malkuth による監視付きデプロイ（ワークスペース規約）

このワークスペースでは arona は **malkuth** 監視下で `arona-malkuth.service` として稼働しています。このパターンはここにデプロイされるすべてのサービスに適用されます:

- malkuth は `arona-server serve` プロセスを監視します — 起動、プローブ、失敗時の再起動、シャットダウン前のコネクションのドレインを行います。
- 監視対象の各サービスは、サービスごとの **info ポート**とサービスごとの **proxy ポート**を通してのみ公開されます。サービス自体は `127.0.0.1` にバインドし、ネットワークから直接到達できません。
- スーパーバイザーは `--watch <deployment-path>` 付きで起動されます。そのパスのバイナリが変更されると — 例えばアップグレードで新しいファイルが上書きコピーされた場合 — malkuth は**ローリング再起動**を実行します。一度に 1 インスタンスずつ、まずコネクションをドレインします。

運用上の影響:

- スーパーバイザーのプロキシの背後で実行する場合は `ARONA_HOST=127.0.0.1` にバインドしてください。プロキシが唯一のネットワーク向けエントリポイントです。
- アップグレードは「新しいバイナリをデプロイパスにコピーする」ことです。ウォッチャーがローリング再起動をトリガーします。監視ユニットを明示的に再起動することもできます。
- スーパーバイザーのヘルスチェックは `/readyz`（または `/api/health`）に向けてください。[Health probes](#health-probes)を参照してください。

## ヘルスプローブ

サーバーは 2 つの認証不要のヘルスファミリーを公開します（どちらも[運用ガイド](./operations.md)で説明されています）:

- `GET /healthz`、`GET /readyz`（エイリアス）、`GET /v1/health` は `200` と `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}` を返します。
- `GET /api/health` は plana の `HealthResponse` シェイプを返します: `status`、`version`、`kind`、`uptime`（秒）、`network`、`build_hash`、`engine_version`。

ロードバランサー、スーパーバイザー、コンテナのヘルスチェックは `/readyz` に向け、uptime と network の詳細が必要な場合は `/api/health` を使用してください。

## アップグレードとバックアップ

- **以前のバイナリを保持する。** `arona-server` を上書きする前に、`cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak` で脇にコピーし、新しいバージョンが起動に失敗した場合に即座にバイナリをロールバックできるようにしてください。
- **PostgreSQL をバックアップする。** データベースがすべての状態（backend、agent ノード、ユーザー、キー、会話、usage）を保持し、自動的なスキーマ変更は起動時マイグレーションだけです。毎回のアップグレード前に `arona` データベースを `pg_dump` してください。
- **データベースユーザーには `CREATE` 権限が必要です**。マイグレーションが起動時に実行されるため、読み取り専用ロールではサーバーを起動できません。
- **グレースフルにシャットダウンする。** サーバーは SIGINT/SIGTERM で進行中のコネクションをドレインするため、`kill -9` よりも `systemctl restart arona` またはスーパーバイザーの再起動を優先してください。

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
