---
title: "Architecture"
description: "How Arona is put together — workspace layout, the request path through the gateway, routing, health probing, memory, sessions and the deliberate design tradeoffs."
---

# Architecture

This page walks through how Arona is structured and how a request flows
through it: the workspace layout, the request path, the gateway and router,
health checking, the memory client, sessions and notifications, and finally
the deliberate limits and tradeoffs the design accepts. See
[quickstart](quickstart.md) for a running example and
[operations](operations.md) for day-to-day runtime concerns.

## Workspace layout

The repository is a Cargo workspace with three packages:

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC,
│            # backends, migration, entities, providers, models, registry,
│            # orchestration
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

- `packages/core` is the library crate (`_core`). It contains everything the
  server needs: the axum gateway (`gateway/`), the model router (`routing/`),
  the backend adapters (`backends/`), billing (`billing/`), auth (`auth.rs`),
  the memory client (`memory/`), the JSON-RPC plane (`gateway/rpc.rs`), the
  schema (`migration/`, `entity/`), model metadata (`models/`, `providers/`,
  `registry/`) and model orchestration (`orchestration/`).
- `packages/agent` builds the `_agent` binary that runs on GPU nodes and
  connects back over `/ws/agent` (see [agent-cluster](agent-cluster.md)).
- `packages/cli` builds the `_cli` binary used for install, deploy, serve,
  migrate and download operations.

There is no web dashboard in this repository anymore: the Vue dashboard was
removed and now lives in
[shittim-chest](https://github.com/celestia-island/shittim-chest) (chest #291),
which talks to Arona over the JSON-RPC surface. Arona itself is a pure backend
(see the [overview](./README.md)).

## Request path

The entry point is the axum router assembled in `GatewayServer::app`
(`packages/core/src/gateway/server.rs`). Its route table (server.rs:128-163)
covers the OpenAI-compatible REST surface (`/v1/chat/completions`,
`/v1/embeddings`, `/v1/models`, `/v1/health`), video generation, the
`/api/rpc` JSON-RPC endpoint (POST + WebSocket upgrade), the SSE sidecar
`/api/rpc/events`, the agent control plane `/ws/agent`, Swagger UI at
`/docs`, and the admin backend/alias management endpoints.

The router is wrapped in a small stack of layers (server.rs:158-162):

1. Auth managers as `Extension`s so the per-handler extractors can reach them.
2. A request-id layer that reuses an inbound `X-Request-ID` header or generates
   one, exposing it to handlers and logs (`gateway/request_id.rs`).
3. A 1 MB request-body limit (`RequestBodyLimitLayer`).
4. A permissive CORS layer (`*` origin, `*` headers).

Because axum applies layers bottom-up, the CORS layer is outermost.

Every `/v1/*` handler then runs through the same skeleton:

1. **Auth extraction** — `ApiKeyAuth` for the key-only endpoints
   (`/v1/chat/completions`, `/v1/embeddings`, video) and `ApiKeyOrJwt` for
   `GET /v1/models`, which must accept both API keys and session JWTs
   (`gateway/middleware.rs`). The extractor resolves the key/JWT to a user
   email, key prefix, a rate-limit key (the SHA-256 hash of the API key, or a
   `u:<email>` label for JWTs so rotating tokens do not reset the window) and
   an optional project scope.
2. **Billing gates** — `enforce_billing_gates` (server.rs:492-539) rejects the
   request with HTTP 429 + `Retry-After` when the user's tier monthly quota or
   per-minute rate limit is exceeded. DB failures fail open: billing is
   best-effort, never a hard dependency of serving a chat.
3. **Memory recall** (chat paths) — if the memory client is configured and the
   request asks for it, relevant long-term memories are injected as a system
   section (see [Memory client](#memory-client) below). Failure never blocks
   the chat; the resulting state is echoed in the `X-Arona-Memory` header.
4. **Conversation persistence** — an optional `conversation_id` is
   ownership-checked, and the user turn is persisted at send time.
5. **Gateway dispatch** — the request is handed to the `Gateway`, which
   resolves a backend, trims the context, and calls the backend trait.
6. **Usage recording** — the returned (or stream-terminal) usage is recorded
   and persisted through the `UsageTracker` under the key prefix.

The `Gateway` itself lives in `AppState` as an `Arc<Gateway>` — there is no
outer mutex; interior mutability keeps concurrent chat/embeddings/stream calls
from ever holding a lock across an upstream HTTP round trip
(`gateway/mod.rs:29-53`).

## Gateway & router

The `Gateway` (`packages/core/src/gateway/mod.rs`) owns:

- **Router state** — the backend list and aliases, guarded by a
  `tokio::sync::RwLock`. Read-side resolution borrows across awaits; mutations
  (register/remove/alias) take a short write lock and never hold it across an
  upstream call.
- **A request counter** (`AtomicU64`) and a `start_time` used by
  `system.status` and the health endpoints.
- **A deployments map** (`model_id → backend name`) for agent-deployed models.
  `register_agent_backend` builds an `ExternalApiBackend` named
  `agent-{model_id}` and inserts it into the router; re-registering the same
  model replaces the previous backend, and `unregister_agent_backend` removes
  it on a `stop_result` frame (see [agent-cluster](agent-cluster.md)).

Backend resolution happens in the `Router` (`packages/core/src/routing/mod.rs`):

1. **Alias resolution** — a configured alias is rewritten to its target.
2. **Session affinity** — when a `conversation_id` is present, the router
   checks a weak-reference map that pins the conversation to the backend it
   was first served by. Weak references keep the map alive only as long as the
   backend is registered or in flight, so removed backends vanish without
   index drift. A tripped circuit breaker or unhealthy pinned backend degrades
   into a fresh selection, which re-binds the conversation.
3. **Candidate filtering** — an optional `provider` hint filters by backend
   name/kind; candidates must be healthy *and* have an open circuit breaker,
   and must list the requested model. Model ids match exactly or through the
   `:latest` suffix convention (a bare `nomic-embed-text` request matches a
   listed `nomic-embed-text:latest`).
4. **Least-loaded pick** — surviving candidates are sorted by their hit
   counter and the least-loaded one is chosen; the conversation pin (if any)
   is recorded at the same time.

Before the backend is called, `RequestPipeline::transform`
(`packages/core/src/pipeline.rs:422+`) trims the message list to the backend's
`max_context_length`: system messages are kept in full, non-system messages
are kept newest-first while they fit, and a single oversized message is
hard-truncated by characters (the heuristic token counter cannot truncate
token-precisely). The call then goes through the `InferenceBackend` trait;
successes and failures are recorded back into the router's per-backend circuit
breaker (3 failures, 30s recovery, 1 half-open call — routing/mod.rs:57-64).

## Health checker & probing

`run_health_checks` (`packages/core/src/gateway/health_checker.rs`) runs as a
background task spawned at startup (run.rs:135-150) and probes every
registered backend once per 60-second interval. Two details matter:

- The backend list is **fetched fresh on every round** through an async
  fetcher closure, so backends registered after startup (e.g. via the admin
  API) are picked up without a restart.
- The first round runs immediately, before the first interval elapses, so
  health state is established as soon as the process starts.

`probe_backend` is the single probe code path. It is reused by the one-off
**register-time probes**: after an admin registers a backend (server.rs:688-693)
or a persisted backend is restored at boot (run.rs:122-127), a fire-and-forget
probe flips the backend healthy within ~1–2s instead of staying fail-closed
until the next 60s round. This is what makes a freshly registered external
backend's model list appear in `GET /v1/models` almost immediately.

The probe itself is a lightweight backend call (e.g. the external backend hits
`/v1/models` with a 2s probe timeout); the result is cached in the backend and
routing only ever selects backends whose cached health is `Healthy` (plus an
open circuit breaker).

## Memory client

The memory client (`packages/core/src/memory/mod.rs`) is constructed from
environment configuration at server startup (server.rs:95-97): when
`ARONA_MEMORY_URL` and `ARONA_MEMORY_TOKEN` are set, chat requests query the
entelecheia Philia memory service over a JSON-RPC WebSocket and
`recall_and_inject` prepends the relevant memories as a system section
(`## Relevant Long-Term Memories`) into the outbound context. Completed turns
are written back as episodes via `writeback_dialogue` — fire-and-forget work
spawned after the assistant reply is persisted, so memory failures never block
or slow the chat response path. `ARONA_MEMORY_WRITEBACK` (default on) toggles
writeback. See [memory-gateway](memory-gateway.md) for the full picture.

Every chat response carries an `X-Arona-Memory` header with one of three
states: `enabled` (recall ran and injected), `disabled` (not configured or the
request passed `memory: false`), or `offline` (configured but the service was
unreachable).

## Sessions & notifications

`AppState` holds a `plana` `SessionManager` (`state.sessions`). Streaming RPCs
such as `chat.send` create a session id (`gateway/rpc.rs:1701`) and push
notifications — `chat.stream` tokens, `models.progress` deploy progress,
`realtime.event` — onto that session's channel. Clients consume them from the
SSE sidecar `GET /api/rpc/events?session=<id>` (server.rs:191-200); see
[events](../api/events.md) for the notification format and the
pre-subscription window caveat.

The session channel is also used for request/response RPC calls: when a client
sends an `x-session-id` header on `POST /api/rpc`, the server pushes the
result onto that session channel as well (server.rs:184-188, rpc.rs:128-144),
so a client can multiplex an RPC response onto an already-open SSE stream.

## Limits & design tradeoffs

The design deliberately accepts a number of limits; know them before
production use:

- **1 MB request body limit** — larger bodies are rejected by the layer; if
  you need big-context calls, this is the first thing to tune.
- **CORS `*`** — the gateway answers cross-origin calls from anywhere. Fine
  for an API, but if you expose it beyond trusted clients, front it with a
  proxy that enforces your own CORS policy.
- **Fail-open billing** — quota/rate-limit enforcement degrades to allowing
  the request when the DB is unavailable. Billing is metering, not access
  control.
- **No overall timeout on SSE streams** — streaming calls carry no total
  deadline (long generations are legal); hang detection relies on a 120s
  per-read idle timeout (`backends/external.rs:24-31`). Non-streaming calls
  get a 600s overall deadline.
- **Tokenizer-estimate usage** — backends that never report usage (ollama,
  ws_engine) are billed from a local CJK-aware tokenizer estimate, recorded
  as-is (see [billing-usage](billing-usage.md)).
- **In-memory rate-limit windows and revocation** — the per-key sliding
  window and the revoked-key set live in process memory (`auth.rs`), so a
  restart resets them. The auth-level limiter bounds requests per key per
  window; the billing-tier limiter is DB-backed (see
  [auth-security](auth-security.md) and [billing-usage](billing-usage.md)).
- **`/ws/agent` is unauthenticated** — the agent control plane accepts any
  WebSocket that speaks the register/heartbeat protocol. It is safe only on a
  network you control.
- **No TLS at the gateway** — the server binds plain HTTP; terminate TLS in
  front (reverse proxy) in any deployment that crosses a network boundary.
  See [deployment](deployment.md).

On the graceful side, the server performs a graceful shutdown: it installs
Ctrl+C and SIGTERM handlers, logs "draining connections", and lets in-flight
requests finish before the process exits (`gateway/run.rs:14-38`, and the
`with_graceful_shutdown` wiring at run.rs:154-159).

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table); gateway/mod.rs:29-53; routing/mod.rs:102-200; health_checker.rs:14-37; run.rs:135-159; memory/mod.rs:207-241; rpc.rs:128-144,1701 -->
