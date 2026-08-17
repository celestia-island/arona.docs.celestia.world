---
title: "Realtime & Video"
description: "全二重の realtime セッション（realtime.start/event/stop）、engine.invoke 認識・制御チャネル、非同期ビデオ生成ジョブ。"
---

# Realtime & Video

Arona はプレーンなテキストチャットに加えて 2 つのマルチモーダルチャネルを公開します: **全二重の realtime セッション**（1 つの双方向チャネルでの音声・ビデオ入出力）と**非同期ビデオ生成**（バックグラウンドで実行され進捗を報告するタスク型ジョブ）です。どちらも要求されたモデルを提供する backend にルーティングされ、どちらも billing レイヤーで計測されます。

## Realtime セッション

realtime セッションは**1 つのクライアント**と**1 つの upstream** の間の双方向チャネルです: クラウド realtime API（OpenAI-Realtime / Qwen-Omni-Realtime WebSocket ボキャブラリー）またはローカル CEP エンジン。クライアントイベントは JSON-RPC 経由で到着し、upstream に転送されます。サーバーイベントはセッション SSE チャネル上の `realtime.event` 通知としてプッシュバックされます。音声は base64 PCM16（16 kHz クライアント→モデル、24 kHz モデル→クライアント）として送信され、クラウドベンダーのワイヤーフォーマットに一致するため、gateway はバイトをそのまま通します（`packages/core/src/backends/realtime.rs:1-19`）。

### `realtime.start`

`model` を提供する backend に対するセッションを開きます（JWT。パラメータは `model`、`config?`、`conversation_id?` — `packages/core/src/gateway/rpc.rs:1890-1898, 1914-1984`）。backend は `realtime` ケーパビリティ（音声・ビデオモダリティ）を宣言する**必要があります**。そうでない場合、呼び出しは `model {model} does not support realtime sessions (no audio/video modality)` で明示的に失敗します — テキストチャットへのサイレントフォールバックはありません（`packages/core/src/gateway/realtime.rs:138-142`）。

2 種類の upstream がサポートされています（`packages/core/src/gateway/realtime.rs:143-167`）:

- **CEP エンジン upstream** — Celestia Engine Protocol の `Engine.InvokeStart` ストリーミングチャネル経由でイベントをルーティングします。ローカルにデプロイされた omni エンジンは、新しいワイヤーフォーマットなしで同じクライアントサーフェスに参加します。
- **クラウド upstream** — クラウド realtime イベントボキャブラリーを話す固定の `wss://` URL（`session.update`、`input_audio_buffer.*`、`response.audio.delta`、...）。クラウド実装は再接続時に `session.update` を再発行します。

レスポンスは `{ "session_id": ..., "stream_session": ... }` です — 呼び出しの前（または直後）に `/api/rpc/events?session=<stream_session>` を購読してサーバーイベントを受け取ってください。任意の `conversation_id` は音声トランスクリプトをアシスタントメッセージとして永続化し、billing 用のtoken usage を記録します（`packages/core/src/gateway/realtime.rs:32-85`）。

### `realtime.event`

セッションに 1 つのクライアントイベントを送信します（JWT。パラメータは `session_id`、`event` — `packages/core/src/gateway/rpc.rs:1989-2013`）。サポートされるイベントには `session.update`、`input_audio_buffer.append` / `.commit` / `.clear`、`input_image_buffer.append`、`response.create`、`response.cancel`、`session.stop` があります。`send_event` は**非ブロッキング**です: イベントは mpsc チャネルにキューされ、フォワーダータスクが upstream に書き込みます（`packages/core/src/gateway/realtime.rs:254-280`）。イベントを送信できるのはセッション所有者だけです。

### `realtime.stop`

セッションを閉じて削除します（JWT。パラメータは `session_id` — `packages/core/src/gateway/rpc.rs:2016-2034`）。各セッションは正確に 1 つの**フォワーダータスク**を所有し、それが upstream を保持して双方向を多重化します: クライアントイベントはキューから消費され、upstream イベントは同じループでポーリングされます。フォワーダーは upstream が閉じるかセッションが停止されると終了し、レジストリエントリを削除します（`packages/core/src/gateway/realtime.rs:195-250`）。

サーバーイベントは、パラメータ `{ session_id, event }` を持つ `realtime.event` 通知としてセッションチャネルにプッシュされます — [イベントと通知](../api/events.md)を参照してください。

## `engine.invoke`

`engine.invoke` は汎用の**同期**エンジンメソッドチャネルです（ADMIN: JWT + `is_admin`。パラメータは `model`、`method`、`params?` — `packages/core/src/gateway/rpc.rs:261-264,2049-2079`）。`model` を提供する backend 上の任意のメソッドを呼び出し、結果を直接返します。高頻度の認識・制御チャネルになります: 20〜30 Hz ループでの `sensor.ingest`、`control.setpoint` スタイルの呼び出しです。汎用呼び出しチャネルを持たない backend（すべての OpenAI 互換 HTTP backend）は `backend does not support generic invocation` で明示的に拒否します（`packages/core/src/backends/mod.rs:573-586`）。

## ビデオ生成（REST）

ビデオジョブは REST サーフェス上の OpenAI スタイルの非同期タスクです（API key 認証 — `packages/core/src/gateway/server.rs:876-993`。[OpenAI 互換 REST API](../api/openai-rest.md) を参照）:

**`POST /v1/video/generations`**

| フィールド | タイプ | メモ |
| --- | --- | --- |
| `model` | string | 必須 — ビデオ対応 backend を選択します。 |
| `prompt` | string | 必須。 |
| `negative_prompt` | string? | |
| `images` | array? | Base64 データ URL（`data:image/png;base64,...`）。`{ data_base64, mime_type }` として運ばれます。 |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | Backend 選択ヒント（backend 名と照合）。 |
| `extra` | object? | Backend 固有のオーバーライド（seed、steps、cfg、...）。 |

レスポンス:

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`** はジョブをポーリングし、`id`、`object`、`model`、`status`、`progress`、`result`、`error`、`cost`、`created_at` を返します。ジョブは呼び出し元にスコープされます: 他のユーザーが所有するジョブは 404 を返します。REST サーフェスはチャットパスと同じ billing ゲート（月次クォータ、毎分のレート制限）を強制します。

## ビデオ生成（RPC）

同じ機能が JSON-RPC でも利用できます（JWT — `packages/core/src/gateway/rpc.rs:371-386,1296-1431`）:

| メソッド | パラメータ | 戻り値 |
| --- | --- | --- |
| `video.create` | REST 呼び出しと同じフィールド | `{ job_id, stream_id }`。 |
| `video.get` | `job_id` | ジョブビュー（status、progress、result、cost、...）。 |
| `video.list` | `limit?`（デフォルト 20、1〜100 にクランプ） | `{ jobs: [...] }`、新しい順。 |
| `video.cancel` | `job_id` | `{ "ok": true }` — キャンセルできるのは所有者だけです。 |

`video.create` は `stream_id` を返します。ジョブ通知（`video.progress` / `video.done` / `video.failed` — [イベントと通知](../api/events.md)を参照）を受け取るには `/api/rpc/events?session=<stream_id>` を購読してください。

## Backend

ビデオ生成は**クラウドのみ**です: MiniMax H3 Open Platform API、backend kind は `minimax-cloud`（`BackendKind::CloudVideo` — `packages/core/src/backends/mod.rs:502-504,720-727,759-761`）。フローはタスク型です:

1. `POST /v1/video_generation_v2` → `task_id`
2. `Success` / `Fail` / まだ `Pending` になるまで `GET /v1/query/video_generation_v2?task_id=...` をポーリング
3. 成功時、`GET /v1/files/{file_id}/content` → `{ file: { download_url } }` でアーティファクトを解決

（`packages/core/src/backends/minimax_cloud.rs:66-210`）。MiniMax backend はチャット / embeddings を提供しません。そのケーパビリティは `supports_video_generation` と `realtime: false` を宣言します（ケーパビリティモデルは[Backends](./backends.md)を参照）。ルーティングはビデオリクエストを `supports_video_generation` を持つ backend に対してのみ解決し、任意の `provider` ヒントを尊重します（`packages/core/src/routing/mod.rs:205-263`）。

**ComfyUI backend は削除されました**（モデルプラットフォームの統合時）: backend kind `"comfyui"` の設定は `comfyui backend removed` で失敗します（`packages/core/src/backends/mod.rs:756-757`）。セルフホストのビデオパスはありません。ビデオは常に `minimax-cloud` backend を経由します。

## ジョブのライフサイクルと価格

ビデオジョブは `queued → running → done | failed | cancelled` を移動します（`packages/core/src/gateway/video.rs`）:

- **create** — ジョブ行が永続化され（`queued`、進捗 0）、ポーラータスクがスポーンされます（`video.rs:109-176`）。
- **running** — ポーラーがタスクを送信し（進捗 5）、その後 1.5 秒ごとにポーリングして、数回の反復ごとに進捗を 5 ずつ **90** まで上げます（`video.rs:178-275`）。ポーリングエラーはログされ、再試行されます。
- **done** — 進捗 100、結果 URL と計算されたコストが永続化され、usage が記録され、`video.done` 通知がファンアウトされます（`video.rs:332-368`）。
- **failed** — 送信またはポーリングの失敗 → `video.failed`。900 回のポーリング反復（約 22.5 分）の後、ジョブは `generation timed out` で失敗します。
- **cancelled** — `video.cancel` がフラグを設定し、ポーラーが次のパスでそれを観察します。ジョブは `cancelled` とマークされ、エラー `cancelled` で `video.failed` が発火します（`video.rs:389-400`）。

usage はビデオ固有のコストで記録されます: `record_video` はゼロtokenと明示的なドルコストを持つリクエストごとの usage レコードを書き込みます（`packages/core/src/billing/mod.rs:496-531`）。

**価格**はモデル固有で、`video_pricing` テーブルにあります（`packages/core/src/billing/video.rs`）:

| モード | 計算式 |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution`（デフォルト） | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` は短辺のピクセルキー（例: `"768"`）を乗数にマッピングし、`"*"` がフォールバックです。設定された行のないモデルはフォールバックします: モード `per_second_resolution`、`base_price` 0.0、`price_per_second` 0.005、`price_per_frame` 0.0、`resolution_coeff {"*": 1.0}`、通貨 USD（`billing/video.rs:20-32`）。行は `billing.video.pricing.get`（JWT）で照会し、`billing.video.pricing.set`（admin token）で upsert します — [JSON-RPC API](../api/jsonrpc.md) を参照してください。usage レコードが月次 billing にどう集約されるかは[Billing & Usage](./billing-usage.md)を参照してください。

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
