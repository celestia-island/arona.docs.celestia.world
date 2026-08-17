---
title: "Events & Notifications"
description: "Server-sent event (SSE) sidecar — chat.stream, models.progress, realtime.event and video notifications."
---

# Events & Notifications

Streaming tokens, deploy progress and realtime events are **not** delivered on
the JSON-RPC WebSocket socket. Each streaming RPC creates a **session id** and
pushes notifications to the SSE endpoint as server-sent events:

```
GET /api/rpc/events?session=<session_id>
```

## Subscribe-before-send recipe

Notifications emitted between the RPC call returning a session id and the SSE
subscription being established are **dropped** (the pre-subscription window).
The reliable pattern is:

1. Open the SSE stream first (it blocks until a session id is attached).
2. Fire the RPC that returns the session id (e.g. `chat.send`,
   `agents.deploy`, `realtime.start`, `video.create`).
3. Read notifications off the SSE stream as they arrive.

Every notification is a JSON-RPC 2.0 style message with `"jsonrpc": "2.0"`,
a `method` and a `params` object.

## Notification catalog

### `chat.stream`

One notification per token, produced by `chat.send` (and any streaming chat
path that uses a session channel):

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — one content delta.
- `is_complete` — `false` until the final chunk (when the upstream attaches a
  finish reason, the final content chunk may already carry `is_complete:
  true` with a non-empty token); the **terminal** notification always follows
  with an empty `token` and `is_complete: true`.
- A stream error is delivered as a terminal notification with
  `token: "Stream error: ..."` and `is_complete: true` (see
  `packages/core/src/gateway/rpc.rs`).

### `models.progress`

Model download progress for `agents.deploy`, forwarded from the agent. The
`stream_id` comes from the `agents.deploy` response.

### `realtime.event`

Server events for an open full-duplex realtime session, pushed to the session
channel (`packages/core/src/gateway/realtime.rs`). Client events sent via
`realtime.event` RPC are forwarded upstream; server events arrive here.

### Video job notifications

`video.create` jobs push progress over the session channel
(`packages/core/src/gateway/video.rs`):

| Method | Payload (params) | Meaning |
| --- | --- | --- |
| `video.progress` | `job_id`, `stream_id`, `status: "running"`, `progress` (0–90) | Job is running. |
| `video.done` | `job_id`, `stream_id`, `result`, `cost` | Job finished; `result` carries the artifact URL. |
| `video.failed` | `job_id`, `stream_id`, `error` | Job failed or was cancelled. |

## Ordering notes

- The SSE stream is ordered per session; tokens arrive in generation order.
- A single session id may only be consumed by one SSE subscriber; open the
  stream before (or immediately after) the RPC that returns the id.
- The `x-session-id` header on `POST /api/rpc` attaches the RPC **response**
  itself to a session channel as well — used by clients that want the
  response echoed over the same stream.

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
