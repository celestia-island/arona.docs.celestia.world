---
title: "Realtime & Video"
description: "Full-duplex realtime sessions (realtime.start/event/stop), the engine.invoke perception/control channel, and asynchronous video generation jobs."
---

# Realtime & Video

Arona exposes two multimodal channels beyond plain text chat: **full-duplex
realtime sessions** (speech/video in and out over one bidirectional channel)
and **asynchronous video generation** (task-style jobs that run in the
background and report progress). Both are routed to the backend that serves
the requested model and both are metered through the billing layer.

## Realtime sessions

A realtime session is a bidirectional channel between **one client** and **one
upstream**: a cloud realtime API (OpenAI-Realtime / Qwen-Omni-Realtime WebSocket
vocabulary) or a local CEP engine. Client events arrive over JSON-RPC and are
forwarded upstream; server events are pushed back as `realtime.event`
notifications over the session SSE channel. Audio travels as base64 PCM16
(16 kHz client→model, 24 kHz model→client), matching the cloud vendors' wire
format so the gateway passes bytes through untouched
(`packages/core/src/backends/realtime.rs:1-19`).

### `realtime.start`

Opens a session against the backend serving `model` (JWT; params `model`,
`config?`, `conversation_id?` — `packages/core/src/gateway/rpc.rs:1890-1898,
1914-1984`). The backend **must** declare the `realtime` capability (audio/video
modalities); otherwise the call fails explicitly with
`model {model} does not support realtime sessions (no audio/video modality)` —
there is no silent fallback to text chat
(`packages/core/src/gateway/realtime.rs:138-142`).

Two upstream kinds are supported (`packages/core/src/gateway/realtime.rs:143-167`):

- **CEP engine upstream** — routes events over the Celestia Engine Protocol
  `Engine.InvokeStart` streaming channel, so a locally deployed omni engine
  joins the same client surface with no new wire format.
- **Cloud upstream** — a fixed `wss://` URL speaking the cloud realtime event
  vocabulary (`session.update`, `input_audio_buffer.*`, `response.audio.delta`,
  ...). The cloud impl re-issues `session.update` on reconnect.

The response is `{ "session_id": ..., "stream_session": ... }` — subscribe to
`/api/rpc/events?session=<stream_session>` before (or immediately after) the
call to receive server events. The optional `conversation_id` persists the
speech transcript as assistant messages and records token usage for billing
(`packages/core/src/gateway/realtime.rs:32-85`).

### `realtime.event`

Sends one client event into the session (JWT; params `session_id`, `event` —
`packages/core/src/gateway/rpc.rs:1989-2013`). Supported events include
`session.update`, `input_audio_buffer.append` / `.commit` / `.clear`,
`input_image_buffer.append`, `response.create`, `response.cancel` and
`session.stop`. `send_event` is **non-blocking**: the event is queued on an
mpsc channel and the forwarder task writes it upstream
(`packages/core/src/gateway/realtime.rs:254-280`). Only the session owner may
send events.

### `realtime.stop`

Closes and removes the session (JWT; params `session_id` —
`packages/core/src/gateway/rpc.rs:2016-2034`). Each session owns exactly one
**forwarder task** that holds the upstream and multiplexes both directions:
client events are consumed from the queue and upstream events are polled in
the same loop. The forwarder exits when the upstream closes or the session is
stopped, removing the registry entry
(`packages/core/src/gateway/realtime.rs:195-250`).

Server events are pushed as `realtime.event` notifications with params
`{ session_id, event }` over the session channel — see
[Events & Notifications](../api/events.md).

## `engine.invoke`

`engine.invoke` is the generic **synchronous** engine-method channel
(ADMIN: JWT + `is_admin`; params `model`, `method`, `params?` —
`packages/core/src/gateway/rpc.rs:261-264,2049-2079`). It invokes an arbitrary
method on the backend serving `model` and returns the result directly, making
it the high-frequency perception/control channel: `sensor.ingest`,
`control.setpoint` style calls in 20-30 Hz loops. Backends without a generic
invocation channel (all OpenAI-compatible HTTP backends) reject explicitly with
`backend does not support generic invocation`
(`packages/core/src/backends/mod.rs:573-586`).

## Video generation (REST)

Video jobs are OpenAI-style async tasks over the REST surface (API key auth —
`packages/core/src/gateway/server.rs:876-993`; see
[OpenAI-compatible REST API](../api/openai-rest.md)):

**`POST /v1/video/generations`**

| Field | Type | Notes |
| --- | --- | --- |
| `model` | string | required — selects a video-capable backend. |
| `prompt` | string | required. |
| `negative_prompt` | string? | |
| `images` | array? | Base64 data URLs (`data:image/png;base64,...`), carried as `{ data_base64, mime_type }`. |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | Backend selection hint (matched against the backend name). |
| `extra` | object? | Backend-specific overrides (seed, steps, cfg, ...). |

Response:

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`** polls the job and returns `id`, `object`,
`model`, `status`, `progress`, `result`, `error`, `cost`, `created_at`. Jobs
are scoped to the caller: a job owned by another user returns 404. The REST
surface enforces the same billing gates (monthly quota, per-minute rate limit)
as the chat path.

## Video generation (RPC)

The same capability is available over JSON-RPC (JWT —
`packages/core/src/gateway/rpc.rs:371-386,1296-1431`):

| Method | Params | Returns |
| --- | --- | --- |
| `video.create` | same fields as the REST call | `{ job_id, stream_id }`. |
| `video.get` | `job_id` | The job view (status, progress, result, cost, ...). |
| `video.list` | `limit?` (default 20, clamped to 1-100) | `{ jobs: [...] }`, newest first. |
| `video.cancel` | `job_id` | `{ "ok": true }` — only the owner may cancel. |

`video.create` returns a `stream_id`; subscribe to
`/api/rpc/events?session=<stream_id>` to receive job notifications
(`video.progress` / `video.done` / `video.failed` — see
[Events & Notifications](../api/events.md)).

## Backend

Video generation is **cloud-only**: the MiniMax H3 Open Platform API, backend
kind `minimax-cloud` (`BackendKind::CloudVideo` —
`packages/core/src/backends/mod.rs:502-504,720-727,759-761`). The flow is
task-style:

1. `POST /v1/video_generation_v2` → `task_id`
2. poll `GET /v1/query/video_generation_v2?task_id=...` until `Success` /
   `Fail` / still `Pending`
3. on success, resolve the artifact via
   `GET /v1/files/{file_id}/content` → `{ file: { download_url } }`

(`packages/core/src/backends/minimax_cloud.rs:66-210`). The MiniMax backend
does not serve chat/embeddings; its capabilities declare
`supports_video_generation` and `realtime: false` (see
[Backends](./backends.md) for the capability model). Routing resolves video
requests only against backends with `supports_video_generation`, honouring the
optional `provider` hint (`packages/core/src/routing/mod.rs:205-263`).

The **ComfyUI backend was removed** during the model-platform convergence:
configuring backend kind `"comfyui"` fails with `comfyui backend removed`
(`packages/core/src/backends/mod.rs:756-757`). There is no self-hosted video
path; video always goes through a `minimax-cloud` backend.

## Job lifecycle & pricing

A video job moves through `queued → running → done | failed | cancelled`
(`packages/core/src/gateway/video.rs`):

- **create** — the job row is persisted (`queued`, progress 0) and a poller
  task is spawned (`video.rs:109-176`).
- **running** — the poller submits the task (progress 5), then polls every
  1.5 s, bumping progress by 5 every few iterations up to **90**
  (`video.rs:178-275`). Poll errors are logged and retried.
- **done** — progress 100, the result URL and the computed cost are persisted,
  usage is recorded, and a `video.done` notification is fanned out
  (`video.rs:332-368`).
- **failed** — submit or poll failure → `video.failed`; after 900 poll
  iterations (~22.5 minutes) the job fails with `generation timed out`.
- **cancelled** — `video.cancel` sets a flag the poller observes on its next
  pass; the job is marked `cancelled` and `video.failed` fires with error
  `cancelled` (`video.rs:389-400`).

Usage is recorded with the video-specific cost: `record_video` writes a
per-request usage record with zero tokens and an explicit dollar cost
(`packages/core/src/billing/mod.rs:496-531`).

**Pricing** is model-specific, in the `video_pricing` table
(`packages/core/src/billing/video.rs`):

| Mode | Formula |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (default) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` maps the short-side pixel key (e.g. `"768"`) to a
multiplier, with `"*"` as the fallback. Models without a configured row fall
back to: mode `per_second_resolution`, `base_price` 0.0,
`price_per_second` 0.005, `price_per_frame` 0.0, `resolution_coeff {"*": 1.0}`,
currency USD (`billing/video.rs:20-32`). Query rows with
`billing.video.pricing.get` (JWT) and upsert them with
`billing.video.pricing.set` (admin token) — see
[JSON-RPC API](../api/jsonrpc.md). See [Billing & Usage](./billing-usage.md)
for how usage records aggregate into monthly billing.

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
