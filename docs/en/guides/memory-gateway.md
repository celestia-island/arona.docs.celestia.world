---
title: "Memory Gateway"
description: "Long-term memory for chat — recall injection, episode writeback, per-request control, header states and the memory.status / memory.delete RPCs."
---

# Memory Gateway

The Memory Gateway gives chat turns access to **long-term memory** stored in
the entelecheia scepter / Philia memory service. On each chat turn Arona
queries the service for memories relevant to the conversation, injects them
into the prompt as a system section, and — after a completed reply — writes
the turn back as an episode so future conversations can recall it.

It is a WebSocket JSON-RPC client to Philia (`Sync.ConnectHandshake`,
`Sync.MemoryQueryRequest`, `Sync.MemoryStoreRequest`,
`Sync.MemoryDeleteRequest`). Connections are established lazily, dropped on
any error and re-established on the next call; every failure degrades
silently and **never blocks the chat path**.

## Configuration

The gateway is controlled by three environment variables:

| Variable | Meaning |
| --- | --- |
| `ARONA_MEMORY_URL` | WebSocket URL of the scepter / Philia service, e.g. `ws://192.0.2.10:8424/ws`. |
| `ARONA_MEMORY_TOKEN` | Token for the memory service. |
| `ARONA_MEMORY_WRITEBACK` | Whether completed turns are written back. Default **on**; set `false` to disable (parsed as a strict boolean — `0` is not accepted). |

Both `ARONA_MEMORY_URL` **and** `ARONA_MEMORY_TOKEN` must be set and
non-empty, otherwise the gateway is **disabled**: recall and writeback are
skipped entirely and every request reports `disabled`. The token is sent both
as a `?token=` query parameter on the WebSocket upgrade and inside the
`Sync.ConnectHandshake` request.

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

See [Configuration](configuration.md) for the full environment reference.

## Recall injection

With the gateway enabled, **every chat turn** — REST non-streaming
`/v1/chat/completions`, REST streaming (SSE), and RPC `chat.send` — queries
the memory service before the request is forwarded:

- The query is the **last user message** of the assembled context.
- Up to **5** memories are requested (`limit = 5`).
- Results are rendered as a markdown system section titled
  `## Relevant Long-Term Memories`, one `- [score] text` bullet per memory
  (scores with two decimals, blank entries skipped), and prepended to the
  message list as a `system` message. Injection is idempotent: a context that
  already carries the section is not re-injected.
- If no relevant memories are returned, nothing is injected and the turn
  proceeds unchanged.

The recall runs before conversation persistence and upstream forwarding; a
slow or failing memory service adds **no latency guarantee** beyond its own
10-second RPC timeout and cannot fail the request.

## Writeback

After a completed assistant reply, the turn is written back to the memory
service as an **episode** node. The episode text is a heuristic transcript of
the turn — `User: <user content>\nAssistant: <assistant content>` (either
side omitted when empty; both empty skips the writeback). Writeback is
**fire-and-forget**: it runs in a spawned task, never blocks the chat
response, and its failures are only logged inside the memory client. (On the
REST streaming path, writeback additionally requires a conversation to be
attached to the request; the non-streaming REST and RPC paths write back
regardless.)

## Per-request control

Both the REST chat request body and the RPC `chat.send` params accept an
optional `memory` field to override the server configuration **per call**:

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` — force recall on / off for this turn.
- omitted (`null`) — follow the server configuration (`req.memory.unwrap_or(true)`),
  i.e. enabled iff the gateway is configured.

The override affects recall; writeback follows only
`ARONA_MEMORY_WRITEBACK` plus the gateway being enabled.

## Header states

REST responses carry the turn's memory state in the **`X-Arona-Memory`**
response header; the RPC `chat.send` response echoes the same value in a
`memory` field of its result. Possible states:

| Value | Meaning |
| --- | --- |
| `enabled` | Memory was requested, the gateway is configured, recall succeeded and at least one memory was injected. |
| `disabled` | Gateway not configured, or `memory: false` on the request, or no user message to query, or recall succeeded but returned **no** relevant memories (nothing to inject). |
| `offline` | Memory was requested and the gateway is configured, but the recall call failed (connection refused, RPC error or timeout). |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## Failure semantics

Everything degrades explicitly, in the same direction — chat always runs:

- **Recall failure** — logged at `warn` level; the request proceeds without
  injected memories and reports `offline` in the header.
- **Writeback failure** — logged inside the memory client; the chat response
  is unaffected.
- **Memory service unconfigured** — recall and writeback are no-ops; every
  request reports `disabled`.

There is no mode in which a memory outage fails or delays a chat request
beyond the client's own bounded timeouts.

## RPC surface

Two management methods are exposed on the JSON-RPC surface (both require a
JWT; see [JSON-RPC API](../api/jsonrpc.md)):

**`memory.status`** — snapshot of the gateway:

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

`events` is an in-memory ring buffer of recent activity — recall, writeback,
delete and error events, newest first, up to the requested count (the status
handler requests the last 50; the buffer itself is capped at 100). It is
**not** a durable audit log — it resets on restart.

**`memory.delete`** — prune a stored node by id:

```json
{ "node_id": "…" }
```

Returns `{ "deleted": true | false }`. Fails with an error when `node_id` is
missing or when the memory service is not configured.

## Related

- [Configuration](configuration.md) — `ARONA_MEMORY_*` variables.
- [Quickstart](quickstart.md) — end-to-end setup of the gateway.
- [Backends](backends.md) — how chat requests are routed before recall runs.
- [Billing & Usage](billing-usage.md) — how the same chat turns are metered.
- [Operations](operations.md) — logs and health for the memory connection.
- [JSON-RPC API](../api/jsonrpc.md) — `memory.status`, `memory.delete`, `chat.send`.
- [Overview](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
