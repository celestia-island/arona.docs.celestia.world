---
title: "OpenAI-compatible REST API"
description: "OpenAI-style /v1/* reference — chat completions, embeddings, model listing, async video generations, error shapes and rate limits."
---

# OpenAI-compatible REST API

Arona exposes an OpenAI-compatible REST surface under `/v1/*` for LLM chat,
embeddings, model listing, health probing and async video generation. Any
OpenAI SDK pointed at the base URL works for chat and embeddings; the video
endpoints follow OpenAI's task-style submit/poll convention.

All request and response bodies are JSON. Errors use a uniform shape (see
[Errors](#errors)); authentication failures at the middleware layer are the one
exception and are returned as plain text (see [Authentication](#authentication)).

## Endpoints at a glance

| Method | Path | Description |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | Chat turn, streaming or non-streaming. |
| `POST` | `/v1/embeddings` | Embedding vectors for one or many inputs. |
| `GET` | `/v1/models` | Router models merged with quick-start models. |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`. |
| `POST` | `/v1/video/generations` | Submit an async video generation task. |
| `GET` | `/v1/video/generations/{id}` | Poll a video task's status / result. |

`/api/health`, `/healthz` and `/readyz` are additional readiness probes
(Kubernetes-style aliases of `/v1/health`).

## Authentication

Chat, embeddings and video endpoints authenticate with an **API key** in the
`Authorization: Bearer` header. API keys are created through the management
plane (`keys.create`, see the [JSON-RPC API](./jsonrpc.md#keys)) and look like
`arona-<uuid>`. They are stored server-side as SHA-256 hashes.

```
Authorization: Bearer arona-CHANGE_ME
```

- **Missing header** → `401` plain text: `Missing Authorization header. Use: Bearer <api_key>`.
- **Invalid or revoked key** → `401` plain text: `Invalid API key`.
- `GET /v1/models` additionally accepts a **JWT** access token (issued by
  `auth.login` / `auth.register`) so the web dashboard can list models with the
  same token it uses for the RPC plane. For that endpoint the messages are
  `Missing Authorization header. Use: Bearer <api_key_or_jwt>` and
  `Invalid API key or JWT`.

Middleware-level rejections are plain-text bodies, not the JSON error shape
described in [Errors](#errors) — the JSON shape is produced only once a request
reaches a handler.

Every authenticated `/v1` request also passes an **in-memory per-key rate
limiter** (default 60 RPM, 60-second window, configurable via
`ARONA_API_RATE_LIMIT_RPM`). Exceeding it returns `429` plain text:
`Rate limit exceeded. Try again later.` Tier-level quota and rate limits are
enforced separately and return JSON 429s with a `Retry-After` header (see
[429 and Retry-After](#429-and-retry-after)).

> Managing API keys, projects and their scoping is covered in
> [Authentication & Security](../guides/auth-security.md).

## POST /v1/chat/completions

The core OpenAI-compatible chat endpoint, with streaming support and
arona-specific extensions (`conversation_id`, `memory`, `extra`, `provider`).

### Request body

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `model` | string | yes | Model id as listed by `GET /v1/models`. |
| `messages` | array | yes | Chat turns, see below. |
| `stream` | boolean | no | Default `false`. When `true` the response is an SSE stream (see [Streaming](#streaming)). |
| `temperature` | number | no | Sampling temperature, forwarded upstream. |
| `max_tokens` | integer | no | Completion token cap, forwarded upstream. |
| `conversation_id` | string | no | Session affinity + persistence. The conversation must exist and belong to the API-key user (`403` `conversation_forbidden` otherwise, `404` `conversation_not_found` if it does not exist). The user turn is persisted at send time and the assistant reply when the turn completes; routing pins the conversation to the backend that first served it. |
| `memory` | boolean | no | Memory gateway override. Defaults to `true` (memory recall is injected when the memory gateway is enabled); `false` disables recall injection for this request. |
| `extra` | object | no | Free-form passthrough merged into the upstream payload top level (see below). |
| `tools` | array | no | OpenAI-style function-call definitions, passed through verbatim to the upstream. |
| `provider` | string | no | Explicit backend selection hint matching a backend **name** (or kind) case-insensitively. When set, only backends matching the hint are candidates. |

**`messages` entries** are `{ "role": "user" | "assistant" | "system", "content": "..." }`.
Two extensions are forwarded upstream for multimodal / agent workloads:

- `images` — attached images for vision requests (an array of
  `{ "media_type", "data", "position" }` objects; the external backend renders
  them as OpenAI `image_url` content parts).
- `tool_calls` — function-call payloads produced by the upstream model, to be
  echoed back on follow-up turns. The external backend MUST forward these or
  agent pipelines (e.g. scepter skill chains) lose every tool invocation.

**`extra` merge rules**: every `extra` key is merged into the upstream request
payload at the top level, with two hard guarantees — the reserved keys
`model`, `messages`, `stream`, `temperature`, `max_tokens` and `options` are
**never** overridden, and neither is any key the gateway itself already set.
Non-object `extra` values are ignored.

**`tools` entries** are `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }` and are forwarded verbatim.

### Non-streaming response

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

- `choices[].message` may carry `tool_calls` for function-calling turns.
- The memory state of the request is echoed in the **`X-Arona-Memory`**
  response header: `enabled` | `disabled` | `offline`.

### Streaming

Set `"stream": true`. The response is a `text/event-stream` SSE stream — one
`data:` line per chunk, each carrying a single JSON `ChatChunk`:

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta` carries `content` (and `tool_calls` deltas with
  `index`/`id`/`type`/`function` for function-calling streams).
- On OpenAI-compatible upstreams the **terminal chunk** carries a `usage`
  field with the real token counts; the gateway records it (falling back to a
  local tokenizer estimate for upstreams that never report usage, e.g. ollama
  / ws_engine).
- The stream terminates with `data: [DONE]`.
- A stream error is delivered as a `data:` event carrying
  `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}`;
  the `[DONE]` event still follows, and usage recording plus assistant
  persistence are skipped for the failed stream.
- The `X-Arona-Memory` header is present on the SSE response as well.

### Example

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

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `model` | string | yes | Embedding model id (e.g. `nomic-embed-text` — a bare name also matches a `:latest` tag). |
| `input` | string or string[] | yes | One input, or many. |

Response: `{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`.

## GET /v1/models

Lists the models routable today: every healthy registered backend's model
listing, merged with the built-in **quick-start models** (always advertised,
even before a backend is registered): `Qwen/Qwen3-0.6B`, `Qwen/Qwen3-1.7B`,
`HuggingFaceTB/SmolLM2-1.7B-Instruct`, `google/gemma-3-1b-it`,
`microsoft/Phi-4-mini-instruct`, `deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`.

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

Quick-start models appear with `owned_by` set to their provider; router models
carry the owning backend's name.

## Video generation

Task-style video endpoints for video-capable backends (e.g. `minimax-cloud`,
see [Backends](../guides/backends.md)). Jobs progress asynchronously; poll the
status endpoint until `done`.

### POST /v1/video/generations

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `model` | string | yes | Video model id registered on a video-capable backend. |
| `prompt` | string | yes | Generation prompt. |
| `negative_prompt` | string | no | Negative prompt. |
| `images` | array | no | Conditioning/reference images as an array of `{ "data_base64": "...", "mime_type": "image/png" }` objects. |
| `duration_seconds` | integer | no | Requested duration. |
| `width` / `height` | integer | no | Output resolution. |
| `provider` | string | no | Explicit backend selection hint (backend name). |
| `extra` | object | no | Backend-specific workflow overrides (seed, steps, cfg, ...). |

Success → `200`:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

Errors: `400` `missing_fields` when `model` or `prompt` is absent; `503`
`video_backend_error` / `no_backend` when no healthy video-capable backend
serves the model; `429` `quota_error` / `quota_exceeded` when the monthly quota
is exhausted.

### GET /v1/video/generations/{id}

Returns the task status:

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

- `status`: `queued` | `running` | `done` | `failed` | `cancelled`; `progress`
  advances 0–90 while running and reaches 100 on `done`.
- `result` (on `done`): `{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }` — `url` points at the generated artifact served by the backend.
- `error` (on `failed` / `cancelled`) and `cost` are populated when applicable.
- Errors: `400` `bad_id` for a non-UUID id; `404` `no_job` when the job does
  not exist or belongs to another API key.

Video jobs also fan progress out over the RPC SSE sidecar
(`video.progress` / `video.done` / `video.failed`, see
[Events & Notifications](./events.md#video-job-notifications)).

## Errors

Gateway-level errors use one shape (`json_error_response`):

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| Status | `type` / `code` | When |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`, `missing_index`, `bad_id`, ... | Malformed or missing request fields. |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id` belongs to another user. |
| `404` | `invalid_request_error` / `model_not_found` | No backend serves the requested model. Message: `No backend available for model: <model>`. |
| `404` | `invalid_request` / `conversation_not_found` | Conversation not found. |
| `404` | `not_found` / `no_job` | Video job not found. |
| `502` | `server_error` / `bad_gateway` | Upstream non-2xx: message `upstream <status>: <detail>` (detail from the upstream error body, bounded to 4 KB). Transport failures (connect/read/timeout) also map to 502 with the error string. |
| `500` | `server_error` / `backend_error` | Other backend failures (e.g. backend does not support the operation). |
| `500` | `server_error` / `internal_error` | Any remaining gateway internal error. |
| `429` | see below | Quota / rate-limit rejections with `Retry-After`. |

## 429 and Retry-After

429 responses include a `Retry-After` header (seconds) so OpenAI-compatible
clients back off:

| Trigger | Status body | `Retry-After` |
| --- | --- | --- |
| Monthly quota exceeded | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | Seconds until the next month. |
| Tier per-minute rate limit | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`. |
| In-memory per-key limiter (60 RPM default) | plain text `Rate limit exceeded. Try again later.` | none (middleware rejection). |

Tiers, quota scoping and usage accounting are described in
[Billing & Usage](../guides/billing-usage.md).

## Usage recording

Every `/v1` request records a usage row under the API-key prefix (`arona-XX`)
when it completes (non-streaming chat, streaming chat at the terminal chunk,
embeddings, and video jobs on completion with their computed cost). See
[Billing & Usage](../guides/billing-usage.md) for the recording model and how
quota is enforced.

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
