---
title: "Eventos y notificaciones"
description: "Sidecar de server-sent events (SSE) — chat.stream, models.progress, realtime.event y notificaciones de vídeo."
---

# Eventos y notificaciones

Los tokens de streaming, el progreso de despliegue y los eventos realtime
**no** se entregan por el socket WebSocket JSON-RPC. Cada RPC streaming crea un
**session id** y empuja notificaciones al endpoint SSE como server-sent events:

```
GET /api/rpc/events?session=<session_id>
```

## Receta de suscribirse-antes-de-enviar

Las notificaciones emitidas entre que la llamada RPC devuelve un session id y
que se establece la suscripción SSE se **pierden** (la ventana de
pre-suscripción). El patrón fiable es:

1. Abra primero el stream SSE (se bloquea hasta que se adjunta un session id).
2. Dispare el RPC que devuelve el session id (p. ej. `chat.send`,
   `agents.deploy`, `realtime.start`, `video.create`).
3. Lea las notificaciones del stream SSE a medida que llegan.

Cada notificación es un mensaje con estilo JSON-RPC 2.0 con `"jsonrpc": "2.0"`,
un `method` y un objeto `params`.

## Catálogo de notificaciones

### `chat.stream`

Una notificación por token, producida por `chat.send` (y cualquier ruta de chat
streaming que use un canal de sesión):

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — un delta de contenido.
- `is_complete` — `false` hasta el chunk final (cuando el upstream adjunta una
  razón de finalización, el chunk de contenido final puede llevar ya
  `is_complete: true` con un token no vacío); la notificación **terminal**
  siempre llega después con un `token` vacío e `is_complete: true`.
- Un error de stream se entrega como notificación terminal con
  `token: "Stream error: ..."` e `is_complete: true` (consulte
  `packages/core/src/gateway/rpc.rs`).

### `models.progress`

Progreso de descarga de modelos para `agents.deploy`, reenviado desde el
agente. El `stream_id` proviene de la respuesta de `agents.deploy`.

### `realtime.event`

Eventos de servidor para una sesión realtime full-duplex abierta, empujados al
canal de sesión (`packages/core/src/gateway/realtime.rs`). Los eventos de
cliente enviados por el RPC `realtime.event` se reenvían al upstream; los
eventos de servidor llegan aquí.

### Notificaciones de trabajos de vídeo

Los trabajos de `video.create` empujan el progreso por el canal de sesión
(`packages/core/src/gateway/video.rs`):

| Método | Payload (params) | Significado |
| --- | --- | --- |
| `video.progress` | `job_id`, `stream_id`, `status: "running"`, `progress` (0-90) | El trabajo se está ejecutando. |
| `video.done` | `job_id`, `stream_id`, `result`, `cost` | El trabajo terminó; `result` lleva la URL del artefacto. |
| `video.failed` | `job_id`, `stream_id`, `error` | El trabajo falló o fue cancelado. |

## Notas de orden

- El stream SSE está ordenado por sesión; los tokens llegan en orden de
  generación.
- Un session id solo puede ser consumido por un suscriptor SSE; abra el stream
  antes (o inmediatamente después) del RPC que devuelve el id.
- La cabecera `x-session-id` en `POST /api/rpc` adjunta también la propia
  **respuesta** RPC a un canal de sesión — la usan los clientes que quieren que
  la respuesta se refleje en el mismo stream.

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
