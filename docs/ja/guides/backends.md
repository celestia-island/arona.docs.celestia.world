---
title: "Backends"
description: "Backend タイプ（external、ollama、engine、minimax-cloud、evernight ブリッジ）、URL セマンティクス、ヘルスプロービング、モデルディスカバリー、エイリアス、ルーティング。"
---

# Backends

**backend** はモデルトラフィックを提供する upstream です。Arona は OpenAI 互換のリクエスト（`/v1/chat/completions`、`/v1/embeddings`、モデル一覧、ビデオジョブ）を登録済みの backend のいずれかにルーティングし、すべてのリクエストを計測し、各 backend のヘルスとモデルインベントリを最新に保ちます。

Backend は管理者が `POST /api/admin/backends` で登録し（[Admin HTTP API](../api/admin-http.md) を参照）、`backend_configs` テーブルに永続化され、起動時に自動的に復元されます。各登録には `name`、`type`、`url`、任意の `api_key`、任意の静的 `models` リストが含まれます。永続化された backend は再起動後も維持されます。復元された backend は fail-closed で始まり、即座にプローブされます。

## Backend タイプ

| `type` | トランスポート | プロトコル | 用途 |
| --- | --- | --- | --- |
| `external` | HTTP(S) | OpenAI 互換 REST | 任意のチャット / embeddings API（クラウドまたはセルフホスト） |
| `ollama` | HTTP(S) | Ollama ネイティブ API（`/api/chat`、`/api/embed`、`/api/tags`） | ローカルまたはリモートの Ollama サーバー。URL だけで構築されます |
| `engine` | `ws://` / `wss://` | CEP（Celestia Engine Protocol）、WebSocket + JSON-RPC | `plana::engine` で定義される CEP 交換標準を話す任意のエンジン |
| `minimax-cloud` | HTTPS | MiniMax H3 タスク型 API（送信 + ポーリング） | クラウドビデオ生成 |
| `evernight://<node>/<service>` | ブリッジ URL | ローカル evernight agent 経由でローカル TCP フォワードに解決 | evernight メッシュ経由でのみ到達可能な産業用・エッジサービス |
| `agent-{model}` | HTTP | OpenAI 互換（external） | GPU agent がモデルをデプロイしたときに自動登録 |

### `external` — 任意の OpenAI 互換 HTTP API

汎用 backend: OpenAI REST シェイプを話す任意のサーバーに対するチャット補完（ストリーミング / 非ストリーミング）と embeddings。ベース `url`、`api_key`（任意）、任意の静的 `models` リストで設定します:

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

静的な `models` リストは権威を持ちます。プローブ時にディスカバリーされたモデルよりも先にマージされます（[モデルディスカバリー](#model-discovery)を参照）。

### `ollama` — URL だけで構築される

Ollama backend は URL だけで構築されます — API key もモデルリストも不要です。Ollama のネイティブプロトコルを話します: チャットは `/api/chat`、embeddings は `/api/embed`、ヘルスプロービングとモデルディスカバリーは `/api/tags`。

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — WebSocket 上の CEP

`engine` backend は `ws://`（または `wss://`）を公開するエンジンに接続し、**Celestia Engine Protocol**（CEP）を話します。これは `plana::engine` で定義される WebSocket + JSON-RPC 2.0 交換標準です。ハンドシェイク → メソッド → ストリーミング通知のフローを実装する任意の言語のエンジンは、Arona 側でアダプターコードを一切書かずにファーストクラスの backend として登録されます。ワイヤーメソッド: `Engine.Handshake`（最初のメッセージ。アイデンティティ + 機能）、`Engine.Chat`、`Engine.ChatStart`（ストリーミング。チャンクは `Engine.ChatChunk` 通知として到着）、`Engine.Embeddings`、`Engine.Models`。接続は最初の使用時に遅延確立され、エラー時には切断されます。次の呼び出しで再接続と再ハンドシェイクが行われます。

### `minimax-cloud` — タスク型ビデオ生成

クラウドビデオ backend は MiniMax H3 Open Platform API を駆動します: 生成タスクを送信し、完了をポーリングし、結果からアーティファクト URL を読み取ります。これは削除された ComfyUI backend の後継です（下記参照）。ビデオジョブは `/v1/video/generations` または `video.*` RPC メソッドで送信され、`video.progress` / `video.done` / `video.failed` 通知で進捗します（[Realtime & Video](realtime-video.md) を参照）。

### `evernight://` ブリッジ URL

`evernight://<node>/<service>` 形式の backend URL は**直接**接続されません。ローカルホストの evernight agent がそれを（agent の WebSocket エンドポイントへの `Bridge.Connect` JSON-RPC 呼び出しで）ローカル TCP フォワードに解決し、backend はハードコードされたリモートアドレスの代わりに `http://127.0.0.1:<local_port>` に対して実行されます。これがシングルパネルアーキテクチャです: Arona パネルは、設定にリモートアドレスを埋め込むことなく、メッシュ経由で他のノードのサービス（CEP エンジン、scepter など）に到達します。

キープアライブタスクは 15 秒ごとにトンネルをプローブします。リモート側が再起動してトンネルが新しいローカルポートで再確立されると、影響を受ける backend は新しい URL で**透過的に再構築**されます。永続化された設定は `evernight://` URL を保持するため、再起動時には再解決されます。`engine` backend の場合、解決された `http://127.0.0.1:<port>` フォワードは WebSocket トランスポート用に `ws://` に適応されます。

### Agent がデプロイしたモデルは自動登録される

GPU agent がモデルのデプロイを完了すると、gateway は `agent-{model_id}` という名前の backend（`http://{agent host}:{port}` 上の `ExternalApiBackend`）を登録し、モデルが即座にルーティング可能になります。デプロイを停止すると再び登録が解除されます。完全なデプロイライフサイクルは[Agent クラスター](agent-cluster.md)を参照してください。

### `comfyui` は拒否される

`comfyui` backend タイプは `comfyui backend removed` エラーで明示的に拒否されます。ComfyUI backend はモデルプラットフォームの統合時に削除され、ビデオ生成は現在 `minimax-cloud` 経由で実行されます。`comfyui` backend の登録は HTTP 400 を返します。

## URL セマンティクス

設定されたベース URL が実際のエンドポイントにどうマッピングされるかは、URL がパスコンポーネントを持つかどうかで決まります:

- **ルート型ベース** — パスが空または `/` の URL はホストルートとして扱われ、OpenAI の `/v1` 規約を維持します: `{base}/v1/chat/completions`、`{base}/v1/models`。例: `http://192.0.2.20:8429`、`https://api.deepseek.com`。
- **パス型ベース** — 非空のパスを持つ URL は、サーバーが実際に提供する完全な API プレフィックスとして扱われ、エンドポイントが直接追記されます: `{base}/chat/completions`、`{base}/models`。これは `/v1` 規約の外にある OpenAI 互換サーバーに必要です。Zhipu GLM コーディングプランが代表例です: API は `https://open.bigmodel.cn/api/coding/paas/v4` にあり、チャットは `{base}/chat/completions` に直接あり、**`/models` エンドポイントはまったくありません** — 標準の `/api/paas/v4` ルートはコーディングプラン用のキーに対して残高エラーを返します。
- 設定されたベース URL の**末尾スラッシュ**は正規化で除去され、結合時に二重スラッシュが生じません。

## プロービングとヘルス

バックグラウンドのヘルスチェッカーは、登録済みのすべての backend を**60 秒**ごとにプローブします。backend リストは各ラウンドで新しく取得されるため、起動後に登録された backend も再起動なしで拾われます。また、各管理者登録でも即時プローブが発火し、次のチェッカーラウンドを待たずに約 1〜2 秒で backend が healthy に切り替わります。

- **External backend** は **2 秒のタイムアウト**で `GET {base}/models`（ルート型ベースの場合は `{base}/v1/models`）をプローブします。**404 は許容されます**: 一部のサーバーはチャットを実装しているがモデル一覧を公開していません（GLM コーディングプランには `/models` エンドポイントがありません）。そのため 404 は backend を healthy とマークし、管理者設定の `models` リストがルーティングのソースになります。タイムアウト、ネットワーク障害、その他の非 2xx 応答は backend を unhealthy とマークします。
- **Ollama backend** は同じ 2 秒のタイムアウトで `/api/tags` をプローブします。
- Backend は最初のプローブ成功まで **fail-closed**（`not probed yet` として報告）で始まるため、新しく登録された（または復元された）backend は検証される前にトラフィックを受け取ることはありません。

ヘルス状態は backend ごとにキャッシュされ、リクエストのたびにルーターが参照します。unhealthy な backend は候補選択から除外されます（[ルーティング](#routing)を参照）。

## モデルディスカバリー

backend は提供するモデル id をアドバタイズし、ルーターはそのアドバタイズに対してリクエストを照合します:

- **External** backend はプローブ応答からパースしたモデル（`data` と `models` の両方の配列が受け入れられます）を、管理者設定の静的リストとマージしてアドバタイズします — 静的な id は順序と優先順位を保ち、動的な id は重複排除されて追記されます。サーバーにモデルエンドポイントがない場合、静的リストだけがルーティングのソースになります。
- **Ollama** backend は `/api/tags` が返すタグをアドバタイズします。
- **Agent がデプロイした**モデルは、デプロイされた `model_id` だけをアドバタイズします。

公開サーフェスは `GET /v1/models`（認証付き）で、すべての healthy な backend のルーティング可能なモデルを一覧表示します（[OpenAI 互換 REST API](../api/openai-rest.md) を参照）。

## エイリアスと名前の正規化

エイリアスは、リクエストされたモデル id をターゲット id にマッピングします。エイリアスはルーティングで最初に解決されるため、エイリアスへのリクエストはターゲットをアドバタイズする backend によって提供されます:

```json
{ "alias": "fast-chat", "target": "deepseek-chat" }
```

エイリアスは `/api/admin/aliases` 管理エンドポイントで管理され、即座に有効になります。

名前マッチングは Ollama スタイルのタグも正規化します: `nomic-embed-text:latest` をリストする backend は、素の `nomic-embed-text` へのリクエストにマッチするため、embedding/chat リクエストは `:latest` サフィックスの簿記なしで解決されます。明示的なタグ（`qwen3:0.6b`）はその正確なタグにのみマッチします。

## ルーティング

すべてのリクエストはルーターを通して解決され、1 つの backend が選択されます:

1. **エイリアス解決** — リクエストされたモデル id はエイリアステーブル（あれば）経由でマッピングされます。
2. **Provider ヒント** — 任意の `provider` フィールドが backend 名（または kind 名、例: ビデオ backend の `cloud`）で候補をフィルタリングします。
3. **healthy な候補のみ** — backend は `Healthy` を報告する*かつ*サーキットブレーカーを通過する（3 回連続の失敗でブレーカーが 30 秒間開き、ハーフオープンのテスト呼び出しが 1 回）必要があります。
4. **最少カウント選択** — モデルを提供する候補を backend ごとのリクエストカウンターでソートし、最も負荷の低いものを選びます。同じモデルを提供する healthy な backend 間に負荷を分散します。
5. **セッションアフィニティ** — リクエストが `conversation_id` を持つ場合、会話は最初に使用した backend にピン留めされます。ピンは `Weak` 参照マップに存在するため、削除された backend はインデックスのずれなしにマップから消えます。アフィニティはベストエフォートです: 会話のターン全体で同じ backend を再利用することで、upstream は会話ごとのランタイム状態（ウォームコンテキスト、KV キャッシュ）を再利用できます。ピン留めされた backend が unhealthy になった場合（またはピンが削除された backend とともに消えた場合）、ルーターは新しい最少カウント選択にフォールバックし、会話を**再バインド**します。

モデルを提供する healthy な backend がない場合、ルーティングは失敗します: 未知のモデルは `model not found`（HTTP 404）、既知だが到達不能なモデルは `all backends unhealthy` となり、500 内部サーバーエラーとして表面化します。HTTP 502 は*到達可能な* upstream が報告した失敗（選択後の非 2xx upstream 応答とトランスポート失敗）のために予約されています。完全なエラーマッピングは[運用](operations.md)を参照してください。

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
