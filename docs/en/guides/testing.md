---
title: "Testing"
description: "The Arona test pyramid — unit tests, hermetic integration, PostgreSQL-gated integration, live-server smoke tests, the mock server, and the real-credential smoke discipline."
---

# Testing

Arona's tests are arranged in layers so that the default `cargo test` run is
fast, hermetic and needs neither a database nor the network, while the
heavier suites are explicit opt-ins that exercise the real wire surface and a
real PostgreSQL. This page maps the layers, the commands that run them, and
the workspace discipline around real-credential smoke runs.

## Unit tests

The bulk of the coverage is plain unit tests inside `packages/core/src`:
217 `#[test]` / `#[tokio::test]` functions, plus ~23 more across
`packages/agent` and `packages/cli`. They run with:

```bash
cargo test --workspace
```

No network, no database. Key suites:

- **auth.rs** — the password policy (≥8 chars AND ≥3 of 4 character
  categories), `::uuid` casts in the raw INSERT/REVOKE SQL, request defaults,
  and admin-flag reads that fall back to `false`.
- **billing/mod.rs** — quota math on cost *or* token dimensions, the monthly
  window (`month_start`, `seconds_until_month_end`), the rate-limit ceiling
  (trips only *at* the RPM, `None` = unlimited), SQL-shape guards for the
  monthly-usage / tier / key-window queries, and `estimate_usage` preferring
  upstream-reported numbers.
- **routing/mod.rs** — alias resolution, `:latest` suffix matching, provider
  hints, least-loaded selection and conversation pinning.
- **gateway/mod.rs** — agent-backend registration: registering
  `agent-{model_id}`, re-registration replacing (not duplicating), and
  unregister restoring the router.

## Hermetic integration (always-run, DB-free)

`packages/core/tests/gateway_integration.rs` contains three always-run tests
that exercise real serialization/contract logic without touching a database:

- **A1** — the JSON-RPC id-echo serialization contract: numeric, string and
  null request ids round-trip through plana's `Id` enum with type fidelity.
- **A2** — the admin-gate error-code contract: `AUTH_ERROR` (-32005, anonymous)
  and `ADMIN_REQUIRED` (-32007, authenticated non-admin) stay distinct, live
  in the implementation-defined range, and never collide with plana's codes
  or the billing `QUOTA_ERROR` (-32006).
- **A3** — `estimate_usage`: upstream-reported usage wins verbatim; without it
  the local tokenizer estimate produces non-zero prompt/completion counts
  whose total is their sum.

`packages/core/tests/smoke.rs` adds three more always-run tests: hardware
detection, the model-registry root path, and config defaults under
`MOCK_MODE=1`.

## PG-gated integration

The full in-process gateway suite — `packages/core/tests/gateway_integration.rs`
— spins the complete axum router on a random loopback port, registers
disposable OpenAI-compatible mock upstreams through the real admin API, and
drives the wire surface with reqwest. Because `AuthManager` talks to
PostgreSQL on every path (even `MOCK_MODE=1` only seeds accounts *into the
database*), this suite is gated behind `ARONA_TEST_PG=1` and skipped by
default. The 10 tests:

- **T1** register + login + `keys.create`/`keys.list` (raw key masked in
  listings, `arona-` prefix).
- **T2** REST chat with usage-record persistence into PostgreSQL.
- **T3** JSON-RPC id echo over the wire (success and error paths).
- **T4** admin gate on `agents.list`: anonymous → `AUTH_ERROR`, non-admin →
  `ADMIN_REQUIRED`.
- **T5** upstream 401 → HTTP 502 `bad_gateway` with the upstream detail.
- **T6** register-time probe publishes models (model appears in
  `GET /v1/models` within 10s without a static model list).
- **T7** conversation persistence through `chat.send` (both turns land in
  `conversations.get`).
- **T8** free-tier billing gate: 10 RPM per key, the 11th request in the
  window is 429 `rate_limit_exceeded`.
- **T9** SSE stream with terminal usage recorded from the upstream.
- **T10** malformed JSON → 400; unknown model → 404 `model_not_found`.

Run it with the disposable-Postgres one-liner from the module docs
(gateway_integration.rs:18-26):

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

These are example credentials for the disposable test container only — never
point this at a real database.

## Live-server smoke

`packages/core/tests/auth_flow.rs` walks the full
`register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` chain against a **live** Arona server, mirroring the deployed
auth loop. It is `#[ignore]`d by default — the plain `cargo test` run never
touches the network. Run it explicitly:

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

Knobs:

- `ARONA_TEST_URL` — base URL (default `http://127.0.0.1:8420`).
- `ARONA_TEST_EXPECT_CHAT=1` — hard-assert `POST /v1/chat/completions` returns
  200. Without it the test only asserts auth passed (not 401/403), because the
  target environment may have no inference provider configured.

The suite also includes negative tests: an unauthenticated chat completion and
an unauthenticated `GET /v1/models` must both be rejected with 401.

## Mock server

`scripts/mock/server.py` is an aiohttp-based OpenAI-compatible fake used by
the quickstart and by smoke runs. It serves `POST /v1/chat/completions`
(non-stream and SSE), `GET /v1/models`, `GET /api/health`, the JSON-RPC
WebSocket/HTTP surface at `/api/rpc`, an SSE sidecar at `/api/rpc/events`, and
`GET /api/test-key`, which returns the mock API key so other services can
discover it. It listens on port 8429 by default (override with
`ARONA_MOCK_PORT`, host with `ARONA_MOCK_HOST`). The [quickstart](quickstart.md)
uses it to stand up an end-to-end environment without real model providers.

## Real-credential smoke discipline

Smoke runs against real providers (DeepSeek / GLM) are deliberately **not**
repository tests — they require real credentials and real money, so they
cannot live in CI or in the git tree. The workspace convention, documented in
the gateway_integration module docs (gateway_integration.rs:54-55), is:

- Evidence files live under `/mnt/work/arona-pr*-smoke.md` — workspace-local,
  never committed to git.
- Credentials come from the environment only; budgets are kept small.
- Each PR that touches the inference path gets a written evidence record.

The mock server is the stand-in for these runs in CI and local development;
the real-credential smoke is a release-time human step.

## CI

`.github/workflows/ci.yml` runs `cargo fmt`, `cargo clippy`, `cargo test
--workspace` and `cargo-deny` on the org's self-hosted runners
(`[self-hosted, linux, x64, local]`); `ci-hosted.yml` mirrors the same checks
on GitHub-hosted runners. `.github/workflows/docs.yml` builds this docs site
with lagrange and deploys it to GitHub Pages on pushes touching `docs/**`.

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
