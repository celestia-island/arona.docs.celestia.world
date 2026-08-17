---
title: "Clúster de agentes"
description: "Clústeres GPU multinodo — descarga de pesos de modelos con la CLI, ejecución del binario _agent en nodos GPU y gestión de despliegues mediante la superficie RPC agents.*."
---

# Clúster de agentes

La historia de despliegue de Arona se divide en dos mitades. El **panel** (el
binario del servidor `arona`) es dueño del routing, el billing, la auth y el
plano de gestión. Cada nodo GPU ejecuta un **proceso `_agent`** que es dueño de
los pesos de los modelos y de los procesos de servicio locales. Los agentes
abren un WebSocket de larga duración de vuelta al plano de control de agentes
del panel (`/ws/agent`); el panel empuja comandos `deploy` / `stop` por ese
socket y el agente transmite hacia arriba el progreso de descarga, los
heartbeats y los resultados de comandos. Una vez que un modelo se ejecuta en un
agente, el panel lo registra como backend enrutable para que el tráfico `/v1/*`
y RPC lo alcance — el plano de control es WebSocket, el plano de datos es HTTP
plano al puerto de motor local del agente.

## Descarga de pesos de modelos (CLI)

El binario `_cli` descarga pesos de modelos desde HuggingFace, ModelScope o
releases de GitHub:

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **Formas de repo** — `hf:owner/repo` (predeterminado; un `owner/repo` simple
  también se resuelve a HuggingFace), `ms:owner/repo` (ModelScope),
  `gh:owner/repo[:tag]` (release de GitHub, tag opcional). También se aceptan
  los prefijos largos `huggingface:`, `modelscope:` y `github:`; un id simple
  sin barra se resuelve al registry de Ollama
  (`packages/core/src/models/download.rs:21-28,55-86`).
- **`--filter <glob|prefix>`** — repetible; solo se descargan los archivos del
  manifest que coinciden con el glob (o prefijo). Sin filtro se selecciona el
  **repo completo**.
- **Confirmación** — una descarga sin filtro siempre pregunta `Continue? [y/N]`
  antes de empezar salvo que se pase `--yes`. Una descarga filtrada nunca
  pregunta; cuando el total seleccionado está en o por encima de 2 GiB, imprime
  un banner informativo en su lugar (`NO_CONFIRM_THRESHOLD`,
  `packages/cli/src/main.rs:12-15, 439-464`).
- **`--out <dir>`** — sobrescribe el destino predeterminado
  `~/.arona/models/<repo-id>`.
- **`--revision <rev>`** — sobrescribe cualquier sufijo `:rev` en línea
  (`hf:owner/repo:rev`).
- **Reanudación** — las descargas interrumpidas se reanudan automáticamente: se
  conserva un archivo `.part` y la descarga continúa desde su longitud actual
  mediante una solicitud Range; los archivos completos se omiten por tamaño y,
  cuando el manifest lleva un digest, se verifican con SHA-256
  (`packages/cli/src/main.rs` `verify_sha256`/`summarize`).
- **Reintentos** — los errores de red reintentan hasta 3 veces con una espera de
  5 segundos; los errores no relacionados con la red fallan de inmediato
  (`packages/cli/src/main.rs:277-302`).
- **`HF_ENDPOINT`** — cambia la URL base de HuggingFace, p. ej. un mirror:

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

Los otros comandos de la CLI (`packages/cli/src/main.rs:28-53`):

| Comando | Propósito |
| --- | --- |
| `install` | Configuración del entorno con un clic: detecta el perfil de hardware e imprime recomendaciones de backend / cuantización. |
| `status` | Imprime el perfil de hardware. |
| `deploy <model>` | Resuelve un modelo localmente e informa si ya está en caché. |
| `download` | Descarga pesos de modelos (arriba). |
| `serve` | Inicia el API server (panel). |
| `connect <url>` | Se conecta a un panel de gestión. |
| `migrate` | Ejecuta migraciones de base de datos. |

## El binario `_agent`

`_agent` se ejecuta en cada nodo GPU y se configura puramente con variables de
entorno (`packages/core/src/config.rs:37-40`):

| Variable | Predeterminado | Significado |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | Id de nodo único; el panel lo usa como `agent_id`. |
| `ARONA_PANEL_URL` | `localhost:8080` | `host:port` del panel; el agente se conecta a `ws://{ARONA_PANEL_URL}/ws/agent`. |

Consulte [Configuración](./configuration.md) para la referencia completa de
variables de entorno (variables del lado del panel, base de datos, secretos).

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

Comportamiento:

- **Conexión de control** — el agente se conecta de vuelta a
  `ws://{ARONA_PANEL_URL}/ws/agent` (`packages/agent/src/panel.rs:23`). Al
  conectarse envía un frame `register` con `agent_name`, `gpu_info` y la lista
  de modelos ya desplegados; el panel registra la dirección TCP del peer del
  agente como su `host`.
- **Backoff de reconexión** — empieza en 1 segundo y se duplica hasta un tope de
  60 segundos (`packages/agent/src/panel.rs:27,33-34,63`).
- **Heartbeats** — cada 30 segundos el agente informa la utilización de GPU, el
  número de modelos cargados y el uptime. El panel considera a un agente
  desconectado cuando su último heartbeat tiene más de 30 segundos.
- **API HTTP local** — se vincula a la dirección **fija** `0.0.0.0:5790`; no
  existe una variable de entorno de dirección de bind
  (`packages/agent/src/main.rs:109`). El panel combina este puerto con el host
  registrado del agente para construir la URL del plano de datos de los modelos
  desplegados.
- **Comandos** — el panel pone en cola comandos `deploy` / `stop` a través del
  socket. Un comando `deploy` lleva `model_id` y un `stream_id`; el progreso de
  descarga se transmite de vuelta como frames `deploy_progress` en el mismo
  socket (el panel los reenvía como notificaciones SSE `models.progress`,
  consulte [Eventos y notificaciones](../api/events.md)), y el frame final
  `deploy_result` informa el `port` y el `backend` del motor local. A `stop` se
  responde con `stop_result`.

Ejecute `_agent` bajo un supervisor de servicios (systemd, malkuth, ...) para
que se reconecte automáticamente; el panel tolera reinicios en cualquiera de los
dos lados (consulte [persistencia de nodos](#node-persistence) más abajo).

## RPC del plano de control de agentes

Toda la superficie de agentes está protegida por admin: cada método requiere un
JWT válido **y** una cuenta de admin (`validate_admin_jwt` comprueba
`is_admin_email`; `packages/core/src/gateway/rpc.rs:106-118,301-337`).

| Método | Params | Devuelve |
| --- | --- | --- |
| `agents.list` | — | Topología del clúster: `id`, `name`, `host`, `status` (`online`/`offline`), resumen de GPU, `models`, `last_heartbeat`, `version`, `connected_at`. |
| `agents.register` | `machine_name`, `version` | `agent_id`, `token`. |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — elimina el nodo. |
| `agents.status` | `agent_id` | `online`, resumen de GPU, `gpu_utilization`, `models`, `host`, `connected_at`, `last_heartbeat`. |
| `agents.deploy` | `model_id`, `agent_id?` | `{ "ok": true, "stream_id" }` — un `agent_id` vacío apunta automáticamente al nodo menos cargado. |
| `agents.stop` | `agent_id`, `model_id` | `{ "ok": true }` — detiene el despliegue. |

`agents.deploy` devuelve un `stream_id`; suscríbase a
`/api/rpc/events?session=<stream_id>` **antes** o inmediatamente después de la
llamada para recibir las notificaciones de descarga `models.progress` (consulte
[Eventos y notificaciones](../api/events.md)). Consulte
[API JSON-RPC](../api/jsonrpc.md) para los detalles de transporte y auth.

## Auto-registro de modelos desplegados

Cuando un frame `deploy_result` informa éxito, el panel registra un
`ExternalApiBackend` llamado **`agent-{model_id}`** en el router del gateway,
con URL base `http://{agent-host}:{port}` — el host registrado del agente más el
puerto del motor que informó (`packages/core/src/gateway/server.rs:310-366`,
`packages/core/src/gateway/mod.rs:253-270`). El modelo desplegado se convierte en
un backend enrutable normal: `/v1/chat/completions`, embeddings y chat RPC lo
alcanzan, los aliases se aplican y el verificador de salud le hace probes
(consulte [Backends](./backends.md) para los tipos de backend y la semántica de
probing).

- Re-desplegar el mismo modelo (p. ej. en un agente distinto) sustituye al
  backend anterior.
- Un `stop_result` con éxito lo des-registra de nuevo
  (`packages/core/src/gateway/mod.rs:274-287`); el id de modelo deja de
  resolverse.

## Colocación

Los despliegues sin `agent_id` explícito pasan por la colocación por menor carga
(`packages/core/src/gateway/tunnel.rs:214-229`): los candidatos son los agentes
cuyo último heartbeat tiene menos de 30 segundos, y se elige el que tenga la
**utilización media de GPU más baja**. Los agentes sin telemetría se ordenan al
final pero siguen siendo seleccionables. Si ningún agente está en línea, el RPC
falla con `No online agents available for deployment`.

En el lado del routing, las conversaciones se **fijan a un backend** (afinidad
de sesión): el primer backend que sirve una conversación se registra y se
reutiliza en los turnos posteriores, de modo que el estado por conversación como
una caché KV de runtime se mantiene caliente
(`packages/core/src/routing/mod.rs:31-34,110-138`). Si el backend fijado se
vuelve no saludable, el routing degrada a una selección nueva y re-fija.

## Persistencia de nodos

Los nodos agente se persisten en la tabla `agent_nodes` (`agent_id`,
`machine_name`, `version`, `host`, `gpu_info`, `models`, `connected_at`,
`last_heartbeat`; `packages/core/src/gateway/tunnel.rs:8-12`). Al arrancar el
panel se restauran las filas persistidas para que los nodos registrados
previamente sigan siendo visibles entre reinicios; las entradas restauradas no
tienen **sender** hasta que cada agente se reconecta por WebSocket
(`packages/core/src/gateway/run.rs:74-85`). Por tanto, desplegar a un nodo
restaurado pero desconectado falla hasta que su `_agent` se reconecte.

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
