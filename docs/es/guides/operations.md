---
title: "Operaciones"
description: "Endpoints de salud, tracing con RUST_LOG, timeouts de upstream, mapeo de errores y solución de problemas para un arona-server en ejecución."
---

# Operaciones

Esta página es para los operadores que ejecutan `arona-server serve`. Cubre los
endpoints de salud que se someten a probe, las líneas de log que merece la pena
buscar con grep, el modelo de timeouts aplicado a los backends upstream, cómo se
asignan los fallos de backend a errores HTTP y las trampas operativas con las
que la gente tropieza. El despliegue en sí se trata en la
[guía de despliegue](./deployment.md).

## Matriz de salud

Los tres endpoints de salud no requieren autenticación y devuelven `200 OK`
siempre que el proceso esté sirviendo — no hay distinción liveness/readiness:

| Endpoint | Respuesta |
| --- | --- |
| `/healthz`, `/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | el mismo cuerpo detallado que arriba |
| `/api/health` | `HealthResponse` de plana: `status`, `version` (`CARGO_PKG_VERSION`), `kind` (`Dev`), `uptime` (segundos), `network` (transporte / región / asn), `build_hash` (`BUILD_HASH`), `engine_version` (`"0.1.0"`) |

`/healthz` y `/readyz` son alias del mismo handler, y `/v1/health` lo comparte,
así que los probes estilo Kubernetes y la ruta de salud compatible con OpenAI
son intercambiables. `/api/health` añade uptime, network y la versión del motor.
Use `/readyz` para los load balancers y supervisores; use `/api/health` cuando
necesite el payload más rico.

## Registro (logging)

El servidor registra a través de `tracing`, filtrado con la variable `RUST_LOG`
estándar (`RUST_LOG=info` es la configuración habitual; `RUST_LOG=debug` revela
el tráfico de probes). Eventos que merece la pena conocer, en orden aproximado
de frecuencia:

| Línea de log | Nivel | Qué le dice |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | Una por solicitud de chat, con `key_prefix`, `model`, `stream` y `request_id` — la pista de auditoría por solicitud más simple. |
| `request completed` | info | La registra el helper `logging_middleware` después de cada respuesta **no streaming** de `/v1/chat/completions` y `/v1/embeddings`: `method`, `path`, `status`, `latency_ms`, `trace_id`. (El chat streaming registra `chat completions SSE request` al inicio en su lugar.) |
| `usage recorded` / `usage persisted` | info | Se registró una fila de uso (en memoria, con tokens/coste) y luego se escribió en la tabla `usage_records`. |
| `external probe: sending` / `external probe: returned` | debug | Un probe de salud del `/v1/models` de un backend external; `matched` dice si el probe se completó dentro del timeout de probe de 2 s. |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | Una solicitud `/v1/*` rechazada por la puerta de billing — el cliente recibió 429 más `Retry-After`. |
| `rpc billing gate rejected: monthly quota exceeded` | warn | La puerta de cuota del lado RPC para los métodos autenticados con JWT (ventana de todo el usuario; respuesta de error JSON-RPC). |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | Restauración al arrancar: backends registrados por el admin y nodos agente cargados desde la base de datos y hechos enrutables de nuevo. |
| `Shutdown signal received, draining connections…` | info | Comenzó el apagado correcto (SIGINT/SIGTERM). |

## Modelo de timeouts

Los timeouts se aplican en el cliente upstream usado para los backends external
(`packages/core/src/backends/external.rs`):

| Timeout | Valor | Se aplica a |
| --- | --- | --- |
| Connect | 10 s | Establecer la conexión TCP/TLS con el upstream. |
| Read idle | 120 s por lectura | Cada llamada al upstream; cada byte recibido reinicia el reloj, así que un stream lento pero vivo nunca se corta. |
| Total no streaming | 600 s | Llamadas de chat/embeddings no streaming — un upstream lento pero vivo no puede mantener una solicitud para siempre. |
| Streaming (SSE) | ninguno | Las llamadas streaming no llevan **plazo total**; las generaciones largas son legales y la detección de cuelgues depende del timeout read-idle. |
| Probe de salud | 2 s | El probe de `/v1/models`. |

## Mapeo de errores

Los fallos de backend se asignan a estados HTTP en los handlers de
chat/embeddings (`packages/core/src/gateway/server.rs`):

| Condición | HTTP | `type` / `code` | Mensaje |
| --- | --- | --- | --- |
| Estado upstream no-2xx (`UpstreamStatus`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| Fallo de transporte upstream (`RequestFailed`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | la cadena del error de transporte |
| Cualquier otro error de backend | **500** | `server_error` / `backend_error` | la cadena del error |
| Sin backend para el modelo (`NoBackend`) | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| API key no válida (`Unauthorized`) | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| Límite de tasa (`RateLimited`) | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

La intención de diseño: los llamadores pueden distinguir "su provider rechazó o
falló" (502) de "el propio gateway está roto" (500). Cada cuerpo de error tiene
la misma forma con estilo OpenAI — `{"error":{"message":...,"type":...,"code":...}}`
(`json_error_response`). Los 429 de la puerta de billing además llevan una
cabecera `Retry-After` y usan `quota_error`/`quota_exceeded` (cuota) y
`rate_limit_error`/`rate_limit_exceeded` (límite de tasa del tier)
respectivamente.

## Solución de problemas

### Un backend recién registrado permanece en fail-closed hasta que se somete a probe

Los backends external arrancan en un estado de salud desconocido e informan
`"<url> not probed yet"`. Pasan a saludables cuando (a) se ejecuta la primera
ronda del verificador de salud — inmediatamente al arrancar y luego cada 60 s —
o (b) el probe fire-and-forget lanzado en el registro o la restauración tiene
éxito, normalmente en ~1-2 segundos. Hasta entonces, las solicitudes enrutadas
al backend fallan en fail-closed por diseño.

### Un 404 en `/models` del probe es normal en algunos backends

El probe external golpea `GET {base}/v1/models` (o `{base}/models` para URLs
base con prefijo de ruta). Algunos servidores compatibles con OpenAI
implementan chat pero no exponen un listado de modelos — el endpoint de
coding-plan de Zhipu GLM es uno. Un **404 se tolera**: el backend se marca
saludable y la lista de modelos configurada por el admin sigue siendo
autoritativa para el routing. Solo los probes realmente fallidos (timeout,
error de red, otro no-2xx) marcan el backend como no saludable.

### Los streams SSE que no producen nada no se facturan

Una respuesta streaming solo se registra en el uso cuando el stream produjo
texto **o** trajo uso terminal; un stream que terminó sin ninguna de las dos
cosas no se registra en absoluto. Si ve una solicitud sin una línea `usage
recorded` correspondiente, compruebe si el stream produjo contenido realmente.

### Informe de versión

`version` en los cuerpos de salud es `CARGO_PKG_VERSION`; `build_hash` es el
valor `BUILD_HASH` de tiempo de compilación emitido por
`packages/core/build.rs`. Compare `build_hash` entre nodos para confirmar que
todos ejecutan el mismo artefacto.

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
