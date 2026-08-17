---
title: "イベントと通知"
description: "Server-sent event（SSE）サイドカー — chat.stream、models.progress、realtime.event、ビデオ通知。"
---

# イベントと通知

ストリーミングtoken、デプロイ進捗、realtime イベントは JSON-RPC WebSocket ソケットでは**配信されません**。各ストリーミング RPC は**セッション id** を作成し、server-sent events として SSE エンドポイントに通知をプッシュします:

```
GET /api/rpc/events?session=<session_id>
```

## 購読してから送信するレシピ

RPC 呼び出しがセッション id を返してから SSE 購読が確立されるまでの間に発行された通知は**ドロップ**されます（事前購読ウィンドウ）。信頼できるパターンは:

1. 最初に SSE ストリームを開きます（セッション id がアタッチされるまでブロックします）。
2. セッション id を返す RPC を発火します（例: `chat.send`、`agents.deploy`、`realtime.start`、`video.create`）。
3. 到着する通知を SSE ストリームから読み取ります。

すべての通知は `"jsonrpc": "2.0"`、`method`、`params` オブジェクトを持つ JSON-RPC 2.0 スタイルのメッセージです。

## 通知カタログ

### `chat.stream`

tokenごとに 1 つの通知で、`chat.send`（およびセッションチャネルを使用する任意のストリーミングチャットパス）が生成します:

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — 1 つのコンテンツデルタ。
- `is_complete` — 最終チャンクまで `false`（upstream が finish reason をアタッチする場合、最終コンテンツチャンクが非空tokenで `is_complete: true` をすでに運ぶことがあります）。**終端**通知は常に後に続き、空の `token` と `is_complete: true` を持ちます。
- ストリームエラーは、`token: "Stream error: ..."` と `is_complete: true` を持つ終端通知として配信されます（`packages/core/src/gateway/rpc.rs` を参照）。

### `models.progress`

`agents.deploy` のモデルダウンロード進捗で、agent から転送されます。`stream_id` は `agents.deploy` レスポンスから取得します。

### `realtime.event`

開いている全二重 realtime セッションのサーバーイベントで、セッションチャネルにプッシュされます（`packages/core/src/gateway/realtime.rs`）。`realtime.event` RPC 経由で送信されたクライアントイベントは upstream に転送されます。サーバーイベントはここに届きます。

### ビデオジョブ通知

`video.create` ジョブはセッションチャネルに進捗をプッシュします（`packages/core/src/gateway/video.rs`）:

| メソッド | ペイロード（params） | 意味 |
| --- | --- | --- |
| `video.progress` | `job_id`、`stream_id`、`status: "running"`、`progress`（0〜90） | ジョブが実行中。 |
| `video.done` | `job_id`、`stream_id`、`result`、`cost` | ジョブが完了。`result` はアーティファクト URL を運びます。 |
| `video.failed` | `job_id`、`stream_id`、`error` | ジョブが失敗またはキャンセルされた。 |

## 順序の注意点

- SSE ストリームはセッションごとに順序付けられ、tokenは生成順に到着します。
- 単一のセッション id は 1 つの SSE サブスクライバーだけが消費できます。id を返す RPC の前（または直後）にストリームを開いてください。
- `POST /api/rpc` の `x-session-id` ヘッダーは、RPC **レスポンス**自体もセッションチャネルにアタッチします — 同じストリーム上でレスポンスをエコーさせたいクライアントが使用します。

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
