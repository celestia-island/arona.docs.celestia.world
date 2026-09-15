---
title: "JSON-RPC API Reference"
description: "Arona management-plane JSON-RPC 2.0 API at /api/rpc — chat, realtime, engine, auth, keys, providers, agents, memory, conversations, usage, billing, video and system methods over HTTP and WebSocket."
---

# JSON-RPC API Reference

## Permission map (full RBAC)

Authorization resolves exclusively from **group default permission sets ⊕ grants** (`rbac_groups` / `rbac_user_groups` / `rbac_grants`). The built-in `administrators` group's default set is the entire kirino permission catalog plus the opaque extras (`credits.mint`, `usage.read.all`) — an "admin" passes because those RBAC permissions are forcibly enabled for the group, not via any boolean. `operators` drop the `system` domain / `rbac.manage` / the opaque extras; `registered` keeps the self-service slice (own-account API keys, read/list, provider use).

| Method | Required permission |
|---|---|
| `engine.invoke` | `system.write` |
| `providers.add` / `providers.update` / `providers.remove` / `providers.test` | `provider.create` / `provider.update` / `provider.delete` / `provider.use` |
| `agents.list` / `agents.status` | `agent.read` (management plane: operators + administrators) |
| `agents.register` / `agents.deregister` / `agents.deploy` / `agents.stop` | `deploy.execute` |
| `keys.list` / `keys.create` / `keys.revoke` | `channel.list` / `channel.create` / `channel.delete` — self-service: the three points are in the `registered` baseline and each handler scopes to the caller's own account (attaching a key to a business group additionally needs that group's administrator) |
| `billing.plan` | `config.read` |
| `billing.plan.set`, video pricing set, model aliases write | `config.write` |
| `group.list` / `group.get` | `workspace.list` |
| `group.create` | `workspace.create` |
| group management (update/delete/members/invites/models) | `workspace.manage` (global, or pinned to the group), or the group's own business admin |
| `group.credits.topup` | `credits.mint` (opaque) |
| admin usage query (`/api/admin/usage/query`) | `usage.read.all` (opaque — part of the administrators baseline) |
| REST backends CRUD | `provider.list` / `provider.create` / `provider.delete` |
| REST aliases | `config.read` / `config.write` |
| `ledger.self`, `auth.*`, chat/embeddings | authenticated (self-scope), no permission point |

Denials return `-32007` `Permission required: <permission>` for authenticated callers and the standard auth error for anonymous ones.


Arona exposes a JSON-RPC 2.0 surface at `/api/rpc` for the management plane:
auth, keys, providers, agents, memory, conversations, usage, billing, video,
realtime and streaming chat. It complements the OpenAI-compatible REST
surface (`/v1/*`, see [OpenAI-compatible REST API](./openai-rest.md)); use
REST for key-authenticated inference workloads and JSON-RPC for
session/account management and streaming control. The
[Quickstart](../guides/quickstart.md) walks through the first end-to-end
turn.

The surface dispatches **39 request methods** plus one anonymous
WebSocket-only liveness method, `system.probe` (40 methods total). Every
request is a JSON-RPC 2.0 object with `jsonrpc: "2.0"`, a `method` string,
an optional `params` object and an optional `id`.

## Transport

- **HTTP POST `/api/rpc`** — request/response. Send `Content-Type:
  application/json`. The JWT travels in the `Authorization: Bearer <jwt>`
  header. The request body is capped at 1 MiB.
- **WebSocket `GET /api/rpc`** — long-lived connection. Browsers cannot set
  custom headers on WebSocket upgrade, so the JWT travels as a `?token=<jwt>`
  query parameter; the server folds it into an `Authorization: Bearer` header
  internally (see `packages/core/src/gateway/server.rs`). Authenticated
  sockets may stay connected indefinitely.
- **Batch requests** — a POST body that is a JSON array is executed element
  by element and answered with a JSON array of responses in the same order.
- **Anonymous access** — over WebSocket without a JWT, the public methods
  (`auth.register`/`auth.login`/`auth.refresh`, `providers.list`,
  `system.status`, `Service.Info`) remain callable, and `system.probe` is
  answered with a single ack before the socket closes. Every other method requires a valid
  JWT; the admin-gated methods additionally require an admin account (see
  the legend below). Anonymous sockets are also bound by a 10-second idle
  timeout.
- **Session attachment** — an `x-session-id` header on `POST /api/rpc`
  additionally pushes the RPC response itself onto that session channel,
  alongside the streaming notifications.

## Ids

Request `id` values are echoed with type fidelity: `null` → `null`, strings →
strings, integers → numbers, and anything else (floats, objects, integers
outside the i64 range) → the JSON string rendering. An omitted `id` is
answered with `null`.

## Server → client notifications (SSE sidecar)

Tokens, deploy progress and realtime events are **not** delivered on the
WebSocket socket. Each streaming RPC creates a session id and pushes
notifications to `GET /api/rpc/events?session=<session_id>` as server-sent
events. Subscribe to the SSE endpoint **before or immediately after** the RPC
call returns a session id — notifications emitted between the call returning
and the SSE subscription being established are dropped (the pre-subscription
window). The recommended pattern is to open the SSE stream first, then fire
the RPC.

Notification methods: `chat.stream` (one token per event from `chat.send`),
`models.progress` (agent model download progress from `agents.deploy`),
`realtime.event` (server events for an open realtime session), and
`video.progress` / `video.done` / `video.failed` (async video jobs). See the
full catalog in [Events & Notifications](./events.md).

## Error codes

| Code | Name | Meaning |
| --- | --- | --- |
| `-32700` | Parse error | Request body is not valid JSON. |
| `-32600` | Invalid request | Request object is malformed, e.g. a missing `method`. |
| `-32601` | Method not found | Unknown `method` string; the message echoes it. |
| `-32602` | Invalid params | `params` failed deserialization for the method. |
| `-32603` | Internal error | Unexpected server failure. |
| `-32000` | `APP_ERROR` | Generic application error — e.g. conversation/provider/agent not found, no online agent available for deploy. |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` — missing or invalid JWT. Also used by admin-token methods when the bearer token does not match `ARONA_ADMIN_TOKEN` (`"Admin access required"`). |
| `-32006` | `QUOTA_ERROR` | Monthly billing quota exceeded for a JWT-gated RPC method (`chat.send`, `realtime.start`). |
| `-32007` | `ADMIN_REQUIRED` | Authenticated **non-admin** calling an admin-gated method (`agents.*`, `engine.invoke`, `providers.*` mutations); the message includes a method-specific hint. |

> The `agents.*`, `engine.invoke` and provider-mutation methods are admin-only: they require a
> JWT of an administrators-group member. An authenticated non-admin
> is rejected with `-32007` (`ADMIN_REQUIRED`); an unauthenticated caller
> gets the standard `AUTH_ERROR` so the server does not reveal that the
> method is privileged.

## Auth legend

| Legend | Credentials |
| --- | --- |
| **public** | No credentials required. |
| **JWT** | `Authorization: Bearer <jwt>` on HTTP, or `?token=<jwt>` on WebSocket. |
| **admin (RBAC permission — see the permission map)** | Bearer JWT of a member of the built-in `administrators` group (or the operator service token, which authenticates as the first administrators member). |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN` (env-configured; when unset the method is always denied, default-deny). |

All example credentials and addresses in this document are placeholders
(RFC 5737 documentation IPs, `sk-xxx` keys). See
[Authentication & Security](../guides/auth-security.md) for the full auth
model behind this legend.

## Chat

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model` (string), `messages` (array of `{ role, content, images?, tool_calls? }`), `temperature?` (number), `max_tokens?` (integer), `conversation_id?` (string), `memory?` (bool), `extra?` (object), `tools?` (array of OpenAI-style function definitions), `provider?` (string) | Send a streaming chat turn. Returns `{ "stream_id", "memory" }` — `memory` is the recall state (`enabled` / `disabled` / `offline`); tokens arrive as `chat.stream` notifications on the SSE sidecar. With a `conversation_id`, the completed persisted history is assembled server-side and the turn is persisted. Billing-gated (monthly quota → `-32006`); usage is recorded under `jwt-<user-uuid>`. |

## Realtime (full-duplex audio/video sessions)

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model` (string), `config?` (session config object), `conversation_id?` (string), `ref_id?` (string, ≤ 64 chars) | Open a full-duplex session against the backend serving `model`. Returns `{ "session_id", "stream_session" }`: use `session_id` for `realtime.event` / `realtime.stop`, and subscribe to `stream_session` on the SSE sidecar to receive `realtime.event` notifications. The optional `ref_id` is a usage attribution reference (typically the caller's conversation UUID) recorded on every usage row of the session — with it, rows are written even for zero-token responses (local speech engines report no tokens), queryable via the admin usage query (`POST /api/admin/usage/query`). Billing-gated like `chat.send` (whole-user monthly quota → `-32006`). |
| `realtime.event` | JWT | `session_id` (string), `event` (client event — audio append/commit/clear, image frame, response create/cancel, session stop) | Send one client event into an open session; it is forwarded to the upstream backend. Returns `{ "ok": true }`. |
| `realtime.stop` | JWT | `session_id` (string) | Close and remove a session. Returns `{ "removed": bool }`. |

## Engine (generic perception/control channel)

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `engine.invoke` | admin (RBAC permission — see the permission map) | `model` (string), `method` (string), `params?` (object) | Synchronous request/response invocation of an arbitrary engine method on the backend serving `model` — the high-frequency channel for `sensor.ingest` / `control.setpoint` style calls (20–30 Hz loops). The result is the backend's raw response. |

## Auth

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `auth.register` | public | `email`, `password`, `name?` | Register an account. Only allowed while registration is open (`ARONA_REGISTRATION_OPEN`); the first registered user becomes the admin. Returns the same token response as `auth.login` (`access_token`, `refresh_token`, `token_type`, `expires_in`, `user`). |
| `auth.login` | public | `email`, `password` | Log in. Returns `access_token`, `refresh_token`, `token_type`, `expires_in`, `user` (`{ id, email, name, permissions }` (effective permission names)). Rate-limited per IP and account. |
| `auth.refresh` | public | `refresh_token` | Exchange a refresh token for a fresh access token (and a new refresh token). Reused or expired refresh tokens are rejected with `AUTH_ERROR`. |
| `auth.me` | JWT | — | Current user profile: `{ "id", "email", "name" }`. |

## Keys

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | List the caller's API keys (id, name, `key_prefix`, project, timestamps, active flag). |
| `keys.create` | JWT | `name`, `project?` | Create an API key. Returns `{ id, name, key, key_prefix, project, created_at }` — the full `arona-<uuid>` secret in `key` is shown **once**; store it immediately. |
| `keys.revoke` | JWT | `key_id` | Revoke an API key. Returns `{ "ok": true }`. |

## Providers

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `group.create` | JWT or admin token | `name`, `description?`, `id?` (uuid, idempotent bridge) | Create a group; the creator becomes owner + admin member. Returns the group. |
| `group.list` | JWT or admin token | — | Groups the caller belongs to (id, name, my_role, member_count, allowlist flag). |
| `group.get` | JWT (member) | `group_id` | Group detail + active members. |
| `group.update` | group admin / platform | `group_id`, `name?`, `description?`, `enforce_model_allowlist?`, `rpm_ceiling?` (null clears) | Update group fields. Returns the group. |
| `group.delete` | group owner | `group_id` | Delete the group (group keys detach, memberships cascade). |
| `group.members.list` | group admin / platform | `group_id` | Active members with role, cost multiplier and state. |
| `group.members.add` | group admin / platform | `group_id`, `email`, `role?` (admin\|member), `cost_multiplier?` | Add an existing platform user (upsert). Returns the member. |
| `group.members.update` | group admin / platform | `group_id`, `user_id` (email), `role?`, `cost_multiplier?`, `is_active?` | Update a member; `is_active: false` is the kill switch that instantly invalidates their group keys. The owner cannot be deactivated or removed. |
| `group.members.remove` | group admin / platform | `group_id`, `user_id` (email) | Remove a member (not the owner). |
| `group.invite.create` | group admin / platform | `group_id`, `role?`, `email?`, `max_uses?`, `ttl_secs?` | Create an invitation; the token is visible exactly once. |
| `group.invite.accept` | JWT | `token` | Accept an invitation; joins the group with the granted role. Single-use, expiry- and email-checked. |
| `group.models.list` | group admin / platform | `group_id` | `{ allowed: [...], denied: [...] }` rows of the group allowlist. |
| `group.models.allow` | group admin / platform | `group_id`, `model_ids[]` | Allow models (explicit deny rows win over allow). |
| `group.models.deny` | group admin / platform | `group_id`, `model_ids[]` | Deny models (deny-override). |
| `group.credits.topup` | platform admin | `group_id`, `points`, `note?` | Credit the group points pool. Returns the new balance. |
| `group.credits.balance` | group admin / platform | `group_id` | Pool balance + recent ledger entries. |
| `group.credits.allocate` | group admin / platform | `group_id`, `user_id` (email), `points`, `note?` | Atomically move points from the group pool into a member's personal wallet (refused whole on a short pool). |
| `ledger.self` | JWT or admin token | — | The caller's personal points wallet: balance + recent entries. |
| `providers.list` | **public** | — | List known providers: built-in official entries plus custom ones, as display metadata (`id`, `name`, `description`, `website_domain`, `is_official`, `is_operator`). Public by design — the list carries no credentials; only the mutations below are admin-gated. |
| `providers.add` | admin (RBAC permission — see the permission map) | `id`, `name`, `description?`, `website_domain?` | Add a custom provider entry. Returns `{ "ok": true }`. |
| `providers.update` | admin (RBAC permission — see the permission map) | `provider_id`, `name?`, `description?`, `website_domain?` | Update a custom provider's fields (only the provided ones). Returns `{ "ok": true }`. |
| `providers.remove` | admin (RBAC permission — see the permission map) | `provider_id` | Remove a custom provider. Returns `{ "ok": true }`. |
| `providers.test` | admin (RBAC permission — see the permission map) | — | Test a provider connection. Stub: returns `{ "ok": true, "message": "Provider connection test not yet implemented" }`. |

## Agents

All `agents.*` methods require the `agent.read` (registry reads) / `deploy.execute` (lifecycle) RBAC permissions. Agent nodes connect
outbound over `GET /ws/agent`; this RPC group controls the registry (see
[Agent Cluster](../guides/agent-cluster.md)).

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `agents.list` | admin (RBAC permission — see the permission map) | — | List registered agent nodes: id, name, host, `online`/`offline` status (heartbeat-based), GPU summary, deployed models, version, timestamps. |
| `agents.register` | admin (RBAC permission — see the permission map) | `machine_name`, `version` | Register an agent node with the tunnel manager. Returns `{ "agent_id", "token" }` (the token is the agent's control-plane credential). |
| `agents.deregister` | admin (RBAC permission — see the permission map) | `agent_id` | Deregister (disconnect) an agent. Returns `{ "ok": true }`. |
| `agents.status` | admin (RBAC permission — see the permission map) | `agent_id` | Per-agent status: online flag, host, GPU summary, loaded models, GPU utilization, heartbeat/connection timestamps. |
| `agents.deploy` | admin (RBAC permission — see the permission map) | `model_id`, `agent_id?` (empty/missing = least-loaded node; errors if none online) | Deploy a model on an agent. Returns `{ "ok": true, "stream_id" }` — subscribe to `stream_id` on the SSE sidecar for `models.progress` download notifications. |
| `agents.stop` | admin (RBAC permission — see the permission map) | `agent_id`, `model_id` | Stop a deployed model. Returns `{ "ok": true, "stream_id": null }` (no progress stream). |

## Memory

Long-term memory is served by the entelecheia Philia service over a
WebSocket; failures never block chat (see
[Memory Gateway](../guides/memory-gateway.md)).

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | Memory gateway state: `{ "enabled", "writeback", "events" }` — flags plus up to 50 recent activity events (newest first). |
| `memory.delete` | JWT | `node_id` | Delete a stored memory node. Returns `{ "deleted": bool }`. |

## Conversations

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | List the caller's conversations with relative-age timestamps. |
| `conversations.create` | JWT | `title?` (default `New Conversation`) | Create a conversation. Returns the new conversation object. |
| `conversations.get` | JWT | `conversation_id` (legacy alias: `id`) | Fetch a conversation with its messages. Ownership-checked; cross-user access is rejected. |
| `conversations.delete` | JWT | `conversation_id` (legacy alias: `id`) | Delete a conversation (owner only). Returns `{ "ok": true }`. |

> `conversations.get` / `conversations.delete` also accept the legacy `id`
> key from older dashboard clients; `conversation_id` wins when both are
> present.

## Usage

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?` (integer, default 50, clamped to 1–200), `offset?` (integer, default 0), `project?` (string) | Paginated usage records for the caller, newest first, covering both API-key rows (`arona-XX` prefix) and JWT-attributed rows (`jwt-<user-uuid>`). Returns `{ "records", "total", "limit", "offset", "project" }`; the `project` filter narrows to key-tagged rows only. Every record includes its `ref_id` field, which is `null` unless the request was tagged with an attribution reference (`x-celestia-ref` header / `realtime.start` `ref_id`). |

## Billing

Tiers, quotas and usage accounting are described in
[Billing & Usage](../guides/billing-usage.md).

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | Current billing state: `{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` — monthly usage (`cost_usd`, tokens, request count) and remaining quota. |
| `billing.plan.set` | admin token | `user_email`, `tier` | Set a user's billing tier. Returns `{ "ok": true }`. Denied with `AUTH_ERROR` when the bearer does not match `ARONA_ADMIN_TOKEN`. |
| `billing.video.pricing.get` | JWT | — | Video pricing table. Returns `{ "pricing": [...] }`. |
| `billing.video.pricing.set` | admin token | `model`, `mode?` (default `per_second_resolution`), `base_price?` (number, default 0), `price_per_second?` (number, default 0), `price_per_frame?` (number, default 0), `resolution_coeff?` (object), `currency?` (default `USD`), `enabled?` (bool, default `true`) | Upsert video pricing for a model. Returns `{ "ok": true }`. Denied with `AUTH_ERROR` when the bearer does not match `ARONA_ADMIN_TOKEN`. |

## Video

Async video generation jobs (see
[Realtime & Video](../guides/realtime-video.md)). Job progress is pushed as
`video.progress` / `video.done` / `video.failed` notifications on the session
channel.

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`, `prompt`, `negative_prompt?`, `images?` (array of `{ data_base64, mime_type }`), `duration_seconds?` (integer), `width?` (integer), `height?` (integer), `provider?` (string), `extra?` (object) | Submit an async video generation job. Returns `{ "job_id", "stream_id" }` — subscribe to `stream_id` for progress notifications. |
| `video.get` | JWT | `job_id` (UUID) | Poll a job's status/result (status, progress, result, error, cost). |
| `video.list` | JWT | `limit?` (integer, default 20) | List the caller's jobs. Returns `{ "jobs": [...] }`. |
| `video.cancel` | JWT | `job_id` (UUID) | Cancel a running job. Returns `{ "ok": true }`. |

## System

| Method | Auth | Params | Description |
| --- | --- | --- | --- |
| `system.status` | public | — | Aggregate gateway status: `{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds", "version", "build_hash", "kind", "engine_version" }` — the last four are the shared version report, added without changing any existing key. |
| `Service.Info` | public | — | The `VersionReport` every health route reports: `version`, `build_hash`, `kind` and an optional `engine_version` that arona omits. |
| `system.probe` | anonymous (WS only) | — | One-shot liveness probe over the WebSocket transport. The server acks `{ "ok": true, "status": "ok" }` and then closes the socket — anonymous visitors never hold an open connection. Any other method on an unauthenticated socket is rejected with `AUTH_ERROR`. |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
