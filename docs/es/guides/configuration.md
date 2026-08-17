---
title: "Configuración"
description: "Referencia de todas las variables de entorno que lee el servidor de Arona, con sus valores predeterminados y su semántica."
---

# Configuración

Arona se configura **enteramente mediante variables de entorno**, leídas una
sola vez al arrancar el proceso (`Config::load` en
`packages/core/src/config.rs`, más unas pocas que se leen en el primer uso).
No hay archivo de configuración: cambie una variable y reinicie el servidor
para que surta efecto.

Esta página es la referencia de todo lo que lee el código del servidor,
agrupado por ámbito. Las variables solo de mock y de runtime se incluyen por
exhaustividad.

## Tabla de referencia

| Variable | Predeterminado | Propósito |
| --- | --- | --- |
| `DATABASE_URL` | *(obligatoria)* | URL de conexión a PostgreSQL. |
| `JWT_SECRET` | *(obligatoria fuera del modo mock)* | Secreto usado para firmar los JWTs. |
| `ARONA_HOST` | `0.0.0.0` | Dirección de bind (recurre a `SHITTIM_CHEST_HOST`). |
| `ARONA_PORT` | `8420` | Puerto de bind (recurre a `SHITTIM_CHEST_PORT`). |
| `ARONA_DATA_DIR` | sin definir | Directorio local de datos. |
| `ARONA_ADMIN_TOKEN` | sin definir | Bearer token para `/api/admin/*` y los métodos RPC de admin. |
| `ARONA_REGISTRATION_OPEN` | `0` | Un valor truthy (`1`/`true`/`yes`/`on`) abre el registro. |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | Límite en memoria de solicitudes por minuto y por clave; `0` bloquea todo. |
| `MOCK_MODE` | sin definir | Su presencia (cualquier valor) activa el modo mock de desarrollo. |
| `MOCK_SEED_PATH` | sin definir | Archivo SQL de seed ejecutado en modo mock. |
| `ARONA_MEMORY_URL` | sin definir | URL WebSocket del gateway de memoria Philia. |
| `ARONA_MEMORY_TOKEN` | sin definir | Token para el gateway de memoria. |
| `ARONA_MEMORY_WRITEBACK` | `true` | Escribe los turnos de chat completados de vuelta a la memoria; acepta `true`/`false` (cualquier otro valor recurre al predeterminado). |
| `ARONA_AGENT_NAME` | `arona-agent` | Identidad del agente en el nodo GPU. |
| `ARONA_PANEL_URL` | `localhost:8080` | Dónde se conecta el agente (`ws://<panel_url>/ws/agent`). |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | Agente evernight local para las URLs de backend `evernight://`. |
| `ARONA_MISTRALRS` | sin definir | Su presencia fuerza el motor Mistral.rs para los planes de modelo Gguf. |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | Interfaz a la que se vincula un servidor de modelo llama.cpp generado. |
| `HF_ENDPOINT` | `https://huggingface.co` | URL base de Hugging Face para las descargas de modelos. |
| `GITHUB_TOKEN` | sin definir | Token de acceso para el registry de modelos de GitHub. |
| `RUST_LOG` | sin definir | Filtro de tracing, p. ej. `info` o `arona=debug,info`. |

## Variables obligatorias

### `DATABASE_URL`

URL de conexión a PostgreSQL. **Obligatoria**: el servidor sale con
`FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.` cuando
está vacía, y el subcomando CLI `migrate` se niega a ejecutarse. El esquema se
crea/actualiza automáticamente con `serve` al arrancar.

### `JWT_SECRET`

Secreto usado para firmar los pares de JWTs access/refresh emitidos por
`auth.login` y `auth.register`. **Obligatoria en producción**: el código
incrusta un fallback de desarrollo
(`dev-secret-not-for-production-use-only-32chars`), pero el servidor se niega a
arrancar con él salvo que `MOCK_MODE=1`:

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

Use un valor largo y aleatorio (p. ej. `openssl rand -hex 32`).

## Servidor

### `ARONA_HOST` / `ARONA_PORT`

Dirección y puerto de bind del gateway. Por compatibilidad heredada recurren a
`SHITTIM_CHEST_HOST` / `SHITTIM_CHEST_PORT`; los valores predeterminados
definitivos son `0.0.0.0:8420`.

### `ARONA_DATA_DIR`

Directorio local de datos opcional, transportado en el estado de la aplicación
para los componentes que necesitan una ubicación de trabajo temporal. Sin
definir por defecto.

## Seguridad y control de acceso

### `ARONA_ADMIN_TOKEN`

Bearer token que protege las rutas HTTP `/api/admin/*` (`POST/GET/DELETE
/api/admin/backends`, `/api/admin/aliases`) y los métodos RPC
`billing.plan.set` / `billing.video.pricing.set`. Cuando está **sin definir**,
todas esas rutas rechazan con `Admin access required` (401) — no hay valor
predeterminado. Defínalo con un valor aleatorio robusto antes de iniciar el
servidor.

### `ARONA_REGISTRATION_OPEN`

Abre el registro de autoservicio mediante `auth.register`. Los valores truthy
son exactamente `1`, `true`, `yes`, `on` (sin distinguir mayúsculas); cualquier
otro valor — incluidos `0`, `false`, `off` o una variable sin definir/vacía —
permanece cerrado. La marca se lee una sola vez al arrancar. Al **primer
usuario registrado siempre se le permite** (incluso con el registro cerrado) y
se convierte en el admin.

### `ARONA_API_RATE_LIMIT_RPM`

Límite de tasa en memoria con ventana deslizante por clave (solicitudes por
minuto), aplicado a cada solicitud `/v1/*` autenticada (chat, embeddings,
vídeo, models), indexado por el hash de la API key (o la etiqueta `u:<email>`
para el `GET /v1/models` que acepta JWT). El plano RPC no está limitado por
este limitador — los extractores de auth de `/v1/*` son los únicos llamadores.
Predeterminado `60`. El valor se parsea una vez en un `OnceLock` de todo el
proceso. **Un valor de `0` bloquea todas las solicitudes** — la comprobación es
`entry.len() >= rpm`, así que con `0` ninguna solicitud puede pasar. Este es el
límite de todo el gateway; los billing tiers imponen sus propios límites por
clave encima.

## Desarrollo

### `MOCK_MODE`

Modo mock de desarrollo, activado por **presencia** — la comprobación es
`std::env::var("MOCK_MODE").is_ok()`, así que *cualquier* valor (incluidos `0`
o un valor vacío pero definido) lo activa. Este modo:

- elimina el requisito de `JWT_SECRET` (el secreto de desarrollo integrado se
  vuelve aceptable);
- siembra cuatro cuentas de demostración (`demiurge@celestia.world`,
  `momoi@celestia.world`, `midori@celestia.world`, `yuzu@celestia.world`,
  contraseña `33550336`);
- espera a que el seed termine antes de vincular el listener.

No lo use nunca en producción.

### `MOCK_SEED_PATH`

Solo en modo mock, apunta a un archivo SQL sin procesar que se ejecuta en lugar
del seed de cuentas integrado (las sentencias se dividen por `;`, los
comentarios `--` se omiten). Si el archivo no se puede leer, se usa el seed
integrado como fallback.

## Gateway de memoria

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

Configuración del gateway de memoria a largo plazo (entelecheia Philia). La
memoria está **deshabilitada por completo** salvo que `ARONA_MEMORY_URL` y
`ARONA_MEMORY_TOKEN` estén definidas y no vacías. Cuando está activada:

- los turnos de chat completados se recuperan e inyectan como contexto, y
- `ARONA_MEMORY_WRITEBACK` (predeterminado `true`) controla si los turnos
  terminados se escriben de vuelta al servicio de memoria; `0` o `false`
  deshabilita el writeback.

Los fallos de memoria nunca bloquean el chat; el estado resultante se refleja
en la cabecera de respuesta `X-Arona-Memory` (`enabled` / `disabled` /
`offline`).

## Identidad del agente y clúster

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

Identidad del binario agente del nodo GPU (`_agent`): `ARONA_AGENT_NAME`
(predeterminado `arona-agent`) se informa al panel como nombre/id del agente, y
`ARONA_PANEL_URL` (predeterminado `localhost:8080`) es a dónde se conecta el
agente (`ws://<panel_url>/ws/agent`).

La API HTTP propia del agente está **codificada** para vincularse a
`0.0.0.0:5790` — no existe una variable de entorno de dirección de bind para
ella.

### `ARONA_AGENT_BIND_ADDR`

Interfaz a la que se vincula un **servidor de modelo llama.cpp generado**
cuando el agente despliega un modelo Gguf, para que el motor sea accesible
desde otras máquinas (p. ej. `0.0.0.0`). Por defecto es `127.0.0.1`. Tenga en
cuenta que *no* es el bind de la API HTTP del agente (que está fijado en
`0.0.0.0:5790`).

## Bridge de evernight

### `ARONA_EVERNIGHT_URL`

URL WebSocket del agente evernight local que se usa para resolver las URLs de
backend `evernight://` en reenvíos TCP locales. Predeterminado
`ws://127.0.0.1:3001/ws`.

## Runtime de modelos y descargas

### `ARONA_MISTRALRS`

Su presencia (cualquier valor) fuerza el motor Mistral.rs para los planes de
modelo Gguf que de otro modo usarían llama.cpp por defecto. Semántica de
presencia igual que `MOCK_MODE`.

### `HF_ENDPOINT`

URL base para las descargas de modelos de Hugging Face (fuentes `hf:`),
predeterminado `https://huggingface.co`. Defínalo en un mirror como
`https://hf-mirror.com` cuando huggingface.co no sea accesible. Lo lee el
descargador de modelos; se recorta una barra final.

### `GITHUB_TOKEN`

Token de acceso que usa el registry de modelos de GitHub (fuentes `gh:`) para
el acceso a la API. Sin definir por defecto; sin él se aplican los límites de
tasa de la API de GitHub.

## Registro (logging)

### `RUST_LOG`

Filtro de tracing estándar aplicado por `tracing_subscriber` al arrancar, p.
ej. `info` o `arona=debug,info`. Sigue la semántica habitual de `RUST_LOG`
(`error`/`warn`/`info`/`debug`/`trace`, con sobrescrituras por destino).

## Valores predeterminados de un vistazo

| Ajuste | Predeterminado |
| --- | --- |
| Dirección / puerto de bind | `0.0.0.0:8420` |
| Límite de tasa de API por clave | 60 RPM |
| Nombre del agente | `arona-agent` |
| URL del panel | `localhost:8080` |
| Writeback de memoria | activado |
| Registro | cerrado |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
