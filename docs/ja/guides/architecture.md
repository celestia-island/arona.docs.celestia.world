---
title: "アーキテクチャ"
description: "Arona の構造 — ワークスペースレイアウト、gateway を通るリクエストパス、ルーティング、ヘルスプロービング、メモリ、セッション、意図的な設計トレードオフ。"
---

# アーキテクチャ

このページでは Arona がどう構造化され、リクエストがどう流れるかを説明します: ワークスペースレイアウト、リクエストパス、gateway とルーター、ヘルスチェック、メモリクライアント、セッションと通知、そして最後に設計が受け入れる意図的な制限とトレードオフです。実行例は[クイックスタート](quickstart.md)、日々のランタイムの関心事は[運用](operations.md)を参照してください。

## ワークスペースレイアウト

リポジトリは 3 つのパッケージを持つ Cargo ワークスペースです:

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC,
│            # backends, migration, entities, providers, models, registry,
│            # orchestration
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

- `packages/core` はライブラリクレート（`_core`）です。サーバーが必要とするすべてを含みます: axum gateway（`gateway/`）、モデルルーター（`routing/`）、backend アダプター（`backends/`）、billing（`billing/`）、auth（`auth.rs`）、メモリクライアント（`memory/`）、JSON-RPC プレーン（`gateway/rpc.rs`）、スキーマ（`migration/`、`entity/`）、モデルメタデータ（`models/`、`providers/`、`registry/`）、モデルオーケストレーション（`orchestration/`）。
- `packages/agent` は GPU ノードで実行され、`/ws/agent` 経由で接続を張り返す `_agent` バイナリをビルドします（[agent-cluster](agent-cluster.md) を参照）。
- `packages/cli` は install、deploy、serve、migrate、download 操作に使用される `_cli` バイナリをビルドします。

このリポジトリにはもう Web ダッシュボードはありません: Vue ダッシュボードは削除され、現在は [shittim-chest](https://github.com/celestia-island/shittim-chest)（chest #291）にあり、JSON-RPC サーフェス経由で Arona と通信します。Arona 自体は純粋な backend です（[概要](./README.md)を参照）。

## リクエストパス

エントリポイントは `GatewayServer::app` で組み立てられる axum ルーターです（`packages/core/src/gateway/server.rs`）。そのルートテーブル（server.rs:128-163）は、OpenAI 互換の REST サーフェス（`/v1/chat/completions`、`/v1/embeddings`、`/v1/models`、`/v1/health`）、ビデオ生成、`/api/rpc` JSON-RPC エンドポイント（POST + WebSocket アップグレード）、SSE サイドカー `/api/rpc/events`、agent 制御プレーン `/ws/agent`、`/docs` の Swagger UI、admin の backend / エイリアス管理エンドポイントをカバーします。

ルーターは小さなレイヤースタックでラップされています（server.rs:158-162）:

1. Auth マネージャーを `Extension` として提供し、ハンドラーごとのエクストラクターが到達できるようにします。
2. インバウンドの `X-Request-ID` ヘッダーを再利用するか生成するリクエスト id レイヤーで、ハンドラーとログに公開します（`gateway/request_id.rs`）。
3. 1 MB のリクエストボディ制限（`RequestBodyLimitLayer`）。
4. 許容的な CORS レイヤー（`*` オリジン、`*` ヘッダー）。

axum はレイヤーを下から上に適用するため、CORS レイヤーが最外殻になります。

すべての `/v1/*` ハンドラーは同じスケルトンを実行します:

1. **認証抽出** — キーのみのエンドポイント（`/v1/chat/completions`、`/v1/embeddings`、ビデオ）では `ApiKeyAuth`、`GET /v1/models` では `ApiKeyOrJwt`。後者は API key とセッション JWT の両方を受け付ける必要があります（`gateway/middleware.rs`）。エクストラクターはキー / JWT をユーザーのメール、キープレフィックス、レート制限キー（API key の SHA-256 ハッシュ、または JWT の場合は `u:<email>` ラベル — ローテーションするtokenがウィンドウをリセットしないように）、任意のプロジェクトスコープに解決します。
2. **Billing ゲート** — `enforce_billing_gates`（server.rs:492-539）は、ユーザーの tier 月次クォータまたは毎分のレート制限を超えた場合、HTTP 429 + `Retry-After` でリクエストを拒否します。DB 障害は fail-open です: billing はベストエフォートであり、チャット提供のハードな依存関係ではありません。
3. **メモリリコール**（チャットパス）— メモリクライアントが設定され、リクエストが求める場合、関連する長期メモリがシステムセクションとして注入されます（下の[メモリクライアント](#memory-client)を参照）。失敗がチャットをブロックすることはなく、結果の状態は `X-Arona-Memory` ヘッダーに反映されます。
4. **会話の永続化** — 任意の `conversation_id` は所有権チェックされ、ユーザーターンは送信時に永続化されます。
5. **Gateway ディスパッチ** — リクエストは `Gateway` に渡され、backend を解決し、コンテキストをトリムし、backend トレイトを呼び出します。
6. **Usage 記録** — 返された（またはストリーム終端の）usage は、`UsageTracker` を通じてキープレフィックス配下に記録・永続化されます。

`Gateway` 自体は `AppState` 内に `Arc<Gateway>` として存在します — 外部ミューテックスはありません。内部可変性により、並行するチャット / embeddings / ストリーム呼び出しが upstream HTTP ラウンドトリップをまたいでロックを保持することが決してありません（`gateway/mod.rs:29-53`）。

## Gateway とルーター

`Gateway`（`packages/core/src/gateway/mod.rs`）は次のものを所有します:

- **ルーター状態** — backend リストとエイリアス。`tokio::sync::RwLock` で保護されます。読み取り側の解決は await をまたいで借用します。変更（register/remove/alias）は短い書き込みロックを取り、upstream 呼び出しをまたいで保持することはありません。
- **リクエストカウンター**（`AtomicU64`）と、`system.status` とヘルスエンドポイントが使う `start_time`。
- **デプロイメントマップ**（`model_id → backend name`）— agent がデプロイしたモデル用。`register_agent_backend` は `agent-{model_id}` という名前の `ExternalApiBackend` を構築してルーターに挿入します。同じモデルの再登録は以前の backend を置き換え、`unregister_agent_backend` は `stop_result` フレームでそれを削除します（[agent-cluster](agent-cluster.md) を参照）。

Backend の解決は `Router`（`packages/core/src/routing/mod.rs`）で行われます:

1. **エイリアス解決** — 設定されたエイリアスはターゲットに書き換えられます。
2. **セッションアフィニティ** — `conversation_id` がある場合、ルーターは会話を最初にサービスを提供した backend にピン留めする弱参照マップをチェックします。弱参照は backend が登録されているか処理中である間だけマップを生かすため、削除された backend はインデックスのずれなしに消えます。作動したサーキットブレーカーや unhealthy なピン留め backend は新しい選択に劣化し、会話を再バインドします。
3. **候補フィルタリング** — 任意の `provider` ヒントは backend 名 / kind でフィルタリングします。候補は healthy *かつ*サーキットブレーカーが開いていて、要求されたモデルをリストしている必要があります。モデル id は完全一致または `:latest` サフィックス規約でマッチします（素の `nomic-embed-text` リクエストはリストされた `nomic-embed-text:latest` にマッチ）。
4. **最少負荷選択** — 生き残った候補はヒットカウンターでソートされ、最も負荷の低いものが選ばれます。会話のピン（あれば）は同時に記録されます。

backend が呼び出される前に、`RequestPipeline::transform`（`packages/core/src/pipeline.rs:422+`）がメッセージリストを backend の `max_context_length` にトリムします: システムメッセージは完全に保持され、非システムメッセージは収まる限り新しい順で保持され、単一の巨大メッセージは文字数でハード切り詰めされます（ヒューリスティックなtokenカウンターはtoken精度で切り詰めることはできません）。呼び出しは `InferenceBackend` トレイトを通り、成功と失敗はルーターの backend ごとのサーキットブレーカー（3 回の失敗、30 秒の回復、1 回のハーフオープン呼び出し — routing/mod.rs:57-64）に記録されます。

## ヘルスチェッカーとプロービング

`run_health_checks`（`packages/core/src/gateway/health_checker.rs`）は起動時にスポーンされるバックグラウンドタスクとして実行され（run.rs:135-150）、登録済みのすべての backend を 60 秒間隔で 1 回プローブします。重要な詳細が 2 つあります:

- backend リストは**各ラウンドで非同期フェッチャークロージャーを通して新しく取得**されるため、起動後に登録された backend（例: admin API 経由）は再起動なしで拾われます。
- 最初のラウンドは最初の間隔が経過する前の即座に実行されるため、プロセス開始と同時にヘルス状態が確立されます。

`probe_backend` は単一のプローブコードパスです。これは一度きりの**登録時プローブ**でも再利用されます: 管理者が backend を登録した後（server.rs:688-693）、または永続化された backend が起動時に復元された後（run.rs:122-127）に、fire-and-forget プローブが次の 60 秒ラウンドまで fail-closed のままにする代わりに、約 1〜2 秒で backend を healthy に切り替えます。これが、新しく登録された external backend のモデルリストがほぼ即座に `GET /v1/models` に現れる理由です。

プローブ自体は軽量な backend 呼び出しです（例: external backend は 2 秒のプローブタイムアウトで `/v1/models` にヒット）。結果は backend にキャッシュされ、ルーティングはキャッシュされたヘルスが `Healthy` の backend（プラス開いたサーキットブレーカー）だけを選択します。

## メモリクライアント

メモリクライアント（`packages/core/src/memory/mod.rs`）はサーバー起動時に環境設定から構築されます（server.rs:95-97）: `ARONA_MEMORY_URL` と `ARONA_MEMORY_TOKEN` が設定されている場合、チャットリクエストは JSON-RPC WebSocket 経由で entelecheia Philia メモリサービスに照会し、`recall_and_inject` は関連メモリをシステムセクション（`## Relevant Long-Term Memories`）として送信コンテキストの先頭に付加します。完了したターンは `writeback_dialogue` 経由でエピソードとして書き戻されます — アシスタント応答が永続化された後にスポーンされる fire-and-forget の作業で、メモリの失敗がチャットレスポンスパスをブロックしたり遅くしたりすることはありません。`ARONA_MEMORY_WRITEBACK`（デフォルトオン）は書き戻しを切り替えます。全体像は[memory-gateway](memory-gateway.md)を参照してください。

すべてのチャットレスポンスは 3 つの状態のいずれかを持つ `X-Arona-Memory` ヘッダーを運びます: `enabled`（リコールが実行され注入された）、`disabled`（未設定またはリクエストが `memory: false` を渡した）、`offline`（設定されているがサービスに到達できなかった）。

## セッションと通知

`AppState` は plana の `SessionManager`（`state.sessions`）を保持します。`chat.send` などのストリーミング RPC はセッション id を作成し（`gateway/rpc.rs:1701`）、`chat.stream` token、`models.progress` デプロイ進捗、`realtime.event` などの通知をそのセッションのチャネルにプッシュします。クライアントは SSE サイドカー `GET /api/rpc/events?session=<id>`（server.rs:191-200）からそれらを消費します。通知形式と事前購読ウィンドウの注意点は[イベント](../api/events.md)を参照してください。

セッションチャネルはリクエスト / レスポンスの RPC 呼び出しにも使用されます: クライアントが `POST /api/rpc` で `x-session-id` ヘッダーを送信すると、サーバーは結果もそのセッションチャネルにプッシュします（server.rs:184-188、rpc.rs:128-144）。したがってクライアントは、すでに開いている SSE ストリームに RPC レスポンスを多重化できます。

## 制限と設計トレードオフ

設計は意図的に多くの制限を受け入れています。本番使用の前に知っておいてください:

- **1 MB のリクエストボディ制限** — より大きいボディはレイヤーに拒否されます。大きなコンテキストの呼び出しが必要なら、これが最初にチューニングすべきものです。
- **CORS `*`** — gateway はどこからのクロスオリジン呼び出しにも応答します。API には問題ありませんが、信頼されたクライアントを超えて公開する場合は、独自の CORS ポリシーを強制するプロキシを前面に置いてください。
- **Fail-open billing** — クォータ / レート制限の強制は、DB が利用できないときにリクエストを許可する方向に劣化します。Billing は計測であり、アクセス制御ではありません。
- **SSE ストリームに全体のタイムアウトなし** — ストリーミング呼び出しは合計デッドラインを持ちません（長い生成は合法）。ハング検出は読み取りごとの 120 秒のアイドルタイムアウトに依存します（`backends/external.rs:24-31`）。非ストリーミング呼び出しは 600 秒の全体デッドラインを得ます。
- **トークナイザー推定の usage** — usage を報告しない backend（ollama、ws_engine）は、ローカルの CJK 対応トークナイザー推定で請求され、そのまま記録されます（[billing-usage](billing-usage.md)を参照）。
- **インメモリのレート制限ウィンドウと失効** — キーごとのスライディングウィンドウと失効キーセットはプロセスメモリにあります（`auth.rs`）。したがって再起動でリセットされます。認証レベルのリミッターはウィンドウごとのキーあたりのリクエストを制限し、billing tier のリミッターは DB バックです（[auth-security](auth-security.md)と[billing-usage](billing-usage.md)を参照）。
- **`/ws/agent` は認証なし** — agent 制御プレーンは register / heartbeat プロトコルを話す任意の WebSocket を受け入れます。自分が制御するネットワーク上でのみ安全です。
- **gateway に TLS なし** — サーバーは平文 HTTP にバインドします。ネットワーク境界を越えるデプロイでは、前面（リバースプロキシ）で TLS を終端してください。[deployment](deployment.md)を参照してください。

優雅な面では、サーバーはグレースフルシャットダウンを実行します: Ctrl+C と SIGTERM ハンドラーをインストールし、「draining connections」をログし、プロセスが終了する前に進行中のリクエストを完了させます（`gateway/run.rs:14-38`、および run.rs:154-159 の `with_graceful_shutdown` 配線）。

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table); gateway/mod.rs:29-53; routing/mod.rs:102-200; health_checker.rs:14-37; run.rs:135-159; memory/mod.rs:207-241; rpc.rs:128-144,1701 -->
