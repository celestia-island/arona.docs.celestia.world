---
title: "Arona"
description: "Self-deployment and remote management platform for AI models — gateway, backends, billing, agents, memory."
---

# Arona

**Self-deployment and remote management platform for AI models.**

Arona is a **pure backend** platform written in Rust (axum): it is an
OpenAI-compatible model gateway *and* a management plane for the models you
run on your own hardware. It serves the `/v1/*` OpenAI-compatible REST API,
the JSON-RPC 2.0 management plane (`/api/rpc`), the agent control plane
(`/ws/agent`) and a Swagger UI at `/docs`.

There is **no bundled web dashboard and no bundled CLI chat** — the chat +
admin UI lives in [shittim-chest](https://github.com/celestia-island/shittim-chest),
which talks to Arona over the RPC surface. Arona focuses on the server side:
routing, billing, auth, model deployment, agents and memory.

## Feature matrix

| Area | What Arona provides |
| --- | --- |
| **Conversation core** | OpenAI-compatible `chat.completions` (stream + non-stream), `embeddings`, `models` listing; streaming with a terminal `[DONE]` chunk and real usage on the final chunk. |
| **Backends** | Admin-registered upstreams: `external` (any OpenAI-compatible HTTP API), `ollama`, CEP `engine` (WebSocket), `minimax-cloud` video, and `evernight://` bridge URLs into industrial/edge services. |
| **Authentication** | JWT access/refresh pairs (15 min / 7 days), API keys `arona-{uuid}` stored as SHA-256 hashes, three admin tiers, password policy, dual-track rate limiting. |
| **Billing & usage** | Seeded tiers (free / pro / enterprise), per-request usage records on every channel, plana pricing table, per-project quota scoping, 429 + `Retry-After`. |
| **Model management** | Model download (`hf:` / `ms:` / `gh:` sources), `_agent` GPU-node deployment, auto-registration of deployed models as routable backends. |
| **Realtime & multimodal** | Full-duplex `realtime.*` sessions, `engine.invoke` perception/control channel, async video generation jobs (MiniMax cloud). |
| **Agent cluster** | GPU nodes connect over `/ws/agent`, least-loaded placement, session affinity, node persistence across restarts. |
| **Memory gateway** | Long-term memory via entelecheia Philia: recall injection, writeback episodes, explicit degradation. |
| **Operations** | Health probes, `RUST_LOG` tracing, upstream error mapping (502 vs 500), graceful shutdown, auto-migration on start. |

## Positioning

Arona is a **gateway + platform**: it routes model traffic to your backends,
deploys models onto your GPU agents, and meters everything.

- vs **pi** — pi is a CLI assistant that talks to models; arona has no CLI
  chat. Arona is the platform pi (and other tools) talks *to*.
- vs **one-api / new-api** — those are API-key gateways in front of model
  providers; arona adds **model deployment** (download weights, run them on
  your agents), a full management RPC plane, billing tiers and memory.
- vs **LiteLLM** — a gateway peer; arona additionally owns the deployment
  lifecycle of the models behind the gateway.

## Start here

- [Quickstart](quickstart.md) — end-to-end with the built-in mock upstream.
- [Configuration](configuration.md) — every environment variable.
- [Deployment](deployment.md) — bare metal, systemd, Docker, supervision.
- [Backends](backends.md) — backend types, URL semantics and probing.
- [OpenAI-compatible REST API](../api/openai-rest.md) — `/v1/*`.
- [JSON-RPC API](../api/jsonrpc.md) — the full management plane.

## Repository layout

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

The web dashboard was removed from this repository and now lives in
[shittim-chest](https://github.com/celestia-island/shittim-chest) (chest #291).

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
