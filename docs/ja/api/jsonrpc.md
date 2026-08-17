---
title: "JSON-RPC API リファレンス"
description: "/api/rpc の Arona 管理プレーン JSON-RPC 2.0 API — HTTP と WebSocket 上の chat、realtime、engine、auth、keys、providers、agents、memory、conversations、usage、billing、video、system メソッド。"
---

# JSON-RPC API リファレンス

Arona は管理プレーン用に `/api/rpc` で JSON-RPC 2.0 サーフェスを公開します: auth、keys、providers、agents、memory、conversations、usage、billing、video、realtime、ストリーミングチャットです。これは OpenAI 互換の REST サーフェス（`/v1/*`。[OpenAI 互換 REST API](./openai-rest.md)を参照）を補完します。キー認証の推論ワークロードには REST を、セッション / アカウント管理とストリーミング制御には JSON-RPC を使用してください。[クイックスタート](../guides/quickstart.md)が最初のエンドツーエンドターンを説明します。

このサーフェスは **39 のリクエストメソッド**と、匿名の WebSocket 専用 liveness メソッド `system.probe`（合計 40 メソッド）をディスパッチします。すべてのリクエストは `jsonrpc: "2.0"`、`method` 文字列、任意の `params` オブジェクト、任意の `id` を持つ JSON-RPC 2.0 オブジェクトです。

## トランスポート

- **HTTP POST `/api/rpc`** — リクエスト / レスポンス。`Content-Type: application/json` を送信します。JWT は `Authorization: Bearer <jwt>` ヘッダーで送信します。リクエストボディは 1 MiB に制限されます。
- **WebSocket `GET /api/rpc`** — 長寿命接続。ブラウザーは WebSocket アップグレードでカスタムヘッダーを設定できないため、JWT は `?token=<jwt>` クエリパラメータで送信され、サーバーは内部で `Authorization: Bearer` ヘッダーに畳み込みます（`packages/core/src/gateway/server.rs` を参照）。認証済みソケットは無期限に接続したままでいられます。
- **バッチリクエスト** — JSON 配列である POST ボディは要素ごとに実行され、同じ順序のレスポンスの JSON 配列で応答されます。
- **匿名アクセス** — JWT なしの WebSocket では、公開メソッド（`auth.register`/`auth.login`/`auth.refresh`、`providers.list`、`system.status`）は呼び出し可能なままで、`system.probe` はソケットが閉じる前に単一の ack で応答されます。他のすべてのメソッドは有効な JWT を要求します。admin ゲートのメソッドはさらに admin アカウントを要求します（下の凡例を参照）。匿名ソケットは 10 秒のアイドルタイムアウトにも拘束されます。
- **セッションアタッチメント** — `POST /api/rpc` の `x-session-id` ヘッダーは、RPC レスポンス自体もストリーミング通知と並んでそのセッションチャネルにプッシュします。

## Id

リクエストの `id` 値は型忠実度でエコーされます: `null` → `null`、文字列 → 文字列、整数 → 数値、その他（float、オブジェクト、i64 範囲外の整数）→ JSON 文字列レンダリング。省略された `id` は `null` で応答されます。

## サーバー → クライアント通知（SSE サイドカー）

token、デプロイ進捗、realtime イベントは WebSocket ソケットでは**配信されません**。各ストリーミング RPC はセッション id を作成し、server-sent events として `GET /api/rpc/events?session=<session_id>` に通知をプッシュします。RPC 呼び出しがセッション id を返した**前または直後**に SSE エンドポイントを購読してください — 呼び出しが返ってから SSE 購読が確立されるまでの間に発行された通知はドロップされます（事前購読ウィンドウ）。推奨パターンは、最初に SSE ストリームを開き、その後 RPC を発火することです。

通知メソッド: `chat.stream`（`chat.send` からのイベントごとに 1 token）、`models.progress`（`agents.deploy` からの agent モデルダウンロード進捗）、`realtime.event`（開いている realtime セッションのサーバーイベント）、`video.progress` / `video.done` / `video.failed`（非同期ビデオジョブ）。完全なカタログは[イベントと通知](./events.md)を参照してください。

## エラーコード

| コード | 名前 | 意味 |
| --- | --- | --- |
| `-32700` | Parse error | リクエストボディが有効な JSON ではありません。 |
| `-32600` | Invalid request | リクエストオブジェクトが不正です（例: `method` の欠落）。 |
| `-32601` | Method not found | 未知の `method` 文字列。メッセージはそれをエコーします。 |
| `-32602` | Invalid params | `params` がメソッドのデシリアライゼーションに失敗しました。 |
| `-32603` | Internal error | 予期しないサーバー障害。 |
| `-32000` | `APP_ERROR` | 汎用アプリケーションエラー — 例: conversation/provider/agent が見つからない、デプロイ可能なオンライン agent がない。 |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` — JWT の欠落または無効。admin token メソッドで bearer token が `ARONA_ADMIN_TOKEN` と一致しない場合（`"Admin access required"`）にも使用されます。 |
| `-32006` | `QUOTA_ERROR` | JWT ゲートの RPC メソッド（`chat.send`）で月次 billing クォータを超過。 |
| `-32007` | `ADMIN_REQUIRED` | 認証済みの**非 admin** が admin ゲートのメソッド（`agents.*`、`engine.invoke`）を呼び出した。メッセージにはメソッド固有のヒントが含まれます。 |

> `agents.*` と `engine.invoke` メソッドは admin のみです: アカウントが `users.is_admin = true` を持つ JWT を要求します。認証済みの非 admin は `-32007`（`ADMIN_REQUIRED`）で拒否されます。未認証の呼び出し元は標準の `AUTH_ERROR` を受け取り、サーバーはメソッドが特権的であることを明かしません。

## 認証の凡例

| 凡例 | クレデンシャル |
| --- | --- |
| **public** | クレデンシャル不要。 |
| **JWT** | HTTP では `Authorization: Bearer <jwt>`、WebSocket では `?token=<jwt>`。 |
| **admin（JWT + is_admin）** | `users.is_admin = true` のアカウントの Bearer JWT。 |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN`（環境設定。未設定の場合はメソッドは常に拒否され、デフォルト拒否）。 |

このドキュメントのすべての例示クレデンシャルとアドレスはプレースホルダーです（RFC 5737 ドキュメント IP、`sk-xxx` キー）。この凡例の背後にある完全な認証モデルは[認証とセキュリティ](../guides/auth-security.md)を参照してください。

## チャット

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model`（string）、`messages`（`{ role, content, images?, tool_calls? }` の配列）、`temperature?`（number）、`max_tokens?`（integer）、`conversation_id?`（string）、`memory?`（bool）、`extra?`（object）、`tools?`（OpenAI スタイル関数定義の配列）、`provider?`（string） | ストリーミングチャットターンを送信します。`{ "stream_id", "memory" }` を返します — `memory` はリコール状態（`enabled` / `disabled` / `offline`）。tokenは SSE サイドカーの `chat.stream` 通知として届きます。`conversation_id` がある場合、完了した永続化履歴がサーバー側で組み立てられ、ターンが永続化されます。Billing ゲート付き（月次クォータ → `-32006`）。usage は `jwt-<user-uuid>` の下に記録されます。 |

## Realtime（全二重の音声・ビデオセッション）

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model`（string）、`config?`（セッション設定オブジェクト）、`conversation_id?`（string） | `model` を提供する backend に対する全二重セッションを開きます。`{ "session_id", "stream_session" }` を返します: `realtime.event` / `realtime.stop` には `session_id` を使用し、`realtime.event` 通知を受け取るには SSE サイドカーの `stream_session` を購読します。 |
| `realtime.event` | JWT | `session_id`（string）、`event`（クライアントイベント — 音声 append/commit/clear、画像フレーム、response create/cancel、セッション停止） | 開いているセッションに 1 つのクライアントイベントを送信します。upstream backend に転送されます。`{ "ok": true }` を返します。 |
| `realtime.stop` | JWT | `session_id`（string） | セッションを閉じて削除します。`{ "removed": bool }` を返します。 |

## Engine（汎用認識・制御チャネル）

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `engine.invoke` | admin（JWT + is_admin） | `model`（string）、`method`（string）、`params?`（object） | `model` を提供する backend 上の任意のエンジンメソッドの同期リクエスト / レスポンス呼び出し — `sensor.ingest` / `control.setpoint` スタイルの呼び出し（20〜30 Hz ループ）用の高頻度チャネルです。結果は backend の生のレスポンスです。 |

## Auth

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `auth.register` | public | `email`、`password`、`name?` | アカウントを登録します。登録が開放されている間（`ARONA_REGISTRATION_OPEN`）のみ許可されます。最初に登録したユーザーが admin になります。`auth.login` と同じtokenレスポンス（`access_token`、`refresh_token`、`token_type`、`expires_in`、`user`）を返します。 |
| `auth.login` | public | `email`、`password` | ログインします。`access_token`、`refresh_token`、`token_type`、`expires_in`、`user`（`{ id, email, name, is_admin }`）を返します。IP とアカウントごとにレート制限されます。 |
| `auth.refresh` | public | `refresh_token` | refresh tokenを新しいaccess token（と新しい refresh token）と交換します。再利用または期限切れの refresh tokenは `AUTH_ERROR` で拒否されます。 |
| `auth.me` | JWT | — | 現在のユーザープロファイル: `{ "id", "email", "name" }`。 |

## Keys

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | 呼び出し元の API key を一覧します（id、name、`key_prefix`、project、タイムスタンプ、active フラグ）。 |
| `keys.create` | JWT | `name`、`project?` | API key を作成します。`{ id, name, key, key_prefix, project, created_at }` を返します — `key` の完全な `arona-<uuid>` シークレットは**一度だけ**表示されます。すぐに保存してください。 |
| `keys.revoke` | JWT | `key_id` | API key を失効させます。`{ "ok": true }` を返します。 |

## Providers

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | 既知のproviderを一覧します: 組み込みの公式エントリとカスタムエントリを表示メタデータとして（`id`、`name`、`description`、`website_domain`、`is_official`、`is_operator`）。設計上公開されています — リストはクレデンシャルを持たず、以下の変更だけが JWT ゲートです。 |
| `providers.add` | JWT | `id`、`name`、`description?`、`website_domain?` | カスタムproviderエントリを追加します。`{ "ok": true }` を返します。 |
| `providers.update` | JWT | `provider_id`、`name?`、`description?`、`website_domain?` | カスタムproviderのフィールドを更新します（指定されたもののみ）。`{ "ok": true }` を返します。 |
| `providers.remove` | JWT | `provider_id` | カスタムproviderを削除します。`{ "ok": true }` を返します。 |
| `providers.test` | JWT | — | provider接続をテストします。スタブ: `{ "ok": true, "message": "Provider connection test not yet implemented" }` を返します。 |

## Agents

すべての `agents.*` メソッドは admin のみです（JWT + `is_admin`）。Agent ノードは `GET /ws/agent` でアウトバウンド接続します。この RPC グループはレジストリを制御します（[Agent クラスター](../guides/agent-cluster.md)を参照）。

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `agents.list` | admin（JWT + is_admin） | — | 登録済みの agent ノードを一覧します: id、name、host、`online`/`offline` ステータス（ハートビートベース）、GPU サマリー、デプロイ済みモデル、バージョン、タイムスタンプ。 |
| `agents.register` | admin（JWT + is_admin） | `machine_name`、`version` | tunnel マネージャーに agent ノードを登録します。`{ "agent_id", "token" }` を返します（token は agent の制御プレーンクレデンシャル）。 |
| `agents.deregister` | admin（JWT + is_admin） | `agent_id` | agent を登録解除（切断）します。`{ "ok": true }` を返します。 |
| `agents.status` | admin（JWT + is_admin） | `agent_id` | Agent ごとのステータス: online フラグ、host、GPU サマリー、ロード済みモデル、GPU 使用率、ハートビート / 接続タイムスタンプ。 |
| `agents.deploy` | admin（JWT + is_admin） | `model_id`、`agent_id?`（空 / 省略 = 最少負荷ノード。オンラインがない場合はエラー） | agent 上でモデルをデプロイします。`{ "ok": true, "stream_id" }` を返します — `models.progress` ダウンロード通知のために SSE サイドカーの `stream_id` を購読します。 |
| `agents.stop` | admin（JWT + is_admin） | `agent_id`、`model_id` | デプロイ済みモデルを停止します。`{ "ok": true, "stream_id": null }` を返します（進捗ストリームなし）。 |

## Memory

長期メモリは entelecheia Philia サービスが WebSocket 経由で提供します。失敗がチャットをブロックすることはありません（[Memory Gateway](../guides/memory-gateway.md)を参照）。

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | Memory gateway の状態: `{ "enabled", "writeback", "events" }` — フラグと、最大 50 件の最近のアクティビティイベント（新しい順）。 |
| `memory.delete` | JWT | `node_id` | 保存されたメモリノードを削除します。`{ "deleted": bool }` を返します。 |

## Conversations

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | 呼び出し元の会話を相対的な経過時間タイムスタンプ付きで一覧します。 |
| `conversations.create` | JWT | `title?`（デフォルト `New Conversation`） | 会話を作成します。新しい会話オブジェクトを返します。 |
| `conversations.get` | JWT | `conversation_id`（レガシーエイリアス: `id`） | メッセージ付きの会話を取得します。所有権チェック付き。ユーザー間のアクセスは拒否されます。 |
| `conversations.delete` | JWT | `conversation_id`（レガシーエイリアス: `id`） | 会話を削除します（所有者のみ）。`{ "ok": true }` を返します。 |

> `conversations.get` / `conversations.delete` は古いダッシュボードクライアントのレガシー `id` キーも受け付けます。両方ある場合は `conversation_id` が優先されます。

## Usage

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?`（integer、デフォルト 50、1〜200 にクランプ）、`offset?`（integer、デフォルト 0）、`project?`（string） | 呼び出し元のページングされた usage レコード。新しい順で、API key 行（`arona-XX` プレフィックス）と JWT 帰属行（`jwt-<user-uuid>`）の両方をカバーします。`{ "records", "total", "limit", "offset", "project" }` を返します。`project` フィルターはキータグ付きの行だけに絞り込みます。 |

## Billing

Tier、クォータ、usage の会計は[Billing & Usage](../guides/billing-usage.md)で説明されています。

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | 現在の billing 状態: `{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` — 月次 usage（`cost_usd`、token、リクエスト数）と残りクォータ。 |
| `billing.plan.set` | admin token | `user_email`、`tier` | ユーザーの billing tier を設定します。`{ "ok": true }` を返します。bearer が `ARONA_ADMIN_TOKEN` と一致しない場合、`AUTH_ERROR` で拒否されます。 |
| `billing.video.pricing.get` | JWT | — | ビデオ価格テーブル。`{ "pricing": [...] }` を返します。 |
| `billing.video.pricing.set` | admin token | `model`、`mode?`（デフォルト `per_second_resolution`）、`base_price?`（number、デフォルト 0）、`price_per_second?`（number、デフォルト 0）、`price_per_frame?`（number、デフォルト 0）、`resolution_coeff?`（object）、`currency?`（デフォルト `USD`）、`enabled?`（bool、デフォルト `true`） | モデルのビデオ価格を upsert します。`{ "ok": true }` を返します。bearer が `ARONA_ADMIN_TOKEN` と一致しない場合、`AUTH_ERROR` で拒否されます。 |

## Video

非同期ビデオ生成ジョブ（[Realtime & Video](../guides/realtime-video.md)を参照）。ジョブ進捗は `video.progress` / `video.done` / `video.failed` 通知としてセッションチャネルにプッシュされます。

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`、`prompt`、`negative_prompt?`、`images?`（`{ data_base64, mime_type }` の配列）、`duration_seconds?`（integer）、`width?`（integer）、`height?`（integer）、`provider?`（string）、`extra?`（object） | 非同期ビデオ生成ジョブを送信します。`{ "job_id", "stream_id" }` を返します — 進捗通知のために `stream_id` を購読します。 |
| `video.get` | JWT | `job_id`（UUID） | ジョブのステータス / 結果をポーリングします（status、progress、result、error、cost）。 |
| `video.list` | JWT | `limit?`（integer、デフォルト 20） | 呼び出し元のジョブを一覧します。`{ "jobs": [...] }` を返します。 |
| `video.cancel` | JWT | `job_id`（UUID） | 実行中のジョブをキャンセルします。`{ "ok": true }` を返します。 |

## System

| メソッド | 認証 | パラメータ | 説明 |
| --- | --- | --- | --- |
| `system.status` | public | — | 集約された gateway ステータス: `{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`。 |
| `system.probe` | anonymous（WS のみ） | — | WebSocket トランスポート上のワンショット liveness プローブ。サーバーは `{ "ok": true, "status": "ok" }` で ack してからソケットを閉じます — 匿名の訪問者は開いた接続を保持しません。未認証ソケット上の他のメソッドは `AUTH_ERROR` で拒否されます。 |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
