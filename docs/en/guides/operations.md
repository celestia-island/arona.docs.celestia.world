---
title: "Operations"
description: "Health endpoints, RUST_LOG tracing, upstream timeouts, error mapping and troubleshooting for a running arona-server."
---

# Operations

This page is for operators running `arona-server serve`. It covers the
health endpoints you probe, the log lines worth grepping, the timeout model
applied to upstream backends, how backend failures map to HTTP errors, and
the operational gotchas that trip people up. Deployment itself is covered
in the [deployment guide](./deployment.md).

## Health matrix

All three health endpoints are unauthenticated and return `200 OK` whenever
the process is serving — there is no liveness/readiness distinction:

| Endpoint | Response |
| --- | --- |
| `/healthz`, `/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | the same detailed body as above |
| `/api/health` | plana `HealthResponse`: `status`, `version` (`CARGO_PKG_VERSION`), `kind` (`Dev`), `uptime` (seconds), `network` (transport / region / asn), `build_hash` (`BUILD_HASH`), `engine_version` (`"0.1.0"`) |

`/healthz` and `/readyz` are aliases of the same handler, and `/v1/health`
shares it, so the Kubernetes-style probes and the OpenAI-compatible health
route are interchangeable. `/api/health` adds uptime, network and engine
version. Use `/readyz` for load balancers and supervisors; use
`/api/health` when you need the richer payload.

## Logging

The server logs through `tracing`, filtered with the standard `RUST_LOG`
variable (`RUST_LOG=info` is the common setting; `RUST_LOG=debug` reveals
probe traffic). Events worth knowing, in rough order of frequency:

| Log line | Level | What it tells you |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | One per chat request, with `key_prefix`, `model`, `stream` and `request_id` — the simplest per-request audit trail. |
| `request completed` | info | Logged by the `logging_middleware` helper after every **non-streaming** `/v1/chat/completions` and `/v1/embeddings` response: `method`, `path`, `status`, `latency_ms`, `trace_id`. (Streaming chat logs `chat completions SSE request` at start instead.) |
| `usage recorded` / `usage persisted` | info | A usage row was recorded (in-memory, with tokens/cost) and then written to the `usage_records` table. |
| `external probe: sending` / `external probe: returned` | debug | A health probe of an external backend's `/v1/models`; `matched` says whether the probe completed within the 2s probe timeout. |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | A `/v1/*` request refused by the billing gate — the client received 429 plus `Retry-After`. |
| `rpc billing gate rejected: monthly quota exceeded` | warn | The RPC-side quota gate for JWT-authenticated methods (whole-user window; JSON-RPC error response). |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | Startup restore: admin-registered backends and agent nodes loaded from the database and made routable again. |
| `Shutdown signal received, draining connections…` | info | Graceful shutdown began (SIGINT/SIGTERM). |

## Timeout model

Timeouts are enforced on the upstream client used for external backends
(`packages/core/src/backends/external.rs`):

| Timeout | Value | Applies to |
| --- | --- | --- |
| Connect | 10s | Establishing the upstream TCP/TLS connection. |
| Read idle | 120s per read | Every upstream call; each received byte resets the clock, so a healthy-but-slow stream is never cut. |
| Non-streaming overall | 600s | Non-streaming chat/embeddings calls — a slow-but-alive upstream cannot hold a request forever. |
| Streaming (SSE) | none | Streaming calls carry **no overall deadline**; long generations are legal and hang detection relies on the read-idle timeout. |
| Health probe | 2s | The `/v1/models` probe. |

## Error mapping

Backend failures map to HTTP statuses in the chat/embeddings handlers
(`packages/core/src/gateway/server.rs`):

| Condition | HTTP | `type` / `code` | Message |
| --- | --- | --- | --- |
| Upstream non-2xx status (`UpstreamStatus`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| Upstream transport failure (`RequestFailed`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | the transport error string |
| Any other backend error | **500** | `server_error` / `backend_error` | the error string |
| No backend for the model (`NoBackend`) | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| Invalid API key (`Unauthorized`) | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| Rate limit (`RateLimited`) | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

The design intent: callers can tell "your provider rejected or failed"
(502) apart from "the gateway itself is broken" (500). Every error body has
the same OpenAI-style shape — `{"error":{"message":...,"type":...,"code":...}}`
(`json_error_response`). The billing-gate 429s additionally carry a
`Retry-After` header and use `quota_error`/`quota_exceeded` (quota) and
`rate_limit_error`/`rate_limit_exceeded` (tier rate limit) respectively.

## Troubleshooting

### A newly registered backend stays fail-closed until probed

External backends start in an unknown health state and report
`"<url> not probed yet"`. They flip healthy when (a) the health checker's
first round runs — immediately at startup, then every 60s — or (b) the
fire-and-forget probe launched at registration or restore time succeeds,
normally within ~1-2 seconds. Until then, requests routed to the backend
fail closed by design.

### A 404 on the probe's `/models` is normal for some backends

The external probe hits `GET {base}/v1/models` (or `{base}/models` for
base URLs with a path prefix). Some OpenAI-compatible servers implement
chat but expose no model listing — the Zhipu GLM coding-plan endpoint is
one. A **404 is tolerated**: the backend is marked healthy and the
admin-configured models list stays authoritative for routing. Only genuinely
failed probes (timeout, network error, other non-2xx) mark the backend
unhealthy.

### SSE streams that produce nothing are not billed

A streaming response is recorded to usage only when the stream produced
text **or** carried terminal usage; a stream that ended with neither is not
recorded at all. If you see a request without a matching `usage recorded`
line, check whether the stream actually produced content.

### Version reporting

`version` in the health bodies is `CARGO_PKG_VERSION`; `build_hash` is the
build-time `BUILD_HASH` value emitted by `packages/core/build.rs`. Compare
`build_hash` across nodes to confirm they all run the same artifact.

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
