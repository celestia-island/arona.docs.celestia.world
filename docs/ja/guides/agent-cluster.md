---
title: "Agent クラスター"
description: "マルチノード GPU クラスター — CLI でのモデル重みのダウンロード、GPU ノードでの _agent バイナリの実行、agents.* RPC サーフェスでのデプロイの駆動。"
---

# Agent クラスター

Arona のデプロイのストーリーは 2 つの半分に分かれます。**パネル**（`arona` サーバーバイナリ）はルーティング、billing、auth、管理プレーンを所有します。各 GPU ノードは 1 つの **`_agent` プロセス**を実行し、モデル重みとローカルの提供プロセスを所有します。Agent はパネルの agent 制御プレーン（`/ws/agent`）への長寿命 WebSocket を開きます。パネルはそのソケットを下って `deploy` / `stop` コマンドをプッシュし、agent はダウンロード進捗、ハートビート、コマンド結果を上ってストリーミングします。モデルが agent 上で稼働すると、パネルはそれをルーティング可能な backend として登録し、`/v1/*` と RPC トラフィックが到達できるようにします — 制御プレーンは WebSocket、データプレーンは agent のローカルエンジンポートへの平文 HTTP です。

## モデル重みのダウンロード（CLI）

`_cli` バイナリは HuggingFace、ModelScope、または GitHub releases からモデル重みをダウンロードします:

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **リポジトリ形式** — `hf:owner/repo`（デフォルト。素の `owner/repo` も HuggingFace に解決されます）、`ms:owner/repo`（ModelScope）、`gh:owner/repo[:tag]`（GitHub release、タグは任意）。長いプレフィックス `huggingface:`、`modelscope:`、`github:` も受け付けられます。スラッシュのない素の id は Ollama レジストリに解決されます（`packages/core/src/models/download.rs:21-28,55-86`）。
- **`--filter <glob|prefix>`** — 繰り返し可能。glob（またはプレフィックス）に一致するマニフェストファイルだけがダウンロードされます。フィルターなしでは**リポジトリ全体**が選択されます。
- **確認** — フィルターなしのダウンロードは、`--yes` が渡されない限り、常に開始前に `Continue? [y/N]` を尋ねます。フィルター付きのダウンロードはプロンプトを出しません。選択された合計が 2 GiB 以上の場合、代わりに情報バナーを表示します（`NO_CONFIRM_THRESHOLD`、`packages/cli/src/main.rs:12-15, 439-464`）。
- **`--out <dir>`** — デフォルトの宛先 `~/.arona/models/<repo-id>` を上書きします。
- **`--revision <rev>`** — インラインの `:rev` サフィックス（`hf:owner/repo:rev`）を上書きします。
- **再開** — 中断されたダウンロードは自動的に再開されます: `.part` ファイルが保持され、Range リクエストで現在の長さからダウンロードが続行されます。完全なファイルはサイズでスキップされ、マニフェストがダイジェストを持つ場合は SHA-256 で検証されます（`packages/cli/src/main.rs` の `verify_sha256`/`summarize`）。
- **再試行** — ネットワークエラーは 5 秒間隔で最大 3 回再試行されます。非ネットワークエラーは即座に失敗します（`packages/cli/src/main.rs:277-302`）。
- **`HF_ENDPOINT`** — HuggingFace のベース URL を切り替えます（例: ミラー）:

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

他の CLI コマンド（`packages/cli/src/main.rs:28-53`）:

| コマンド | 用途 |
| --- | --- |
| `install` | ワンクリック環境セットアップ: ハードウェアプロファイルを検出し、backend / 量子化の推奨を表示します。 |
| `status` | ハードウェアプロファイルを表示します。 |
| `deploy <model>` | モデルをローカルで解決し、すでにキャッシュされているかどうかを報告します。 |
| `download` | モデル重みをダウンロードします（上記）。 |
| `serve` | API サーバー（パネル）を起動します。 |
| `connect <url>` | 管理パネルに接続します。 |
| `migrate` | データベースマイグレーションを実行します。 |

## `_agent` バイナリ

`_agent` は各 GPU ノードで実行され、純粋に環境変数で設定されます（`packages/core/src/config.rs:37-40`）:

| 変数 | デフォルト | 意味 |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | 一意のノード id。パネルはこれを `agent_id` として使用します。 |
| `ARONA_PANEL_URL` | `localhost:8080` | パネルの `host:port`。agent は `ws://{ARONA_PANEL_URL}/ws/agent` に接続します。 |

完全な環境変数リファレンス（パネル側の変数、データベース、シークレット）は[設定](./configuration.md)を参照してください。

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

挙動:

- **制御接続** — agent は `ws://{ARONA_PANEL_URL}/ws/agent` に接続を張り返します（`packages/agent/src/panel.rs:23`）。接続時に `agent_name`、`gpu_info`、すでにデプロイ済みのモデルのリストを運ぶ `register` フレームを送信します。パネルは agent の TCP ピアアドレスをその `host` として記録します。
- **再接続バックオフ** — 1 秒から始まり、60 秒の上限まで倍増します（`packages/agent/src/panel.rs:27,33-34,63`）。
- **ハートビート** — 30 秒ごとに agent は GPU 使用率、ロード済みモデル数、uptime を報告します。パネルは最後のハートビートが 30 秒より古い場合、agent をオフラインと見なします。
- **ローカル HTTP API** — **固定**アドレス `0.0.0.0:5790` にバインドします。バインドアドレスの環境変数はありません（`packages/agent/src/main.rs:109`）。パネルはこのポートと agent の記録済み host を組み合わせて、デプロイ済みモデルのデータプレーン URL を構築します。
- **コマンド** — パネルはソケット越しに `deploy` / `stop` コマンドをキューします。`deploy` コマンドは `model_id` と `stream_id` を運びます。ダウンロード進捗は同じソケット上で `deploy_progress` フレームとしてストリーミングされ（パネルはそれを `models.progress` SSE 通知として転送します。[イベントと通知](../api/events.md)を参照）、最後の `deploy_result` フレームはローカルエンジンの `port` と `backend` を報告します。`stop` には `stop_result` で応答します。

`_agent` はサービススーパーバイザー（systemd、malkuth など）の下で実行し、自動的に再接続できるようにしてください。パネルはどちら側の再起動も許容します（下の[ノード永続化](#node-persistence)を参照）。

## Agent 制御プレーン RPC

agent サーフェス全体は admin ゲート付きです。すべてのメソッドは有効な JWT **かつ** admin アカウントを要求します（`validate_admin_jwt` は `is_admin_email` をチェックします。`packages/core/src/gateway/rpc.rs:106-118,301-337`）。

| メソッド | パラメータ | 戻り値 |
| --- | --- | --- |
| `agents.list` | — | クラスタートポロジー: `id`、`name`、`host`、`status`（`online`/`offline`）、GPU サマリー、`models`、`last_heartbeat`、`version`、`connected_at`。 |
| `agents.register` | `machine_name`、`version` | `agent_id`、`token`。 |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — ノードを削除します。 |
| `agents.status` | `agent_id` | `online`、GPU サマリー、`gpu_utilization`、`models`、`host`、`connected_at`、`last_heartbeat`。 |
| `agents.deploy` | `model_id`、`agent_id?` | `{ "ok": true, "stream_id" }` — 空の `agent_id` は最少負荷ノードを自動ターゲットします。 |
| `agents.stop` | `agent_id`、`model_id` | `{ "ok": true }` — デプロイを停止します。 |

`agents.deploy` は `stream_id` を返します。呼び出しの**前**または直後に `/api/rpc/events?session=<stream_id>` を購読して、`models.progress` ダウンロード通知を受け取ってください（[イベントと通知](../api/events.md)を参照）。トランスポートと認証の詳細は[JSON-RPC API](../api/jsonrpc.md)を参照してください。

## デプロイ済みモデルの自動登録

`deploy_result` フレームが成功を報告すると、パネルは **`agent-{model_id}`** という名前の `ExternalApiBackend` を gateway ルーターに登録します。ベース URL は `http://{agent-host}:{port}` — agent の記録済み host と、それが報告したエンジンポートです（`packages/core/src/gateway/server.rs:310-366`、`packages/core/src/gateway/mod.rs:253-270`）。デプロイ済みモデルは通常のルーティング可能な backend になります: `/v1/chat/completions`、embeddings、RPC チャットがすべて到達でき、エイリアスが適用され、ヘルスチェッカーがプローブします（backend タイプとプロービングのセマンティクスは[Backends](./backends.md)を参照）。

- 同じモデルを再デプロイすると（例: 別の agent 上で）、以前の backend が置き換えられます。
- 成功した `stop_result` は再び登録を解除します（`packages/core/src/gateway/mod.rs:274-287`）。モデル id は解決されなくなります。

## 配置

明示的な `agent_id` のないデプロイは最少負荷配置を経由します（`packages/core/src/gateway/tunnel.rs:214-229`）: 候補は最後のハートビートが 30 秒未満の agent で、**平均 GPU 使用率が最も低い**ものが選ばれます。テレメトリのない agent は最後にソートされますが、選択可能なままです。オンラインの agent がない場合、RPC は `No online agents available for deployment` で失敗します。

ルーティング側では、会話は**1 つの backend にピン留めされます**（セッションアフィニティ）: 会話に最初にサービスを提供した backend が記録され、後続のターンで再利用されるため、ランタイム KV キャッシュなどの会話ごとの状態はウォームに保たれます（`packages/core/src/routing/mod.rs:31-34,110-138`）。ピン留めされた backend が unhealthy になった場合、ルーティングは新しい選択に劣化し、再ピン留めします。

## ノード永続化

Agent ノードは `agent_nodes` テーブルに永続化されます（`agent_id`、`machine_name`、`version`、`host`、`gpu_info`、`models`、`connected_at`、`last_heartbeat`。`packages/core/src/gateway/tunnel.rs:8-12`）。パネル起動時、永続化された行が復元され、以前に登録されたノードが再起動をまたいで表示され続けます。復元されたエントリは、各 agent が WebSocket で再接続するまで **sender なし**です（`packages/core/src/gateway/run.rs:74-85`）。したがって、復元されたが切断されているノードへのデプロイは、その `_agent` が再接続するまで失敗します。

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
