---
title: "Autenticación y seguridad"
description: "Sesiones JWT, API keys, las tres puertas de admin, política de contraseñas, limitación de tasa de doble vía y el modelo de seguridad."
---

# Autenticación y seguridad

Arona autentica a los llamadores en dos vías: **tokens de sesión JWT** para los
clientes interactivos (el chat + la interfaz de admin, llamadas RPC) y **API
keys** (`arona-…`) para el tráfico programático compatible con OpenAI. Un token
de admin separado protege las superficies administrativas. Esta página
documenta la mecánica, el modelo de seguridad y los restos conocidos de bajo
riesgo de una auditoría de seguridad.

## Sesiones JWT

Las sesiones usan pares de tokens JWT access/refresh emitidos por el gestor de
tokens `kirino_session`:

- **TTL del access token: 900 segundos (15 minutos).**
- **TTL del refresh token: 604,800 segundos (7 días).**

Los access tokens autentican el plano JSON-RPC (`/api/rpc`) y `GET /v1/models`;
el sidecar SSE (`/api/rpc/events`) está indexado por su session id, una
capacidad acuñada durante las llamadas RPC autenticadas más que una credencial
bearer. Los endpoints `/v1/chat/completions`, `/v1/embeddings` y `/v1/video/*`
requieren una **API key** (allí no se acepta un JWT). Los access tokens son de
corta duración para que un token robado solo sea utilizable brevemente. Los
refresh tokens se canjean por pares nuevos mediante `auth.refresh`.

El refresh usa **rotación de familia de tokens**: consumir un refresh token lo
invalida y emite uno nuevo, y reutilizar un refresh token ya consumido revoca
toda la familia — `auth.refresh` responde con `AUTH_ERROR` y el mensaje
`Refresh token reused` (el error subyacente es `TokenReused`, "refresh token
has been reused — token family revoked"), y la cuenta debe iniciar sesión de
nuevo. La revocación de familia es **en memoria** (un conjunto
`revoked_families`): un reinicio del servidor la borra, así que la protección es
de mejor esfuerzo entre reinicios (el estado de sesión por usuario no sobrevive
a un reinicio).

El secreto de firma proviene de la variable de entorno `JWT_SECRET`. Fuera de
`MOCK_MODE=1`, el servidor **se niega a arrancar** si `JWT_SECRET` no está
definida o sigue siendo igual al secreto de desarrollo integrado, de modo que
una instancia de producción nunca puede servir por accidente tokens firmados con
una constante pública. Use un secreto robusto y aleatorio y no lo incluya nunca
en el repositorio.

## API keys

Las API keys son la credencial de máquina para la superficie compatible con
OpenAI:

- **Formato:** `arona-{uuid}`.
- **Almacenamiento:** solo se almacena el **hash SHA-256** de la clave en la
  tabla `api_keys` — el texto plano se devuelve exactamente una vez, en la
  respuesta de `keys.create`, y nunca se puede recuperar después.
- **Prefijo de clave:** los primeros 8 caracteres (`key_prefix`) se almacenan en
  claro para su visualización y atribución de uso; la interfaz muestra una forma
  enmascarada como `arona-XXXX…abcd`.
- **Revocación:** la búsqueda de claves une `api_keys.is_active = TRUE`, así que
  una clave revocada deja de validarse inmediatamente — no hay TTL de caché que
  esperar.

## Niveles de admin

Hay tres puertas de admin distintas, cada una con su propia credencial:

1. **Rutas `/api/admin/*`** — la gestión de backends y aliases
   (`POST/GET/DELETE /api/admin/backends`, `POST/GET/DELETE /api/admin/aliases`)
   requiere la cabecera `Authorization: Bearer ARONA_ADMIN_TOKEN`. Cuando
   `ARONA_ADMIN_TOKEN` no está definida, `check_admin` siempre falla y cada ruta
   de admin devuelve **401 "Admin access required"** — toda la superficie de
   gestión queda deshabilitada en lugar de abrirse.

2. **Métodos RPC `agents.*` y `engine.invoke`** — el clúster de agentes y el
   plano de control del motor requieren un JWT cuya cuenta tenga
   `users.is_admin = true`. Un no-admin autenticado se rechaza con el código
   definido por la implementación **-32007 (`ADMIN_REQUIRED`)** más una pista
   específica del método (p. ej. `agents.deploy starts model deployments on GPU
   nodes`); un llamador **sin autenticar** recibe el estándar **-32005
   (`AUTH_ERROR`)** para que el servidor no revele que el método es privilegiado.

3. **Métodos RPC `billing.plan.set` y `billing.video.pricing.set`** — las
   mutaciones de billing requieren el mismo Bearer `ARONA_ADMIN_TOKEN` que las
   rutas HTTP de admin; sin él devuelven `AUTH_ERROR` "Admin access required".

El **primer usuario registrado se convierte en el admin**
(`users.is_admin = true`). Cada registro posterior es un usuario normal, y el
registro solo está abierto mientras `ARONA_REGISTRATION_OPEN` esté definida con
un valor truthy.

## Política de contraseñas

Las contraseñas deben cumplir **ambas** reglas (aplicadas en el registro y en
cualquier ruta de cambio de contraseña):

- al menos **8 caracteres**, y
- al menos **3 de las 4 categorías de caracteres**: mayúsculas, minúsculas,
  dígitos, especiales.

## Limitación de tasa

La limitación de tasa se ejecuta en dos vías independientes; cualquiera de las
dos puede rechazar una solicitud con **429**:

### 1. Ventana deslizante en memoria (por identidad)

Cada solicitud `/v1` autenticada pasa por un limitador de ventana deslizante en
memoria indexado por la identidad del llamador:

- **Las llamadas con API key** se indexan por el **hash SHA-256** de la clave;
- **Las llamadas con JWT** se indexan por `u:<email>` — los JWTs rotan cada 15
  minutos, así que indexar la ventana por la instancia del token la
  restablecería en silencio en cada refresh.

El presupuesto predeterminado es de **60 solicitudes por minuto**, sobrescribible
con `ARONA_API_RATE_LIMIT_RPM` (defínalo más alto para pipelines de agentes que
disparan muchas llamadas LLM en paralelo). Definirlo en **0 bloquea todas las
solicitudes**.

### 2. Límite de tasa del tier (por clave, desde la base de datos)

Los billing tiers llevan un `rate_limit_rpm` por clave. La comprobación cuenta
las filas de `usage_records` del prefijo de la clave en la **ventana de los
últimos 60 segundos** (el uso se persiste después de cada respuesta, así que la
ventana se retrasa como mucho una solicitud en vuelo; los fallos de base de
datos abren (fail open)). El **tier free predefinido es de 10 RPM**; los tiers
pro/enterprise elevan el techo. La aplicación de la cuota mensual comparte la
misma ruta de rechazo.

### Limitación de tasa en el inicio de sesión

La adivinación de credenciales se limita en el endpoint de login: **5 intentos
fallidos por ventana de 5 minutos y por email** y **20 por ventana de 5 minutos
y por IP**, cada uno seguido de un bloqueo de 15 minutos.

### Retry-After

Cada respuesta 429 lleva una cabecera `Retry-After` para que los clientes
compatibles con OpenAI retrocedan en lugar de martillear el endpoint: los
rechazos por cuota la fijan en **segundos hasta el final del mes**; los rechazos
por límite de tasa la fijan en **60**. Consulte [Billing y uso](billing-usage.md)
para el modelo de cuotas.

## Notas sobre el modelo de seguridad

- **CORS permite cualquier origen** (`allow_origin(Any)`) — Arona es un backend
  consumido por muchos clientes de primera y tercera parte; si su despliegue
  debe restringir orígenes, póngalo detrás de un reverse proxy que aplique CORS.
- **Los cuerpos de solicitud están limitados a 1 MB** (`RequestBodyLimitLayer`),
  lo que acota el uso de memoria en el gateway.
- **El gateway no termina TLS** — escucha en HTTP plano. Póngalo detrás de un
  reverse proxy (consulte [Despliegue](deployment.md)) que termine HTTPS.
- **Los secretos provienen solo del entorno**: `ARONA_ADMIN_TOKEN` y
  `JWT_SECRET` se leen de variables de entorno y deben ser valores aleatorios
  robustos que nunca se incluyan en el repositorio.
- La dirección de bind predeterminada del servidor es `0.0.0.0`; restrinja la
  exposición en la capa de red.

## Restos conocidos de bajo riesgo (de la auditoría)

Lo siguiente se documenta tal cual; es intencional o aceptado por ahora, pero
vale la pena saberlo cuando exponga una instancia más allá de una red de
confianza:

- **`providers.list` es público**, mientras que `providers.add` /
  `providers.update` / `providers.remove` / `providers.test` requieren un JWT.
  La ruta de lectura pública revela el catálogo de providers pero nada secreto.
- **`/ws/agent` es un plano de control sin autenticar**: los agentes GPU se
  conectan sin credencial y se auto-registran (frames de `register` / `heartbeat`
  / resultado de comando). Cualquiera que pueda alcanzar el puerto WebSocket
  puede registrar un agente falso. Consulte [Clúster de agentes](agent-cluster.md)
  para las compensaciones operativas.
- **`memory.delete` es solo-JWT sin comprobación de propiedad**: cualquier
  usuario autenticado puede eliminar un nodo de memoria por `node_id`. Eliminar
  memoria requiere haber iniciado sesión, pero no ser dueño del nodo.

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
