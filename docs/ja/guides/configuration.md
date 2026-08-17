---
title: "設定"
description: "Arona サーバーが読み込むすべての環境変数のリファレンス — デフォルト値とセマンティクス付き。"
---

# 設定

Arona は**完全に環境変数**で設定され、プロセス起動時に一度だけ読み込まれます（`packages/core/src/config.rs` の `Config::load` と、最初の使用時に読み込まれる少数の変数）。設定ファイルはありません。変数を変更したら、サーバーを再起動して反映させてください。

このページは、サーバーコードが読み込むすべての項目を関心事ごとにまとめたリファレンスです。mock 専用とランタイムの変数も完全性のために含めています。

## リファレンステーブル

| 変数 | デフォルト | 用途 |
| --- | --- | --- |
| `DATABASE_URL` | *(必須)* | PostgreSQL 接続 URL。 |
| `JWT_SECRET` | *(mock モード以外では必須)* | JWT の署名に使用するシークレット。 |
| `ARONA_HOST` | `0.0.0.0` | バインドアドレス（`SHITTIM_CHEST_HOST` にフォールバック）。 |
| `ARONA_PORT` | `8420` | バインドポート（`SHITTIM_CHEST_PORT` にフォールバック）。 |
| `ARONA_DATA_DIR` | 未設定 | ローカルデータディレクトリ。 |
| `ARONA_ADMIN_TOKEN` | 未設定 | `/api/admin/*` と admin RPC メソッド用の Bearer token。 |
| `ARONA_REGISTRATION_OPEN` | `0` | Truthy（`1`/`true`/`yes`/`on`）でサインアップを開放。 |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | キーごとのインメモリリクエスト制限（毎分）; `0` はすべてをブロック。 |
| `MOCK_MODE` | 未設定 | 存在する（任意の値）だけで開発用 mock モードを有効化。 |
| `MOCK_SEED_PATH` | 未設定 | mock モードで実行する生 SQL シードファイル。 |
| `ARONA_MEMORY_URL` | 未設定 | Philia memory gateway の WebSocket URL。 |
| `ARONA_MEMORY_TOKEN` | 未設定 | memory gateway 用のtoken。 |
| `ARONA_MEMORY_WRITEBACK` | `true` | 完了したチャットターンを memory に書き戻すかどうか。`true`/`false` を受け付けます（その他の値はデフォルトにフォールバック）。 |
| `ARONA_AGENT_NAME` | `arona-agent` | GPU ノード agent のアイデンティティ。 |
| `ARONA_PANEL_URL` | `localhost:8080` | agent が接続する場所（`ws://<panel_url>/ws/agent`）。 |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | `evernight://` backend URL 用のローカル evernight agent。 |
| `ARONA_MISTRALRS` | 未設定 | 存在すると Gguf モデルプランで Mistral.rs エンジンを強制。 |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | 起動した llama.cpp モデルサーバーがバインドするインターフェース。 |
| `HF_ENDPOINT` | `https://huggingface.co` | モデルダウンロード用の Hugging Face ベース URL。 |
| `GITHUB_TOKEN` | 未設定 | GitHub モデルレジストリ用のaccess token。 |
| `RUST_LOG` | 未設定 | トレーシングフィルター（例: `info` や `arona=debug,info`）。 |

## 必須変数

### `DATABASE_URL`

PostgreSQL 接続 URL。**必須**: 空の場合、サーバーは `FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.` で終了し、`migrate` CLI サブコマンドも実行を拒否します。スキーマは `serve` が起動時に自動的に作成・更新します。

### `JWT_SECRET`

`auth.login` と `auth.register` が発行する access/refresh JWT ペアの署名に使用するシークレット。**本番では必須**: コードには開発用フォールバック（`dev-secret-not-for-production-use-only-32chars`）が埋め込まれていますが、`MOCK_MODE=1` でない限りサーバーはそれで起動することを拒否します:

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

長くてランダムな値を使用してください（例: `openssl rand -hex 32`）。

## サーバー

### `ARONA_HOST` / `ARONA_PORT`

gateway のバインドアドレスとポート。レガシー互換のため `SHITTIM_CHEST_HOST` / `SHITTIM_CHEST_PORT` にフォールバックします。最終的なデフォルトは `0.0.0.0:8420` です。

### `ARONA_DATA_DIR`

オプションのローカルデータディレクトリで、スクラッチ領域が必要なコンポーネントのためにアプリ状態に保持されます。デフォルトでは未設定です。

## セキュリティとアクセス制御

### `ARONA_ADMIN_TOKEN`

`/api/admin/*` HTTP ルート（`POST/GET/DELETE /api/admin/backends`、`/api/admin/aliases`）と `billing.plan.set` / `billing.video.pricing.set` RPC メソッドを保護する Bearer token。**未設定**の場合、それらのルートはすべて `Admin access required`（401）で拒否されます — デフォルト値はありません。サーバーを起動する前に、強力なランダム値を設定してください。

### `ARONA_REGISTRATION_OPEN`

`auth.register` によるセルフサービスのサインアップを開放します。Truthy な値は正確に `1`、`true`、`yes`、`on`（大文字小文字を区別しない）です。それ以外 — `0`、`false`、`off`、未設定・空の変数を含む — は閉じたままです。このフラグは起動時に一度だけ読み込まれます。**最初に登録したユーザーは常に許可され**（登録が閉じていても）、admin になります。

### `ARONA_API_RATE_LIMIT_RPM`

キーごとのインメモリスライディングウィンドウのレート制限（毎分のリクエスト数）で、認証済みのすべての `/v1/*` リクエスト（chat、embeddings、video、models）に適用され、API key のハッシュ（JWT を受け付ける `GET /v1/models` では `u:<email>` ラベル）でキーイングされます。RPC プレーンはこのリミッターによるレート制限を受けません — `/v1/*` の auth エクストラクターだけが呼び出し元です。デフォルトは `60`。値はプロセス全体の `OnceLock` に一度だけパースされます。**`0` を設定するとすべてのリクエストをブロックします** — チェックは `entry.len() >= rpm` なので、`0` ではどのリクエストも通過できません。これは gateway 全体の制限です。billing tier はさらに独自のキーごとの制限を上乗せします。

## 開発

### `MOCK_MODE`

開発用 mock モードで、**存在**によって有効になります — チェックは `std::env::var("MOCK_MODE").is_ok()` なので、*任意の*値（`0` や空だが設定された値も含む）で有効になります。次のことを行います:

- `JWT_SECRET` の要件を緩和します（組み込みの開発シークレットが許容されます）;
- 4 つのデモアカウントをシードします（`demiurge@celestia.world`、`momoi@celestia.world`、`midori@celestia.world`、`yuzu@celestia.world`、パスワード `33550336`）;
- リスナーをバインドする前にシードの完了を待ちます。

本番では絶対に使用しないでください。

### `MOCK_SEED_PATH`

mock モードのみで、組み込みのアカウントシードの代わりに実行される生 SQL ファイルを指します（文は `;` で分割、`--` コメントはスキップ）。ファイルを読み込めない場合はフォールバックとして組み込みのシードが使われます。

## Memory gateway

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

長期メモリ gateway（entelecheia Philia）の設定。`ARONA_MEMORY_URL` と `ARONA_MEMORY_TOKEN` の両方が設定され、非空でない限り、memory は**完全に無効**です。有効な場合:

- 完了したチャットターンがリコールされ、コンテキストとして注入され、
- `ARONA_MEMORY_WRITEBACK`（デフォルト `true`）が、完了したターンを memory サービスに書き戻すかどうかを制御します。`0` または `false` で書き戻しを無効にします。

memory の障害がチャットをブロックすることはありません。結果の状態は `X-Arona-Memory` レスポンスヘッダー（`enabled` / `disabled` / `offline`）に反映されます。

## Agent のアイデンティティとクラスター

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

GPU ノード agent バイナリ（`_agent`）のアイデンティティ: `ARONA_AGENT_NAME`（デフォルト `arona-agent`）は agent の名前/id としてパネルに報告され、`ARONA_PANEL_URL`（デフォルト `localhost:8080`）は agent が接続する場所です（`ws://<panel_url>/ws/agent`）。

agent 自身の HTTP API は `0.0.0.0:5790` にバインドされるよう**ハードコード**されています — バインドアドレスの環境変数はありません。

### `ARONA_AGENT_BIND_ADDR`

agent が Gguf モデルをデプロイするときに、**起動される llama.cpp モデルサーバー**がバインドするインターフェースで、エンジンが他のマシンから到達可能になるようにします（例: `0.0.0.0`）。デフォルトは `127.0.0.1`。これは agent HTTP API のバインド（`0.0.0.0:5790` に固定）では*ない*ことに注意してください。

## Evernight ブリッジ

### `ARONA_EVERNIGHT_URL`

`evernight://` backend URL をローカル TCP フォワードに解決するために使われるローカル evernight agent の WebSocket URL。デフォルトは `ws://127.0.0.1:3001/ws`。

## モデルランタイムとダウンロード

### `ARONA_MISTRALRS`

存在（任意の値）すると、通常 llama.cpp になる Gguf モデルプランで Mistral.rs エンジンを強制します。存在セマンティクスは `MOCK_MODE` と同じです。

### `HF_ENDPOINT`

Hugging Face のモデルダウンロード（`hf:` ソース）のベース URL。デフォルトは `https://huggingface.co`。huggingface.co に到達できない場合は `https://hf-mirror.com` などのミラーに設定してください。モデルダウンローダーが読み込み、末尾のスラッシュはトリムされます。

### `GITHUB_TOKEN`

GitHub モデルレジストリ（`gh:` ソース）が API アクセスに使用するaccess token。デフォルトでは未設定。これがないと GitHub API のレート制限が適用されます。

## ロギング

### `RUST_LOG`

起動時に `tracing_subscriber` が適用する標準のトレーシングフィルター（例: `info` や `arona=debug,info`）。通常の `RUST_LOG` セマンティクスに従います（`error`/`warn`/`info`/`debug`/`trace`、ターゲットごとの上書き）。

## デフォルト値の一覧

| 設定 | デフォルト |
| --- | --- |
| バインドアドレス / ポート | `0.0.0.0:8420` |
| キーごとの API レート制限 | 60 RPM |
| Agent 名 | `arona-agent` |
| パネル URL | `localhost:8080` |
| Memory 書き戻し | オン |
| 登録 | クローズ |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
