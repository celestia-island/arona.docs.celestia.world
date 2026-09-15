---
title: "Billing & Usage"
description: "Usage records, per-model cost, billing tiers, quota and rate-limit enforcement, project-scoped keys, video pricing and the usage.list RPC."
---

# Billing & Usage

Arona meters every model request and enforces tiered quotas and rate limits
at the gateway. Per-model prices come from the shared plana pricing table
(never reimplemented in arona), usage is persisted as `usage_records` rows,
and the whole monthly picture is exposed through the `usage.list` RPC.

## Usage records

Every metered request ends up as one row in the `usage_records` table
(`m20250101_000006_create_usage_records`):

| Column | Type | Meaning |
| --- | --- | --- |
| `id` | `UUID` | Primary key, generated. |
| `api_key_id` | `VARCHAR(64)` | The **masked key id** — the first 8 characters of the API key, an ellipsis, its last 4 (keys look like `arona-{uuid}`) — or a synthetic `jwt-<user-uuid>` id for JWT-attributed RPC channels. Display only: attribution uses `api_key_row_id` below. |
| `api_key_row_id` | `UUID` | The `api_keys` row the request was authenticated with. Quota and usage queries join on it; rows written before the column existed carry `NULL` and fall back to the 8-character `api_key_id` match, which buckets 256 keys together. |
| `model` | `VARCHAR(128)` | Model id the request was routed to. |
| `backend` | `VARCHAR(64)` | Backend kind: `gateway`, `rpc`, `realtime`, or the backend capability name. |
| `prompt_tokens` | `INTEGER` | Input tokens, upstream-reported or estimated. |
| `completion_tokens` | `INTEGER` | Output tokens, upstream-reported or estimated. |
| `total_tokens` | `INTEGER` | Sum of the two. |
| `cost` | `DOUBLE PRECISION` | Computed USD cost; never `NULL` for a metered request (a model with no pricing row is billed at the fallback rate below). |
| `created_at` | `TIMESTAMPTZ` | When the request completed. |

Indexes exist on `api_key_id`, `model` and `created_at` (the columns the
monthly aggregation and rate-limit windows scan).

## Recording channels

Usage is recorded on every metered channel:

1. **REST non-streaming** — `POST /v1/chat/completions` and
   `POST /v1/embeddings` record the upstream-reported usage once the response
   has been produced. A backend that reports nothing usable is estimated
   locally instead: the shipped self-hosted backends answer `0/0/0` rather
   than omitting `usage`, and a report like that must not bill the request at
   exactly zero.
2. **REST streaming (SSE)** — the upstream-reported usage wins when the
   stream carried it (OpenAI-compatible terminal chunk `usage` field);
   otherwise — including when it reports `0/0/0` — a local CJK-aware
   tokenizer estimate (`estimate_usage`) is recorded as-is. A stream that ends
   without a clean finish — the upstream
   errored mid-stream, or the client disconnected and dropped the response
   body — is billed the same way for whatever was generated before it
   stopped, through a billing guard owned by the response body. Only a
   stream that produced neither text nor usage is **not** recorded at all.
3. **RPC `chat.send`** — the same estimate-vs-upstream logic applies; rows
   are attributed with the synthetic `jwt-<user-uuid>` id so they join back
   to the user.
4. **Realtime sessions** — each completed `response_done` transcript records
   its token usage (when non-zero) under `jwt-<user-uuid>` with backend
   `realtime`.
5. **Video jobs** — a completed job records an explicit dollar cost
   (see [Video pricing](#video-pricing)); token counts are zero.

Recording is best-effort: a failed insert is logged and never fails the
request.

## Cost calculation

Cost is computed from the canonical per-1M-token pricing table
(`plana_llm_provider::metering::lookup_pricing`, shared across all
celestia-island services):

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

Model matching in the table is substring-based on the lowercased model id
(more specific families win). A model with **no pricing row is not free**:
arona charges plana's own fallback estimate
(`plana_llm_provider::metering::estimate_cost`, provider-keyed, generic arm
`3.00` / `15.00` USD per 1M input / output tokens), logs a warning naming
that model once per process, and counts the resulting cost toward the
quota. This is what keeps the self-hosted quick-start models and the
embedding families from writing `NULL`-cost rows that consume nothing (on a
tier with no quota, that used to be free forever). **Do not reimplement
pricing in arona — add the missing row to plana's table.**

## Tiers

Tiers live in the `billing_tiers` table, seeded on first migration
(`m20250101_000007_create_billing_tiers`). A `NULL` quota column means
"unlimited" for that dimension. Users without a `tier_id` fall back to the
seeded `free` tier.

| Tier | Monthly USD quota | Monthly token quota | Per-key RPM |
| --- | --- | --- | --- |
| `free` | $1.00 | 100,000 | 10 |
| `pro` | $20.00 | 5,000,000 | 120 |
| `enterprise` | unlimited (`NULL`) | unlimited (`NULL`) | 1000 |

Tier assignment is an admin operation (`billing.plan.set` RPC); the current
tier and usage are surfaced through `billing.plan`.

## Quota and rate-limit enforcement

### REST (`/v1/*`)

Before every **metered** REST endpoint — `/v1/chat/completions`,
`/v1/embeddings` and `/v1/video/generations` — the gateway enforces two
gates for key-authenticated requests:

- **Monthly quota** — the tier's `monthly_quota_usd` and `quota_tokens`
  limits are checked against usage accumulated since the first instant of the
  current calendar month. Either dimension reaching its limit trips the gate.
- **Per-key rate limit** — the tier's `rate_limit_rpm` is checked against the
  number of requests recorded for this key prefix in the trailing 60-second
  window. (`/v1/models` and the health endpoints are not metered and not
  gated.)

A rejection is an HTTP **429 Too Many Requests** with a `Retry-After` header
and an OpenAI-style error body:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| Rejection | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| Monthly quota exhausted | `quota_error` / `quota_exceeded` | Seconds until the start of the **next calendar month** |
| Tier rate limit exceeded | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

JWT-authenticated `chat.send` goes through the same monthly quota gate, but
against the **whole-user** window (the call carries no API key). A rejection
is a JSON-RPC error with the implementation-defined code `-32006`
(`QUOTA_ERROR`) and the same message as the REST quota rejection. There is no
per-key rate limit on the RPC path — rate limiting is key-scoped and RPC
calls have no key. `realtime.start` passes through the same whole-user monthly quota gate (a
session open with an exhausted quota is rejected with `-32006`);
`video.create` checks the quota at job creation.

## Points ledger (prepaid, tier as backstop)

Groups and users carry points wallets (`ledger_accounts`, one personal
wallet per user and one pool per group). Settlement is **points-first**:
every metered request tries to deduct
`round(cost_usd × POINTS_PER_USD × member_multiplier)` from the drawing
account (the group pool for group keys, the personal wallet otherwise) —
atomic conditional updates, so concurrent deductions can never overdraw.
When the wallet cannot cover the charge its whole remaining balance is
debited: the wallet is **clamped to zero** (the ledger records a
`consume_shortfall` entry with the gap, the usage row carries the points
actually debited) and the rest of the charge rides the tier monthly quota
(postpaid). The next request is therefore refused as soon as the tier quota
is exhausted too, instead of riding a leftover sub-charge balance forever.
When BOTH are drained the REST gate answers **402 Payment Required**
(`insufficient_credits` / `payment_required`, `Retry-After` until month
end); the RPC path keeps the `-32006` `QUOTA_ERROR` shape.

`POINTS_PER_USD` comes from the env (default 100 — one point = one cent).
Management surface: `group.credits.topup` (platform admin),
`group.credits.allocate` (group admin, atomic pool→wallet transfer),
`group.credits.balance`, `ledger.self`; `billing.plan` reports
`points_balance` and `points_per_usd`. Usage rows settled from the ledger
carry `points` and `account_type` (`user` | `group`).

## Fail-open tradeoff

Billing is **best-effort by design**. If the database query behind a quota or
rate-limit check fails, the check returns `Unknown` and the request is
**allowed** (only logged) instead of blocking chat. An operator can rely on
429s to protect capacity, but must not treat them as a hard guarantee when
the database is unhealthy — the documented tradeoff is availability of the
chat path over strict metering.

## Project-scoped keys

API keys can be created with a `project` label (`api_keys.project`,
`default` when unset). Quota enforcement honors it:

- A key tagged with a project other than `default` checks its quota against
  usage attributed to **that project's own bucket** (`PROJECT_MONTHLY_USAGE_SQL`).
- `default` / untagged keys keep the **whole-user** window, matching
  pre-project behavior.

JWT-attributed RPC rows (`jwt-<user-uuid>`) carry no project label and are
**intentionally excluded** from project windows — they still count toward the
whole-user window, so a project cannot be "hidden" by sending traffic over
the RPC channel.

## Video pricing

Video generation uses model-specific, task-style pricing (per-token pricing
makes no sense for a video). Pricing rows live in the `video_pricing` table;
`compute_cost` falls back to a placeholder default when no row is configured.

| Mode | Cost |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (default) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` is a JSON object keyed by the short-side pixel value
(e.g. `"768"`); the `"*"` key is the fallback. The default pricing row is
mode `per_second_resolution`, `base_price` 0.0, `price_per_second` 0.005,
`resolution_coeff {"*": 1.0}`. Rows are managed through
`billing.video.pricing.get` (any JWT) and `billing.video.pricing.set`
(Bearer `ARONA_ADMIN_TOKEN`); the computed cost is recorded against the
job's key when the job completes.

## usage.list

`usage.list` (JWT) returns the caller's paginated usage records covering
**both** API-key rows (joined via key prefix) and JWT-attributed rows (joined
via the synthetic `jwt-<user-uuid>` id), newest first:

| Param | Default | Notes |
| --- | --- | --- |
| `limit` | `50` | Clamped to `1..=200`. |
| `offset` | `0` | Page offset. |
| `project` | unset | When set, only records attributed to keys with that project label (JWT rows are excluded). |

The response is `{ "records": [...], "total", "limit", "offset", "project" }`
where each record carries `id`, `model`, `backend`, `prompt_tokens`,
`completion_tokens`, `total_tokens`, `cost` and `created_at`. The monthly
quota aggregation uses the same join shape, so quota math and the usage view
always agree on scope.

## Related

- [Quickstart](quickstart.md) — get a key and make your first metered request.
- [Configuration](configuration.md) — environment variables for the gateway.
- [Authentication & Security](auth-security.md) — API key creation and the `project` label.
- [Realtime & Video](realtime-video.md) — the video job lifecycle behind video pricing.
- [Operations](operations.md) — health probes and observability.
- [OpenAI-compatible REST API](../api/openai-rest.md) — the `/v1/*` surface.
- [JSON-RPC API](../api/jsonrpc.md) — `usage.list`, `billing.plan`, `billing.video.pricing.*`.
- [Overview](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: chat.send and realtime.start are quota-gated on the RPC path against the whole-user monthly window; video.create checks quota at job creation. The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
