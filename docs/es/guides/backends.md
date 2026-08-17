---
title: "Backends"
description: "Tipos de backend (external, ollama, engine, minimax-cloud, bridges de evernight), semántica de URLs, probes de salud, descubrimiento de modelos, aliases y routing."
---

# Backends

Un **backend** es un upstream que sirve tráfico de modelos. Arona enruta las
solicitudes compatibles con OpenAI (`/v1/chat/completions`, `/v1/embeddings`,
listado de modelos, trabajos de vídeo) hacia uno de los backends registrados,
contabiliza cada solicitud y mantiene actualizados la salud y el inventario de
modelos de cada backend.

Los backends los registra un admin mediante `POST /api/admin/backends`
(consulte la [API HTTP de administración](../api/admin-http.md)), se persisten
en la tabla `backend_configs` y se restauran automáticamente al arrancar. Cada
registro lleva un `name`, un `type`, una `url`, un `api_key` opcional y una
lista estática `models` opcional. Los backends persistidos sobreviven a los
reinicios; los backends restaurados arrancan en fail-closed y se someten a un
probe inmediatamente.

## Tipos de backend

| `type` | Transporte | Protocolo | Propósito |
| --- | --- | --- | --- |
| `external` | HTTP(S) | REST compatible con OpenAI | Cualquier API de chat/embeddings (cloud o autoalojada) |
| `ollama` | HTTP(S) | API nativa de Ollama (`/api/chat`, `/api/embed`, `/api/tags`) | Un servidor Ollama local o remoto; se construye solo con la URL |
| `engine` | `ws://` / `wss://` | CEP (Celestia Engine Protocol), WebSocket + JSON-RPC | Cualquier motor que hable el estándar de intercambio CEP (`plana::engine`) |
| `minimax-cloud` | HTTPS | API estilo task de MiniMax H3 (submit + poll) | Generación de vídeo en la nube |
| `evernight://<node>/<service>` | URL de bridge | Se resuelve a través del agente evernight local en un reenvío TCP local | Servicios industriales/edge accesibles solo a través de la mesh de evernight |
| `agent-{model}` | HTTP | OpenAI-compatible (external) | Auto-registrado cuando un agente GPU despliega un modelo |

### `external` — cualquier API HTTP compatible con OpenAI

El backend de propósito general: chat completions (streaming y no streaming) y
embeddings contra cualquier servidor que hable la forma REST de OpenAI.
Configúrelo con una `url` base, un `api_key` (opcional) y una lista estática
`models` opcional:

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

La lista estática `models` es autoritativa: se fusiona por delante de cualquier
modelo descubierto en el momento del probe (consulte
[Descubrimiento de modelos](#model-discovery)).

### `ollama` — se construye solo con la URL

Un backend Ollama se construye solo con la URL — sin API key, sin lista de
modelos. Habla los protocolos nativos de Ollama: `/api/chat` para chat,
`/api/embed` para embeddings y `/api/tags` para probes de salud y
descubrimiento de modelos.

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — CEP sobre WebSocket

Un backend `engine` se conecta a un motor que expone `ws://` (o `wss://`) y
habla el **Celestia Engine Protocol** (CEP): un estándar de intercambio
WebSocket + JSON-RPC 2.0 definido en `plana::engine`. Cualquier motor escrito
en cualquier lenguaje que implemente el flujo handshake → métodos →
notificaciones de streaming se registra como backend de primera clase sin
código de adaptación en Arona. Métodos de la vía: `Engine.Handshake` (primer
mensaje; identidad + capacidades), `Engine.Chat`, `Engine.ChatStart` (streaming;
los chunks llegan como notificaciones `Engine.ChatChunk`), `Engine.Embeddings` y
`Engine.Models`. Las conexiones se establecen de forma diferida en el primer uso
y se cierran ante cualquier error; la siguiente llamada reconecta y rehace el
handshake.

### `minimax-cloud` — generación de vídeo estilo task

El backend de vídeo en la nube maneja la API de la plataforma abierta MiniMax
H3: enviar un task de generación, hacer poll hasta la finalización y luego leer
la URL del artefacto del resultado. Esto es lo que sustituyó al backend ComfyUI
eliminado (consulte más abajo); los trabajos de vídeo se envían mediante
`/v1/video/generations` o los métodos RPC `video.*` y avanzan con las
notificaciones `video.progress` / `video.done` / `video.failed` (consulte
[Realtime y vídeo](realtime-video.md)).

### URLs de bridge `evernight://`

Una URL de backend de la forma `evernight://<node>/<service>` **no** se
contacta directamente. El agente evernight del host local la resuelve (una
llamada JSON-RPC `Bridge.Connect` sobre el endpoint WebSocket del agente) en un
reenvío TCP local, y el backend opera contra `http://127.0.0.1:<local_port>` en
lugar de una dirección remota fija. Esta es la arquitectura de panel único: el
panel de Arona alcanza servicios en otros nodos (motores CEP, scepter, ...) a
través de la mesh sin incrustar nunca una dirección remota en una configuración.

Una tarea keepalive somete a probe el túnel cada 15 segundos; cuando el lado
remoto se reinicia y el túnel se restablece en un puerto local nuevo, el backend
afectado se **reconstruye de forma transparente** con la nueva URL — la
configuración persistida conserva la URL `evernight://` para que los reinicios
la vuelvan a resolver. Para los backends `engine`, el reenvío resuelto
`http://127.0.0.1:<port>` se adapta a `ws://` para el transporte WebSocket.

### Los modelos desplegados por agentes se auto-registran

Cuando un agente GPU termina de desplegar un modelo, el gateway registra un
backend llamado `agent-{model_id}` (un `ExternalApiBackend` sobre
`http://{agent host}:{port}`) para que el modelo sea enrutable de inmediato;
detener el despliegue lo des-registra de nuevo. Consulte
[Clúster de agentes](agent-cluster.md) para el ciclo de vida completo del
despliegue.

### `comfyui` es rechazado

El tipo de backend `comfyui` se rechaza explícitamente con el error
`comfyui backend removed`: el backend ComfyUI se eliminó durante la
convergencia de la plataforma de modelos, y la generación de vídeo ahora se
ejecuta a través de `minimax-cloud`. Registrar un backend `comfyui` devuelve un
HTTP 400.

## Semántica de URLs

El modo en que una URL base configurada se asigna a endpoints reales lo decide
si la URL tiene un componente de ruta:

- **Base de estilo raíz** — una URL cuya ruta está vacía o es `/` se trata como
  raíz de host y conserva la convención OpenAI `/v1`: `{base}/v1/chat/completions`,
  `{base}/v1/models`. Ejemplos: `http://192.0.2.20:8429`,
  `https://api.deepseek.com`.
- **Base de estilo ruta** — una URL con una ruta no vacía se trata como el
  prefijo completo de la API que el servidor sirve realmente, y el endpoint se
  añade directamente: `{base}/chat/completions`, `{base}/models`. Esto es lo
  que necesitan los servidores compatibles con OpenAI fuera de la convención
  `/v1`. El plan de coding Zhipu GLM es el ejemplo canónico: su API vive en
  `https://open.bigmodel.cn/api/coding/paas/v4` con chat directamente en
  `{base}/chat/completions` y **ningún endpoint `/models` en absoluto** — la
  raíz estándar `/api/paas/v4` devuelve errores de saldo para las claves del
  plan de coding.
- Una **barra final** en la URL base configurada se normaliza para que la
  unión nunca produzca una doble barra.

## Probing y salud

Un verificador de salud en segundo plano somete a probe cada backend registrado
cada **60 segundos**; la lista de backends se obtiene de nuevo en cada ronda,
así que los backends registrados después del arranque se recogen sin reiniciar.
Cada registro de admin también dispara un probe inmediato para que el backend
pase a saludable en ~1-2 segundos en lugar de esperar a la siguiente ronda del
verificador.

- **Los backends external** someten a probe `GET {base}/models` (o
  `{base}/v1/models` para bases de estilo raíz) con un **timeout de 2
  segundos**. Un **404 se tolera**: algunos servidores implementan chat pero no
  exponen un listado de modelos (el plan de coding GLM no tiene endpoint
  `/models`), así que un 404 marca el backend como saludable y la lista `models`
  configurada por el admin se convierte en la fuente del routing. Los timeouts,
  los fallos de red y otras respuestas no-2xx marcan el backend como no
  saludable.
- **Los backends Ollama** someten a probe `/api/tags` con el mismo timeout de 2
  segundos.
- Los backends arrancan en **fail-closed** — informados como `not probed yet` —
  hasta el primer probe con éxito, así que un backend recién registrado (o
  restaurado) nunca recibe tráfico antes de haber sido verificado.

El estado de salud se cachea por backend y el router lo consulta en cada
solicitud; los backends no saludables se excluyen de la selección de candidatos
(consulte [Routing](#routing)).

## Descubrimiento de modelos

Un backend anuncia los ids de modelo que sirve, y el router compara las
solicitudes con ese anuncio:

- **External**: anuncian los modelos parseados de la respuesta del probe (se
  aceptan tanto un array `data` como uno `models`), fusionados con la lista
  estática configurada por el admin — los ids estáticos conservan el orden y la
  precedencia, los ids dinámicos se deduplican y se añaden al final. Cuando un
  servidor no tiene endpoint de modelos, la lista estática por sí sola es la
  fuente del routing.
- **Ollama**: anuncian las tags devueltas por `/api/tags`.
- **Desplegados por agentes**: anuncian exactamente el `model_id` desplegado.

La superficie pública es `GET /v1/models` (autenticado), que lista los modelos
enrutables de cada backend saludable (consulte la
[API REST compatible con OpenAI](../api/openai-rest.md)).

## Aliases y normalización de nombres

Los aliases asignan un id de modelo solicitado a un id de destino. El alias se
resuelve primero en el routing, así que una solicitud para el alias la sirve el
backend que anuncie el destino:

```json
{ "alias": "fast-chat", "target": "deepseek-chat" }
```

Los aliases se gestionan mediante los endpoints de admin `/api/admin/aliases` y
surtan efecto de inmediato.

La coincidencia de nombres también normaliza las tags estilo Ollama: un backend
que lista `nomic-embed-text:latest` coincide con una solicitud simple de
`nomic-embed-text`, de modo que las solicitudes de embeddings/chat se resuelven
sin llevar la contabilidad del sufijo `:latest`. Una tag explícita
(`qwen3:0.6b`) coincide solo con esa tag exacta.

## Routing

Cada solicitud se resuelve a través del router, que selecciona un backend:

1. **Resolución de alias** — el id de modelo solicitado se asigna mediante la
   tabla de aliases (si existe).
2. **Pista de provider** — un campo `provider` opcional filtra los candidatos
   por nombre de backend (o nombre de tipo, p. ej. `cloud` para los backends de
   vídeo).
3. **Solo candidatos saludables** — un backend debe informar `Healthy` *y*
   superar su circuit breaker (3 fallos consecutivos abren el breaker durante 30
   segundos, con una llamada de prueba half-open) para ser seleccionable.
4. **Selección por menor carga** — los candidatos que sirven el modelo se
   ordenan por su contador de solicitudes por backend y se elige el menos
   cargado. Esto reparte la carga entre los backends saludables que sirven el
   mismo modelo.
5. **Afinidad de sesión** — cuando una solicitud lleva un `conversation_id`, la
   conversación se fija al backend que la usó por primera vez. La fijación vive
   en un mapa de referencias `Weak`, así que un backend eliminado desaparece del
   mapa sin deriva de índices. La afinidad es de mejor esfuerzo: reutilizar el
   mismo backend entre los turnos de una conversación permite que el upstream
   reutilice el estado de runtime por conversación (contextos calientes, cachés
   KV). Si el backend fijado se ha vuelto no saludable (o la fijación ha muerto
   con un backend eliminado), el router recurre a una selección nueva por menor
   carga y **re-fija** la conversación.

Si ningún backend saludable sirve el modelo, el routing falla: un modelo
desconocido produce `model not found` (HTTP 404), un modelo conocido pero
inalcanzable produce `all backends unhealthy`, que se muestra como un error 500
de servidor interno. El HTTP 502 se reserva para los fallos informados por un
upstream *alcanzable* (respuestas upstream no-2xx y fallos de transporte después
de la selección). Consulte [Operaciones](operations.md) para el mapeo de errores
completo.

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
