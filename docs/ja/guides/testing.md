---
title: "テスト"
description: "Arona のテストピラミッド — ユニットテスト、密閉型インテグレーション、PostgreSQL ゲートのインテグレーション、ライブサーバースモークテスト、mock サーバー、実クレデンシャルスモークの規律。"
---

# テスト

Arona のテストはレイヤー状に配置されているため、デフォルトの `cargo test` 実行は高速で密閉型（hermetic）であり、データベースもネットワークも必要としません。一方、より重いスイートは明示的なオプトインで、実際のワイヤーサーフェスと実 PostgreSQL を試します。このページではレイヤー、それらを実行するコマンド、実クレデンシャルスモーク実行に関するワークスペースの規律を説明します。

## ユニットテスト

カバレッジの大部分は `packages/core/src` 内のプレーンなユニットテストです: 217 個の `#[test]` / `#[tokio::test]` 関数と、`packages/agent` と `packages/cli` に約 23 個。次のコマンドで実行します:

```bash
cargo test --workspace
```

ネットワークなし、データベースなし。主要なスイート:

- **auth.rs** — パスワードポリシー（≥8 文字 AND 4 つの文字カテゴリのうち ≥3）、生の INSERT/REVOKE SQL の `::uuid` キャスト、リクエストのデフォルト、`false` にフォールバックする admin フラグの読み取り。
- **billing/mod.rs** — コスト*または*token次元でのクォータ計算、月次ウィンドウ（`month_start`、`seconds_until_month_end`）、レート制限の上限（RPM *ちょうど*で作動、`None` = 無制限）、月次 usage / tier / キーウィンドウクエリの SQL シェイプガード、upstream 報告値を優先する `estimate_usage`。
- **routing/mod.rs** — エイリアス解決、`:latest` サフィックスマッチング、provider ヒント、最少負荷選択、会話ピン留め。
- **gateway/mod.rs** — agent-backend 登録: `agent-{model_id}` の登録、置き換える（複製しない）再登録、ルーターを復元する登録解除。

## 密閉型インテグレーション（常時実行、DB なし）

`packages/core/tests/gateway_integration.rs` には、データベースに触れずに実際のシリアライゼーション / コントラクトロジックを試す 3 つの常時実行テストが含まれます:

- **A1** — JSON-RPC の id エコーシリアライゼーションコントラクト: 数値、文字列、null のリクエスト id が、型忠実度を保って plana の `Id` 列挙型を往復します。
- **A2** — admin ゲートのエラーコードコントラクト: `AUTH_ERROR`（-32005、匿名）と `ADMIN_REQUIRED`（-32007、認証済み非 admin）が別個のままで、実装定義の範囲にあり、plana のコードや billing の `QUOTA_ERROR`（-32006）と衝突しないこと。
- **A3** — `estimate_usage`: upstream 報告の usage がそのまま優先されます。それがない場合、ローカルのトークナイザー推定が非ゼロの prompt/completion 数を生成し、その合計はそれらの和になります。

`packages/core/tests/smoke.rs` はさらに 3 つの常時実行テストを追加します: ハードウェア検出、モデルレジストリのルートパス、`MOCK_MODE=1` 下の設定デフォルト。

## PG ゲートのインテグレーション

完全なインプロセス gateway スイート — `packages/core/tests/gateway_integration.rs` — は、完全な axum ルーターをランダムなループバックポートで起動し、使い捨ての OpenAI 互換 mock upstream を実際の admin API 経由で登録し、reqwest でワイヤーサーフェスを駆動します。`AuthManager` はすべてのパスで PostgreSQL と通信するため（`MOCK_MODE=1` でもアカウントを*データベースに*シードするだけ）、このスイートは `ARONA_TEST_PG=1` の背後にゲートされ、デフォルトではスキップされます。10 個のテスト:

- **T1** 登録 + ログイン + `keys.create`/`keys.list`（一覧では生のキーがマスクされ、`arona-` プレフィックス）。
- **T2** PostgreSQL への usage レコード永続化付きの REST チャット。
- **T3** ワイヤー上の JSON-RPC id エコー（成功パスとエラーパス）。
- **T4** `agents.list` の admin ゲート: 匿名 → `AUTH_ERROR`、非 admin → `ADMIN_REQUIRED`。
- **T5** upstream 401 → HTTP 502 `bad_gateway` と upstream の詳細。
- **T6** 登録時プローブがモデルを公開する（静的モデルリストなしで 10 秒以内にモデルが `GET /v1/models` に現れる）。
- **T7** `chat.send` による会話の永続化（両方のターンが `conversations.get` に届く）。
- **T8** free tier の billing ゲート: キーごとに 10 RPM、ウィンドウ内の 11 番目のリクエストは 429 `rate_limit_exceeded`。
- **T9** upstream から報告された終端 usage を持つ SSE ストリーム。
- **T10** 不正な JSON → 400。未知のモデル → 404 `model_not_found`。

モジュールドキュメント（gateway_integration.rs:18-26）の使い捨て PostgreSQL ワンライナーで実行します:

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

これらは使い捨てテストコンテナ専用の例示クレデンシャルです — 実データベースに向けないでください。

## ライブサーバースモーク

`packages/core/tests/auth_flow.rs` は**ライブ** Arona サーバーに対して完全な `register → login → keys.create → /v1/models → /v1/chat/completions → usage.list` チェーンを歩き、デプロイされた認証ループを反映します。デフォルトでは `#[ignore]` されています — プレーンな `cargo test` 実行はネットワークに触れません。明示的に実行します:

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

ノブ:

- `ARONA_TEST_URL` — ベース URL（デフォルト `http://127.0.0.1:8420`）。
- `ARONA_TEST_EXPECT_CHAT=1` — `POST /v1/chat/completions` が 200 を返すことをハードアサートします。これがないと、テストは認証が通ったことだけをアサートします（401/403 ではない）。対象環境に推論providerが設定されていない可能性があるためです。

スイートにはネガティブテストも含まれます: 未認証のチャット補完と未認証の `GET /v1/models` はどちらも 401 で拒否されなければなりません。

## Mock サーバー

`scripts/mock/server.py` は aiohttp ベースの OpenAI 互換フェイクで、クイックスタートとスモーク実行で使用されます。`POST /v1/chat/completions`（非ストリームと SSE）、`GET /v1/models`、`GET /api/health`、`/api/rpc` の JSON-RPC WebSocket/HTTP サーフェス、`/api/rpc/events` の SSE サイドカー、そして `GET /api/test-key`（他のサービスが発見できるように mock API key を返す）を提供します。デフォルトでポート 8429 をリッスンします（`ARONA_MOCK_PORT` で上書き、ホストは `ARONA_MOCK_HOST`）。[クイックスタート](quickstart.md)はこれを使って、実モデルproviderなしでエンドツーエンド環境を立ち上げます。

## 実クレデンシャルスモークの規律

実provider（DeepSeek / GLM）に対するスモーク実行は、意図的に**リポジトリのテストではありません** — 実クレデンシャルと実コストが必要なため、CI や git ツリーには置けません。gateway_integration モジュールドキュメント（gateway_integration.rs:54-55）に文書化されたワークスペース規約は次のとおりです:

- エビデンスファイルは `/mnt/work/arona-pr*-smoke.md` に置きます — ワークスペースローカルで、git にコミットされることはありません。
- クレデンシャルは環境からのみ取得されます。バジェットは小さく保ちます。
- 推論パスに触れる各 PR には、書面のエビデンスレコードが付けられます。

mock サーバーは CI とローカル開発でのこれらの実行の代役です。実クレデンシャルスモークはリリース時の人間のステップです。

## CI

`.github/workflows/ci.yml` は `cargo fmt`、`cargo clippy`、`cargo test --workspace`、`cargo-deny` を組織のセルフホストランナー（`[self-hosted, linux, x64, local]`）で実行します。`ci-hosted.yml` は同じチェックを GitHub ホストランナーでミラーします。`.github/workflows/docs.yml` は lagrange でこのドキュメントサイトをビルドし、`docs/**` に触れるプッシュで GitHub Pages にデプロイします。

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
