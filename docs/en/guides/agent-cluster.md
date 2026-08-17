---
title: "Agent Cluster"
description: "Multi-node GPU clusters — downloading model weights with the CLI, running the _agent binary on GPU nodes, and driving deployments through the agents.* RPC surface."
---

# Agent Cluster

Arona's deployment story splits into two halves. The **panel** (the `arona`
server binary) owns routing, billing, auth and the management plane. Each GPU
node runs one **`_agent` process** that owns the model weights and the local
serving processes. Agents open a long-lived WebSocket back to the panel's
agent control plane (`/ws/agent`); the panel pushes `deploy` / `stop` commands
down that socket and the agent streams download progress, heartbeats and
command results back up. Once a model is running on an agent, the panel
registers it as a routable backend so `/v1/*` and RPC traffic reach it — the
control plane is WebSocket, the data plane is plain HTTP to the agent's local
engine port.

## Downloading model weights (CLI)

The `_cli` binary downloads model weights from HuggingFace, ModelScope or
GitHub releases:

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **Repo forms** — `hf:owner/repo` (default; a bare `owner/repo` also resolves
  to HuggingFace), `ms:owner/repo` (ModelScope), `gh:owner/repo[:tag]`
  (GitHub release, tag optional). The long prefixes `huggingface:`,
  `modelscope:` and `github:` are accepted too; a bare id with no slash
  resolves to the Ollama registry
  (`packages/core/src/models/download.rs:21-28,55-86`).
- **`--filter <glob|prefix>`** — repeatable; only manifest files matching the
  glob (or prefix) are downloaded. With no filter the **whole repo** is
  selected.
- **Confirmation** — an unfiltered download always asks `Continue? [y/N]`
  before starting unless `--yes` is passed. A filtered download never prompts;
  when the selected total is at or above 2 GiB it prints an informational
  banner instead (`NO_CONFIRM_THRESHOLD`, `packages/cli/src/main.rs:12-15,
  439-464`).
- **`--out <dir>`** — overrides the default destination
  `~/.arona/models/<repo-id>`.
- **`--revision <rev>`** — overrides any inline `:rev` suffix
  (`hf:owner/repo:rev`).
- **Resume** — interrupted downloads resume automatically: a `.part` file is
  kept and the download continues from its current length via a Range request;
  complete files are skipped by size and, when the manifest carries a digest,
  SHA-256-verified (`packages/cli/src/main.rs` `verify_sha256`/`summarize`).
- **Retries** — network errors retry up to 3 times with a 5-second delay;
  non-network errors fail immediately (`packages/cli/src/main.rs:277-302`).
- **`HF_ENDPOINT`** — switches the HuggingFace base URL, e.g. a mirror:

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

The other CLI commands (`packages/cli/src/main.rs:28-53`):

| Command | Purpose |
| --- | --- |
| `install` | One-click environment setup: detects the hardware profile and prints backend / quantization recommendations. |
| `status` | Prints the hardware profile. |
| `deploy <model>` | Resolves a model locally and reports whether it is already cached. |
| `download` | Download model weights (above). |
| `serve` | Starts the API server (panel). |
| `connect <url>` | Connects to a management panel. |
| `migrate` | Runs database migrations. |

## The `_agent` binary

`_agent` runs on each GPU node and is configured purely by environment
variables (`packages/core/src/config.rs:37-40`):

| Variable | Default | Meaning |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | Unique node id; the panel uses it as the `agent_id`. |
| `ARONA_PANEL_URL` | `localhost:8080` | Panel `host:port`; the agent connects to `ws://{ARONA_PANEL_URL}/ws/agent`. |

See [Configuration](./configuration.md) for the full environment-variable
reference (panel-side variables, database, secrets).

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

Behaviour:

- **Control connection** — the agent connects back to
  `ws://{ARONA_PANEL_URL}/ws/agent` (`packages/agent/src/panel.rs:23`). On
  connect it sends a `register` frame carrying `agent_name`, `gpu_info` and the
  list of already-deployed models; the panel records the agent's TCP peer
  address as its `host`.
- **Reconnect backoff** — starts at 1 second and doubles up to a 60-second
  cap (`packages/agent/src/panel.rs:27,33-34,63`).
- **Heartbeats** — every 30 seconds the agent reports GPU utilization,
  loaded-model count and uptime. The panel considers an agent offline when its
  last heartbeat is older than 30 seconds.
- **Local HTTP API** — binds the **fixed** address `0.0.0.0:5790`; there is no
  bind-address environment variable (`packages/agent/src/main.rs:109`). The
  panel combines this port with the agent's recorded host to build the
  data-plane URL for deployed models.
- **Commands** — the panel queues `deploy` / `stop` commands over the socket.
  A `deploy` command carries `model_id` and a `stream_id`; download progress is
  streamed back as `deploy_progress` frames on the same socket (the panel
  forwards them as `models.progress` SSE notifications, see
  [Events & Notifications](../api/events.md)), and the final `deploy_result`
  frame reports the local engine `port` and `backend`. `stop` is answered with
  `stop_result`.

Run `_agent` under a service supervisor (systemd, malkuth, ...) so it
reconnects automatically; the panel tolerates restarts on either side (see
[node persistence](#node-persistence) below).

## Agent control plane RPC

The whole agent surface is admin-gated: every method requires a valid JWT
**and** an admin account (`validate_admin_jwt` checks `is_admin_email`;
`packages/core/src/gateway/rpc.rs:106-118,301-337`).

| Method | Params | Returns |
| --- | --- | --- |
| `agents.list` | — | Cluster topology: `id`, `name`, `host`, `status` (`online`/`offline`), GPU summary, `models`, `last_heartbeat`, `version`, `connected_at`. |
| `agents.register` | `machine_name`, `version` | `agent_id`, `token`. |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — removes the node. |
| `agents.status` | `agent_id` | `online`, GPU summary, `gpu_utilization`, `models`, `host`, `connected_at`, `last_heartbeat`. |
| `agents.deploy` | `model_id`, `agent_id?` | `{ "ok": true, "stream_id" }` — empty `agent_id` auto-targets the least-loaded node. |
| `agents.stop` | `agent_id`, `model_id` | `{ "ok": true }` — halts the deployment. |

`agents.deploy` returns a `stream_id`; subscribe to
`/api/rpc/events?session=<stream_id>` **before** or immediately after the call
to receive `models.progress` download notifications (see
[Events & Notifications](../api/events.md)). See
[JSON-RPC API](../api/jsonrpc.md) for the transport and auth details.

## Deployed-model auto-registration

When a `deploy_result` frame reports success, the panel registers an
`ExternalApiBackend` named **`agent-{model_id}`** into the gateway router,
with base URL `http://{agent-host}:{port}` — the agent's recorded host plus
the engine port it reported (`packages/core/src/gateway/server.rs:310-366`,
`packages/core/src/gateway/mod.rs:253-270`). The deployed model becomes a
normal routable backend: `/v1/chat/completions`, embeddings and RPC chat all
reach it, aliases apply, and the health checker probes it (see
[Backends](./backends.md) for backend types and probing semantics).

- Re-deploying the same model (e.g. on a different agent) replaces the
  previous backend.
- A successful `stop_result` unregisters it again
  (`packages/core/src/gateway/mod.rs:274-287`); the model id stops resolving.

## Placement

Deployments without an explicit `agent_id` go through least-loaded placement
(`packages/core/src/gateway/tunnel.rs:214-229`): candidates are agents whose
last heartbeat is under 30 seconds, and the one with the **lowest average GPU
utilization** is picked. Agents without telemetry sort last but remain
selectable. If no agent is online the RPC fails with
`No online agents available for deployment`.

On the routing side, conversations are **pinned to one backend** (session
affinity): the first backend that serves a conversation is recorded and
reused for subsequent turns, so per-conversation state such as a runtime KV
cache stays warm (`packages/core/src/routing/mod.rs:31-34,110-138`). If the
pinned backend becomes unhealthy, routing degrades to a fresh selection and
re-pins.

## Node persistence

Agent nodes persist in the `agent_nodes` table (`agent_id`, `machine_name`,
`version`, `host`, `gpu_info`, `models`, `connected_at`, `last_heartbeat`;
`packages/core/src/gateway/tunnel.rs:8-12`). At panel startup the persisted
rows are restored so previously registered nodes stay visible across restarts;
restored entries are **sender-less** until each agent reconnects over
WebSocket (`packages/core/src/gateway/run.rs:74-85`). Deploying to a restored
but disconnected node therefore fails until its `_agent` reconnects.

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
