---
title: "Billing y uso"
description: "Registros de uso, coste por modelo, billing tiers, aplicación de cuotas y límites de tasa, claves delimitadas por proyecto, precios de vídeo y el RPC usage.list."
---

# Billing y uso

Arona contabiliza cada solicitud de modelo y aplica cuotas y límites de tasa
por tiers en el gateway. Los precios por modelo provienen de la tabla de precios
compartida de plana (nunca reimplementada en arona), el uso se persiste como
filas de `usage_records` y todo el panorama mensual se expone a través del RPC
`usage.list`.

## Registros de uso

Cada solicitud contabilizada termina como una fila en la tabla `usage_records`
(`m20250101_000006_create_usage_records`):

| Columna | Tipo | Significado |
| --- | --- | --- |
| `id` | `UUID` | Clave primaria, generada. |
| `api_key_id` | `VARCHAR(64)` | El **prefijo de clave** — los primeros 8 caracteres de la API key (las claves tienen la forma `arona-{uuid}`) — o un id sintético `jwt-<user-uuid>` para los canales RPC atribuidos por JWT. |
| `model` | `VARCHAR(128)` | Id de modelo al que se enrutó la solicitud. |
| `backend` | `VARCHAR(64)` | Tipo de backend: `gateway`, `rpc`, `realtime` o el nombre de la capacidad del backend. |
| `prompt_tokens` | `INTEGER` | Tokens de entrada, informados por el upstream o estimados. |
| `completion_tokens` | `INTEGER` | Tokens de salida, informados por el upstream o estimados. |
| `total_tokens` | `INTEGER` | Suma de ambos. |
| `cost` | `DOUBLE PRECISION` | Coste USD calculado; `NULL` cuando el modelo no tiene fila de precios. |
| `created_at` | `TIMESTAMPTZ` | Cuándo se completó la solicitud. |

Existen índices sobre `api_key_id`, `model` y `created_at` (las columnas que
escanean la agregación mensual y las ventanas de límite de tasa).

## Canales de registro

El uso se registra en cada canal contabilizado:

1. **REST no streaming** — `POST /v1/chat/completions` y `POST /v1/embeddings`
   registran el uso exacto informado por el upstream una vez producida la
   respuesta.
2. **REST streaming (SSE)** — el uso informado por el upstream gana cuando el
   stream lo trajo (campo `usage` del chunk terminal compatible con OpenAI); en
   caso contrario se registra tal cual una estimación local de un tokenizador
   consciente de CJK (`estimate_usage`). Los streams que no produjeron ni texto
   ni uso **no** se registran en absoluto.
3. **RPC `chat.send`** — se aplica la misma lógica estimación-vs-upstream; las
   filas se atribuyen con el id sintético `jwt-<user-uuid>` para que se unan de
   vuelta al usuario.
4. **Sesiones realtime** — cada transcripción `response_done` completada
   registra su uso de tokens (cuando no es cero) bajo `jwt-<user-uuid>` con el
   backend `realtime`.
5. **Trabajos de vídeo** — un trabajo completado registra un coste en dólares
   explícito (consulte [Precios de vídeo](#video-pricing)); los recuentos de
   tokens son cero.

El registro es de mejor esfuerzo: un insert fallido se registra en el log y
nunca hace fallar la solicitud.

## Cálculo del coste

El coste se calcula a partir de la tabla canónica de precios por millón de
tokens (`plana_llm_provider::metering::lookup_pricing`, compartida por todos los
servicios de celestia-island):

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

La coincidencia de modelos en la tabla se basa en subcadenas sobre el id de
modelo en minúsculas (ganan las familias más específicas). Cuando un modelo no
tiene fila de precios, `cost` es `NULL`. **No reimplemente los precios en arona
— actualice la tabla de plana.**

## Tiers

Los tiers viven en la tabla `billing_tiers`, sembrada en la primera migración
(`m20250101_000007_create_billing_tiers`). Una columna de cuota `NULL` significa
"ilimitado" para esa dimensión. Los usuarios sin `tier_id` recurren al tier
`free` predefinido.

| Tier | Cuota USD mensual | Cuota de tokens mensual | RPM por clave |
| --- | --- | --- | --- |
| `free` | $1.00 | 100,000 | 10 |
| `pro` | $20.00 | 5,000,000 | 120 |
| `enterprise` | ilimitado (`NULL`) | ilimitado (`NULL`) | 1000 |

La asignación de tier es una operación de admin (RPC `billing.plan.set`); el
tier actual y el uso se muestran mediante `billing.plan`.

## Aplicación de cuotas y límites de tasa

### REST (`/v1/*`)

Antes de cada endpoint REST **contabilizado** — `/v1/chat/completions`,
`/v1/embeddings` y `/v1/video/generations` — el gateway aplica dos puertas a las
solicitudes autenticadas con clave:

- **Cuota mensual** — los límites `monthly_quota_usd` y `quota_tokens` del tier
  se comprueban contra el uso acumulado desde el primer instante del mes
  natural actual. Que cualquiera de las dos dimensiones alcance su límite
  dispara la puerta.
- **Límite de tasa por clave** — el `rate_limit_rpm` del tier se comprueba
  contra el número de solicitudes registradas para este prefijo de clave en la
  ventana de los últimos 60 segundos. (`/v1/models` y los endpoints de salud no
  se contabilizan ni se someten a puerta.)

Un rechazo es un **429 Too Many Requests** HTTP con una cabecera `Retry-After`
y un cuerpo de error con estilo OpenAI:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| Rechazo | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| Cuota mensual agotada | `quota_error` / `quota_exceeded` | Segundos hasta el inicio del **mes natural siguiente** |
| Límite de tasa del tier superado | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

El `chat.send` autenticado con JWT pasa por la misma puerta de cuota mensual,
pero contra la ventana de **todo el usuario** (la llamada no lleva API key). Un
rechazo es un error JSON-RPC con el código definido por la implementación
`-32006` (`QUOTA_ERROR`) y el mismo mensaje que el rechazo de cuota REST. No hay
límite de tasa por clave en la ruta RPC — la limitación de tasa está delimitada
por clave y las llamadas RPC no tienen clave. Los métodos **RPC** de realtime y
vídeo no se someten a puerta de cuota.

## Compensación fail-open

El billing es **de mejor esfuerzo por diseño**. Si la consulta de base de datos
detrás de una comprobación de cuota o límite de tasa falla, la comprobación
devuelve `Unknown` y la solicitud se **permite** (solo se registra en el log) en
lugar de bloquear el chat. Un operador puede confiar en los 429 para proteger la
capacidad, pero no debe tratarlos como una garantía dura cuando la base de datos
no está saludable — la compensación documentada es la disponibilidad de la ruta
de chat por encima de la contabilización estricta.

## Claves delimitadas por proyecto

Las API keys se pueden crear con una etiqueta `project` (`api_keys.project`,
`default` cuando no está definida). La aplicación de cuotas la respeta:

- Una clave etiquetada con un proyecto distinto de `default` comprueba su cuota
  contra el uso atribuido al **bucket propio de ese proyecto**
  (`PROJECT_MONTHLY_USAGE_SQL`).
- Las claves `default` / sin etiqueta conservan la ventana de **todo el
  usuario**, con el comportamiento anterior a los proyectos.

Las filas RPC atribuidas por JWT (`jwt-<user-uuid>`) no llevan etiqueta de
proyecto y se **excluyen a propósito** de las ventanas de proyecto — siguen
contando para la ventana de todo el usuario, así que un proyecto no se puede
"ocultar" enviando tráfico por el canal RPC.

## Precios de vídeo

La generación de vídeo usa precios específicos de modelo, estilo task (los
precios por token no tienen sentido para un vídeo). Las filas de precios viven
en la tabla `video_pricing`; `compute_cost` recurre a un valor predeterminado
placeholder cuando no hay fila configurada.

| Modo | Coste |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (predeterminado) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` es un objeto JSON indexado por el valor de píxel del lado
corto (p. ej. `"768"`); la clave `"*"` es el fallback. La fila de precios
predeterminada es el modo `per_second_resolution`, `base_price` 0.0,
`price_per_second` 0.005, `resolution_coeff {"*": 1.0}`. Las filas se gestionan
mediante `billing.video.pricing.get` (cualquier JWT) y
`billing.video.pricing.set` (Bearer `ARONA_ADMIN_TOKEN`); el coste calculado se
registra contra la clave del trabajo cuando el trabajo se completa.

## usage.list

`usage.list` (JWT) devuelve los registros de uso paginados del llamador que
cubren **tanto** las filas de API key (unidas por el prefijo de clave) como las
filas atribuidas por JWT (unidas por el id sintético `jwt-<user-uuid>`), las
más recientes primero:

| Parámetro | Predeterminado | Notas |
| --- | --- | --- |
| `limit` | `50` | Acotado a `1..=200`. |
| `offset` | `0` | Desplazamiento de página. |
| `project` | sin definir | Cuando está definido, solo los registros atribuidos a claves con esa etiqueta de proyecto (las filas JWT se excluyen). |

La respuesta es `{ "records": [...], "total", "limit", "offset", "project" }`,
donde cada registro lleva `id`, `model`, `backend`, `prompt_tokens`,
`completion_tokens`, `total_tokens`, `cost` y `created_at`. La agregación de la
cuota mensual usa la misma forma de unión, así que el cálculo de la cuota y la
vista de uso siempre coinciden en el ámbito.

## Relacionados

- [Inicio rápido](quickstart.md) — obtenga una clave y haga su primera solicitud contabilizada.
- [Configuración](configuration.md) — variables de entorno para el gateway.
- [Autenticación y seguridad](auth-security.md) — creación de API keys y la etiqueta `project`.
- [Realtime y vídeo](realtime-video.md) — el ciclo de vida de los trabajos de vídeo detrás de los precios de vídeo.
- [Operaciones](operations.md) — probes de salud y observabilidad.
- [API REST compatible con OpenAI](../api/openai-rest.md) — la superficie `/v1/*`.
- [API JSON-RPC](../api/jsonrpc.md) — `usage.list`, `billing.plan`, `billing.video.pricing.*`.
- [Visión general](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
