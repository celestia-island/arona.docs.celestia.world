---
title: "クイックスタート"
description: "組み込みの mock upstream を使った Arona のエンドツーエンド手順 — マイグレーション、サーバー起動、backend 登録、API key 作成、チャットまで。"
---

# クイックスタート

このガイドでは、**組み込みの mock upstream** を使って、1 台のマシン上で Arona の完全なエンドツーエンドセットアップを進めます。実際のモデル重み、GPU、外部 API アカウントは不要です。この手順を終えると、次のものが揃います。

- 稼働中の Arona gateway（`/v1/*` OpenAI 互換 REST API と `/api/rpc` JSON-RPC 管理プレーン）、
- `external` backend として登録された mock upstream、
- ユーザーアカウントと API key、
- mock に対するストリーミング**および**非ストリーミングのチャットターン、
- `usage.list` で確認できる usage レコード。

## 前提条件

- **Rust ツールチェーン**（リポジトリルートの `rust-toolchain.toml` を参照）。
- **Python 3** と `aiohttp` — mock upstream にのみ必要です（`pip install aiohttp`）。
- 稼働中の **PostgreSQL** インスタンスとその接続 URL。

## 1. 環境変数を設定する

Arona は**プロセス起動時**に環境変数から設定を読み込みます。必須は `DATABASE_URL` と `JWT_SECRET` の 2 つで、これらがないとサーバーは起動を拒否します（`MOCK_MODE=1` の場合を除く）。`ARONA_ADMIN_TOKEN` は強く推奨されます。これがないと、`/api/admin/*` の全ルートが 401 を返します。

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

これらの変数はプロセスの起動時に一度だけ読み込まれます。変更した場合はサーバーを再起動してください。完全な変数リファレンスは[設定](configuration.md)を参照してください。

## 2. マイグレーションを実行してサーバーを起動する

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

新しいデータベースであれば `serve` だけで十分です。起動時に自動マイグレーションが実行されます。サーバーはデフォルトで `0.0.0.0:8420` にバインドします（`ARONA_HOST` / `ARONA_PORT` で上書き可能）。

## 3. mock upstream を起動する

別のターミナルで:

```bash
python3 scripts/mock/server.py
```

mock はデフォルトで `127.0.0.1:8429` をリッスンする aiohttp サーバーです（ポートは `ARONA_MOCK_PORT` で上書き可能）。起動時に API key を出力し、`GET /api/test-key` も提供します。これは `{"api_key": ..., "base_url": ...}` を返します。また、後述で使う `gpt-5.5` を含むいくつかのモデル id を公開し、通常・ストリーミングの両方のチャット補完に応答します。

出力されたキーを取得します:

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. mock を external backend として登録する

Backend は管理 HTTP API を通じて登録します:

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

backend は登録時に即座にプローブされ、約 1〜2 秒で healthy に切り替わります。そのプローブが完了するまでは、fail-closed の「未プローブ（not probed yet）」状態のままです（下のトラブルシューティングのボックスを参照）。設定は永続化されるため、backend は再起動後も維持されます。

## 5. アカウントを登録してログインする

アカウントは JSON-RPC プレーン `POST /api/rpc` 上にあります。`ARONA_REGISTRATION_OPEN=1` が設定されているため `auth.register` は開放されており、最初に登録したユーザーが admin になります。

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

パスワードは最低 8 文字**かつ**、4 つの文字カテゴリ（大文字、小文字、数字、記号）のうち少なくとも 3 つを含む必要があります。次にログインして JWT ペアを取得します:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

レスポンスから `access_token` をエクスポートします:

```bash
export JWT="<access_token from the login response>"
```

## 6. API key を作成する

`keys.create` は JWT 認証で、完全な `arona-{uuid}` シークレットを**一度だけ**返します。データベースには SHA-256 ハッシュしか保存されないため、今すぐコピーしてください:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. チャット（非ストリーミング）

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

`choices[0].message` と `usage` ブロックを持つ OpenAI スタイルの補完オブジェクトが返ります。

## 8. チャット（ストリーミング）

同じエンドポイントに `"stream": true` を付けると、server-sent events で応答します。tokenごとに 1 つの `data:` チャンクで、最後に `data: [DONE]` チャンクで終了します:

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. usage を確認する

すべてのチャットターンは、キーのプレフィックス配下に usage 行を記録します。JWT で照会します:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

上で行った `gpt-5.5` リクエストのレコードが 1 件以上表示されるはずです。

## トラブルシューティング

- **`No backend available for model: gpt-5.5`（HTTP 404、`code: model_not_found`）** — そのモデル id を提供する backend が登録されていません。backend が登録されていない（または `models` リストにその id が含まれていない）、あるいは登録呼び出しが失敗した可能性があります。`GET /api/admin/backends`（admin token）で確認してください。
- **`all backends unhealthy`（HTTP 500、`backend_error`）** — モデルに対して backend は*登録されています*が、healthy な候補がありません。登録直後の external backend は fail-closed の「未プローブ（not probed yet）」状態で始まり、登録時プローブが完了すると約 1〜2 秒後に healthy に切り替わります。その間のウィンドウでチャットするとこのエラーに当たります。少し待って再試行するか、mock が実際に `127.0.0.1:8429` で稼働しているか確認してください。
- **`/v1/*` で HTTP 401** — `Authorization` ヘッダーがないと `Missing Authorization header. Use: Bearer <api_key>`、未知のキーだと `Invalid API key` が返ります。`$AR_KEY`（プレフィックスではなく完全なシークレット）を再確認してください。
- **`/api/admin/*` で HTTP 401 `Admin access required`** — bearer token が `ARONA_ADMIN_TOKEN` と一致しないか、変数が未設定です（その場合ルートは常に拒否します）。設定後にサーバーを再起動してください。
- **`auth.register` が「Registration is closed」で失敗する** — `ARONA_REGISTRATION_OPEN` が truthy でない場合、登録は無効です。サーバー起動**前**に `ARONA_REGISTRATION_OPEN=1` を設定してください（起動時に読み込まれます）。または、最初のユーザーになってください — 最初に登録したユーザーは常に許可され、admin になります。
- **HTTP 429 レート制限** — 独立した 3 つの制限が発火し得ます:
  - キーごとのインメモリ制限。デフォルト 60 RPM（`ARONA_API_RATE_LIMIT_RPM`）→ `Rate limit exceeded. Try again later.`;
  - free billing tier のキーごとの 10 RPM 制限 → `Retry-After: 60` ヘッダー付きの 429;
  - free tier の月額 $1 / 10 万tokenのクォータ → 次の請求期間を指す `Retry-After` 付きの 429。

## 次のステップ

- [設定](configuration.md) — すべての環境変数。
- [Backends](backends.md) — backend タイプ、URL セマンティクス、プロービング。
- [デプロイ](deployment.md) — ベアメタル、systemd、Docker。
- [OpenAI 互換 REST API](../api/openai-rest.md) — 完全な `/v1/*` サーフェス。
- [JSON-RPC API](../api/jsonrpc.md) — 上で使用した管理プレーン。

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
