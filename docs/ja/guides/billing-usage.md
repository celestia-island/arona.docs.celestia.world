---
title: "Billing & Usage"
description: "Usage レコード、モデルごとのコスト、billing tier、クォータとレート制限の強制、プロジェクトスコープのキー、ビデオ価格、usage.list RPC。"
---

# Billing & Usage

Arona はすべてのモデルリクエストを計測し、gateway で階層化されたクォータとレート制限を強制します。モデルごとの価格は共有の plana 価格テーブルから取得され（arona で再実装されることは決してありません）、usage は `usage_records` 行として永続化され、月次の全体像は `usage.list` RPC で公開されます。

## Usage レコード

計測対象のすべてのリクエストは `usage_records` テーブルの 1 行になります（`m20250101_000006_create_usage_records`）:

| カラム | タイプ | 意味 |
| --- | --- | --- |
| `id` | `UUID` | 生成される主キー。 |
| `api_key_id` | `VARCHAR(64)` | **キープレフィックス** — API key の最初の 8 文字（キーは `arona-{uuid}` の形）— または JWT 帰属の RPC チャネル用の合成 `jwt-<user-uuid>` id。 |
| `model` | `VARCHAR(128)` | リクエストがルーティングされたモデル id。 |
| `backend` | `VARCHAR(64)` | Backend の kind: `gateway`、`rpc`、`realtime`、または backend のケーパビリティ名。 |
| `prompt_tokens` | `INTEGER` | 入力token。upstream 報告または推定。 |
| `completion_tokens` | `INTEGER` | 出力token。upstream 報告または推定。 |
| `total_tokens` | `INTEGER` | 両者の合計。 |
| `cost` | `DOUBLE PRECISION` | 計算された USD コスト。モデルに価格行がない場合は `NULL`。 |
| `created_at` | `TIMESTAMPTZ` | リクエストが完了した時刻。 |

`api_key_id`、`model`、`created_at`（月次集計とレート制限ウィンドウがスキャンするカラム）にインデックスがあります。

## 記録チャネル

usage はすべての計測対象チャネルで記録されます:

1. **REST 非ストリーミング** — `POST /v1/chat/completions` と `POST /v1/embeddings` は、レスポンスが生成された後に upstream 報告の正確な usage を記録します。
2. **REST ストリーミング（SSE）** — ストリームが usage を運んだ場合（OpenAI 互換の終端チャンクの `usage` フィールド）は upstream 報告の usage が優先されます。それ以外はローカルの CJK 対応トークナイザー推定（`estimate_usage`）がそのまま記録されます。テキストも usage も生成しなかったストリームは**まったく**記録されません。
3. **RPC `chat.send`** — 同じ推定 vs upstream のロジックが適用され、行は合成の `jwt-<user-uuid>` id で帰属されるためユーザーに結合できます。
4. **Realtime セッション** — 完了した各 `response_done` トランスクリプトは（非ゼロの場合）token usage を `jwt-<user-uuid>` と backend `realtime` で記録します。
5. **ビデオジョブ** — 完了したジョブは明示的なドルコストを記録します（[ビデオ価格](#video-pricing)を参照）。token数はゼロです。

記録はベストエフォートです: 失敗した挿入はログに記録され、リクエストを失敗させることはありません。

## コスト計算

コストは正規の 100 万tokenあたりの価格テーブル（`plana_llm_provider::metering::lookup_pricing`。celestia-island の全サービスで共有）から計算されます:

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

テーブル内のモデルマッチングは、小文字化されたモデル id に対する部分文字列ベースです（より具体的なファミリーが優先されます）。モデルに価格行がない場合、`cost` は `NULL` です。**arona で価格を再実装しないでください — plana のテーブルを更新してください。**

## Tier

Tier は `billing_tiers` テーブルにあり、最初のマイグレーションでシードされます（`m20250101_000007_create_billing_tiers`）。`NULL` のクォータカラムは、その次元で「無制限」を意味します。`tier_id` のないユーザーはシード済みの `free` tier にフォールバックします。

| Tier | 月次 USD クォータ | 月次tokenクォータ | キーごとの RPM |
| --- | --- | --- | --- |
| `free` | $1.00 | 100,000 | 10 |
| `pro` | $20.00 | 5,000,000 | 120 |
| `enterprise` | 無制限（`NULL`） | 無制限（`NULL`） | 1000 |

Tier の割り当ては管理者操作です（`billing.plan.set` RPC）。現在の tier と usage は `billing.plan` で公開されます。

## クォータとレート制限の強制

### REST（`/v1/*`）

**計測対象**の REST エンドポイント — `/v1/chat/completions`、`/v1/embeddings`、`/v1/video/generations` — のそれぞれの前に、gateway はキー認証リクエストに対して 2 つのゲートを強制します:

- **月次クォータ** — tier の `monthly_quota_usd` と `quota_tokens` の制限が、現在の暦月の最初の瞬間以降に蓄積された usage に対してチェックされます。どちらかの次元が制限に達するとゲートが作動します。
- **キーごとのレート制限** — tier の `rate_limit_rpm` が、直近 60 秒のウィンドウでこのキープレフィックスに記録されたリクエスト数に対してチェックされます。（`/v1/models` とヘルスエンドポイントは計測対象でもゲート対象でもありません。）

拒否は **HTTP 429 Too Many Requests** で、`Retry-After` ヘッダーと OpenAI スタイルのエラーボディが付きます:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| 拒否 | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| 月次クォータ枯渇 | `quota_error` / `quota_exceeded` | **次の暦月**の開始までの秒数 |
| Tier レート制限超過 | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

JWT 認証の `chat.send` は同じ月次クォータゲートを通過しますが、**ユーザー全体**のウィンドウに対してです（呼び出しは API key を持ちません）。拒否は実装定義コード `-32006`（`QUOTA_ERROR`）の JSON-RPC エラーで、REST クォータ拒否と同じメッセージです。RPC パスにはキーごとのレート制限はありません — レート制限はキースコープであり、RPC 呼び出しにはキーがないためです。Realtime とビデオの **RPC** メソッドはクォータゲートの対象ではありません。

## Fail-open のトレードオフ

Billing は**設計上ベストエフォート**です。クォータまたはレート制限チェックの背後にあるデータベースクエリが失敗した場合、チェックは `Unknown` を返し、リクエストは（ログのみ記録されて）**許可**され、チャットをブロックしません。オペレーターは容量を保護するために 429 を当てにできますが、データベースが不健康なときはそれをハードな保証として扱ってはなりません — 文書化されたトレードオフは、厳密な計測よりもチャットパスの可用性です。

## プロジェクトスコープのキー

API key は `project` ラベル（`api_keys.project`。未設定時は `default`）付きで作成できます。クォータの強制はこれを尊重します:

- `default` 以外のプロジェクトがタグ付けされたキーは、**そのプロジェクト自身のバケット**に帰属する usage に対してクォータをチェックします（`PROJECT_MONTHLY_USAGE_SQL`）。
- `default` / タグなしのキーは**ユーザー全体**のウィンドウを維持し、プロジェクト前の挙動と一致します。

JWT 帰属の RPC 行（`jwt-<user-uuid>`）はプロジェクトラベルを持たず、プロジェクトウィンドウから**意図的に除外されます** — それでもユーザー全体のウィンドウにはカウントされるため、RPC チャネル経由でトラフィックを送ることでプロジェクトを「隠す」ことはできません。

## ビデオ価格

ビデオ生成はモデル固有のタスク型価格を使用します（ビデオにtoken単位の価格は意味をなしません）。価格行は `video_pricing` テーブルにあります。`compute_cost` は行が設定されていない場合、プレースホルダーのデフォルトにフォールバックします。

| モード | コスト |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution`（デフォルト） | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` は短辺のピクセル値（例: `"768"`）をキーとする JSON オブジェクトで、`"*"` キーがフォールバックです。デフォルトの価格行はモード `per_second_resolution`、`base_price` 0.0、`price_per_second` 0.005、`resolution_coeff {"*": 1.0}` です。行は `billing.video.pricing.get`（任意の JWT）と `billing.video.pricing.set`（Bearer `ARONA_ADMIN_TOKEN`）で管理されます。計算されたコストはジョブ完了時にジョブのキーに対して記録されます。

## usage.list

`usage.list`（JWT）は、呼び出し元のページングされた usage レコードを返します。API key 行（キープレフィックス経由で結合）と JWT 帰属行（合成の `jwt-<user-uuid>` id 経由で結合）の**両方**をカバーし、新しい順です:

| パラメータ | デフォルト | メモ |
| --- | --- | --- |
| `limit` | `50` | `1..=200` にクランプ。 |
| `offset` | `0` | ページオフセット。 |
| `project` | 未設定 | 設定すると、そのプロジェクトラベルのキーに帰属するレコードのみ（JWT 行は除外）。 |

レスポンスは `{ "records": [...], "total", "limit", "offset", "project" }` で、各レコードは `id`、`model`、`backend`、`prompt_tokens`、`completion_tokens`、`total_tokens`、`cost`、`created_at` を持ちます。月次クォータ集計は同じ結合シェイプを使用するため、クォータ計算と usage ビューは常にスコープが一致します。

## 関連

- [クイックスタート](quickstart.md) — キーを取得して最初の計測対象リクエストを行う。
- [設定](configuration.md) — gateway の環境変数。
- [認証とセキュリティ](auth-security.md) — API key の作成と `project` ラベル。
- [Realtime & Video](realtime-video.md) — ビデオ価格の背後にあるビデオジョブのライフサイクル。
- [運用](operations.md) — ヘルスプローブと可観測性。
- [OpenAI 互換 REST API](../api/openai-rest.md) — `/v1/*` サーフェス。
- [JSON-RPC API](../api/jsonrpc.md) — `usage.list`、`billing.plan`、`billing.video.pricing.*`。
- [概要](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
