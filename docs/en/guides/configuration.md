---
title: "Configuration"
description: "Reference for every environment variable read by the Arona server, with defaults and semantics."
---

# Configuration

Arona is configured **entirely through environment variables**, read once at
process startup (`Config::load` in `packages/core/src/config.rs`, plus a few
read at first use). There is no config file: change a variable and restart
the server for it to take effect.

This page is the reference for everything the server code reads, grouped by
concern. Mock-only and runtime variables are included for completeness.

## Reference table

| Variable | Default | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | *(required)* | PostgreSQL connection URL. |
| `JWT_SECRET` | *(required outside mock mode)* | Secret used to sign JWTs. |
| `ARONA_HOST` | `0.0.0.0` | Bind address (falls back to `SHITTIM_CHEST_HOST`). |
| `ARONA_PORT` | `8420` | Bind port (falls back to `SHITTIM_CHEST_PORT`). |
| `ARONA_DATA_DIR` | unset | Local data directory. |
| `ARONA_ADMIN_TOKEN` | unset | Bearer token for `/api/admin/*` and the admin RPC methods. |
| `ARONA_REGISTRATION_OPEN` | `0` | Truthy (`1`/`true`/`yes`/`on`) opens sign-up. |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | Per-key in-memory request limit per minute; `0` blocks everything. |
| `MOCK_MODE` | unset | Presence (any value) enables dev mock mode. |
| `MOCK_SEED_PATH` | unset | Raw SQL seed file executed in mock mode. |
| `ARONA_MEMORY_URL` | unset | Philia memory gateway WebSocket URL. |
| `ARONA_MEMORY_TOKEN` | unset | Token for the memory gateway. |
| `ARONA_MEMORY_WRITEBACK` | `true` | Write completed chat turns back to memory; accepts `true`/`false` (any other value falls back to the default). |
| `ARONA_AGENT_NAME` | `arona-agent` | GPU-node agent identity. |
| `ARONA_PANEL_URL` | `localhost:8080` | Where the agent connects (`ws://<panel_url>/ws/agent`). |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | Local evernight agent for `evernight://` backend URLs. |
| `ARONA_MISTRALRS` | unset | Presence forces the Mistral.rs engine for Gguf model plans. |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | Interface a spawned llama.cpp model server binds to. |
| `HF_ENDPOINT` | `https://huggingface.co` | Hugging Face base URL for model downloads. |
| `GITHUB_TOKEN` | unset | Access token for the GitHub model registry. |
| `RUST_LOG` | unset | Tracing filter, e.g. `info` or `arona=debug,info`. |

## Required variables

### `DATABASE_URL`

PostgreSQL connection URL. **Required**: the server exits with
`FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.` when it
is empty, and the `migrate` CLI subcommand refuses to run. The schema is
created/updated automatically by `serve` on startup.

### `JWT_SECRET`

Secret used to sign the access/refresh JWT pairs issued by `auth.login` and
`auth.register`. **Required in production**: the code embeds a development
fallback (`dev-secret-not-for-production-use-only-32chars`), but the server
refuses to start with it unless `MOCK_MODE=1`:

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

Use a long, random value (e.g. `openssl rand -hex 32`).

## Server

### `ARONA_HOST` / `ARONA_PORT`

Bind address and port for the gateway. For legacy compatibility they fall
back to `SHITTIM_CHEST_HOST` / `SHITTIM_CHEST_PORT`; the ultimate defaults
are `0.0.0.0:8420`.

### `ARONA_DATA_DIR`

Optional local data directory, carried on the app state for components that
need a scratch location. Unset by default.

## Security & access control

### `ARONA_ADMIN_TOKEN`

Bearer token guarding the `/api/admin/*` HTTP routes (`POST/GET/DELETE
/api/admin/backends`, `/api/admin/aliases`) and the
`billing.plan.set` / `billing.video.pricing.set` RPC methods. When it is
**unset**, every one of those routes rejects with
`Admin access required` (401) — there is no default. Set it to a strong
random value before starting the server.

### `ARONA_REGISTRATION_OPEN`

Opens self-service sign-up through `auth.register`. Truthy values are
exactly `1`, `true`, `yes`, `on` (case-insensitive); anything else —
including `0`, `false`, `off`, or an unset/empty variable — stays closed.
The flag is read once at startup. The **first registered user is always
allowed** (even when registration is closed) and becomes the admin.

### `ARONA_API_RATE_LIMIT_RPM`

Per-key in-memory sliding-window rate limit (requests per minute), applied
to every authenticated `/v1/*` request (chat, embeddings, video, models),
keyed by the API-key hash (or the `u:<email>` label for the JWT-accepting
`GET /v1/models`). The RPC plane is not rate-limited by this limiter — the
`/v1/*` auth extractors are the only callers. Default `60`. The value is
parsed once into a process-wide `OnceLock`. **A value of `0` blocks every
request** — the check is `entry.len() >= rpm`, so with `0` no request can
pass. This is the gateway-wide limit; the billing tiers impose their own
per-key limits on top.

## Development

### `MOCK_MODE`

Dev mock mode, enabled by **presence** — the check is
`std::env::var("MOCK_MODE").is_ok()`, so *any* value (including `0` or an
empty-but-set value) enables it. It:

- lifts the `JWT_SECRET` requirement (the built-in dev secret becomes
  acceptable);
- seeds four demo accounts (`demiurge@celestia.world`, `momoi@celestia.world`,
  `midori@celestia.world`, `yuzu@celestia.world`, password `33550336`);
- waits for the seed to complete before binding the listener.

Never use it in production.

### `MOCK_SEED_PATH`

In mock mode only, points at a raw SQL file executed instead of the built-in
account seed (statements split on `;`, `--` comments skipped). If the file
cannot be read the built-in seed is used as a fallback.

## Memory gateway

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

Configuration for the long-term memory gateway (entelecheia Philia). Memory
is **disabled entirely** unless both `ARONA_MEMORY_URL` and
`ARONA_MEMORY_TOKEN` are set and non-empty. When enabled:

- completed chat turns are recalled and injected as context, and
- `ARONA_MEMORY_WRITEBACK` (default `true`) controls whether finished turns
  are written back to the memory service; `0` or `false` disables writeback.

Memory failures never block chat; the resulting state is echoed in the
`X-Arona-Memory` response header (`enabled` / `disabled` / `offline`).

## Agent identity & cluster

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

Identity of the GPU-node agent binary (`_agent`): `ARONA_AGENT_NAME`
(default `arona-agent`) is reported to the panel as the agent's name/id, and
`ARONA_PANEL_URL` (default `localhost:8080`) is where the agent connects
(`ws://<panel_url>/ws/agent`).

The agent's own HTTP API is **hardcoded** to bind `0.0.0.0:5790` — there is
no bind-address environment variable for it.

### `ARONA_AGENT_BIND_ADDR`

Interface that a **spawned llama.cpp model server** binds to when the agent
deploys a Gguf model, so the engine can be reached from other machines
(e.g. `0.0.0.0`). Defaults to `127.0.0.1`. Note this is *not* the agent HTTP
API bind (which is fixed at `0.0.0.0:5790`).

## Evernight bridge

### `ARONA_EVERNIGHT_URL`

WebSocket URL of the local evernight agent used to resolve `evernight://`
backend URLs into local TCP forwards. Default `ws://127.0.0.1:3001/ws`.

## Model runtime & downloads

### `ARONA_MISTRALRS`

Presence (any value) forces the Mistral.rs engine for Gguf model plans that
would otherwise default to llama.cpp. Presence semantics like `MOCK_MODE`.

### `HF_ENDPOINT`

Base URL for Hugging Face model downloads (`hf:` sources), default
`https://huggingface.co`. Set it to a mirror such as
`https://hf-mirror.com` when huggingface.co is unreachable. Read by the
model downloader; a trailing slash is trimmed.

### `GITHUB_TOKEN`

Access token used by the GitHub model registry (`gh:` sources) for API
access. Unset by default; without it GitHub API rate limits apply.

## Logging

### `RUST_LOG`

Standard tracing filter applied by `tracing_subscriber` at startup, e.g.
`info` or `arona=debug,info`. Follows the usual `RUST_LOG` semantics
(`error`/`warn`/`info`/`debug`/`trace`, per-target overrides).

## Defaults at a glance

| Setting | Default |
| --- | --- |
| Bind address / port | `0.0.0.0:8420` |
| Per-key API rate limit | 60 RPM |
| Agent name | `arona-agent` |
| Panel URL | `localhost:8080` |
| Memory writeback | on |
| Registration | closed |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
