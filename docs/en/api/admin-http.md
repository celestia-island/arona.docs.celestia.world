---
title: "Admin HTTP API"
description: "Bearer-token admin surface — register/list/remove backends and manage model aliases over /api/admin/*."
---

# Admin HTTP API

The `/api/admin/*` surface manages the gateway's **backends** (upstream model
providers), **aliases** (model-name → model-id redirection) and **usage
queries** (aggregated metering). It is the HTTP counterpart of the RPC
management plane (see the [JSON-RPC API](./jsonrpc.md)) and is primarily used
by operators and the admin UI.

## Authentication

Every `/api/admin/*` route requires:

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN` is read from the environment at process start
(`GatewayServer::new`). If the variable is **unset**, or the presented token
does not match, the request is rejected with `401`:

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

The bearer prefix is matched case-insensitively (`Bearer` or `bearer`).

> Unlike the `/v1/*` surface, admin auth never falls back to API keys or JWTs,
> and it is enforced with an exact-token comparison — rotate the token by
> restarting the process with a new value.

## Backends

Backends are the routable upstreams behind the gateway. Registration makes a
backend routable immediately, persists its config for restart restore, probes
it (flips healthy within ~1–2 s) and, for bridge URLs, keeps the tunnel alive.
Backend types and URL semantics are detailed in
[Backends](../guides/backends.md).

### POST /api/admin/backends — register a backend

Request body (all fields optional except where noted):

| Field | Type | Notes |
| --- | --- | --- |
| `type` | string | Backend kind. One of `external` (any OpenAI-compatible HTTP API), `ollama` (local or remote ollama server), `engine` (CEP engine over `ws://`/`wss://`), `minimax-cloud` (cloud video API). MDD engine names (`llama_cpp`, `vllm`, `ollama`, `cloud`, `external_api`, `candle`, `native`, ...) resolve through the planner. `comfyui` is **rejected** (`comfyui backend removed`); anything else → `400` `unknown_type`. Defaults to `ollama` when missing. |
| `url` | string | Backend base URL. `evernight://<node>/<service>` bridge URLs are resolved through the local evernight agent into a local TCP forward (resolution failure → `502` `evernight_unreachable`). Defaults to `http://localhost:11434`. |
| `api_key` | string | Optional upstream API key, sent as `Authorization: Bearer` on upstream calls. |
| `name` | string | Backend name. Defaults to the `type` value when missing. Used as the routing `provider` hint and for config row identity. |
| `models` | string[] | Static model list. The routing source when probing discovers none. For `external` backends, discovered models are merged after the static list (static ids keep precedence); `engine` backends return their discovered model cache first and append static ids after; `minimax-cloud` performs no model discovery (its probe only health-pings `/v1/query/available_models`) and serves the static list alone. Ignored by `ollama`, which discovers models from `/api/tags`. |
| `workflow` | object | Optional. Legacy — historically consumed by the removed ComfyUI backend; no current backend reads it (kept for `backend_configs` column compatibility). |

Example:

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

Success → `200`:

```json
{ "status": "ok", "message": "backend registered" }
```

Registration side effects:

- The backend is **registered and routable immediately** (no restart needed).
- The config is **persisted** to the `backend_configs` table and restored at
  startup (a DB failure is logged but never blocks the response).
- A fire-and-forget **probe** runs right away so the backend flips healthy
  within ~1–2 s instead of staying fail-closed until the next 60 s
  health-checker round.
- For `evernight://` URLs, a **keepalive task** watches the tunnel: on
  reconnect with a new local port it transparently rebuilds and re-registers
  the backend under the same name.

### GET /api/admin/backends — list backends

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

- `backends.count` — number of **healthy** backends.
- `backends.health` — per-backend `backend_<index>:<kind>` label and health
  state (`Healthy` / `Degraded` / `Unhealthy`). The `<index>` is the router
  registration index used by `DELETE /api/admin/backends`.
- `models` — every model id routable today (same listing as
  `GET /v1/models`, without the quick-start merge; see
  [OpenAI-compatible REST](./openai-rest.md#get-v1models)).

### DELETE /api/admin/backends — remove a backend

Identified by its router **index** in the JSON body — not by name:

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `index` | integer | yes | Router registration index, matching the `backend_<index>` label in the `GET /api/admin/backends` health report. |

- Missing `index` → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`.
- Index out of range → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`.
- Success → `200` `{ "status": "ok", "message": "backend removed" }`.
- The persisted `backend_configs` row is deleted best-effort: the backend
  name is recovered from the `owned_by` of its model listing; a mismatch
  leaves the row in the store (DB failures are logged, never fatal).

## Aliases

Aliases map one model name to another (`alias` → `target`) so requests for
one model id route to a different backend model. Aliases are resolved before
routing, so they apply uniformly to chat, embeddings and video lookups.

> Aliases are **in-memory router state only** — they are not persisted and are
> lost on restart. Register them after startup or recreate them from your own
> provisioning state.

### POST /api/admin/aliases — add an alias

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `alias` | string | yes | The model name clients will request. |
| `target` | string | yes | The model id requests are routed to. |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- Missing `alias` → `400` `missing_alias`; missing `target` → `400`
  `missing_target`.
- Success → `200` `{ "status": "ok", "message": "alias added" }`.
- Adding an existing alias replaces its target.

### GET /api/admin/aliases — list aliases

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

Pairs are returned sorted by alias.

### DELETE /api/admin/aliases — remove an alias

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `alias` | string | yes | The alias to remove. |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- Missing `alias` → `400` `missing_alias`.
- Removing an unknown alias is a no-op success → `200`
  `{ "status": "ok", "message": "alias removed" }`.

## Usage

Arona is the metering source of truth: every proxied request that produces a
usage row is recorded in `usage_records`. Requests can carry an
**attribution reference** — the `x-celestia-ref` header on
`POST /v1/chat/completions` (stream and non-stream) and `POST /v1/embeddings`,
or the `ref_id` param on the JSON-RPC `realtime.start` method (typically a
conversation UUID chosen by the calling service) — stored in the row's
`ref_id` column. `POST /api/admin/usage/query` aggregates over those
references so downstream services (e.g. shittim-chest) can reconcile usage
they attributed, without keeping local metering of their own.

### POST /api/admin/usage/query — aggregate usage query

Request body (all fields optional):

| Field | Type | Notes |
| --- | --- | --- |
| `ref_ids` | string[] | Attribution references to aggregate over (`WHERE ref_id = ANY(...)`). Absent / empty / blank-only → **global** (all rows). Entries are trimmed; each is capped at 64 characters. |
| `since` | string | Lower bound on `created_at`, inclusive. `YYYY-MM-DD` (start of that day, UTC) or an RFC3339 timestamp. |
| `until` | string | Upper bound on `created_at`, exclusive. `YYYY-MM-DD` covers that **whole day** (exclusive next-day midnight UTC); an RFC3339 timestamp is the exclusive instant itself. |
| `group_by` | string | Additionally aggregate per key: `model` \| `backend` \| `ref` \| `day` (`day` = UTC day bucket of `created_at`, key formatted `YYYY-MM-DD`). Anything else → `400` `bad_group_by`. |
| `limit` | integer | Page size for `include_records`, clamped to 1–500, default 100. |
| `offset` | integer | Page offset for `include_records`, default 0. |
| `include_records` | bool | Also return paginated raw rows plus `records_total`. Default `false`. |

All filters are ANDed. Aggregation happens entirely in SQL (`GROUP BY`,
no in-memory scan); groups are ordered by `total_tokens` descending (ties
broken by group key ascending), raw rows newest first (ties broken by row
id) — both orderings are deterministic for stable pagination.

```bash
curl -X POST http://192.0.2.10:8080/api/admin/usage/query \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "ref_ids": ["0b914cb6-2d71-4c55-9f1a-3f2b1c9d8e7f"],
    "since": "2026-08-01",
    "until": "2026-08-21",
    "group_by": "model",
    "include_records": false
  }'
```

Success → `200` — `totals` is always present; `groups` only when
`group_by` is set; `records` / `records_total` / `limit` / `offset` only when
`include_records` is `true`:

```json
{
  "totals": {
    "requests": 12,
    "prompt_tokens": 3400,
    "completion_tokens": 810,
    "total_tokens": 4210,
    "cost_usd": 0.0121
  },
  "groups": [
    {
      "key": "Qwen/Qwen3-1.7B",
      "requests": 12,
      "prompt_tokens": 3400,
      "completion_tokens": 810,
      "total_tokens": 4210,
      "cost_usd": 0.0121
    }
  ]
}
```

With `include_records: true` each row carries `id`, `ref_id`, `api_key_id`,
`model`, `backend`, `prompt_tokens`, `completion_tokens`, `total_tokens`,
`cost` and `created_at` (RFC3339), newest first:

```json
{
  "totals": { "requests": 1, "prompt_tokens": 120, "completion_tokens": 30,
              "total_tokens": 150, "cost_usd": 0.0004 },
  "records_total": 1,
  "limit": 100,
  "offset": 0,
  "records": [
    {
      "id": "8f14e45f-ceea-467a-9eaa-3728a80d62a1",
      "ref_id": "0b914cb6-2d71-4c55-9f1a-3f2b1c9d8e7f",
      "api_key_id": "arona-Xy",
      "model": "Qwen/Qwen3-1.7B",
      "backend": "gateway",
      "prompt_tokens": 120,
      "completion_tokens": 30,
      "total_tokens": 150,
      "cost": 0.0004,
      "created_at": "2026-08-20T09:30:12+00:00"
    }
  ]
}
```

Notes:

- `cost_usd` is `SUM(cost)` with `NULL` costs ignored (rows for unpriced
  models and zero-token realtime rows contribute `0`).
- Grouping by `ref` yields `"key": null` for the bucket of reference-less
  rows when the query is global.
- Realtime sessions started with a `ref_id` write usage rows even when the
  engine reported zero tokens (local CEP speech engines) — those rows show
  `0` tokens and `null` cost.
- Malformed `since`/`until` → `400` `bad_since` / `bad_until`;
  unknown `group_by` → `400` `bad_group_by`; DB failure → `500`
  `internal_error` (details in the server log).

## Persistence summary

| Resource | Persisted? | Restore on restart |
| --- | --- | --- |
| Backends | Yes — `backend_configs` table (`name` key, upsert on register, delete on removal). | Yes: restored at startup; external backends start fail-closed and flip healthy after the first probe round. `evernight://` URLs are re-resolved through the bridge at startup. |
| Aliases | No — in-memory `Router.aliases` only. | No. |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler), admin_usage_query (POST /api/admin/usage/query); packages/core/src/billing/usage_query.rs (query parsing, SQL builders, executor); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
