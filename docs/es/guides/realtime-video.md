---
title: "Realtime y vídeo"
description: "Sesiones realtime full-duplex (realtime.start/event/stop), el canal de percepción/control engine.invoke y trabajos asíncronos de generación de vídeo."
---

# Realtime y vídeo

Arona expone dos canales multimodales más allá del chat de texto plano:
**sesiones realtime full-duplex** (entrada y salida de voz/vídeo por un único
canal bidireccional) y **generación de vídeo asíncrona** (trabajos estilo task
que se ejecutan en segundo plano e informan progreso). Ambos se enrutan al
backend que sirve el modelo solicitado y ambos se contabilizan a través de la
capa de billing.

## Sesiones realtime

Una sesión realtime es un canal bidireccional entre **un cliente** y **un
upstream**: una API realtime en la nube (vocabulario WebSocket
OpenAI-Realtime / Qwen-Omni-Realtime) o un motor CEP local. Los eventos del
cliente llegan por JSON-RPC y se reenvían al upstream; los eventos del servidor
se empujan de vuelta como notificaciones `realtime.event` por el canal SSE de la
sesión. El audio viaja como PCM16 en base64 (16 kHz cliente→modelo, 24 kHz
modelo→cliente), lo que coincide con el formato de vía de los vendors en la
nube para que el gateway pase los bytes sin tocarlos
(`packages/core/src/backends/realtime.rs:1-19`).

### `realtime.start`

Abre una sesión contra el backend que sirve `model` (JWT; params `model`,
`config?`, `conversation_id?` — `packages/core/src/gateway/rpc.rs:1890-1898,
1914-1984`). El backend **debe** declarar la capacidad `realtime` (modalidades
audio/vídeo); de lo contrario la llamada falla explícitamente con
`model {model} does not support realtime sessions (no audio/video modality)` —
no hay fallback silencioso al chat de texto
(`packages/core/src/gateway/realtime.rs:138-142`).

Se soportan dos tipos de upstream (`packages/core/src/gateway/realtime.rs:143-167`):

- **Upstream de motor CEP** — enruta los eventos por el canal de streaming
  `Engine.InvokeStart` del Celestia Engine Protocol, así que un motor omni
  desplegado localmente se une a la misma superficie de cliente sin un nuevo
  formato de vía.
- **Upstream en la nube** — una URL `wss://` fija que habla el vocabulario de
  eventos realtime en la nube (`session.update`, `input_audio_buffer.*`,
  `response.audio.delta`, ...). La implementación en la nube re-emite
  `session.update` al reconectar.

La respuesta es `{ "session_id": ..., "stream_session": ... }` — suscríbase a
`/api/rpc/events?session=<stream_session>` antes (o inmediatamente después) de
la llamada para recibir los eventos del servidor. El `conversation_id` opcional
persiste la transcripción de voz como mensajes de asistente y registra el uso de
tokens para billing (`packages/core/src/gateway/realtime.rs:32-85`).

### `realtime.event`

Envía un evento de cliente a la sesión (JWT; params `session_id`, `event` —
`packages/core/src/gateway/rpc.rs:1989-2013`). Los eventos soportados incluyen
`session.update`, `input_audio_buffer.append` / `.commit` / `.clear`,
`input_image_buffer.append`, `response.create`, `response.cancel` y
`session.stop`. `send_event` es **no bloqueante**: el evento se pone en cola en
un canal mpsc y la tarea forwarder lo escribe al upstream
(`packages/core/src/gateway/realtime.rs:254-280`). Solo el dueño de la sesión
puede enviar eventos.

### `realtime.stop`

Cierra y elimina la sesión (JWT; params `session_id` —
`packages/core/src/gateway/rpc.rs:2016-2034`). Cada sesión es dueña de
exactamente una **tarea forwarder** que mantiene el upstream y multiplexa ambas
direcciones: los eventos del cliente se consumen de la cola y los eventos del
upstream se hacen poll en el mismo bucle. El forwarder sale cuando el upstream
se cierra o la sesión se detiene, eliminando la entrada del registry
(`packages/core/src/gateway/realtime.rs:195-250`).

Los eventos del servidor se empujan como notificaciones `realtime.event` con
params `{ session_id, event }` por el canal de la sesión — consulte
[Eventos y notificaciones](../api/events.md).

## `engine.invoke`

`engine.invoke` es el canal genérico **síncrono** de métodos de motor (ADMIN:
JWT + `is_admin`; params `model`, `method`, `params?` —
`packages/core/src/gateway/rpc.rs:261-264,2049-2079`). Invoca un método
arbitrario en el backend que sirve `model` y devuelve el resultado
directamente, lo que lo convierte en el canal de percepción/control de alta
frecuencia: llamadas de estilo `sensor.ingest`, `control.setpoint` en bucles de
20-30 Hz. Los backends sin un canal de invocación genérico (todos los backends
HTTP compatibles con OpenAI) rechazan explícitamente con
`backend does not support generic invocation`
(`packages/core/src/backends/mod.rs:573-586`).

## Generación de vídeo (REST)

Los trabajos de vídeo son tareas asíncronas con estilo OpenAI sobre la
superficie REST (auth con API key — `packages/core/src/gateway/server.rs:876-993`;
consulte [API REST compatible con OpenAI](../api/openai-rest.md)):

**`POST /v1/video/generations`**

| Campo | Tipo | Notas |
| --- | --- | --- |
| `model` | string | obligatorio — selecciona un backend con capacidad de vídeo. |
| `prompt` | string | obligatorio. |
| `negative_prompt` | string? | |
| `images` | array? | URLs de datos Base64 (`data:image/png;base64,...`), transportadas como `{ data_base64, mime_type }`. |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | Pista de selección de backend (se compara con el nombre del backend). |
| `extra` | object? | Sobrescrituras específicas del backend (seed, steps, cfg, ...). |

Respuesta:

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`** hace poll del trabajo y devuelve `id`,
`object`, `model`, `status`, `progress`, `result`, `error`, `cost`,
`created_at`. Los trabajos están delimitados al llamador: un trabajo propiedad
de otro usuario devuelve 404. La superficie REST aplica las mismas puertas de
billing (cuota mensual, límite de tasa por minuto) que la ruta de chat.

## Generación de vídeo (RPC)

La misma capacidad está disponible por JSON-RPC (JWT —
`packages/core/src/gateway/rpc.rs:371-386,1296-1431`):

| Método | Params | Devuelve |
| --- | --- | --- |
| `video.create` | los mismos campos que la llamada REST | `{ job_id, stream_id }`. |
| `video.get` | `job_id` | La vista del trabajo (status, progress, result, cost, ...). |
| `video.list` | `limit?` (predeterminado 20, acotado a 1-100) | `{ jobs: [...] }`, los más recientes primero. |
| `video.cancel` | `job_id` | `{ "ok": true }` — solo el dueño puede cancelar. |

`video.create` devuelve un `stream_id`; suscríbase a
`/api/rpc/events?session=<stream_id>` para recibir las notificaciones de
trabajo (`video.progress` / `video.done` / `video.failed` — consulte
[Eventos y notificaciones](../api/events.md)).

## Backend

La generación de vídeo es **solo en la nube**: la API de la plataforma abierta
MiniMax H3, tipo de backend `minimax-cloud` (`BackendKind::CloudVideo` —
`packages/core/src/backends/mod.rs:502-504,720-727,759-761`). El flujo es
estilo task:

1. `POST /v1/video_generation_v2` → `task_id`
2. poll de `GET /v1/query/video_generation_v2?task_id=...` hasta `Success` /
   `Fail` o que siga `Pending`
3. en caso de éxito, resuelva el artefacto mediante
   `GET /v1/files/{file_id}/content` → `{ file: { download_url } }`

(`packages/core/src/backends/minimax_cloud.rs:66-210`). El backend MiniMax no
sirve chat/embeddings; sus capacidades declaran `supports_video_generation` y
`realtime: false` (consulte [Backends](./backends.md) para el modelo de
capacidades). El routing resuelve las solicitudes de vídeo solo contra backends
con `supports_video_generation`, respetando la pista `provider` opcional
(`packages/core/src/routing/mod.rs:205-263`).

El **backend ComfyUI se eliminó** durante la convergencia de la plataforma de
modelos: configurar el tipo de backend `"comfyui"` falla con `comfyui backend
removed` (`packages/core/src/backends/mod.rs:756-757`). No hay ruta de vídeo
autoalojada; el vídeo siempre pasa por un backend `minimax-cloud`.

## Ciclo de vida de los trabajos y precios

Un trabajo de vídeo avanza por `queued → running → done | failed | cancelled`
(`packages/core/src/gateway/video.rs`):

- **create** — la fila del trabajo se persiste (`queued`, progress 0) y se
  genera una tarea poller (`video.rs:109-176`).
- **running** — el poller envía el task (progress 5) y luego hace poll cada
  1.5 s, subiendo el progress de 5 en 5 cada pocas iteraciones hasta **90**
  (`video.rs:178-275`). Los errores de poll se registran y se reintentan.
- **done** — progress 100, se persisten la URL del resultado y el coste
  calculado, se registra el uso y se difunde una notificación `video.done`
  (`video.rs:332-368`).
- **failed** — fallo de envío o de poll → `video.failed`; después de 900
  iteraciones de poll (~22.5 minutos) el trabajo falla con
  `generation timed out`.
- **cancelled** — `video.cancel` define una marca que el poller observa en su
  siguiente pasada; el trabajo se marca `cancelled` y se dispara `video.failed`
  con el error `cancelled` (`video.rs:389-400`).

El uso se registra con el coste específico de vídeo: `record_video` escribe un
registro de uso por solicitud con cero tokens y un coste explícito en dólares
(`packages/core/src/billing/mod.rs:496-531`).

**Precios** específicos de modelo, en la tabla `video_pricing`
(`packages/core/src/billing/video.rs`):

| Modo | Fórmula |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (predeterminado) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` asigna la clave de píxel del lado corto (p. ej. `"768"`) a un
multiplicador, con `"*"` como fallback. Los modelos sin fila configurada
recurren a: modo `per_second_resolution`, `base_price` 0.0,
`price_per_second` 0.005, `price_per_frame` 0.0, `resolution_coeff {"*": 1.0}`,
moneda USD (`billing/video.rs:20-32`). Consulte las filas con
`billing.video.pricing.get` (JWT) y haga upsert con
`billing.video.pricing.set` (token de admin) — consulte
[API JSON-RPC](../api/jsonrpc.md). Consulte [Billing y uso](./billing-usage.md)
para saber cómo se agregan los registros de uso a la facturación mensual.

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
