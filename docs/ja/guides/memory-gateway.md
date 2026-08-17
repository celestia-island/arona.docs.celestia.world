---
title: "Memory Gateway"
description: "チャット用の長期メモリ — リコール注入、エピソード書き戻し、リクエストごとの制御、ヘッダー状態、memory.status / memory.delete RPC。"
---

# Memory Gateway

Memory Gateway は、チャットターンに entelecheia scepter / Philia メモリサービスに保存された**長期メモリ**へのアクセスを提供します。各チャットターンで Arona は会話に関連するメモリをサービスに照会し、システムセクションとしてプロンプトに注入し、応答完了後に — ターンをエピソードとして書き戻し、将来の会話がそれをリコールできるようにします。

これは Philia への WebSocket JSON-RPC クライアントです（`Sync.ConnectHandshake`、`Sync.MemoryQueryRequest`、`Sync.MemoryStoreRequest`、`Sync.MemoryDeleteRequest`）。接続は遅延確立され、エラー時には切断され、次の呼び出しで再確立されます。すべての失敗は静かに劣化し、**チャットパスをブロックすることは決してありません**。

## 設定

gateway は 3 つの環境変数で制御されます:

| 変数 | 意味 |
| --- | --- |
| `ARONA_MEMORY_URL` | scepter / Philia サービスの WebSocket URL（例: `ws://192.0.2.10:8424/ws`）。 |
| `ARONA_MEMORY_TOKEN` | メモリサービス用のtoken。 |
| `ARONA_MEMORY_WRITEBACK` | 完了したターンを書き戻すかどうか。デフォルトは**オン**。無効にするには `false` を設定します（厳密な boolean としてパースされます — `0` は受け付けられません）。 |

`ARONA_MEMORY_URL` **と** `ARONA_MEMORY_TOKEN` の両方が設定され、非空である必要があります。そうでない場合、gateway は**無効**になります: リコールと書き戻しは完全にスキップされ、すべてのリクエストが `disabled` を報告します。tokenは WebSocket アップグレードの `?token=` クエリパラメータと、`Sync.ConnectHandshake` リクエスト内の両方で送信されます。

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

完全な環境リファレンスは[設定](configuration.md)を参照してください。

## リコール注入

gateway が有効な場合、**すべてのチャットターン** — REST 非ストリーミングの `/v1/chat/completions`、REST ストリーミング（SSE）、RPC の `chat.send` — は、リクエストが転送される前にメモリサービスに照会します:

- クエリは組み立てられたコンテキストの**最後のユーザーメッセージ**です。
- 最大 **5** 件のメモリが要求されます（`limit = 5`）。
- 結果は `## Relevant Long-Term Memories` というタイトルの markdown システムセクションとしてレンダリングされ、メモリごとに 1 つの `- [score] text` 箇条書き（スコアは小数 2 桁、空のエントリはスキップ）になり、`system` メッセージとしてメッセージリストの先頭に付加されます。注入は冪等です: すでにセクションを持つコンテキストは再注入されません。
- 関連メモリが返されない場合、何も注入されず、ターンはそのまま進行します。

リコールは会話の永続化と upstream 転送の前に実行されます。遅いまたは失敗するメモリサービスは、自身の 10 秒の RPC タイムアウトを超える**レイテンシーの保証を追加せず**、リクエストを失敗させることもできません。

## 書き戻し

アシスタント応答の完了後、ターンは**エピソード**ノードとしてメモリサービスに書き戻されます。エピソードテキストはターンのヒューリスティックなトランスクリプトです — `User: <user content>\nAssistant: <assistant content>`（どちらかが空なら省略。両方が空なら書き戻しをスキップ）。書き戻しは **fire-and-forget** です: スポーンされたタスクで実行され、チャットレスポンスをブロックせず、失敗はメモリクライアント内でログされるだけです。（REST ストリーミングパスでは、書き戻しはさらにリクエストに会話がアタッチされている必要があります。非ストリーミング REST と RPC のパスは関係なく書き戻します。）

## リクエストごとの制御

REST チャットリクエストボディと RPC の `chat.send` パラメータの両方が、任意の `memory` フィールドを受け付け、サーバー設定を**呼び出しごとに**上書きできます:

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` — このターンのリコールを強制的にオン / オフにします。
- 省略（`null`）— サーバー設定に従います（`req.memory.unwrap_or(true)`）。つまり gateway が設定されていれば有効です。

上書きはリコールに影響します。書き戻しは `ARONA_MEMORY_WRITEBACK` と gateway の有効化にのみ従います。

## ヘッダー状態

REST レスポンスはターンのメモリ状態を **`X-Arona-Memory`** レスポンスヘッダーで運びます。RPC の `chat.send` レスポンスは同じ値を結果の `memory` フィールドでエコーします。可能な状態:

| 値 | 意味 |
| --- | --- |
| `enabled` | メモリが要求され、gateway が設定され、リコールが成功し、少なくとも 1 件のメモリが注入されました。 |
| `disabled` | gateway が未設定、またはリクエストの `memory: false`、または照会するユーザーメッセージがない、またはリコールは成功したが関連メモリが**返されなかった**（注入するものがない）。 |
| `offline` | メモリが要求され、gateway は設定されているが、リコール呼び出しが失敗しました（接続拒否、RPC エラー、またはタイムアウト）。 |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## 失敗のセマンティクス

すべてが同じ方向に明示的に劣化します — チャットは常に実行されます:

- **リコール失敗** — `warn` レベルでログ。リクエストは注入なしで進行し、ヘッダーに `offline` を報告します。
- **書き戻し失敗** — メモリクライアント内でログ。チャットレスポンスは影響を受けません。
- **メモリサービス未設定** — リコールと書き戻しは no-op。すべてのリクエストが `disabled` を報告します。

メモリの障害がチャットリクエストを失敗させたり遅延させたりするモードは、クライアント自身の制限付きタイムアウトを超えて存在しません。

## RPC サーフェス

JSON-RPC サーフェスには 2 つの管理メソッドが公開されています（どちらも JWT を要求します。[JSON-RPC API](../api/jsonrpc.md) を参照）:

**`memory.status`** — gateway のスナップショット:

```json
{
  "enabled": true,
  "writeback": true,
  "events": [
    { "kind": "recall", "detail": "…query…", "at": "2026-01-01T00:00:00.000Z" },
    { "kind": "writeback", "detail": "…node id…", "at": "…" },
    { "kind": "error", "detail": "…", "at": "…" }
  ]
}
```

`events` は最近のアクティビティのインメモリリングバッファです — リコール、書き戻し、削除、エラーイベントが新しい順で、要求された数まで（ステータスハンドラーは最後の 50 件を要求します。バッファ自体は 100 件で上限）。これは**耐久性のある監査ログではありません** — 再起動でリセットされます。

**`memory.delete`** — 保存されたノードを id で削除します:

```json
{ "node_id": "…" }
```

`{ "deleted": true | false }` を返します。`node_id` がない場合、またはメモリサービスが設定されていない場合はエラーで失敗します。

## 関連

- [設定](configuration.md) — `ARONA_MEMORY_*` 変数。
- [クイックスタート](quickstart.md) — gateway のエンドツーエンドセットアップ。
- [Backends](backends.md) — リコールの実行前にチャットリクエストがどうルーティングされるか。
- [Billing & Usage](billing-usage.md) — 同じチャットターンがどう計測されるか。
- [運用](operations.md) — メモリ接続のログとヘルス。
- [JSON-RPC API](../api/jsonrpc.md) — `memory.status`、`memory.delete`、`chat.send`。
- [概要](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
