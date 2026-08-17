---
title: "計費與用量"
description: "Usage records、每模型成本、billing tiers、quota 與速率限制強制、專案範圍 keys、視訊定價與 usage.list RPC。"
---

# 計費與用量

Arona 為每個模型請求計量，並在 gateway 強制分層 quota 與速率限制。每模型
價格來自共用的 plana 定價表（絕不在 arona 重新實作），usage 以
`usage_records` 列持久化，整個月度的全貌透過 `usage.list` RPC 暴露。

## Usage records

每個被計量的請求最終都成為 `usage_records` 資料表中的一列
（`m20250101_000006_create_usage_records`）：

| 欄位 | 類型 | 意義 |
| --- | --- | --- |
| `id` | `UUID` | 主鍵，自動產生。 |
| `api_key_id` | `VARCHAR(64)` | **key 前綴**——API key 的前 8 個字元（keys 長得像 `arona-{uuid}`）——或 JWT 歸因的 RPC 通道使用的合成 `jwt-<user-uuid>` id。 |
| `model` | `VARCHAR(128)` | 請求被路由到的模型 id。 |
| `backend` | `VARCHAR(64)` | Backend 類型：`gateway`、`rpc`、`realtime`，或 backend 能力名稱。 |
| `prompt_tokens` | `INTEGER` | 輸入 tokens，upstream 回報或估算。 |
| `completion_tokens` | `INTEGER` | 輸出 tokens，upstream 回報或估算。 |
| `total_tokens` | `INTEGER` | 兩者之和。 |
| `cost` | `DOUBLE PRECISION` | 計算出的 USD 成本；模型沒有定價列時為 `NULL`。 |
| `created_at` | `TIMESTAMPTZ` | 請求完成的時間。 |

`api_key_id`、`model` 與 `created_at` 上建有索引（月度彙總與速率限制
視窗掃描的欄位）。

## 記錄通道

Usage 在每個被計量的通道上記錄：

1. **REST 非串流** — `POST /v1/chat/completions` 與
   `POST /v1/embeddings` 在回應產生後記錄 upstream 確切回報的 usage。
2. **REST 串流（SSE）** — 當串流攜帶 upstream 回報的 usage 時以其為準
   （OpenAI 相容的終端 chunk `usage` 欄位）；否則記錄本機 CJK 感知
   tokenizer 的估算值（`estimate_usage`）。既沒產生文字也沒產生 usage 的
   串流**完全不記錄**。
3. **RPC `chat.send`** — 套用相同的估算 vs upstream 邏輯；列以合成的
   `jwt-<user-uuid>` id 歸因，因此可 join 回使用者。
4. **Realtime session** — 每個完成的 `response_done` 轉錄（transcript）
   在非零時記錄其 token usage，以 `jwt-<user-uuid>` 歸因、backend 為
   `realtime`。
5. **視訊任務** — 完成的任務記錄明確的美元成本
   （見[視訊定價](#video-pricing)）；token 數為零。

記錄是盡力而為：失敗的 insert 會被記錄，但絕不使請求失敗。

## 成本計算

成本從每 1M token 的標準定價表計算
（`plana_llm_provider::metering::lookup_pricing`，所有 celestia-island
服務共用）：

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

表中的模型比對以小寫模型 id 做子字串比對（更具體的 family 勝出）。當模型
沒有定價列時，`cost` 為 `NULL`。**不要在 arona 重新實作定價——更新
plana 的表。**

## Tiers

Tiers 存在 `billing_tiers` 資料表，首次 migration 時 seed
（`m20250101_000007_create_billing_tiers`）。`NULL` 的 quota 欄位表示該
維度「無限制」。沒有 `tier_id` 的使用者退回 seed 的 `free` tier。

| Tier | 每月 USD quota | 每月 token quota | 每 key RPM |
| --- | --- | --- | --- |
| `free` | $1.00 | 100,000 | 10 |
| `pro` | $20.00 | 5,000,000 | 120 |
| `enterprise` | 無限制（`NULL`） | 無限制（`NULL`） | 1000 |

Tier 指派是管理操作（`billing.plan.set` RPC）；目前的 tier 與用量透過
`billing.plan` 呈現。

## Quota 與速率限制強制

### REST（`/v1/*`）

在每個**被計量**的 REST endpoint——`/v1/chat/completions`、
`/v1/embeddings` 與 `/v1/video/generations`——之前，gateway 對
key 認證的請求強制兩道閘門：

- **每月 quota** — 以目前行事曆月第一刻以來累積的 usage 檢查 tier 的
  `monthly_quota_usd` 與 `quota_tokens` 限制。任一維度達到限制即觸發閘門。
- **Per-key 速率限制** — 以最近 60 秒視窗內此 key 前綴記錄的請求數檢查
  tier 的 `rate_limit_rpm`。（`/v1/models` 與健康 endpoint 不計量、
  不受閘門限制。）

拒絕是 HTTP **429 Too Many Requests**，帶 `Retry-After` header
與 OpenAI 風格的錯誤內文：

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| 拒絕 | `type`／`code` | `Retry-After` |
| --- | --- | --- |
| 每月 quota 用盡 | `quota_error`／`quota_exceeded` | 距**下一個行事曆月**開始的秒數 |
| Tier 速率限制超限 | `rate_limit_error`／`rate_limit_exceeded` | `60` |

### RPC

JWT 認證的 `chat.send` 會通過相同的每月 quota 閘門，但對照的是
**整個使用者**的視窗（呼叫不攜帶 API key）。拒絕是 JSON-RPC 錯誤，
code 為實作定義的 `-32006`（`QUOTA_ERROR`），訊息與 REST quota 拒絕相同。
RPC 路徑沒有 per-key 速率限制——速率限制以 key 為範圍，而 RPC 呼叫沒有
key。Realtime 與視訊的 **RPC** 方法不受 quota 閘門限制。

## Fail-open 取捨

計費**刻意設計為盡力而為**。若 quota 或速率限制檢查背後的資料庫查詢失敗，
檢查回傳 `Unknown`，請求**被允許**（僅記錄）而非阻擋聊天。操作員可以依賴
429 來保護容量，但資料庫不健康時不能將其視為硬保證——此處記載的取捨是
聊天路徑的可用性優先於嚴格計量。

## 專案範圍 keys

API keys 可以帶 `project` 標籤建立（`api_keys.project`，
未設定時為 `default`）。Quota 強制會尊重它：

- 標記為 `default` 以外專案的 key，對照歸因於**該專案自己的 bucket**
  的 usage 檢查 quota（`PROJECT_MONTHLY_USAGE_SQL`）。
- `default`／未標記的 keys 維持**整個使用者**的視窗，與引入專案前的
  行為一致。

JWT 歸因的 RPC 列（`jwt-<user-uuid>`）不攜帶專案標籤，且**刻意排除**
在專案視窗之外——它們仍計入整個使用者的視窗，因此專案無法藉由將流量
改走 RPC 通道來「隱藏」。

## 視訊定價

視訊生成使用模型特定的 task 型定價（對視訊而言 per-token 定價沒有意義）。
定價列存在 `video_pricing` 資料表；沒有設定列時 `compute_cost` 退回
佔位符預設值。

| 模式 | 成本 |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution`（預設） | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` 是以短邊像素值為鍵的 JSON 物件（例如 `"768"`）；
`"*"` 鍵是後備值。預設定價列是模式 `per_second_resolution`、
`base_price` 0.0、`price_per_second` 0.005、`resolution_coeff {"*": 1.0}`。
列透過 `billing.video.pricing.get`（任何 JWT）與
`billing.video.pricing.set`（Bearer `ARONA_ADMIN_TOKEN`）管理；
任務完成時計算出的成本記錄在任務的 key 之下。

## usage.list

`usage.list`（JWT）回傳呼叫者的分頁 usage records，涵蓋**同時**包含
API-key 列（經由 key 前綴 join）與 JWT 歸因列（經由合成的
`jwt-<user-uuid>` id join），最新在前：

| 參數 | 預設值 | 備註 |
| --- | --- | --- |
| `limit` | `50` | 限制在 `1..=200`。 |
| `offset` | `0` | 頁面偏移。 |
| `project` | 未設定 | 設定時，只回傳歸因於帶該專案標籤的 keys 的記錄（JWT 列排除）。 |

回應是 `{ "records": [...], "total", "limit", "offset", "project" }`，
其中每筆記錄攜帶 `id`、`model`、`backend`、`prompt_tokens`、
`completion_tokens`、`total_tokens`、`cost` 與 `created_at`。每月 quota
彙總使用相同的 join 形狀，因此 quota 數學與用量檢視在範圍上永遠一致。

## 相關

- [快速入門](quickstart.md) — 取得 key 並發出你的第一個被計量請求。
- [設定](configuration.md) — gateway 的環境變數。
- [認證與安全性](auth-security.md) — API key 建立與 `project` 標籤。
- [Realtime 與視訊](realtime-video.md) — 視訊定價背後的視訊任務生命週期。
- [營運](operations.md) — 健康探測與可觀測性。
- [OpenAI 相容 REST API](../api/openai-rest.md) — `/v1/*` 介面。
- [JSON-RPC API](../api/jsonrpc.md) — `usage.list`、`billing.plan`、`billing.video.pricing.*`。
- [總覽](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
