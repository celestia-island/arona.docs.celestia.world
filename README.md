<!-- markdownlint-disable MD033 MD041 MD036 -->
<p align="center"><img src="https://raw.githubusercontent.com/celestia-island/docs.celestia.world/master/res/logo/arona.webp" alt="Arona" width="200" /></p>

<h1 align="center">Arona</h1>

<p align="center"><strong>Self-deployment and remote management platform for AI models.</strong></p>

<div align="center">

[![License: BUSL-1.1](https://img.shields.io/badge/License-BUSL--1.1-blue.svg)](https://spdx.org/licenses/BUSL-1.1.html)
[![GitHub](https://img.shields.io/badge/github-celestia--island%2Farona-blue.svg)](https://github.com/celestia-island/arona)
[![Docs](https://img.shields.io/badge/docs-arona.docs.celestia.world-blue.svg)](https://arona.docs.celestia.world)
[![Install](https://img.shields.io/badge/install-script-orange.svg)](scripts/install.sh)

</div>

<div align="center">

**English** ·
[简体中文](./docs/zh-Hans/guides/README.md) ·
[繁體中文](./docs/zh-Hant/guides/README.md) ·
[日本語](./docs/ja/guides/README.md) ·
[한국어](./docs/ko/guides/README.md) ·
[Français](./docs/fr/guides/README.md) ·
[Español](./docs/es/guides/README.md) ·
[Русский](./docs/ru/guides/README.md) ·
[العربية](./docs/ar/guides/README.md) ·
[Deutsch](./docs/de/guides/README.md) ·
[Português](./docs/pt/guides/README.md)

</div>
<!-- markdownlint-enable MD033 MD041 MD036 -->

<p align="center">
  <img src="https://raw.githubusercontent.com/celestia-island/docs.celestia.world/master/res/screenshots/arona-terminal.svg"
       alt="Arona serving the OpenAI-compatible /v1/models and /v1/chat/completions endpoints against the built-in mock upstream"
       width="840" />
</p>

> Early development. Not yet ready for public use.

## What Arona is

Arona is a **pure backend** platform for running and managing AI models on
your own hardware: an OpenAI-compatible model gateway *and* a management
plane. It serves the `/v1/*` OpenAI-compatible REST API, a JSON-RPC 2.0
management plane (`/api/rpc`), the agent control plane (`/ws/agent`) and API
docs. There is **no bundled web dashboard and no CLI chat** — the chat +
admin UI lives in [shittim-chest](https://github.com/celestia-island/shittim-chest),
which talks to Arona over the RPC surface.

Positioning: where one-api / new-api and LiteLLM are API-key gateways in
front of model providers, Arona additionally owns **model deployment** —
downloading weights, deploying them onto GPU-node agents, routing traffic,
and metering usage. pi is a CLI assistant that talks to models; Arona is the
platform pi (and other tools) talk *to*.

The full documentation lives at [arona.docs.celestia.world](https://arona.docs.celestia.world)
(English, with translations in progress).

## Install

Pick one:

```bash
# 1. Install script — downloads the prebuilt release asset
#    arona-server-<tag>-linux-x86_64 (see scripts/install.sh; override the
#    tag with ARONA_VERSION, the install dir with ARONA_BIN_DIR):
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash

# 2. Direct release asset:
VERSION=v0.1.25
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
  -o /usr/local/bin/arona-server && chmod +x /usr/local/bin/arona-server

# 3. From source:
cargo build --release -p _cli
```

The binary serves as the API server (`serve` / `migrate`) and as a model
tool (`install` / `status` / `deploy` / `download` / `connect`).

## Quickstart

The end-to-end quickstart (mock upstream → register backend → chat) is in the
[documentation](https://arona.docs.celestia.world/guides/quickstart.html).
In short:

```bash
export DATABASE_URL="postgres://user:password@localhost:5432/arona"
export JWT_SECRET="replace-with-a-long-random-secret"
export ARONA_ADMIN_TOKEN="replace-with-an-admin-token"
cargo run -p _cli -- migrate
cargo run -p _cli -- serve          # binds 0.0.0.0:8420

python3 scripts/mock/server.py      # mock OpenAI upstream on 127.0.0.1:8429
curl -s http://127.0.0.1:8429/api/test-key   # -> {"api_key": ..., "base_url": ...}

curl -s -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"type":"external","url":"http://127.0.0.1:8429","api_key":"<mock key>","name":"mock","models":["gpt-5.5"]}'
```

Then open registration (`ARONA_REGISTRATION_OPEN=1` **before** starting the
server — the first registered user becomes admin), `auth.register` /
`auth.login` over `/api/rpc`, `keys.create` (returns `arona-{uuid}` once),
and chat against `/v1/chat/completions` with `Authorization: Bearer $AR_KEY`.
See the [quickstart](https://arona.docs.celestia.world/guides/quickstart.html)
for the full walkthrough and troubleshooting.

## API surface

| Endpoint | Method(s) | Auth | Description |
| --- | --- | --- | --- |
| `/v1/chat/completions` | POST | API key (`arona-…`) | OpenAI-compatible chat, streaming and non-streaming. |
| `/v1/embeddings` | POST | API key (`arona-…`) | OpenAI-compatible embeddings. |
| `/v1/models` | GET | API key or JWT | Model list (routable backends + quick-start models). |
| `/v1/video/generations` | POST | API key (`arona-…`) | Submit an async video generation task. |
| `/v1/video/generations/{id}` | GET | API key (`arona-…`) | Poll a video task's status / result. |
| `/api/rpc` | POST, GET (WS) | JWT (Bearer header; WS `?token=`) | JSON-RPC 2.0 management plane. Anonymous sockets may only send `system.probe`. |
| `/api/rpc/events?session=` | GET (SSE) | session id | Server → client notifications (`chat.stream`, `models.progress`, `realtime.event`, video progress). |
| `/ws/agent` | GET (WS) | — | Agent control plane; GPU-node agents connect here. |
| `/docs` | GET | — | Swagger UI. |
| `/api-docs/openapi.json` | GET | — | OpenAPI 3.1 spec. |
| `/api/admin/backends` | POST / GET / DELETE | `ARONA_ADMIN_TOKEN` | Register / list / remove upstream backends. |
| `/api/admin/aliases` | POST / GET / DELETE | `ARONA_ADMIN_TOKEN` | Model alias management. |
| `/v1/health`, `/healthz`, `/readyz`, `/api/health` | GET | — | Health / readiness probes. |

## Billing & usage

- **Attribution** — REST `/v1/*` requests are recorded under the API-key
  prefix (`arona-XX`, first 8 chars of the `arona-{uuid}` secret); JWT-authenticated RPC
  channels (`chat.send`, realtime sessions) are recorded under
  `jwt-<user-uuid>`. Both row kinds join back to the user in the monthly
  aggregation and in `usage.list`.
- **Tiers** — every account starts on the `free` tier (seeded in the
  database, mirrored by `free_tier()` in `billing/mod.rs`): **$1/month,
  100k tokens, 10 requests per minute** (`pro`: $20 / 5M tokens / 120 RPM;
  `enterprise`: unlimited / 1000 RPM). Exceeding the monthly quota or the
  per-minute rate limit returns **429** with a `Retry-After` header.
- **Fail-open** — billing is best-effort: a database failure during a quota /
  rate-limit check logs and **allows** the request rather than blocking chat.
- **Token estimation** — when an upstream never reports usage (ollama,
  ws_engine), the recorded usage falls back to the local CJK-aware tokenizer
  estimate; upstream-reported usage always wins when present.

## Long-term memory (Memory Gateway)

Chat turns can be persisted as conversations and recalled as long-term
memories across sessions. Recall is injected as a `## Relevant Long-Term
Memories` system section; completed turns are written back as episodes.
Everything degrades explicitly — failures never block the chat. Point arona
at an entelecheia scepter instance with the PhiLia memory agent:

```bash
export ARONA_MEMORY_URL="ws://<scepter-host>:8424/ws"
export ARONA_MEMORY_TOKEN="<scepter connection token>"
export ARONA_MEMORY_WRITEBACK="true"     # default on; "false" disables (strict boolean)
```

Every chat response carries an `X-Arona-Memory` header: `enabled` (recall
ran), `disabled` (gateway not configured, or `memory: false` on the request)
or `offline` (gateway configured but unreachable). REST and RPC `chat.send`
requests may pass `memory: true|false` per call.

## Agent cluster (multi-node deployment)

The `_agent` binary runs on GPU nodes and connects back to the panel over
WebSocket (`ws://<panel>/ws/agent`); deployed models auto-register into the
gateway router with least-count cross-node load balancing. The agent's HTTP
API is hardcoded to bind `0.0.0.0:5790` (there is no bind-address env var).

```bash
# On each GPU node:
export ARONA_AGENT_NAME="node-<id>"
export ARONA_PANEL_URL="<panel-host>:8420"   # ws://<panel>/ws/agent is used
_agent
```

- Deploy/stop models via the `agents.deploy` / `agents.stop` RPCs (empty
  `agent_id` auto-targets the least-loaded node), or list/register nodes with
  `agents.list` / `agents.register`.
- Agent nodes persist across panel restarts (`agent_nodes` table); models
  deployed on agents become routable backends automatically
  (`agent-{model}`), removed on stop.
- Conversations are pinned to one backend (session affinity) so runtime KV
  caches can be reused.

## Structure

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

The Vue web dashboard was removed from this repository and now lives in
shittim-chest (chest #291).

## Testing

- Unit tests: `cargo test --workspace` (no network) — 217 tests in
  `packages/core/src` plus hermetic integration (id-echo, admin-gate error
  codes, usage estimation).
- PG-gated gateway integration (register → login → keys → chat → usage,
  upstream 502 mapping, billing gates): `ARONA_TEST_PG=1 cargo test -p _core
  --test gateway_integration -- --ignored` (docker one-liner in the module
  docs).
- Live auth-flow integration: `ARONA_TEST_RUN=1 cargo test -p _core --test
  auth_flow -- --ignored` against a running server (`ARONA_TEST_URL` to
  override the default `http://127.0.0.1:8420`).

## License

BUSL-1.1 — Business Source License 1.1. See [LICENSE](LICENSE).
