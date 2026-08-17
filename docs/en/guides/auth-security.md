---
title: "Authentication & Security"
description: "JWT sessions, API keys, the three admin gates, password policy, dual-track rate limiting and the security model."
---

# Authentication & Security

Arona authenticates callers on two tracks: **JWT session tokens** for
interactive clients (the chat + admin UI, RPC calls) and **API keys**
(`arona-…`) for programmatic OpenAI-compatible traffic. A separate admin token
guards the administrative surfaces. This page documents the mechanics, the
security model, and the known low-risk leftovers from a security audit.

## JWT sessions

Sessions use JWT access/refresh token pairs issued by the `kirino_session`
token manager:

- **Access token TTL: 900 seconds (15 minutes).**
- **Refresh token TTL: 604,800 seconds (7 days).**

Access tokens authenticate the JSON-RPC plane (`/api/rpc`) and
`GET /v1/models`; the SSE sidecar (`/api/rpc/events`) is keyed by its session
id, a capability minted during authenticated RPC calls rather than a bearer
credential. The `/v1/chat/completions`, `/v1/embeddings` and `/v1/video/*`
endpoints require an **API key** (a JWT is not accepted there).
Access tokens are short-lived so a stolen token is usable only briefly.
Refresh tokens are exchanged for fresh pairs via `auth.refresh`.

Refresh uses **token-family rotation**: consuming a refresh token invalidates
it and issues a new one, and reusing a consumed refresh token revokes the
whole family — `auth.refresh` answers with `AUTH_ERROR` and the message
`Refresh token reused` (the underlying error is `TokenReused`, "refresh token
has been reused — token family revoked"), and the account must log in again.
Family revocation is **in-memory** (a `revoked_families` set): a server
restart clears it, so the protection is best-effort across restarts (per-user
session state does not survive a restart).

The signing secret comes from the `JWT_SECRET` environment variable. Outside
`MOCK_MODE=1` the server **refuses to start** if `JWT_SECRET` is unset or
still equals the built-in development secret, so a production instance can
never accidentally serve tokens signed with a public constant. Use a strong,
random secret and never commit it.

## API keys

API keys are the machine credential for the OpenAI-compatible surface:

- **Format:** `arona-{uuid}`.
- **Storage:** only the **SHA-256 hash** of the key is stored in the
  `api_keys` table — the plaintext is returned exactly once, in the
  `keys.create` response, and can never be recovered later.
- **Key prefix:** the first 8 characters (`key_prefix`) are stored in clear
  for display and usage attribution; the UI shows a masked form such as
  `arona-XXXX…abcd`.
- **Revocation:** key lookup joins `api_keys.is_active = TRUE`, so a revoked
  key stops validating immediately — there is no cache TTL to wait out.

## Admin tiers

There are three distinct admin gates, each with its own credential:

1. **`/api/admin/*` routes** — backend and alias management
   (`POST/GET/DELETE /api/admin/backends`, `POST/GET/DELETE /api/admin/aliases`)
   require the `Authorization: Bearer ARONA_ADMIN_TOKEN` header. When
   `ARONA_ADMIN_TOKEN` is unset, `check_admin` always fails and every admin
   route returns **401 "Admin access required"** — the whole management
   surface is disabled rather than opened up.

2. **`agents.*` and `engine.invoke` RPC methods** — the agent cluster and
   engine control plane require a JWT whose account has `users.is_admin =
   true`. An authenticated non-admin is rejected with the implementation-
   defined code **-32007 (`ADMIN_REQUIRED`)** plus a method-specific hint
   (e.g. `agents.deploy starts model deployments on GPU nodes`); an
   **unauthenticated** caller gets the standard **-32005 (`AUTH_ERROR`)** so
   the server does not reveal that the method is privileged at all.

3. **`billing.plan.set` and `billing.video.pricing.set` RPC methods** —
   billing mutations require the same Bearer `ARONA_ADMIN_TOKEN` as the admin
   HTTP routes; without it they return `AUTH_ERROR` "Admin access required".

The **first registered user becomes the admin** (`users.is_admin = true`).
Every later registration is a regular user, and registration is only open
while `ARONA_REGISTRATION_OPEN` is set to a truthy value.

## Password policy

Passwords must satisfy **both** rules (enforced at registration and on any
password-change path):

- at least **8 characters**, and
- at least **3 of the 4 character categories**: uppercase, lowercase, digit,
  special.

## Rate limiting

Rate limiting runs on two independent tracks; either one can reject a request
with **429**:

### 1. In-memory sliding window (per identity)

Every authenticated `/v1` request passes an in-memory sliding-window limiter
keyed by the caller's identity:

- **API-key calls** are keyed by the key's **SHA-256 hash**;
- **JWT calls** are keyed by `u:<email>` — JWTs rotate every 15 minutes, so
  keying the window by the token instance would silently reset it on every
  refresh.

The default budget is **60 requests per minute**, overridable with
`ARONA_API_RATE_LIMIT_RPM` (set higher for agent pipelines that fan out many
parallel LLM calls). Setting it to **0 blocks every request**.

### 2. Tier rate limit (per key, from the database)

Billing tiers carry a per-key `rate_limit_rpm`. The check counts
`usage_records` rows for the key's prefix in the **trailing 60 seconds**
(usage is persisted after each response, so the window lags by at most one
in-flight request; DB failures fail open). The seeded **free tier is 10
RPM**; pro/enterprise tiers raise the ceiling. Monthly quota enforcement
shares the same rejection path.

### Login rate limiting

Credential-guessing is throttled at the login endpoint: **5 failed attempts
per 5-minute window per email** and **20 per 5-minute window per IP**, each
followed by a 15-minute lockout.

### `Retry-After`

Every 429 response carries a `Retry-After` header so OpenAI-compatible
clients back off instead of hammering the endpoint: quota rejections set it
to **seconds until the end of the month**; rate-limit rejections set it to
**60**. See [Billing & Usage](billing-usage.md) for the quota model.

## Security model notes

- **CORS allows any origin** (`allow_origin(Any)`) — Arona is a backend
  consumed by many first-party and third-party clients; if your deployment
  must restrict origins, front it with a reverse proxy that enforces CORS.
- **Request bodies are limited to 1 MB** (`RequestBodyLimitLayer`), bounding
  memory use on the gateway.
- **The gateway terminates no TLS** — it listens on plain HTTP. Put it behind
  a reverse proxy (see [Deployment](deployment.md)) that terminates HTTPS.
- **Secrets come from the environment only**: `ARONA_ADMIN_TOKEN` and
  `JWT_SECRET` are read from env vars, and must be strong random values never
  committed to the repository.
- The default server bind address is `0.0.0.0`; restrict exposure at the
  network layer.

## Known low-risk leftovers (from the audit)

The following are documented as-is; they are intentional or accepted for now,
but worth knowing when you expose an instance beyond a trusted network:

- **`providers.list` is public**, while `providers.add` / `providers.update` /
  `providers.remove` / `providers.test` require a JWT. The public read path
  reveals the provider catalog but nothing secret.
- **`/ws/agent` is an unauthenticated control plane**: GPU agents connect
  with no credential and self-register (`register` / `heartbeat` /
  command-result frames). Anyone who can reach the WebSocket port can
  register a fake agent. See [Agent Cluster](agent-cluster.md) for the
  operational trade-offs.
- **`memory.delete` is JWT-only with no ownership check**: any authenticated
  user can delete a memory node by `node_id`. Deleting memory requires being
  logged in, but not owning the node.

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
