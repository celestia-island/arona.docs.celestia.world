---
title: "OpenAI 互換 REST API"
description: "OpenAI スタイルの /v1/* リファレンス — チャット補完、embeddings、モデル一覧、非同期ビデオ生成、エラーシェイプ、レート制限。"
---

# OpenAI 互換 REST API

Arona は LLM チャット、embeddings、モデル一覧、ヘルスプロービング、非同期ビデオ生成のために `/v1/*` 配下に OpenAI 互換の REST サーフェスを公開します。ベース URL を指す任意の OpenAI SDK がチャットと embeddings で動作します。ビデオエンドポイントは OpenAI のタスク型 submit/poll 規約に従います。

すべてのリクエストとレスポンスのボディは JSON です。エラーは統一されたシェイプを使用します（[エラー](#errors)を参照）。ミドルウェアレイヤーでの認証失敗が唯一の例外で、プレーンテキストで返されます（[認証](#authentication)を参照）。

## エンドポイント一覧

| メソッド | パス | 説明 |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | チャットターン。ストリーミングまたは非ストリーミング。 |
| `POST` | `/v1/embeddings` | 1 つまたは多数の入力の埋め込みベクトル。 |
| `GET` | `/v1/models` | ルーターモデルとクイックスタートモデルをマージした一覧。 |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`。 |
| `POST` | `/v1/video/generations` | 非同期ビデオ生成タスクを送信。 |
| `GET` | `/v1/video/generations/{id}` | ビデオタスクのステータス / 結果をポーリング。 |

`/api/health`、`/healthz`、`/readyz` は追加の readiness プローブです（`/v1/health` の Kubernetes スタイルエイリアス）。

## 認証

チャット、embeddings、ビデオエンドポイントは `Authorization: Bearer` ヘッダーの **API key** で認証します。API key は管理プレーン（`keys.create`。[JSON-RPC API](./jsonrpc.md#keys) を参照）で作成され、`arona-<uuid>` の形をしています。サーバー側では SHA-256 ハッシュとして保存されます。

```
Authorization: Bearer arona-CHANGE_ME
```

- **ヘッダーなし** → `401` プレーンテキスト: `Missing Authorization header. Use: Bearer <api_key>`。
- **無効または失効したキー** → `401` プレーンテキスト: `Invalid API key`。
- `GET /v1/models` はさらに **JWT** access token（`auth.login` / `auth.register` が発行）を受け付けます。Web ダッシュボードが RPC プレーンに使うのと同じtokenでモデルを一覧できるようにするためです。このエンドポイントではメッセージは `Missing Authorization header. Use: Bearer <api_key_or_jwt>` と `Invalid API key or JWT` です。

ミドルウェアレベルの拒否はプレーンテキストボディで、[エラー](#errors)で説明する JSON エラーシェイプではありません — JSON シェイプはリクエストがハンドラーに到達して初めて生成されます。

認証済みのすべての `/v1` リクエストは**インメモリのキーごとのレートリミッター**（デフォルト 60 RPM、60 秒ウィンドウ、`ARONA_API_RATE_LIMIT_RPM` で設定可能）も通過します。超過すると `429` プレーンテキスト `Rate limit exceeded. Try again later.` が返ります。Tier レベルのクォータとレート制限は別途強制され、`Retry-After` ヘッダー付きの JSON 429 を返します（[429 と Retry-After](#429-and-retry-after)を参照）。

> API key、プロジェクト、そのスコープの管理は[認証とセキュリティ](../guides/auth-security.md)で説明されています。

## POST /v1/chat/completions

ストリーミングサポートと arona 固有の拡張（`conversation_id`、`memory`、`extra`、`provider`）を持つ、中核の OpenAI 互換チャットエンドポイントです。

### リクエストボディ

| フィールド | タイプ | 必須 | メモ |
| --- | --- | --- | --- |
| `model` | string | はい | `GET /v1/models` が一覧するモデル id。 |
| `messages` | array | はい | チャットターン。下記参照。 |
| `stream` | boolean | いいえ | デフォルト `false`。`true` の場合、レスポンスは SSE ストリームになります（[ストリーミング](#streaming)を参照）。 |
| `temperature` | number | いいえ | サンプリング温度。upstream に転送されます。 |
| `max_tokens` | integer | いいえ | 補完tokenの上限。upstream に転送されます。 |
| `conversation_id` | string | いいえ | セッションアフィニティ + 永続化。会話は存在し、API key ユーザーに属する必要があります（それ以外は `403` `conversation_forbidden`、存在しない場合は `404` `conversation_not_found`）。ユーザーターンは送信時に永続化され、アシスタント応答はターン完了時に永続化されます。ルーティングは会話を最初にサービスを提供した backend にピン留めします。 |
| `memory` | boolean | いいえ | Memory gateway のオーバーライド。デフォルト `true`（memory gateway が有効な場合にメモリリコールが注入されます）。`false` はこのリクエストのリコール注入を無効にします。 |
| `extra` | object | いいえ | upstream ペイロードのトップレベルにマージされる自由形式のパススルー（下記参照）。 |
| `tools` | array | いいえ | OpenAI スタイルの関数呼び出し定義。そのまま upstream に渡されます。 |
| `provider` | string | いいえ | backend **名**（または kind）と大文字小文字を区別せず一致する明示的な backend 選択ヒント。設定すると、ヒントに一致する backend だけが候補になります。 |

**`messages` エントリ**は `{ "role": "user" | "assistant" | "system", "content": "..." }` です。マルチモーダル / agent ワークロード用に 2 つの拡張が upstream に転送されます:

- `images` — ビジョンリクエスト用の添付画像（`{ "media_type", "data", "position" }` オブジェクトの配列。external backend はそれらを OpenAI の `image_url` コンテンツパーツとしてレンダリングします）。
- `tool_calls` — upstream モデルが生成した関数呼び出しペイロードで、後続ターンでエコーバックされます。external backend はこれらを転送する必要があります。そうしないと agent パイプライン（例: scepter スキルチェーン）がすべてのツール呼び出しを失います。

**`extra` のマージルール**: すべての `extra` キーは upstream リクエストペイロードのトップレベルにマージされますが、2 つのハードな保証があります — 予約キー `model`、`messages`、`stream`、`temperature`、`max_tokens`、`options` は**決して**上書きされず、gateway 自身がすでに設定したキーも上書きされません。オブジェクトでない `extra` 値は無視されます。

**`tools` エントリ**は `{ "type": "function", "function": { "name", "description"?, "parameters"? } }` で、そのまま転送されます。

### 非ストリーミングレスポンス

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1720000000,
  "model": "Qwen/Qwen3-1.7B",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 12, "completion_tokens": 2, "total_tokens": 14 }
}
```

- `choices[].message` は関数呼び出しターンで `tool_calls` を運ぶことがあります。
- リクエストのメモリ状態は **`X-Arona-Memory`** レスポンスヘッダーに反映されます: `enabled` | `disabled` | `offline`。

### ストリーミング

`"stream": true` を設定します。レスポンスは `text/event-stream` の SSE ストリームです — チャンクごとに 1 つの `data:` 行で、それぞれ単一の JSON `ChatChunk` を運びます:

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta` は `content` を運びます（関数呼び出しストリームでは `index`/`id`/`type`/`function` 付きの `tool_calls` デルタも）。
- OpenAI 互換 upstream では**終端チャンク**が実際のtoken数を持つ `usage` フィールドを運びます。gateway はそれを記録します（usage を報告しない upstream — 例: ollama / ws_engine — ではローカルトークナイザー推定にフォールバック）。
- ストリームは `data: [DONE]` で終了します。
- ストリームエラーは `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}` を運ぶ `data:` イベントとして配信されます。その後も `[DONE]` イベントは続き、失敗したストリームでは usage 記録とアシスタント永続化はスキップされます。
- `X-Arona-Memory` ヘッダーは SSE レスポンスにも存在します。

### 例

```bash
curl http://192.0.2.10:8080/v1/chat/completions \
  -H "Authorization: Bearer arona-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-1.7B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

## POST /v1/embeddings

| フィールド | タイプ | 必須 | メモ |
| --- | --- | --- | --- |
| `model` | string | はい | 埋め込みモデル id（例: `nomic-embed-text` — 素の名前は `:latest` タグにもマッチします）。 |
| `input` | string または string[] | はい | 1 つの入力、または多数。 |

レスポンス: `{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`。

## GET /v1/models

今日ルーティング可能なモデルを一覧します: すべての healthy な登録済み backend のモデル一覧と、組み込みの**クイックスタートモデル**（backend が登録される前でも常にアドバタイズされる）をマージしたものです: `Qwen/Qwen3-0.6B`、`Qwen/Qwen3-1.7B`、`HuggingFaceTB/SmolLM2-1.7B-Instruct`、`google/gemma-3-1b-it`、`microsoft/Phi-4-mini-instruct`、`deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`。

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

クイックスタートモデルは `owned_by` がproviderに設定されて表示されます。ルーターモデルは所有する backend の名前を運びます。

## ビデオ生成

ビデオ対応 backend（例: `minimax-cloud`。[Backends](../guides/backends.md)を参照）用のタスク型ビデオエンドポイントです。ジョブは非同期で進行します。`done` になるまでステータスエンドポイントをポーリングしてください。

### POST /v1/video/generations

| フィールド | タイプ | 必須 | メモ |
| --- | --- | --- | --- |
| `model` | string | はい | ビデオ対応 backend に登録されたビデオモデル id。 |
| `prompt` | string | はい | 生成プロンプト。 |
| `negative_prompt` | string | いいえ | ネガティブプロンプト。 |
| `images` | array | いいえ | コンディショニング / 参照画像。`{ "data_base64": "...", "mime_type": "image/png" }` オブジェクトの配列。 |
| `duration_seconds` | integer | いいえ | 要求された長さ。 |
| `width` / `height` | integer | いいえ | 出力解像度。 |
| `provider` | string | いいえ | 明示的な backend 選択ヒント（backend 名）。 |
| `extra` | object | いいえ | Backend 固有のワークフローオーバーライド（seed、steps、cfg、...）。 |

成功 → `200`:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

エラー: `model` または `prompt` がないと `400` `missing_fields`。モデルを提供する healthy なビデオ対応 backend がないと `503` `video_backend_error` / `no_backend`。月次クォータが枯渇すると `429` `quota_error` / `quota_exceeded`。

### GET /v1/video/generations/{id}

タスクのステータスを返します:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "running",
  "progress": 40,
  "result": null,
  "error": null,
  "cost": 0.0,
  "created_at": 1720000000
}
```

- `status`: `queued` | `running` | `done` | `failed` | `cancelled`。`progress` は実行中に 0〜90 を進み、`done` で 100 に達します。
- `result`（`done` 時）: `{ "url", "duration_seconds"?, "width"?, "height"?, "format"? }` — `url` は backend が提供する生成アーティファクトを指します。
- `error`（`failed` / `cancelled` 時）と `cost` は該当する場合に設定されます。
- エラー: 非 UUID の id は `400` `bad_id`。ジョブが存在しないか別の API key に属する場合は `404` `no_job`。

ビデオジョブは RPC SSE サイドカーにも進捗をファンアウトします（`video.progress` / `video.done` / `video.failed`。[イベントと通知](./events.md#video-job-notifications)を参照）。

## エラー

Gateway レベルのエラーは 1 つのシェイプを使用します（`json_error_response`）:

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| ステータス | `type` / `code` | いつ |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`、`missing_index`、`bad_id`、... | 不正または欠落したリクエストフィールド。 |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id` が別のユーザーに属する。 |
| `404` | `invalid_request_error` / `model_not_found` | 要求されたモデルを提供する backend がない。メッセージ: `No backend available for model: <model>`。 |
| `404` | `invalid_request` / `conversation_not_found` | 会話が見つからない。 |
| `404` | `not_found` / `no_job` | ビデオジョブが見つからない。 |
| `502` | `server_error` / `bad_gateway` | Upstream の非 2xx: メッセージ `upstream <status>: <detail>`（detail は upstream のエラーボディから、4 KB に制限）。トランスポート障害（connect/read/timeout）もエラー文字列付きで 502 にマッピングされます。 |
| `500` | `server_error` / `backend_error` | その他の backend 障害（例: backend が操作をサポートしていない）。 |
| `500` | `server_error` / `internal_error` | 残りの gateway 内部エラー。 |
| `429` | 下記参照 | `Retry-After` 付きのクォータ / レート制限拒否。 |

## 429 と Retry-After

429 レスポンスには `Retry-After` ヘッダー（秒）が含まれ、OpenAI 互換クライアントがバックオフできるようにします:

| トリガー | ステータスボディ | `Retry-After` |
| --- | --- | --- |
| 月次クォータ超過 | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | 翌月までの秒数。 |
| Tier の毎分レート制限 | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`。 |
| インメモリのキーごとのリミッター（デフォルト 60 RPM） | プレーンテキスト `Rate limit exceeded. Try again later.` | なし（ミドルウェア拒否）。 |

Tier、クォータスコープ、usage の会計は[Billing & Usage](../guides/billing-usage.md)で説明されています。

## Usage 記録

すべての `/v1` リクエストは、完了時に API key プレフィックス（`arona-XX`）配下に usage 行を記録します（非ストリーミングチャット、終端チャンクでのストリーミングチャット、embeddings、計算されたコスト付きの完了時ビデオジョブ）。記録モデルとクォータの強制方法は[Billing & Usage](../guides/billing-usage.md)を参照してください。

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
