---
title: "API REST compatible con OpenAI"
description: "Referencia /v1/* con estilo OpenAI — chat completions, embeddings, listado de modelos, generaciones de vídeo asíncronas, formas de error y límites de tasa."
---

# API REST compatible con OpenAI

Arona expone una superficie REST compatible con OpenAI bajo `/v1/*` para chat
LLM, embeddings, listado de modelos, probes de salud y generación de vídeo
asíncrona. Cualquier SDK de OpenAI apuntado a la URL base funciona para chat y
embeddings; los endpoints de vídeo siguen la convención de submit/poll estilo
task de OpenAI.

Todos los cuerpos de solicitud y respuesta son JSON. Los errores usan una forma
uniforme (consulte [Errores](#errors)); los fallos de autenticación en la capa
de middleware son la única excepción y se devuelven como texto plano (consulte
[Autenticación](#authentication)).

## Endpoints de un vistazo

| Método | Ruta | Descripción |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | Turno de chat, streaming o no streaming. |
| `POST` | `/v1/embeddings` | Vectores de embedding para una o muchas entradas. |
| `GET` | `/v1/models` | Modelos del router fusionados con los modelos de inicio rápido. |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`. |
| `POST` | `/v1/video/generations` | Enviar un task asíncrono de generación de vídeo. |
| `GET` | `/v1/video/generations/{id}` | Hacer poll del estado / resultado de un task de vídeo. |

`/api/health`, `/healthz` y `/readyz` son probes de readiness adicionales
(alias estilo Kubernetes de `/v1/health`).

## Autenticación

Los endpoints de chat, embeddings y vídeo se autentican con una **API key** en
la cabecera `Authorization: Bearer`. Las API keys se crean a través del plano
de gestión (`keys.create`, consulte la [API JSON-RPC](./jsonrpc.md#keys)) y
tienen la forma `arona-<uuid>`. Se almacenan del lado del servidor como hashes
SHA-256.

```
Authorization: Bearer arona-CHANGE_ME
```

- **Cabecera ausente** → `401` en texto plano: `Missing Authorization header. Use: Bearer <api_key>`.
- **Clave no válida o revocada** → `401` en texto plano: `Invalid API key`.
- `GET /v1/models` además acepta un **JWT** de access (emitido por `auth.login`
  / `auth.register`) para que el panel web pueda listar modelos con el mismo
  token que usa para el plano RPC. Para ese endpoint los mensajes son
  `Missing Authorization header. Use: Bearer <api_key_or_jwt>` y
  `Invalid API key or JWT`.

Los rechazos a nivel de middleware son cuerpos de texto plano, no la forma de
error JSON descrita en [Errores](#errors) — la forma JSON solo se produce una
vez que una solicitud llega a un handler.

Cada solicitud `/v1` autenticada también pasa por un **limitador de tasa en
memoria por clave** (predeterminado 60 RPM, ventana de 60 segundos,
configurable mediante `ARONA_API_RATE_LIMIT_RPM`). Superarlo devuelve `429` en
texto plano: `Rate limit exceeded. Try again later.` Las cuotas y límites de
tasa a nivel de tier se aplican por separado y devuelven 429 JSON con una
cabecera `Retry-After` (consulte [429 y Retry-After](#429-and-retry-after)).

> La gestión de API keys, proyectos y su delimitación se trata en
> [Autenticación y seguridad](../guides/auth-security.md).

## POST /v1/chat/completions

El endpoint de chat compatible con OpenAI central, con soporte de streaming y
extensiones específicas de arona (`conversation_id`, `memory`, `extra`,
`provider`).

### Cuerpo de la solicitud

| Campo | Tipo | Obligatorio | Notas |
| --- | --- | --- | --- |
| `model` | string | sí | Id de modelo tal como lo lista `GET /v1/models`. |
| `messages` | array | sí | Turnos de chat, consulte más abajo. |
| `stream` | boolean | no | Predeterminado `false`. Cuando es `true`, la respuesta es un stream SSE (consulte [Streaming](#streaming)). |
| `temperature` | number | no | Temperatura de muestreo, reenviada al upstream. |
| `max_tokens` | integer | no | Tope de tokens de completion, reenviado al upstream. |
| `conversation_id` | string | no | Afinidad de sesión + persistencia. La conversación debe existir y pertenecer al usuario de la API key (si no, `403` `conversation_forbidden`; `404` `conversation_not_found` si no existe). El turno del usuario se persiste en el momento del envío y la respuesta del asistente cuando el turno se completa; el routing fija la conversación al backend que la sirvió primero. |
| `memory` | boolean | no | Sobrescritura del gateway de memoria. Predeterminado `true` (la recuperación de memoria se inyecta cuando el gateway de memoria está activado); `false` deshabilita la inyección de recuperación para esta solicitud. |
| `extra` | object | no | Passthrough de forma libre fusionado en el nivel superior del payload del upstream (consulte más abajo). |
| `tools` | array | no | Definiciones de function-call con estilo OpenAI, pasadas tal cual al upstream. |
| `provider` | string | no | Pista explícita de selección de backend que coincide con un **nombre** de backend (o tipo) sin distinguir mayúsculas. Cuando está definido, solo los backends que coinciden con la pista son candidatos. |

**Las entradas `messages`** son `{ "role": "user" | "assistant" | "system", "content": "..." }`.
Se reenvían dos extensiones al upstream para cargas de trabajo multimodales /
de agentes:

- `images` — imágenes adjuntas para solicitudes de visión (un array de objetos
  `{ "media_type", "data", "position" }`; el backend external las representa
  como partes de contenido `image_url` de OpenAI).
- `tool_calls` — payloads de function-call producidos por el modelo del
  upstream, para devolverlos en los turnos posteriores. El backend external
  DEBE reenviarlos o los pipelines de agentes (p. ej. las cadenas de skills de
  scepter) pierden cada invocación de tool.

**Reglas de fusión de `extra`**: cada clave de `extra` se fusiona en el payload
de la solicitud del upstream en el nivel superior, con dos garantías duras — las
claves reservadas `model`, `messages`, `stream`, `temperature`, `max_tokens` y
`options` **nunca** se sobrescriben, ni tampoco ninguna clave que el propio
gateway ya haya definido. Los valores `extra` que no son objetos se ignoran.

**Las entradas `tools`** son `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }` y se reenvían tal cual.

### Respuesta no streaming

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1720000000,
  "model": "Qwen/Qwen3-1.7B",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 12, "completion_tokens": 2, "total_tokens": 14 }
}
```

- `choices[].message` puede llevar `tool_calls` para turnos de function-calling.
- El estado de memoria de la solicitud se refleja en la cabecera de respuesta
  **`X-Arona-Memory`**: `enabled` | `disabled` | `offline`.

### Streaming

Defina `"stream": true`. La respuesta es un stream SSE `text/event-stream` —
una línea `data:` por chunk, cada una con un único `ChatChunk` JSON:

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta` lleva `content` (y deltas de `tool_calls` con
  `index`/`id`/`type`/`function` para streams de function-calling).
- En los upstreams compatibles con OpenAI, el **chunk terminal** lleva un campo
  `usage` con los recuentos reales de tokens; el gateway lo registra (con un
  fallback a una estimación local del tokenizador para los upstreams que nunca
  informan uso, p. ej. ollama / ws_engine).
- El stream termina con `data: [DONE]`.
- Un error de stream se entrega como un evento `data:` que lleva
  `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}`;
  el evento `[DONE]` sigue igualmente, y el registro de uso más la persistencia
  del asistente se omiten para el stream fallido.
- La cabecera `X-Arona-Memory` también está presente en la respuesta SSE.

### Ejemplo

```bash
curl http://192.0.2.10:8080/v1/chat/completions \
  -H "Authorization: Bearer arona-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-1.7B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

## POST /v1/embeddings

| Campo | Tipo | Obligatorio | Notas |
| --- | --- | --- | --- |
| `model` | string | sí | Id de modelo de embedding (p. ej. `nomic-embed-text` — un nombre simple también coincide con una tag `:latest`). |
| `input` | string o string[] | sí | Una entrada, o muchas. |

Respuesta: `{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`.

## GET /v1/models

Lista los modelos enrutables hoy: el listado de modelos de cada backend
saludable registrado, fusionado con los **modelos de inicio rápido** integrados
(siempre anunciados, incluso antes de registrar un backend): `Qwen/Qwen3-0.6B`,
`Qwen/Qwen3-1.7B`, `HuggingFaceTB/SmolLM2-1.7B-Instruct`,
`google/gemma-3-1b-it`, `microsoft/Phi-4-mini-instruct`,
`deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`.

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

Los modelos de inicio rápido aparecen con `owned_by` definido a su provider;
los modelos del router llevan el nombre del backend propietario.

## Generación de vídeo

Endpoints de vídeo estilo task para backends con capacidad de vídeo (p. ej.
`minimax-cloud`, consulte [Backends](../guides/backends.md)). Los trabajos
avanzan de forma asíncrona; haga poll del endpoint de estado hasta `done`.

### POST /v1/video/generations

| Campo | Tipo | Obligatorio | Notas |
| --- | --- | --- | --- |
| `model` | string | sí | Id de modelo de vídeo registrado en un backend con capacidad de vídeo. |
| `prompt` | string | sí | Prompt de generación. |
| `negative_prompt` | string | no | Prompt negativo. |
| `images` | array | no | Imágenes de acondicionamiento/referencia como un array de objetos `{ "data_base64": "...", "mime_type": "image/png" }`. |
| `duration_seconds` | integer | no | Duración solicitada. |
| `width` / `height` | integer | no | Resolución de salida. |
| `provider` | string | no | Pista explícita de selección de backend (nombre del backend). |
| `extra` | object | no | Sobrescrituras de workflow específicas del backend (seed, steps, cfg, ...). |

Éxito → `200`:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

Errores: `400` `missing_fields` cuando `model` o `prompt` están ausentes; `503`
`video_backend_error` / `no_backend` cuando ningún backend saludable con
capacidad de vídeo sirve el modelo; `429` `quota_error` / `quota_exceeded`
cuando la cuota mensual está agotada.

### GET /v1/video/generations/{id}

Devuelve el estado del task:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "running",
  "progress": 40,
  "result": null,
  "error": null,
  "cost": 0.0,
  "created_at": 1720000000
}
```

- `status`: `queued` | `running` | `done` | `failed` | `cancelled`; `progress`
  avanza 0-90 mientras se ejecuta y llega a 100 en `done`.
- `result` (en `done`): `{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }` — `url` apunta al artefacto generado servido por el backend.
- `error` (en `failed` / `cancelled`) y `cost` se rellenan cuando corresponde.
- Errores: `400` `bad_id` para un id que no es UUID; `404` `no_job` cuando el
  trabajo no existe o pertenece a otra API key.

Los trabajos de vídeo también difunden el progreso por el sidecar SSE RPC
(`video.progress` / `video.done` / `video.failed`, consulte
[Eventos y notificaciones](./events.md#video-job-notifications)).

## Errores

Los errores a nivel de gateway usan una sola forma (`json_error_response`):

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| Estado | `type` / `code` | Cuándo |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`, `missing_index`, `bad_id`, ... | Campos de solicitud malformados o ausentes. |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id` pertenece a otro usuario. |
| `404` | `invalid_request_error` / `model_not_found` | Ningún backend sirve el modelo solicitado. Mensaje: `No backend available for model: <model>`. |
| `404` | `invalid_request` / `conversation_not_found` | Conversación no encontrada. |
| `404` | `not_found` / `no_job` | Trabajo de vídeo no encontrado. |
| `502` | `server_error` / `bad_gateway` | Upstream no-2xx: mensaje `upstream <status>: <detail>` (detalle del cuerpo de error del upstream, acotado a 4 KB). Los fallos de transporte (connect/read/timeout) también se asignan a 502 con la cadena del error. |
| `500` | `server_error` / `backend_error` | Otros fallos de backend (p. ej. el backend no soporta la operación). |
| `500` | `server_error` / `internal_error` | Cualquier error interno restante del gateway. |
| `429` | consulte más abajo | Rechazos de cuota / límite de tasa con `Retry-After`. |

## 429 y Retry-After

Las respuestas 429 incluyen una cabecera `Retry-After` (segundos) para que los
clientes compatibles con OpenAI retrocedan:

| Disparador | Cuerpo del estado | `Retry-After` |
| --- | --- | --- |
| Cuota mensual superada | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | Segundos hasta el mes siguiente. |
| Límite de tasa por minuto del tier | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`. |
| Limitador en memoria por clave (60 RPM predeterminado) | texto plano `Rate limit exceeded. Try again later.` | ninguno (rechazo de middleware). |

Los tiers, la delimitación de cuotas y la contabilidad de uso se describen en
[Billing y uso](../guides/billing-usage.md).

## Registro de uso

Cada solicitud `/v1` registra una fila de uso bajo el prefijo de la API key
(`arona-XX`) cuando se completa (chat no streaming, chat streaming en el chunk
terminal, embeddings, y trabajos de vídeo al completarse con su coste
calculado). Consulte [Billing y uso](../guides/billing-usage.md) para el modelo
de registro y cómo se aplica la cuota.

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
