---
title: "Referencia de la API JSON-RPC"
description: "API JSON-RPC 2.0 del plano de gestión de Arona en /api/rpc — métodos de chat, realtime, engine, auth, keys, providers, agents, memory, conversations, usage, billing, video y system por HTTP y WebSocket."
---

# Referencia de la API JSON-RPC

Arona expone una superficie JSON-RPC 2.0 en `/api/rpc` para el plano de
gestión: auth, keys, providers, agents, memory, conversations, usage, billing,
video, realtime y chat streaming. Complementa la superficie REST compatible con
OpenAI (`/v1/*`, consulte [API REST compatible con OpenAI](./openai-rest.md));
use REST para las cargas de trabajo de inferencia autenticadas con clave y
JSON-RPC para la gestión de sesiones/cuentas y el control de streaming. El
[Inicio rápido](../guides/quickstart.md) recorre el primer turno de extremo a
extremo.

La superficie despacha **39 métodos de solicitud** más un método de liveness
anónimo solo-WebSocket, `system.probe` (40 métodos en total). Cada solicitud es
un objeto JSON-RPC 2.0 con `jsonrpc: "2.0"`, una cadena `method`, un objeto
`params` opcional y un `id` opcional.

## Transporte

- **HTTP POST `/api/rpc`** — solicitud/respuesta. Envíe `Content-Type:
  application/json`. El JWT viaja en la cabecera `Authorization: Bearer <jwt>`.
  El cuerpo de la solicitud está limitado a 1 MiB.
- **WebSocket `GET /api/rpc`** — conexión de larga duración. Los navegadores no
  pueden definir cabeceras personalizadas en la actualización WebSocket, así que
  el JWT viaja como parámetro de consulta `?token=<jwt>`; el servidor lo pliega
  en una cabecera `Authorization: Bearer` internamente (consulte
  `packages/core/src/gateway/server.rs`). Los sockets autenticados pueden
  permanecer conectados indefinidamente.
- **Solicitudes en lote** — un cuerpo POST que es un array JSON se ejecuta
  elemento a elemento y se responde con un array JSON de respuestas en el mismo
  orden.
- **Acceso anónimo** — por WebSocket sin JWT, los métodos públicos
  (`auth.register`/`auth.login`/`auth.refresh`, `providers.list`,
  `system.status`) siguen siendo invocables, y `system.probe` se responde con un
  único ack antes de que el socket se cierre. Todos los demás métodos requieren
  un JWT válido; los métodos protegidos por admin además requieren una cuenta de
  admin (consulte la leyenda más abajo). Los sockets anónimos también están
  sujetos a un timeout de inactividad de 10 segundos.
- **Adjunto de sesión** — una cabecera `x-session-id` en `POST /api/rpc`
  además empuja la propia respuesta RPC a ese canal de sesión, junto con las
  notificaciones de streaming.

## Ids

Los valores `id` de las solicitudes se reflejan con fidelidad de tipo: `null` →
`null`, cadenas → cadenas, enteros → números, y cualquier otra cosa (floats,
objetos, enteros fuera del rango i64) → la representación JSON de la cadena. Un
`id` omitido se responde con `null`.

## Notificaciones de servidor → cliente (sidecar SSE)

Los tokens, el progreso de despliegue y los eventos realtime **no** se entregan
por el socket WebSocket. Cada RPC streaming crea un session id y empuja
notificaciones a `GET /api/rpc/events?session=<session_id>` como server-sent
events. Suscríbase al endpoint SSE **antes o inmediatamente después** de que la
llamada RPC devuelva un session id — las notificaciones emitidas entre que la
llamada devuelve y que se establece la suscripción SSE se pierden (la ventana de
pre-suscripción). El patrón recomendado es abrir primero el stream SSE y luego
disparar el RPC.

Métodos de notificación: `chat.stream` (un token por evento de `chat.send`),
`models.progress` (progreso de descarga de modelos de agente desde
`agents.deploy`), `realtime.event` (eventos de servidor para una sesión realtime
abierta) y `video.progress` / `video.done` / `video.failed` (trabajos de vídeo
asíncronos). Consulte el catálogo completo en
[Eventos y notificaciones](./events.md).

## Códigos de error

| Código | Nombre | Significado |
| --- | --- | --- |
| `-32700` | Parse error | El cuerpo de la solicitud no es JSON válido. |
| `-32600` | Invalid request | El objeto de solicitud está malformado, p. ej. falta `method`. |
| `-32601` | Method not found | Cadena `method` desconocida; el mensaje la refleja. |
| `-32602` | Invalid params | `params` falló la deserialización para el método. |
| `-32603` | Internal error | Fallo inesperado del servidor. |
| `-32000` | `APP_ERROR` | Error genérico de aplicación — p. ej. conversación/provider/agente no encontrado, ningún agente en línea disponible para desplegar. |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` — JWT ausente o no válido. También lo usan los métodos de admin token cuando el bearer token no coincide con `ARONA_ADMIN_TOKEN` (`"Admin access required"`). |
| `-32006` | `QUOTA_ERROR` | Cuota de billing mensual superada para un método RPC protegido por JWT (`chat.send`). |
| `-32007` | `ADMIN_REQUIRED` | Un **no-admin** autenticado llama a un método protegido por admin (`agents.*`, `engine.invoke`); el mensaje incluye una pista específica del método. |

> Los métodos `agents.*` y `engine.invoke` son solo-admin: requieren un JWT cuya
> cuenta tenga `users.is_admin = true`. Un no-admin autenticado se rechaza con
> `-32007` (`ADMIN_REQUIRED`); un llamador sin autenticar recibe el estándar
> `AUTH_ERROR` para que el servidor no revele que el método es privilegiado.

## Leyenda de auth

| Leyenda | Credenciales |
| --- | --- |
| **public** | No se requieren credenciales. |
| **JWT** | `Authorization: Bearer <jwt>` en HTTP, o `?token=<jwt>` en WebSocket. |
| **admin (JWT + is_admin)** | JWT bearer de una cuenta con `users.is_admin = true`. |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN` (configurado por entorno; cuando no está definido, el método siempre se deniega, denegación por defecto). |

Todas las credenciales y direcciones de ejemplo de este documento son
placeholders (IPs de documentación RFC 5737, claves `sk-xxx`). Consulte
[Autenticación y seguridad](../guides/auth-security.md) para el modelo de auth
completo detrás de esta leyenda.

## Chat

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model` (string), `messages` (array de `{ role, content, images?, tool_calls? }`), `temperature?` (number), `max_tokens?` (integer), `conversation_id?` (string), `memory?` (bool), `extra?` (object), `tools?` (array de definiciones de función con estilo OpenAI), `provider?` (string) | Envía un turno de chat streaming. Devuelve `{ "stream_id", "memory" }` — `memory` es el estado de recuperación (`enabled` / `disabled` / `offline`); los tokens llegan como notificaciones `chat.stream` en el sidecar SSE. Con un `conversation_id`, el historial persistido completado se ensambla del lado del servidor y el turno se persiste. Con puerta de billing (cuota mensual → `-32006`); el uso se registra bajo `jwt-<user-uuid>`. |

## Realtime (sesiones de audio/vídeo full-duplex)

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model` (string), `config?` (objeto de configuración de sesión), `conversation_id?` (string) | Abre una sesión full-duplex contra el backend que sirve `model`. Devuelve `{ "session_id", "stream_session" }`: use `session_id` para `realtime.event` / `realtime.stop`, y suscríbase a `stream_session` en el sidecar SSE para recibir las notificaciones `realtime.event`. |
| `realtime.event` | JWT | `session_id` (string), `event` (evento de cliente — append/commit/clear de audio, frame de imagen, create/cancel de respuesta, stop de sesión) | Envía un evento de cliente a una sesión abierta; se reenvía al backend del upstream. Devuelve `{ "ok": true }`. |
| `realtime.stop` | JWT | `session_id` (string) | Cierra y elimina una sesión. Devuelve `{ "removed": bool }`. |

## Engine (canal genérico de percepción/control)

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `engine.invoke` | admin (JWT + is_admin) | `model` (string), `method` (string), `params?` (object) | Invocación síncrona de solicitud/respuesta de un método de motor arbitrario en el backend que sirve `model` — el canal de alta frecuencia para llamadas de estilo `sensor.ingest` / `control.setpoint` (bucles de 20-30 Hz). El resultado es la respuesta cruda del backend. |

## Auth

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `auth.register` | public | `email`, `password`, `name?` | Registra una cuenta. Solo se permite mientras el registro esté abierto (`ARONA_REGISTRATION_OPEN`); el primer usuario registrado se convierte en el admin. Devuelve la misma respuesta de tokens que `auth.login` (`access_token`, `refresh_token`, `token_type`, `expires_in`, `user`). |
| `auth.login` | public | `email`, `password` | Inicia sesión. Devuelve `access_token`, `refresh_token`, `token_type`, `expires_in`, `user` (`{ id, email, name, is_admin }`). Con límite de tasa por IP y por cuenta. |
| `auth.refresh` | public | `refresh_token` | Canjea un refresh token por un access token nuevo (y un refresh token nuevo). Los refresh tokens reutilizados o caducados se rechazan con `AUTH_ERROR`. |
| `auth.me` | JWT | — | Perfil del usuario actual: `{ "id", "email", "name" }`. |

## Keys

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | Lista las API keys del llamador (id, name, `key_prefix`, project, timestamps, marca de activa). |
| `keys.create` | JWT | `name`, `project?` | Crea una API key. Devuelve `{ id, name, key, key_prefix, project, created_at }` — el secreto completo `arona-<uuid>` en `key` se muestra **una vez**; guárdelo de inmediato. |
| `keys.revoke` | JWT | `key_id` | Revoca una API key. Devuelve `{ "ok": true }`. |

## Providers

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | Lista los providers conocidos: entradas oficiales integradas más las personalizadas, como metadatos de visualización (`id`, `name`, `description`, `website_domain`, `is_official`, `is_operator`). Público por diseño — la lista no lleva credenciales; solo las mutaciones siguientes están protegidas por JWT. |
| `providers.add` | JWT | `id`, `name`, `description?`, `website_domain?` | Añade una entrada de provider personalizada. Devuelve `{ "ok": true }`. |
| `providers.update` | JWT | `provider_id`, `name?`, `description?`, `website_domain?` | Actualiza los campos de un provider personalizado (solo los proporcionados). Devuelve `{ "ok": true }`. |
| `providers.remove` | JWT | `provider_id` | Elimina un provider personalizado. Devuelve `{ "ok": true }`. |
| `providers.test` | JWT | — | Prueba una conexión de provider. Stub: devuelve `{ "ok": true, "message": "Provider connection test not yet implemented" }`. |

## Agents

Todos los métodos `agents.*` son solo-admin (JWT + `is_admin`). Los nodos
agente se conectan salientes por `GET /ws/agent`; este grupo RPC controla el
registry (consulte [Clúster de agentes](../guides/agent-cluster.md)).

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `agents.list` | admin (JWT + is_admin) | — | Lista los nodos agente registrados: id, name, host, estado `online`/`offline` (basado en heartbeat), resumen de GPU, modelos desplegados, version, timestamps. |
| `agents.register` | admin (JWT + is_admin) | `machine_name`, `version` | Registra un nodo agente con el gestor de túneles. Devuelve `{ "agent_id", "token" }` (el token es la credencial del plano de control del agente). |
| `agents.deregister` | admin (JWT + is_admin) | `agent_id` | Des-registra (desconecta) un agente. Devuelve `{ "ok": true }`. |
| `agents.status` | admin (JWT + is_admin) | `agent_id` | Estado por agente: marca online, host, resumen de GPU, modelos cargados, utilización de GPU, timestamps de heartbeat/conexión. |
| `agents.deploy` | admin (JWT + is_admin) | `model_id`, `agent_id?` (vacío/ausente = nodo menos cargado; error si ninguno está en línea) | Despliega un modelo en un agente. Devuelve `{ "ok": true, "stream_id" }` — suscríbase a `stream_id` en el sidecar SSE para las notificaciones de descarga `models.progress`. |
| `agents.stop` | admin (JWT + is_admin) | `agent_id`, `model_id` | Detiene un modelo desplegado. Devuelve `{ "ok": true, "stream_id": null }` (sin stream de progreso). |

## Memory

La memoria a largo plazo la sirve el servicio Philia de entelecheia por un
WebSocket; los fallos nunca bloquean el chat (consulte
[Gateway de memoria](../guides/memory-gateway.md)).

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | Estado del gateway de memoria: `{ "enabled", "writeback", "events" }` — marcas más hasta 50 eventos de actividad recientes (los más recientes primero). |
| `memory.delete` | JWT | `node_id` | Elimina un nodo de memoria almacenado. Devuelve `{ "deleted": bool }`. |

## Conversaciones

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | Lista las conversaciones del llamador con timestamps de edad relativa. |
| `conversations.create` | JWT | `title?` (predeterminado `New Conversation`) | Crea una conversación. Devuelve el objeto de conversación nuevo. |
| `conversations.get` | JWT | `conversation_id` (alias heredado: `id`) | Obtiene una conversación con sus mensajes. Con comprobación de propiedad; el acceso entre usuarios se rechaza. |
| `conversations.delete` | JWT | `conversation_id` (alias heredado: `id`) | Elimina una conversación (solo el dueño). Devuelve `{ "ok": true }`. |

> `conversations.get` / `conversations.delete` también aceptan la clave `id`
> heredada de los clientes antiguos de dashboard; `conversation_id` gana cuando
> ambas están presentes.

## Usage

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?` (integer, predeterminado 50, acotado a 1-200), `offset?` (integer, predeterminado 0), `project?` (string) | Registros de uso paginados del llamador, los más recientes primero, que cubren tanto las filas de API key (prefijo `arona-XX`) como las filas atribuidas por JWT (`jwt-<user-uuid>`). Devuelve `{ "records", "total", "limit", "offset", "project" }`; el filtro `project` reduce solo a las filas etiquetadas por clave. |

## Billing

Los tiers, las cuotas y la contabilidad de uso se describen en
[Billing y uso](../guides/billing-usage.md).

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | Estado de billing actual: `{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` — uso mensual (`cost_usd`, tokens, recuento de solicitudes) y cuota restante. |
| `billing.plan.set` | admin token | `user_email`, `tier` | Define el billing tier de un usuario. Devuelve `{ "ok": true }`. Denegado con `AUTH_ERROR` cuando el bearer no coincide con `ARONA_ADMIN_TOKEN`. |
| `billing.video.pricing.get` | JWT | — | Tabla de precios de vídeo. Devuelve `{ "pricing": [...] }`. |
| `billing.video.pricing.set` | admin token | `model`, `mode?` (predeterminado `per_second_resolution`), `base_price?` (number, predeterminado 0), `price_per_second?` (number, predeterminado 0), `price_per_frame?` (number, predeterminado 0), `resolution_coeff?` (object), `currency?` (predeterminado `USD`), `enabled?` (bool, predeterminado `true`) | Hace upsert de los precios de vídeo de un modelo. Devuelve `{ "ok": true }`. Denegado con `AUTH_ERROR` cuando el bearer no coincide con `ARONA_ADMIN_TOKEN`. |

## Video

Trabajos asíncronos de generación de vídeo (consulte
[Realtime y vídeo](../guides/realtime-video.md)). El progreso de los trabajos
se empuja como notificaciones `video.progress` / `video.done` / `video.failed`
en el canal de sesión.

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`, `prompt`, `negative_prompt?`, `images?` (array de `{ data_base64, mime_type }`), `duration_seconds?` (integer), `width?` (integer), `height?` (integer), `provider?` (string), `extra?` (object) | Envía un trabajo asíncrono de generación de vídeo. Devuelve `{ "job_id", "stream_id" }` — suscríbase a `stream_id` para las notificaciones de progreso. |
| `video.get` | JWT | `job_id` (UUID) | Hace poll del estado/resultado de un trabajo (status, progress, result, error, cost). |
| `video.list` | JWT | `limit?` (integer, predeterminado 20) | Lista los trabajos del llamador. Devuelve `{ "jobs": [...] }`. |
| `video.cancel` | JWT | `job_id` (UUID) | Cancela un trabajo en ejecución. Devuelve `{ "ok": true }`. |

## System

| Método | Auth | Params | Descripción |
| --- | --- | --- | --- |
| `system.status` | public | — | Estado agregado del gateway: `{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`. |
| `system.probe` | anónimo (solo WS) | — | Probe de liveness de un solo disparo por el transporte WebSocket. El servidor hace ack de `{ "ok": true, "status": "ok" }` y luego cierra el socket — los visitantes anónimos nunca mantienen una conexión abierta. Cualquier otro método en un socket sin autenticar se rechaza con `AUTH_ERROR`. |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
