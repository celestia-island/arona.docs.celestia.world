---
title: "Backends"
description: "Backend types (external, ollama, engine, minimax-cloud, evernight bridges), URL semantics, health probing, model discovery, aliases and routing."
---

# Backends

A **backend** is an upstream that serves model traffic. Arona routes
OpenAI-compatible requests (`/v1/chat/completions`, `/v1/embeddings`, model
listing, video jobs) to one of the registered backends, meters every request,
and keeps each backend's health and model inventory fresh.

Backends are registered by an admin through
`POST /api/admin/backends` (see the [Admin HTTP API](../api/admin-http.md)),
persisted to the `backend_configs` table, and restored automatically at
startup. Each registration carries a `name`, a `type`, a `url`, an optional
`api_key` and an optional static `models` list. Persisted backends survive
restarts; restored backends start fail-closed and are probed immediately.

## Backend types

| `type` | Transport | Protocol | Purpose |
| --- | --- | --- | --- |
| `external` | HTTP(S) | OpenAI-compatible REST | Any chat/embeddings API (cloud or self-hosted) |
| `ollama` | HTTP(S) | Ollama native API (`/api/chat`, `/api/embed`, `/api/tags`) | A local or remote Ollama server; built from the URL alone |
| `engine` | `ws://` / `wss://` | CEP (Celestia Engine Protocol), WebSocket + JSON-RPC | Any engine speaking the CEP interchange standard (`plana::engine`) |
| `minimax-cloud` | HTTPS | MiniMax H3 task-style API (submit + poll) | Cloud video generation |
| `evernight://<node>/<service>` | bridge URL | Resolved through the local evernight agent into a local TCP forward | Industrial/edge services reachable only over the evernight mesh |
| `agent-{model}` | HTTP | OpenAI-compatible (external) | Auto-registered when a GPU agent deploys a model |

### `external` — any OpenAI-compatible HTTP API

The general-purpose backend: chat completions (streaming and non-streaming)
and embeddings against any server that speaks the OpenAI REST shape. Configure
it with a base `url`, an `api_key` (optional) and an optional static `models`
list:

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

The static `models` list is authoritative: it is merged ahead of any models
discovered at probe time (see [Model discovery](#model-discovery)).

### `ollama` — built from the URL alone

An Ollama backend is constructed from the URL alone — no API key, no model
list. It speaks Ollama's native protocols: `/api/chat` for chat, `/api/embed`
for embeddings, and `/api/tags` for health probing and model discovery.

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — CEP over WebSocket

A `engine` backend connects to an engine exposing `ws://` (or `wss://`) and
speaks the **Celestia Engine Protocol** (CEP): a WebSocket + JSON-RPC 2.0
interchange standard defined in `plana::engine`. Any engine written in any
language that implements the handshake → methods → streaming-notifications
flow registers as a first-class backend with zero adapter code in Arona.
Wire methods: `Engine.Handshake` (first message; identity + capabilities),
`Engine.Chat`, `Engine.ChatStart` (streaming; chunks arrive as
`Engine.ChatChunk` notifications), `Engine.Embeddings` and `Engine.Models`.
Connections are established lazily on first use and dropped on any error; the
next call reconnects and re-handshakes.

### `minimax-cloud` — task-style video generation

The cloud video backend drives the MiniMax H3 Open Platform API: submit a
generation task, poll for completion, then read the artifact URL from the
result. This is what replaced the removed ComfyUI backend (see below); video
jobs are submitted through `/v1/video/generations` or the `video.*` RPC
methods and progress through `video.progress` / `video.done` / `video.failed`
notifications (see [Realtime & Video](realtime-video.md)).

### `evernight://` bridge URLs

A backend URL of the form `evernight://<node>/<service>` is **not** contacted
directly. The local host evernight agent resolves it (a `Bridge.Connect`
JSON-RPC call over the agent's WebSocket endpoint) into a local TCP forward,
and the backend runs against `http://127.0.0.1:<local_port>` instead of a
hardcoded remote address. This is the single-panel architecture: the Arona
panel reaches services on other nodes (CEP engines, scepter, ...) through the
mesh without ever embedding a remote address in a config.

A keepalive task probes the tunnel every 15 seconds; when the remote side
restarts and the tunnel is re-established on a new local port, the affected
backend is **transparently rebuilt** with the new URL — the persisted config
keeps the `evernight://` URL so restarts re-resolve it. For `engine` backends
the resolved `http://127.0.0.1:<port>` forward is adapted to `ws://` for the
WebSocket transport.

### Agent-deployed models auto-register

When a GPU agent finishes deploying a model, the gateway registers a backend
named `agent-{model_id}` (an `ExternalApiBackend` over `http://{agent host}:{port}`)
so the model becomes routable immediately; stopping the deployment
unregisters it again. See [Agent Cluster](agent-cluster.md) for the full
deployment lifecycle.

### `comfyui` is rejected

The `comfyui` backend type is explicitly refused with the error
`comfyui backend removed`: the ComfyUI backend was removed during the
model-platform convergence, and video generation now runs through
`minimax-cloud`. Registering a `comfyui` backend returns an HTTP 400.

## URL semantics

How a configured base URL maps to actual endpoints is decided by whether the
URL has a path component:

- **Root-style base** — a URL whose path is empty or `/` is treated as a host
  root and keeps the OpenAI `/v1` convention: `{base}/v1/chat/completions`,
  `{base}/v1/models`. Examples: `http://192.0.2.20:8429`,
  `https://api.deepseek.com`.
- **Path-style base** — a URL with a non-empty path is treated as the full API
  prefix the server actually serves, and the endpoint is appended directly:
  `{base}/chat/completions`, `{base}/models`. This is what OpenAI-compatible
  servers outside the `/v1` convention need. The Zhipu GLM coding plan is the
  canonical example: its API lives at
  `https://open.bigmodel.cn/api/coding/paas/v4` with chat directly at
  `{base}/chat/completions` and **no `/models` endpoint at all** — the
  standard `/api/paas/v4` root returns balance errors for coding-plan keys.
- A **trailing slash** on the configured base URL is normalized away so the
  join never produces a double slash.

## Probing & health

A background health checker probes every registered backend every **60
seconds**; the backend list is fetched fresh on each round, so backends
registered after startup are picked up without a restart. Each admin
registration also fires an immediate probe so the backend flips healthy within
~1–2 seconds instead of waiting for the next checker round.

- **External backends** probe `GET {base}/models` (or `{base}/v1/models` for
  root-style bases) with a **2-second timeout**. A **404 is tolerated**: some
  servers implement chat but expose no model listing (the GLM coding plan has
  no `/models` endpoint), so a 404 marks the backend healthy and the
  admin-configured `models` list becomes the routing source. Timeouts, network
  failures and other non-2xx responses mark the backend unhealthy.
- **Ollama backends** probe `/api/tags` with the same 2-second timeout.
- Backends start **fail-closed** — reported as `not probed yet` — until the
  first successful probe, so a freshly registered (or restored) backend never
  receives traffic before it has been verified.

Health state is cached per backend and consulted by the router on every
request; unhealthy backends are excluded from candidate selection (see
[Routing](#routing)).

## Model discovery

A backend advertises the model ids it serves, and the router matches requests
against that advertisement:

- **External** backends advertise the models parsed from the probe response
  (both a `data` and a `models` array are accepted), merged with the
  admin-configured static list — static ids keep order and precedence,
  dynamic ids are deduplicated and appended. When a server has no models
  endpoint, the static list alone is the routing source.
- **Ollama** backends advertise the tags returned by `/api/tags`.
- **Agent-deployed** models advertise exactly the deployed `model_id`.

The public surface is `GET /v1/models` (authenticated), which lists the
routable models of every healthy backend (see the
[OpenAI-compatible REST API](../api/openai-rest.md)).

## Aliases & name normalization

Aliases map a requested model id to a target id. The alias is resolved first
in routing, so a request for the alias is served by whatever backend
advertises the target:

```json
{ "alias": "fast-chat", "target": "deepseek-chat" }
```

Aliases are managed through the `/api/admin/aliases` admin endpoints and take
effect immediately.

Name matching also normalizes Ollama-style tags: a backend listing
`nomic-embed-text:latest` matches a bare request for `nomic-embed-text`, so
embedding/chat requests resolve without `:latest` suffix bookkeeping. An
explicit tag (`qwen3:0.6b`) matches only that exact tag.

## Routing

Every request resolves through the router, which selects one backend:

1. **Alias resolution** — the requested model id is mapped through the alias
   table (if any).
2. **Provider hint** — an optional `provider` field filters candidates by
   backend name (or kind name, e.g. `cloud` for video backends).
3. **Healthy candidates only** — a backend must report `Healthy` *and* pass
   its circuit breaker (3 consecutive failures open the breaker for 30
   seconds, with one half-open test call) to be selectable.
4. **Least-count pick** — candidates serving the model are sorted by their
   per-backend request counter and the least-loaded one is chosen. This
   spreads load across healthy backends serving the same model.
5. **Session affinity** — when a request carries a `conversation_id`, the
   conversation is pinned to the backend it first used. The pin lives in a
   `Weak`-reference map, so a removed backend vanishes from the map without
   index drift. Affinity is best-effort: reusing the same backend across the
   turns of a conversation lets the upstream reuse per-conversation runtime
   state (warm contexts, KV caches). If the pinned backend has become
   unhealthy (or the pin has died with a removed backend), the router falls
   back to a fresh least-count selection and **re-binds** the conversation.

If no healthy backend serves the model, routing fails: an unknown model yields
`model not found` (HTTP 404), a known-but-unreachable model yields `all
backends unhealthy`, which is surfaced as a 500 internal server error. HTTP
502 is reserved for failures reported by a *reachable* upstream (non-2xx
upstream responses and transport failures after selection). See
[Operations](operations.md) for the full error mapping.

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
