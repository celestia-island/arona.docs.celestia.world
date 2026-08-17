---
title: "Admin HTTP API"
description: "Bearer token の管理サーフェス — /api/admin/* で backend の登録・一覧・削除とモデルエイリアスの管理。"
---

# Admin HTTP API

`/api/admin/*` サーフェスは gateway の **backend**（upstream モデルprovider）と **エイリアス**（モデル名 → モデル id のリダイレクト）を管理します。これは RPC 管理プレーンの HTTP 側の対応物で（[JSON-RPC API](./jsonrpc.md)を参照）、主にオペレーターと管理 UI が使用します。

## 認証

すべての `/api/admin/*` ルートは次を要求します:

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN` はプロセス起動時に環境から読み込まれます（`GatewayServer::new`）。変数が**未設定**の場合、または提示されたtokenが一致しない場合、リクエストは `401` で拒否されます:

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

bearer プレフィックスは大文字小文字を区別せずマッチします（`Bearer` または `bearer`）。

> `/v1/*` サーフェスと異なり、admin 認証は API key や JWT にフォールバックせず、正確なtoken比較で強制されます — 新しい値でプロセスを再起動してtokenをローテーションしてください。

## Backend

Backend は gateway の背後にあるルーティング可能な upstream です。登録により backend は即座にルーティング可能になり、再起動復元のために設定が永続化され、プローブされ（約 1〜2 秒で healthy に切り替わる）、ブリッジ URL の場合はトンネルが生きたまま維持されます。Backend タイプと URL セマンティクスの詳細は[Backends](../guides/backends.md)にあります。

### POST /api/admin/backends — backend を登録する

リクエストボディ（注記がある場合を除き、すべてのフィールドは任意）:

| フィールド | タイプ | メモ |
| --- | --- | --- |
| `type` | string | Backend の kind。`external`（任意の OpenAI 互換 HTTP API）、`ollama`（ローカルまたはリモートの ollama サーバー）、`engine`（`ws://`/`wss://` 上の CEP エンジン）、`minimax-cloud`（クラウドビデオ API）のいずれか。MDD エンジン名（`llama_cpp`、`vllm`、`ollama`、`cloud`、`external_api`、`candle`、`native`、...）はプランナーを通して解決されます。`comfyui` は**拒否**されます（`comfyui backend removed`）。それ以外は `400` `unknown_type`。欠落時は `ollama` がデフォルト。 |
| `url` | string | Backend のベース URL。`evernight://<node>/<service>` ブリッジ URL はローカル evernight agent 経由でローカル TCP フォワードに解決されます（解決失敗 → `502` `evernight_unreachable`）。欠落時は `http://localhost:11434` がデフォルト。 |
| `api_key` | string | 任意の upstream API key。upstream 呼び出しで `Authorization: Bearer` として送信されます。 |
| `name` | string | Backend 名。欠落時は `type` 値がデフォルト。ルーティングの `provider` ヒントと設定行のアイデンティティとして使用されます。 |
| `models` | string[] | 静的モデルリスト。プロービングで何も発見されない場合のルーティングソース。`external` backend では、発見されたモデルは静的リストの後にマージされます（静的な id が優先順位を保持）。`engine` backend は発見されたモデルキャッシュを先に返し、静的な id を後から追加します。`minimax-cloud` はモデルディスカバリーを行いません（そのプローブは `/v1/query/available_models` へのヘルスピングのみ）し、静的リストだけを提供します。`ollama` では無視され、`/api/tags` からモデルを発見します。 |
| `workflow` | object | 任意。レガシー — 歴史的に削除された ComfyUI backend が消費していました。現在の backend は読み取りません（`backend_configs` カラム互換のために保持）。 |

例:

```bash
curl -X POST http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "name": "mock-upstream",
    "url": "http://192.0.2.20:11434",
    "api_key": "sk-xxx",
    "models": ["Qwen/Qwen3-1.7B"]
  }'
```

成功 → `200`:

```json
{ "status": "ok", "message": "backend registered" }
```

登録の副作用:

- backend は**即座に登録されルーティング可能になります**（再起動不要）。
- 設定は `backend_configs` テーブルに**永続化**され、起動時に復元されます（DB 失敗はログされるだけで、レスポンスをブロックしません）。
- 次の 60 秒のヘルスチェッカーラウンドまで fail-closed のままにする代わりに、fire-and-forget の**プローブ**がすぐに実行され、backend は約 1〜2 秒で healthy に切り替わります。
- `evernight://` URL の場合、**キープアライブタスク**がトンネルを監視します: 新しいローカルポートでの再接続時、同じ名前の下で backend を透過的に再構築して再登録します。

### GET /api/admin/backends — backend を一覧する

```json
{
  "backends": {
    "count": 2,
    "health": [
      ["backend_0:ExternalApi", "Healthy"],
      ["backend_1:Ollama", { "Unhealthy": "connection refused" }]
    ]
  },
  "models": [
    { "id": "Qwen/Qwen3-1.7B", "object": "model", "owned_by": "mock-upstream" }
  ]
}
```

- `backends.count` — **healthy** な backend の数。
- `backends.health` — backend ごとの `backend_<index>:<kind>` ラベルとヘルス状態（`Healthy` / `Degraded` / `Unhealthy`）。`<index>` は `DELETE /api/admin/backends` が使用するルーター登録インデックスです。
- `models` — 今日ルーティング可能なすべてのモデル id（`GET /v1/models` と同じ一覧。クイックスタートのマージなし。[OpenAI 互換 REST](./openai-rest.md#get-v1models)を参照）。

### DELETE /api/admin/backends — backend を削除する

JSON ボディのルーター**インデックス**で識別されます — 名前ではありません:

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| フィールド | タイプ | 必須 | メモ |
| --- | --- | --- | --- |
| `index` | integer | はい | `GET /api/admin/backends` のヘルスレポートの `backend_<index>` ラベルと一致するルーター登録インデックス。 |

- `index` の欠落 → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`。
- インデックスが範囲外 → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`。
- 成功 → `200` `{ "status": "ok", "message": "backend removed" }`。
- 永続化された `backend_configs` 行はベストエフォートで削除されます: backend 名はモデル一覧の `owned_by` から復元され、不一致の場合は行がストアに残ります（DB 失敗はログされるだけで、致命的ではありません）。

## エイリアス

エイリアスは 1 つのモデル名を別のモデルにマッピングします（`alias` → `target`）。あるモデル id へのリクエストが別の backend モデルにルーティングされるようにします。エイリアスはルーティングの前に解決されるため、チャット、embeddings、ビデオのルックアップに一様に適用されます。

> エイリアスは**インメモリのルーター状態のみ**です — 永続化されず、再起動で失われます。起動後に登録するか、独自のプロビジョニング状態から再作成してください。

### POST /api/admin/aliases — エイリアスを追加する

| フィールド | タイプ | 必須 | メモ |
| --- | --- | --- | --- |
| `alias` | string | はい | クライアントがリクエストするモデル名。 |
| `target` | string | はい | リクエストがルーティングされるモデル id。 |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- `alias` の欠落 → `400` `missing_alias`。`target` の欠落 → `400` `missing_target`。
- 成功 → `200` `{ "status": "ok", "message": "alias added" }`。
- 既存のエイリアスの追加はそのターゲットを置き換えます。

### GET /api/admin/aliases — エイリアスを一覧する

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

ペアはエイリアス順にソートされて返されます。

### DELETE /api/admin/aliases — エイリアスを削除する

| フィールド | タイプ | 必須 | メモ |
| --- | --- | --- | --- |
| `alias` | string | はい | 削除するエイリアス。 |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- `alias` の欠落 → `400` `missing_alias`。
- 未知のエイリアスの削除は no-op の成功です → `200` `{ "status": "ok", "message": "alias removed" }`。

## 永続化のまとめ

| リソース | 永続化? | 再起動時の復元 |
| --- | --- | --- |
| Backend | はい — `backend_configs` テーブル（`name` キー、登録時に upsert、削除時に delete）。 | はい: 起動時に復元。external backend は fail-closed で始まり、最初のプローブラウンド後に healthy に切り替わります。`evernight://` URL は起動時にブリッジ経由で再解決されます。 |
| エイリアス | いいえ — インメモリの `Router.aliases` のみ。 | いいえ。 |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
