---
title: "Quickstart"
description: "End-to-end Arona walkthrough with the built-in mock upstream: migrate, serve, register a backend, create an API key and chat."
---

# Quickstart

This guide walks you through a complete end-to-end Arona setup on a single
machine using the **built-in mock upstream** — no real model weights, GPU or
external API account required. By the end you will have:

- a running Arona gateway (`/v1/*` OpenAI-compatible REST API plus the
  `/api/rpc` JSON-RPC management plane),
- the mock upstream registered as an `external` backend,
- a user account and an API key,
- a working non-streaming **and** streaming chat turn against the mock,
- usage records visible through `usage.list`.

## Prerequisites

- **Rust toolchain** (see `rust-toolchain.toml` at the repository root).
- **Python 3** with `aiohttp` — only needed for the mock upstream
  (`pip install aiohttp`).
- A **running PostgreSQL** instance and the connection URL for it.

## 1. Set the environment

Arona reads its configuration from environment variables **at process
startup**. Two are mandatory: `DATABASE_URL` and `JWT_SECRET` — the server
refuses to start without them (unless `MOCK_MODE=1`). `ARONA_ADMIN_TOKEN` is
strongly recommended: without it, every `/api/admin/*` route returns 401.

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

These variables are read once when the process starts — if you change them,
restart the server. See [Configuration](configuration.md) for the full
variable reference.

## 2. Migrate and start the server

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

`serve` alone is enough for a fresh database: it auto-migrates on startup.
The server binds `0.0.0.0:8420` by default (override with `ARONA_HOST` /
`ARONA_PORT`).

## 3. Start the mock upstream

In a second terminal:

```bash
python3 scripts/mock/server.py
```

The mock is an aiohttp server that listens on `127.0.0.1:8429` by default
(`ARONA_MOCK_PORT` overrides the port). It prints its API key on startup and
also serves `GET /api/test-key`, which returns
`{"api_key": ..., "base_url": ...}`. It exposes a handful of model ids —
including `gpt-5.5`, used below — and answers both plain and streaming chat
completions.

Capture the printed key:

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. Register the mock as an external backend

Backends are registered through the admin HTTP API:

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

The backend is probed immediately on registration and flips healthy within
~1-2 seconds; until that probe completes it stays in a fail-closed "not
probed yet" state (see the troubleshooting box below). The configuration is
persisted, so the backend survives a restart.

## 5. Register an account and log in

Accounts live on the JSON-RPC plane, `POST /api/rpc`. Because
`ARONA_REGISTRATION_OPEN=1` is set, `auth.register` is open; the first
registered user becomes the admin.

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

Passwords must be at least 8 characters **and** contain at least 3 of the 4
character categories (uppercase, lowercase, digit, special). Then log in to
obtain the JWT pair:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

Export the `access_token` from the response:

```bash
export JWT="<access_token from the login response>"
```

## 6. Create an API key

`keys.create` is JWT-authenticated and returns the **full** `arona-{uuid}`
secret exactly once — the database only stores its SHA-256 hash, so copy it
now:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. Chat (non-streaming)

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

You get back an OpenAI-style completion object with a `choices[0].message`
and a `usage` block.

## 8. Chat (streaming)

The same endpoint with `"stream": true` answers with server-sent events:
one `data:` chunk per token, terminated by a final `data: [DONE]` chunk:

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. Verify usage

Every chat turn records a usage row under the key's prefix. Query it with
the JWT:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

You should see one or more records for the `gpt-5.5` requests made above.

## Troubleshooting

- **`No backend available for model: gpt-5.5` (HTTP 404, `code:
  model_not_found`)** — no registered backend serves that model id. Either
  the backend was never registered (or its `models` list does not include
  the id), or the registration call failed. Check with
  `GET /api/admin/backends` (admin token).
- **`all backends unhealthy` (HTTP 500, `backend_error`)** — a backend *is*
  registered for the model but no candidate is healthy. A freshly registered
  external backend starts in a fail-closed "not probed yet" state and flips
  healthy once the register-time probe completes, ~1-2 s later; if you chat
  inside that window you hit this error. Retry after a moment, or check the
  mock is actually running on `127.0.0.1:8429`.
- **HTTP 401 on `/v1/*`** — a missing `Authorization` header yields
  `Missing Authorization header. Use: Bearer <api_key>`; an unknown key
  yields `Invalid API key`. Double-check `$AR_KEY` (full secret, not the
  prefix).
- **HTTP 401 `Admin access required` on `/api/admin/*`** — the bearer token
  does not match `ARONA_ADMIN_TOKEN`, or the variable is unset (then the
  route always rejects). Restart the server after setting it.
- **`auth.register` fails with "Registration is closed"** — registration is
  disabled when `ARONA_REGISTRATION_OPEN` is not truthy. Set
  `ARONA_REGISTRATION_OPEN=1` **before** starting the server (it is read at
  startup), or be the very first user — the first registered user is always
  allowed and becomes the admin.
- **HTTP 429 rate limits** — three independent limits can fire:
  - the per-key in-memory limit, 60 RPM by default
    (`ARONA_API_RATE_LIMIT_RPM`) → `Rate limit exceeded. Try again later.`;
  - the free billing tier's per-key 10 RPM limit → 429 with a
    `Retry-After: 60` header;
  - the free tier's monthly $1 / 100k-token quota → 429 with `Retry-After`
    pointing at the next billing period.

## Next steps

- [Configuration](configuration.md) — every environment variable.
- [Backends](backends.md) — backend types, URL semantics and probing.
- [Deployment](deployment.md) — bare metal, systemd, Docker.
- [OpenAI-compatible REST API](../api/openai-rest.md) — the full `/v1/*` surface.
- [JSON-RPC API](../api/jsonrpc.md) — the management plane used above.

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
