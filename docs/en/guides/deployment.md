---
title: "Deployment"
description: "Install arona-server from a release asset, script or source; run it on bare metal with systemd, in Docker Compose, or under malkuth supervision; and upgrade it safely."
---

# Deployment

Arona ships as a **single self-contained Rust binary** built from the `_cli`
crate. The release workflow publishes it as
`arona-server-<tag>-linux-x86_64`
(`.github/workflows/release.yml`), and the same binary plays both roles:

- **API server** — `serve` (the gateway, `migrate` applies schema
  migrations explicitly);
- **Model tooling** — `install` (hardware detection), `status`, `deploy
  <model>`, `download <repo>`, and `connect <panel-url>`.

The server requires PostgreSQL. The schema is created and migrated
automatically on startup, so deployment is mostly "get the binary, point it
at a database, run it". Start with the [quickstart](./quickstart.md) for an
end-to-end walkthrough, then come back here for the production layout.

## Requirements

- **Linux x86_64** for the prebuilt release asset; any platform supported by
  Rust 1.91+ can build from source.
- **PostgreSQL** — the Docker Compose example uses `postgres:16-alpine`;
  any recent version works. The server refuses to start without
  `DATABASE_URL`.
- **`curl`, `python3`, `ca-certificates`** if you use the install script
  (on Debian/RedHat the script installs them itself via apt/dnf).
- A writable location for the binary (e.g. `/usr/local/bin`) and, if you
  run model downloads, disk space for the model cache under `ARONA_DATA_DIR`.

## Install

### From a release asset

Download the asset for the tag you want and make it executable:

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### From the install script

The repository ships `scripts/install.sh`: it resolves the latest release
tag from the GitHub API (or honors the `ARONA_VERSION` override), downloads
the matching asset, installs it as `arona-server` in `~/.local/bin` by
default (override the directory with `ARONA_BIN_DIR`), and prints the next
steps:

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### From source

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` is exactly what the release workflow and
the Dockerfile build.

## Configuration

All configuration is via environment variables; the [configuration
reference](./configuration.md) documents every variable, and
`.env.example` at the repository root is a working starting point. The
minimum set:

| Variable | Required? | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | yes | PostgreSQL connection string; the server exits if unset. |
| `JWT_SECRET` | yes | Token-signing secret; the server refuses to run with the built-in development secret unless `MOCK_MODE=1`. |
| `ARONA_ADMIN_TOKEN` | strongly recommended | Shared bearer token for `/api/admin/*` routes; without it those routes always return 401. |

Optional: `ARONA_HOST` (default `0.0.0.0`), `ARONA_PORT` (default `8420`),
`ARONA_REGISTRATION_OPEN` (truthy `1`/`true`/`yes`/`on` — opens signup;
the first registered user becomes admin), `ARONA_DATA_DIR` (model cache
root), `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`
(memory gateway), `ARONA_API_RATE_LIMIT_RPM` (per-key rate limit) and
`RUST_LOG` (tracing filter).

## Database migrations

`serve` connects to the database and applies all pending schema migrations
on startup (`init_database` → `Migrator::up`), so there is no separate
deploy step. To apply migrations explicitly — for example to verify them
before the first start — run:

```bash
arona-server migrate
```

The database user needs **`CREATE` privilege** on the target schema,
because the startup migration creates tables. There is no data migration
beyond the schema: the database *is* the state, so back it up before
upgrades (see [Upgrade and backup](#upgrade-and-backup)).

## Bare metal with systemd

Example unit file (`/etc/systemd/system/arona.service`). **All secret
values below are placeholders — replace `CHANGE_ME` before use:**

```ini
[Unit]
Description=Arona API server
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=arona
Environment=DATABASE_URL=postgres://arona:CHANGE_ME@127.0.0.1:5432/arona
Environment=JWT_SECRET=CHANGE_ME
Environment=ARONA_ADMIN_TOKEN=CHANGE_ME
Environment=ARONA_HOST=0.0.0.0
Environment=ARONA_PORT=8420
Environment=RUST_LOG=info
# Optional:
# Environment=ARONA_REGISTRATION_OPEN=1
# Environment=ARONA_MEMORY_URL=ws://127.0.0.1:8424/ws
# Environment=ARONA_MEMORY_TOKEN=CHANGE_ME
# Environment=ARONA_MEMORY_WRITEBACK=1
# Environment=ARONA_DATA_DIR=/var/lib/arona
ExecStart=/usr/local/bin/arona-server serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Then enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

The repository root ships a `docker-compose.yml` with two services:

- **`arona`** — builds the image from the `Dockerfile` (tag `arona:local`),
  publishes `${ARONA_PORT:-8420}:8420`, and requires `DATABASE_URL`,
  `JWT_SECRET` and `ARONA_ADMIN_TOKEN` — Compose fails fast with a
  `:?`-style error if any is missing from `.env`. Its healthcheck runs
  `curl -fsS http://127.0.0.1:8420/readyz`.
- **`postgres`** — `postgres:16-alpine` with placeholder-only credentials
  (`POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`; the default
  password is `change-me` — override it) and a named volume
  `arona-pgdata`.

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

The bundled postgres service is reachable at host `postgres` inside the
Compose network. The `Dockerfile` builds `_cli` and `_agent` in a
`rust:1.91-slim-bookworm` builder, installs `ca-certificates`, `curl` and
`python3` in a `debian:bookworm-slim` runtime, copies both binaries in,
exposes ports 8420 (server) and 5790 (the bundled `_agent`'s local API),
and runs `arona-server serve` as the entrypoint.

## Supervised deployment with malkuth (workspace convention)

In this workspace arona runs under **malkuth** supervision as
`arona-malkuth.service`. The pattern applies to any service deployed here:

- malkuth supervises the `arona-server serve` process — it spawns it,
  probes it, restarts it on failure and drains connections before
  shutdown.
- Each supervised service is exposed only through a per-service **info
  port** and a per-service **proxy port**; the service itself binds
  `127.0.0.1` and is never reachable directly from the network.
- The supervisor is started with `--watch <deployment-path>`: when the
  binary at that path changes — e.g. an upgrade copies a new file over
  it — malkuth performs a **rolling restart**, one instance at a time,
  draining connections first.

Operational consequences:

- Bind `ARONA_HOST=127.0.0.1` when running behind the supervisor proxy;
  the proxy is the only network-facing entry point.
- Upgrading is "copy the new binary onto the deployment path": the watcher
  triggers the rolling restart. You can also restart the supervised unit
  explicitly.
- Point the supervisor's health check at `/readyz` (or `/api/health`); see
  [Health probes](#health-probes).

## Health probes

The server exposes two unauthenticated health families (both also covered
in the [operations guide](./operations.md)):

- `GET /healthz`, `GET /readyz` (aliases) and `GET /v1/health` return
  `200` with `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`.
- `GET /api/health` returns the plana `HealthResponse` shape: `status`,
  `version`, `kind`, `uptime` (seconds), `network`, `build_hash` and
  `engine_version`.

Point load balancers, supervisors and container healthchecks at
`/readyz`; use `/api/health` when you need uptime and network detail.

## Upgrade and backup

- **Keep the previous binary.** Before overwriting `arona-server`, copy it
  aside — `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak` —
  so you can roll the binary back immediately if the new version fails to
  start.
- **Back up PostgreSQL.** The database holds all state — backends, agent
  nodes, users, keys, conversations and usage — and the only automatic
  schema change is the startup migration. `pg_dump` the `arona` database
  before every upgrade.
- **The database user needs `CREATE` rights**, because migrations run on
  startup; a read-only role cannot boot the server.
- **Shut down gracefully.** The server drains in-flight connections on
  SIGINT/SIGTERM, so prefer `systemctl restart arona` or a supervisor
  restart over `kill -9`.

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
