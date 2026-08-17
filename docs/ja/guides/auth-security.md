---
title: "認証とセキュリティ"
description: "JWT セッション、API key、3 つの admin ゲート、パスワードポリシー、二系統のレート制限、セキュリティモデル。"
---

# 認証とセキュリティ

Arona は 2 つの系統で呼び出し元を認証します: 対話型クライアント（チャット + 管理 UI、RPC 呼び出し）用の **JWT セッションtoken**と、プログラムによる OpenAI 互換トラフィック用の **API key**（`arona-…`）です。管理サーフェスは別の admin token で保護されます。このページでは、その仕組み、セキュリティモデル、そしてセキュリティ監査から判明した既知の低リスク残件を説明します。

## JWT セッション

セッションは `kirino_session` tokenマネージャーが発行する JWT access/refresh tokenペアを使用します:

- **Access tokenの TTL: 900 秒（15 分）。**
- **Refresh tokenの TTL: 604,800 秒（7 日）。**

Access token は JSON-RPC プレーン（`/api/rpc`）と `GET /v1/models` を認証します。SSE サイドカー（`/api/rpc/events`）はセッション id でキーイングされます。これは Bearer クレデンシャルではなく、認証済み RPC 呼び出し中に鋳造されるケーパビリティです。`/v1/chat/completions`、`/v1/embeddings`、`/v1/video/*` エンドポイントは **API key** を要求します（そこでは JWT は受け付けられません）。Access token は短命なので、盗まれたtokenは短時間しか使えません。Refresh tokenは `auth.refresh` で新しいペアと交換されます。

Refresh は**tokenファミリーのローテーション**を使用します: refresh tokenを消費するとそれが無効化され新しいものが発行され、消費済みの refresh tokenを再利用するとファミリー全体が失効します — `auth.refresh` は `AUTH_ERROR` とメッセージ `Refresh token reused` で応答します（基になるエラーは `TokenReused`、「refresh token has been reused — token family revoked」）、アカウントは再ログインが必要です。ファミリーの失効は**インメモリ**です（`revoked_families` セット）: サーバーの再起動でクリアされるため、保護は再起動をまたいでベストエフォートです（ユーザーごとのセッション状態は再起動を生き延びません）。

署名シークレットは `JWT_SECRET` 環境変数から取得されます。`MOCK_MODE=1` 以外では、`JWT_SECRET` が未設定か組み込みの開発シークレットのままの場合、サーバーは**起動を拒否**します。したがって、本番インスタンスが公開定数で署名されたtokenを誤って提供することは決してありません。強力でランダムなシークレットを使用し、コミットしてはなりません。

## API key

API key は OpenAI 互換サーフェス用のマシンクレデンシャルです:

- **形式:** `arona-{uuid}`。
- **保存:** `api_keys` テーブルにはキーの **SHA-256 ハッシュ**だけが保存されます — 平文は `keys.create` のレスポンスで一度だけ返され、後から復元することはできません。
- **キープレフィックス:** 最初の 8 文字（`key_prefix`）は表示と usage の帰属のために平文で保存されます。UI は `arona-XXXX…abcd` のようなマスク形式を表示します。
- **失効:** キーのルックアップは `api_keys.is_active = TRUE` と結合するため、失効したキーは即座に検証に失敗します — 待つべきキャッシュ TTL はありません。

## Admin ティア

それぞれ独自のクレデンシャルを持つ 3 つの異なる admin ゲートがあります:

1. **`/api/admin/*` ルート** — backend とエイリアスの管理（`POST/GET/DELETE /api/admin/backends`、`POST/GET/DELETE /api/admin/aliases`）は `Authorization: Bearer ARONA_ADMIN_TOKEN` ヘッダーを要求します。`ARONA_ADMIN_TOKEN` が未設定の場合、`check_admin` は常に失敗し、すべての admin ルートは **401「Admin access required」** を返します — 管理サーフェス全体が開放されるのではなく無効化されます。

2. **`agents.*` と `engine.invoke` RPC メソッド** — agent クラスターとエンジン制御プレーンは、アカウントが `users.is_admin = true` を持つ JWT を要求します。認証済みの非 admin は実装定義コード **-32007（`ADMIN_REQUIRED`）** とメソッド固有のヒント（例: `agents.deploy starts model deployments on GPU nodes`）で拒否されます。**未認証**の呼び出し元は標準の **-32005（`AUTH_ERROR`）** を受け取り、サーバーはメソッドが特権的であることを一切明かしません。

3. **`billing.plan.set` と `billing.video.pricing.set` RPC メソッド** — billing の変更は admin HTTP ルートと同じ Bearer `ARONA_ADMIN_TOKEN` を要求します。これがないと `AUTH_ERROR`「Admin access required」を返します。

**最初に登録したユーザーが admin になります**（`users.is_admin = true`）。以降の登録はすべて通常ユーザーで、登録が開放されるのは `ARONA_REGISTRATION_OPEN` が truthy な値に設定されている間だけです。

## パスワードポリシー

パスワードは**両方の**ルールを満たす必要があります（登録時とパスワード変更パスで強制されます）:

- 最低 **8 文字**、かつ
- **4 つの文字カテゴリのうち少なくとも 3 つ**: 大文字、小文字、数字、記号。

## レート制限

レート制限は 2 つの独立した系統で実行され、どちらか一方が **429** でリクエストを拒否できます:

### 1. インメモリスライディングウィンドウ（アイデンティティごと）

認証済みのすべての `/v1` リクエストは、呼び出し元のアイデンティティでキーイングされたインメモリスライディングウィンドウリミッターを通過します:

- **API key 呼び出し**はキーの **SHA-256 ハッシュ**でキーイングされます;
- **JWT 呼び出し**は `u:<email>` でキーイングされます — JWT は 15 分ごとにローテーションするため、token インスタンスでウィンドウをキーイングすると、更新のたびにウィンドウが静かにリセットされます。

デフォルトのバジェットは**毎分 60 リクエスト**で、`ARONA_API_RATE_LIMIT_RPM` で上書きできます（多数の並列 LLM 呼び出しをファンアウトする agent パイプラインでは高めに設定します）。**0 に設定するとすべてのリクエストをブロックします**。

### 2. Tier レート制限（キーごと、データベースから）

Billing tier はキーごとの `rate_limit_rpm` を持ちます。チェックは**直近 60 秒間**のキープレフィックスに対する `usage_records` 行を数えます（usage は各応答の後に永続化されるため、ウィンドウは最大 1 つの進行中リクエスト分だけ遅れます。DB 障害は fail-open します）。シード済みの **free tier は 10 RPM** です。pro/enterprise tier は上限を引き上げます。月次クォータの強制も同じ拒否パスを共有します。

### ログインレート制限

クレデンシャルの推測はログインエンドポイントで抑制されます: **メールごとに 5 分ウィンドウあたり 5 回の失敗**、**IP ごとに 5 分ウィンドウあたり 20 回**、それぞれ 15 分のロックアウトが続きます。

### `Retry-After`

すべての 429 レスポンスは `Retry-After` ヘッダーを持つため、OpenAI 互換クライアントはエンドポイントを叩き続けるのではなくバックオフします: クォータ拒否は**月末までの秒数**、レート制限拒否は**60** に設定されます。クォータモデルは[Billing & Usage](billing-usage.md)を参照してください。

## セキュリティモデルのメモ

- **CORS は任意のオリジンを許可します**（`allow_origin(Any)`）— Arona は多くのファーストパーティ・サードパーティクライアントが利用する backend です。デプロイメントでオリジンを制限する必要がある場合は、CORS を強制するリバースプロキシを前面に置いてください。
- **リクエストボディは 1 MB に制限されています**（`RequestBodyLimitLayer`）— gateway 上のメモリ使用を制限します。
- **gateway は TLS を終端しません** — 平文 HTTP でリッスンします。HTTPS を終端するリバースプロキシの背後に置いてください（[デプロイ](deployment.md)を参照）。
- **シークレットは環境からのみ取得されます**: `ARONA_ADMIN_TOKEN` と `JWT_SECRET` は環境変数から読み込まれ、リポジトリにコミットしてはならない強力なランダム値でなければなりません。
- デフォルトのサーバーバインドアドレスは `0.0.0.0` です。ネットワーク層で露出を制限してください。

## 既知の低リスク残件（監査より）

以下は現状のまま文書化されています。意図的なものか、今のところ受け入れられているものですが、信頼されたネットワークの外にインスタンスを公開する場合は知っておく価値があります:

- **`providers.list` は公開**ですが、`providers.add` / `providers.update` / `providers.remove` / `providers.test` は JWT を要求します。公開された読み取りパスはprovider カタログを明かしますが、秘密は何も含みません。
- **`/ws/agent` は認証なしの制御プレーンです**: GPU agent はクレデンシャルなしで接続し、自己登録します（`register` / `heartbeat` / command-result フレーム）。WebSocket ポートに到達できる人は誰でも偽の agent を登録できます。運用上のトレードオフは[Agent クラスター](agent-cluster.md)を参照してください。
- **`memory.delete` は所有権チェックなしの JWT のみです**: 認証済みのユーザーなら誰でも `node_id` でメモリノードを削除できます。メモリの削除にはログインが必要ですが、ノードの所有は不要です。

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
