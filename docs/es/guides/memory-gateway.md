---
title: "Gateway de memoria"
description: "Memoria a largo plazo para el chat — inyección de recuperación, writeback de episodios, control por solicitud, estados de cabecera y los RPC memory.status / memory.delete."
---

# Gateway de memoria

El Gateway de Memoria da a los turnos de chat acceso a la **memoria a largo
plazo** almacenada en el servicio de memoria scepter / Philia de entelecheia.
En cada turno de chat, Arona consulta al servicio las memorias relevantes para
la conversación, las inyecta en el prompt como una sección de sistema y —
después de una respuesta completada — escribe el turno de vuelta como un
episodio para que las conversaciones futuras puedan recuperarlo.

Es un cliente JSON-RPC por WebSocket hacia Philia (`Sync.ConnectHandshake`,
`Sync.MemoryQueryRequest`, `Sync.MemoryStoreRequest`,
`Sync.MemoryDeleteRequest`). Las conexiones se establecen de forma diferida, se
cierran ante cualquier error y se restablecen en la siguiente llamada; cada
fallo degrada en silencio y **nunca bloquea la ruta de chat**.

## Configuración

El gateway se controla con tres variables de entorno:

| Variable | Significado |
| --- | --- |
| `ARONA_MEMORY_URL` | URL WebSocket del servicio scepter / Philia, p. ej. `ws://192.0.2.10:8424/ws`. |
| `ARONA_MEMORY_TOKEN` | Token para el servicio de memoria. |
| `ARONA_MEMORY_WRITEBACK` | Si los turnos completados se escriben de vuelta. Predeterminado **activado**; defínalo en `false` para deshabilitarlo (se parsea como booleano estricto — `0` no se acepta). |

Deben estar definidas y no vacías tanto `ARONA_MEMORY_URL` **como**
`ARONA_MEMORY_TOKEN`; de lo contrario el gateway está **deshabilitado**: la
recuperación y el writeback se omiten por completo y cada solicitud informa
`disabled`. El token se envía tanto como parámetro de consulta `?token=` en la
actualización WebSocket como dentro de la solicitud `Sync.ConnectHandshake`.

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

Consulte [Configuración](configuration.md) para la referencia completa de
entorno.

## Inyección de recuperación

Con el gateway activado, **cada turno de chat** — REST no streaming
`/v1/chat/completions`, REST streaming (SSE) y RPC `chat.send` — consulta al
servicio de memoria antes de reenviar la solicitud:

- La consulta es el **último mensaje de usuario** del contexto ensamblado.
- Se solicitan hasta **5** memorias (`limit = 5`).
- Los resultados se representan como una sección de sistema markdown titulada
  `## Relevant Long-Term Memories`, un bullet `- [score] text` por memoria
  (puntuaciones con dos decimales, las entradas vacías se omiten), y se
  anteponen a la lista de mensajes como un mensaje `system`. La inyección es
  idempotente: un contexto que ya lleva la sección no se vuelve a inyectar.
- Si no se devuelve ninguna memoria relevante, no se inyecta nada y el turno
  continúa sin cambios.

La recuperación se ejecuta antes de la persistencia de la conversación y del
reenvío al upstream; un servicio de memoria lento o con fallos **no añade
garantía de latencia** más allá de su propio timeout RPC de 10 segundos y no
puede hacer fallar la solicitud.

## Writeback

Después de una respuesta de asistente completada, el turno se escribe de vuelta
al servicio de memoria como un nodo **episodio**. El texto del episodio es una
transcripción heurística del turno — `User: <user content>\nAssistant:
<assistant content>` (cualquiera de los lados se omite cuando está vacío; si
ambos están vacíos se omite el writeback). El writeback es **fire-and-forget**:
se ejecuta en una tarea generada, nunca bloquea la respuesta de chat y sus
fallos solo se registran dentro del cliente de memoria. (En la ruta REST
streaming, el writeback además requiere que una conversación esté adjunta a la
solicitud; las rutas REST no streaming y RPC escriben de vuelta
independientemente.)

## Control por solicitud

Tanto el cuerpo de la solicitud de chat REST como los params de `chat.send`
RPC aceptan un campo `memory` opcional para sobrescribir la configuración del
servidor **por llamada**:

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` — fuerza la recuperación activada/desactivada
  para este turno.
- omitido (`null`) — sigue la configuración del servidor
  (`req.memory.unwrap_or(true)`), es decir, activada si y solo si el gateway
  está configurado.

La sobrescritura afecta a la recuperación; el writeback solo sigue a
`ARONA_MEMORY_WRITEBACK` más el hecho de que el gateway esté activado.

## Estados de la cabecera

Las respuestas REST llevan el estado de memoria del turno en la cabecera de
respuesta **`X-Arona-Memory`**; la respuesta RPC `chat.send` refleja el mismo
valor en un campo `memory` de su resultado. Estados posibles:

| Valor | Significado |
| --- | --- |
| `enabled` | La memoria se solicitó, el gateway está configurado, la recuperación tuvo éxito y se inyectó al menos una memoria. |
| `disabled` | Gateway no configurado, o `memory: false` en la solicitud, o ningún mensaje de usuario que consultar, o la recuperación tuvo éxito pero no devolvió **ninguna** memoria relevante (nada que inyectar). |
| `offline` | La memoria se solicitó y el gateway está configurado, pero la llamada de recuperación falló (conexión rechazada, error RPC o timeout). |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## Semántica de fallos

Todo degrada explícitamente, en la misma dirección — el chat siempre se ejecuta:

- **Fallo de recuperación** — se registra a nivel `warn`; la solicitud continúa
  sin memorias inyectadas e informa `offline` en la cabecera.
- **Fallo de writeback** — se registra dentro del cliente de memoria; la
  respuesta de chat no se ve afectada.
- **Servicio de memoria no configurado** — la recuperación y el writeback son
  no-ops; cada solicitud informa `disabled`.

No existe ningún modo en el que una caída de memoria haga fallar o retrase una
solicitud de chat más allá de los timeouts acotados del propio cliente.

## Superficie RPC

Se exponen dos métodos de gestión en la superficie JSON-RPC (ambos requieren un
JWT; consulte [API JSON-RPC](../api/jsonrpc.md)):

**`memory.status`** — instantánea del gateway:

```json
{
  "enabled": true,
  "writeback": true,
  "events": [
    { "kind": "recall", "detail": "…query…", "at": "2026-01-01T00:00:00.000Z" },
    { "kind": "writeback", "detail": "…node id…", "at": "…" },
    { "kind": "error", "detail": "…", "at": "…" }
  ]
}
```

`events` es un buffer circular en memoria de la actividad reciente — eventos de
recuperación, writeback, borrado y error, los más recientes primero, hasta el
recuento solicitado (el handler de status solicita los últimos 50; el propio
buffer está limitado a 100). **No** es un registro de auditoría duradero — se
restablece al reiniciar.

**`memory.delete`** — poda un nodo almacenado por id:

```json
{ "node_id": "…" }
```

Devuelve `{ "deleted": true | false }`. Falla con un error cuando `node_id`
falta o cuando el servicio de memoria no está configurado.

## Relacionados

- [Configuración](configuration.md) — variables `ARONA_MEMORY_*`.
- [Inicio rápido](quickstart.md) — configuración de extremo a extremo del gateway.
- [Backends](backends.md) — cómo se enrutan las solicitudes de chat antes de la recuperación.
- [Billing y uso](billing-usage.md) — cómo se contabilizan esos mismos turnos de chat.
- [Operaciones](operations.md) — logs y salud de la conexión de memoria.
- [API JSON-RPC](../api/jsonrpc.md) — `memory.status`, `memory.delete`, `chat.send`.
- [Visión general](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
