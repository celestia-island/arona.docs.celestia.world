---
title: "運用"
description: "稼働中の arona-server のヘルスエンドポイント、RUST_LOG トレーシング、upstream タイムアウト、エラーマッピング、トラブルシューティング。"
---

# 運用

このページは `arona-server serve` を運用するオペレーター向けです。プローブするヘルスエンドポイント、grep する価値のあるログ行、upstream backend に適用されるタイムアウトモデル、backend の失敗が HTTP エラーにどうマッピングされるか、そして人を躓かせる運用上の落とし穴を説明します。デプロイ自体は[デプロイガイド](./deployment.md)で説明されています。

## ヘルスマトリクス

3 つのヘルスエンドポイントはすべて認証不要で、プロセスが提供中であれば常に `200 OK` を返します — liveness / readiness の区別はありません:

| エンドポイント | レスポンス |
| --- | --- |
| `/healthz`、`/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | 上と同じ詳細ボディ |
| `/api/health` | plana `HealthResponse`: `status`、`version`（`CARGO_PKG_VERSION`）、`kind`（`Dev`）、`uptime`（秒）、`network`（transport / region / asn）、`build_hash`（`BUILD_HASH`）、`engine_version`（`"0.1.0"`） |

`/healthz` と `/readyz` は同じハンドラーのエイリアスで、`/v1/health` もそれを共有するため、Kubernetes スタイルのプローブと OpenAI 互換のヘルスルートは互換です。`/api/health` は uptime、network、エンジンバージョンを追加します。ロードバランサーとスーパーバイザーには `/readyz` を、よりリッチなペイロードが必要な場合は `/api/health` を使用してください。

## ロギング

サーバーは `tracing` 経由でログし、標準の `RUST_LOG` 変数でフィルタリングされます（`RUST_LOG=info` が一般的な設定。`RUST_LOG=debug` はプローブトラフィックを明かします）。知っておく価値のあるイベントを、おおよそ頻度順に:

| ログ行 | レベル | 何を伝えるか |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | チャットリクエストごとに 1 行。`key_prefix`、`model`、`stream`、`request_id` 付き — 最もシンプルなリクエストごとの監査証跡です。 |
| `request completed` | info | すべての**非ストリーミング**の `/v1/chat/completions` と `/v1/embeddings` レスポンスの後に `logging_middleware` ヘルパーがログします: `method`、`path`、`status`、`latency_ms`、`trace_id`。（ストリーミングチャットは代わりに開始時に `chat completions SSE request` をログします。） |
| `usage recorded` / `usage persisted` | info | usage 行が記録され（インメモリ、token / コスト付き）、その後 `usage_records` テーブルに書き込まれました。 |
| `external probe: sending` / `external probe: returned` | debug | external backend の `/v1/models` のヘルスプローブ。`matched` はプローブが 2 秒のプローブタイムアウト内に完了したかどうかを示します。 |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | billing ゲートに拒否された `/v1/*` リクエスト — クライアントは 429 と `Retry-After` を受け取りました。 |
| `rpc billing gate rejected: monthly quota exceeded` | warn | JWT 認証メソッドの RPC 側クォータゲート（ユーザー全体のウィンドウ。JSON-RPC エラーレスポンス）。 |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | 起動時の復元: 管理者登録の backend と agent ノードがデータベースから読み込まれ、再びルーティング可能になりました。 |
| `Shutdown signal received, draining connections…` | info | グレースフルシャットダウンが始まりました（SIGINT/SIGTERM）。 |

## タイムアウトモデル

タイムアウトは external backend に使用される upstream クライアントで強制されます（`packages/core/src/backends/external.rs`）:

| タイムアウト | 値 | 適用対象 |
| --- | --- | --- |
| 接続 | 10s | upstream の TCP/TLS 接続の確立。 |
| 読み取りアイドル | 読み取りごとに 120s | すべての upstream 呼び出し。受信バイトごとにクロックがリセットされるため、healthy だが遅いストリームが切断されることはありません。 |
| 非ストリーミング全体 | 600s | 非ストリーミングのチャット / embeddings 呼び出し — 遅いが生きている upstream がリクエストを永遠に保持することはできません。 |
| ストリーミング（SSE） | なし | ストリーミング呼び出しは**全体のデッドラインを持ちません**。長い生成は合法で、ハング検出は読み取りアイドルタイムアウトに依存します。 |
| ヘルスプローブ | 2s | `/v1/models` プローブ。 |

## エラーマッピング

Backend の失敗はチャット / embeddings ハンドラーで HTTP ステータスにマッピングされます（`packages/core/src/gateway/server.rs`）:

| 条件 | HTTP | `type` / `code` | メッセージ |
| --- | --- | --- | --- |
| Upstream の非 2xx ステータス（`UpstreamStatus`） | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| Upstream のトランスポート失敗（`RequestFailed`） | **502 Bad Gateway** | `server_error` / `bad_gateway` | トランスポートエラー文字列 |
| その他の backend エラー | **500** | `server_error` / `backend_error` | エラー文字列 |
| モデルの backend がない（`NoBackend`） | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| 無効な API key（`Unauthorized`） | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| レート制限（`RateLimited`） | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

設計意図: 呼び出し元は「providerが拒否または失敗した」（502）と「gateway 自体が壊れている」（500）を区別できます。すべてのエラーボディは同じ OpenAI スタイルのシェイプを持ちます — `{"error":{"message":...,"type":...,"code":...}}`（`json_error_response`）。billing ゲートの 429 はさらに `Retry-After` ヘッダーを持ち、それぞれ `quota_error`/`quota_exceeded`（クォータ）と `rate_limit_error`/`rate_limit_exceeded`（tier レート制限）を使用します。

## トラブルシューティング

### 新しく登録した backend はプローブされるまで fail-closed のまま

External backend は未知のヘルス状態で始まり、`"<url> not probed yet"` を報告します。次の場合に healthy に切り替わります: (a) ヘルスチェッカーの最初のラウンドが実行されたとき — 起動時に即座に、その後 60 秒ごと — または (b) 登録時または復元時に起動される fire-and-forget プローブが成功したとき（通常は約 1〜2 秒以内）。それまでは、backend にルーティングされたリクエストは設計上 fail-closed になります。

### プローブの `/models` での 404 は一部の backend では正常

External プローブは `GET {base}/v1/models`（パスプレフィックス付きのベース URL では `{base}/models`）にヒットします。一部の OpenAI 互換サーバーはチャットを実装していますがモデル一覧を公開していません — Zhipu GLM コーディングプランのエンドポイントがその一例です。**404 は許容されます**: backend は healthy とマークされ、管理者設定の models リストがルーティングの権威のままです。真に失敗したプローブ（タイムアウト、ネットワークエラー、その他の非 2xx）だけが backend を unhealthy とマークします。

### 何も生成しない SSE ストリームは請求されない

ストリーミングレスポンスが usage に記録されるのは、ストリームがテキスト**または**終端の usage を生成した場合だけです。どちらもなしで終了したストリームはまったく記録されません。一致する `usage recorded` 行のないリクエストを見つけたら、ストリームが実際にコンテンツを生成したかどうかを確認してください。

### バージョン報告

ヘルスボディの `version` は `CARGO_PKG_VERSION` です。`build_hash` は `packages/core/build.rs` が出力するビルド時の `BUILD_HASH` 値です。ノード間で `build_hash` を比較して、すべてが同じアーティファクトを実行していることを確認してください。

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
