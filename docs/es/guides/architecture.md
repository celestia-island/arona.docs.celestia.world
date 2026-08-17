---
title: "Arquitectura"
description: "Cómo está ensamblado Arona — estructura del workspace, la ruta de la solicitud a través del gateway, routing, probes de salud, memoria, sesiones y las compensaciones de diseño deliberadas."
---

# Arquitectura

Esta página recorre cómo está estructurado Arona y cómo fluye una solicitud a
través de él: la estructura del workspace, la ruta de la solicitud, el gateway
y el router, la comprobación de salud, el cliente de memoria, las sesiones y
las notificaciones y, por último, los límites y compensaciones deliberados que
acepta el diseño. Consulte [inicio rápido](quickstart.md) para un ejemplo en
funcionamiento y [operaciones](operations.md) para las preocupaciones de
runtime del día a día.

## Estructura del workspace

El repositorio es un workspace de Cargo con tres paquetes:

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC,
│            # backends, migration, entities, providers, models, registry,
│            # orchestration
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

- `packages/core` es el crate de biblioteca (`_core`). Contiene todo lo que el
  servidor necesita: el gateway axum (`gateway/`), el router de modelos
  (`routing/`), los adaptadores de backend (`backends/`), billing (`billing/`),
  auth (`auth.rs`), el cliente de memoria (`memory/`), el plano JSON-RPC
  (`gateway/rpc.rs`), el esquema (`migration/`, `entity/`), los metadatos de
  modelos (`models/`, `providers/`, `registry/`) y la orquestación de modelos
  (`orchestration/`).
- `packages/agent` compila el binario `_agent` que se ejecuta en los nodos GPU y
  se conecta de vuelta por `/ws/agent` (consulte
  [clúster de agentes](agent-cluster.md)).
- `packages/cli` compila el binario `_cli` usado para las operaciones install,
  deploy, serve, migrate y download.

Ya no hay panel web en este repositorio: el dashboard Vue se eliminó y ahora
vive en [shittim-chest](https://github.com/celestia-island/shittim-chest)
(chest #291), que se comunica con Arona por la superficie JSON-RPC. Arona en sí
es puramente backend (consulte la [visión general](./README.md)).

## Ruta de la solicitud

El punto de entrada es el router axum ensamblado en `GatewayServer::app`
(`packages/core/src/gateway/server.rs`). Su tabla de rutas (server.rs:128-163)
cubre la superficie REST compatible con OpenAI (`/v1/chat/completions`,
`/v1/embeddings`, `/v1/models`, `/v1/health`), la generación de vídeo, el
endpoint JSON-RPC `/api/rpc` (POST + actualización WebSocket), el sidecar SSE
`/api/rpc/events`, el plano de control de agentes `/ws/agent`, la interfaz
Swagger en `/docs` y los endpoints de admin de gestión de backends/aliases.

El router está envuelto en una pequeña pila de capas (server.rs:158-162):

1. Gestores de auth como `Extension`s para que los extractores por handler
   puedan alcanzarlos.
2. Una capa de request-id que reutiliza una cabecera `X-Request-ID` entrante o
   genera una, exponiéndola a los handlers y los logs (`gateway/request_id.rs`).
3. Un límite de cuerpo de solicitud de 1 MB (`RequestBodyLimitLayer`).
4. Una capa CORS permisiva (origen `*`, cabeceras `*`).

Como axum aplica las capas de abajo hacia arriba, la capa CORS es la más
externa.

Cada handler `/v1/*` pasa entonces por el mismo esqueleto:

1. **Extracción de auth** — `ApiKeyAuth` para los endpoints solo de clave
   (`/v1/chat/completions`, `/v1/embeddings`, vídeo) y `ApiKeyOrJwt` para
   `GET /v1/models`, que debe aceptar tanto API keys como JWTs de sesión
   (`gateway/middleware.rs`). El extractor resuelve la clave/JWT en un email de
   usuario, un prefijo de clave, una clave de límite de tasa (el hash SHA-256 de
   la API key, o una etiqueta `u:<email>` para los JWTs para que los tokens
   rotatorios no restablezcan la ventana) y un ámbito de proyecto opcional.
2. **Puertas de billing** — `enforce_billing_gates` (server.rs:492-539) rechaza
   la solicitud con HTTP 429 + `Retry-After` cuando se supera la cuota mensual
   del tier del usuario o el límite de tasa por minuto. Los fallos de base de
   datos abren: el billing es de mejor esfuerzo, nunca una dependencia dura de
   servir un chat.
3. **Recuperación de memoria** (rutas de chat) — si el cliente de memoria está
   configurado y la solicitud lo pide, las memorias relevantes a largo plazo se
   inyectan como sección de sistema (consulte
   [Cliente de memoria](#memory-client) más abajo). El fallo nunca bloquea el
   chat; el estado resultante se refleja en la cabecera `X-Arona-Memory`.
4. **Persistencia de conversación** — un `conversation_id` opcional se
   comprueba en cuanto a propiedad, y el turno del usuario se persiste en el
   momento del envío.
5. **Despacho del gateway** — la solicitud se entrega al `Gateway`, que resuelve
   un backend, recorta el contexto y llama al trait del backend.
6. **Registro de uso** — el uso devuelto (o el terminal del stream) se registra
   y se persiste a través del `UsageTracker` bajo el prefijo de clave.

El `Gateway` en sí vive en `AppState` como un `Arc<Gateway>` — no hay mutex
externo; la mutabilidad interior evita que las llamadas concurrentes de
chat/embeddings/stream tengan que mantener un lock a través de un round trip
HTTP al upstream (`gateway/mod.rs:29-53`).

## Gateway y router

El `Gateway` (`packages/core/src/gateway/mod.rs`) es dueño de:

- **Estado del router** — la lista de backends y los aliases, protegidos por un
  `tokio::sync::RwLock`. La resolución del lado de lectura toma prestado a
  través de awaits; las mutaciones (register/remove/alias) toman un write lock
  corto y nunca lo mantienen a través de una llamada al upstream.
- **Un contador de solicitudes** (`AtomicU64`) y un `start_time` usados por
  `system.status` y los endpoints de salud.
- **Un mapa de despliegues** (`model_id → nombre de backend`) para los modelos
  desplegados por agentes. `register_agent_backend` construye un
  `ExternalApiBackend` llamado `agent-{model_id}` y lo inserta en el router;
  re-registrar el mismo modelo sustituye al backend anterior, y
  `unregister_agent_backend` lo elimina en un frame `stop_result` (consulte
  [clúster de agentes](agent-cluster.md)).

La resolución de backends ocurre en el `Router` (`packages/core/src/routing/mod.rs`):

1. **Resolución de alias** — un alias configurado se reescribe a su destino.
2. **Afinidad de sesión** — cuando hay un `conversation_id`, el router
   comprueba un mapa de referencias débiles que fija la conversación al backend
   que la sirvió por primera vez. Las referencias débiles mantienen vivo el mapa
   solo mientras el backend esté registrado o en vuelo, así que los backends
   eliminados desaparecen sin deriva de índices. Un circuit breaker disparado o
   un backend fijado no saludable degrada a una selección nueva, que re-fija la
   conversación.
3. **Filtrado de candidatos** — una pista `provider` opcional filtra por
   nombre/tipo de backend; los candidatos deben estar saludables *y* tener un
   circuit breaker abierto, y deben listar el modelo solicitado. Los ids de
   modelo coinciden exactamente o mediante la convención del sufijo `:latest`
   (una solicitud simple de `nomic-embed-text` coincide con un
   `nomic-embed-text:latest` listado).
4. **Selección por menor carga** — los candidatos supervivientes se ordenan por
   su contador de hits y se elige el menos cargado; la fijación de conversación
   (si existe) se registra al mismo tiempo.

Antes de llamar al backend, `RequestPipeline::transform`
(`packages/core/src/pipeline.rs:422+`) recorta la lista de mensajes al
`max_context_length` del backend: los mensajes de sistema se conservan
completos, los mensajes que no son de sistema se conservan de los más recientes
a los más antiguos mientras quepan, y un único mensaje demasiado grande se
trunca por caracteres (el contador heurístico de tokens no puede truncar con
precisión de token). La llamada pasa entonces por el trait `InferenceBackend`;
los éxitos y fallos se registran de vuelta en el circuit breaker por backend del
router (3 fallos, 30 s de recuperación, 1 llamada half-open —
routing/mod.rs:57-64).

## Verificador de salud y probing

`run_health_checks` (`packages/core/src/gateway/health_checker.rs`) se ejecuta
como una tarea en segundo plano generada al arrancar (run.rs:135-150) y somete
a probe cada backend registrado una vez por intervalo de 60 segundos. Dos
detalles importan:

- La lista de backends se **obtiene de nuevo en cada ronda** a través de un
  closure fetcher asíncrono, así que los backends registrados después del
  arranque (p. ej. vía la API de admin) se recogen sin reiniciar.
- La primera ronda se ejecuta inmediatamente, antes de que transcurra el primer
  intervalo, de modo que el estado de salud se establece en cuanto arranca el
  proceso.

`probe_backend` es la única ruta de código del probe. Se reutiliza para los
**probes en el momento del registro** de una sola vez: después de que un admin
registra un backend (server.rs:688-693) o un backend persistido se restaura al
arrancar (run.rs:122-127), un probe fire-and-forget hace pasar el backend a
saludable en ~1-2 s en lugar de permanecer en fail-closed hasta la siguiente
ronda de 60 s. Esto es lo que hace que la lista de modelos de un backend
external recién registrado aparezca en `GET /v1/models` casi de inmediato.

El probe en sí es una llamada ligera al backend (p. ej. el backend external
golpea `/v1/models` con un timeout de probe de 2 s); el resultado se cachea en
el backend y el routing solo selecciona backends cuya salud cacheada sea
`Healthy` (además de un circuit breaker abierto).

## Cliente de memoria

El cliente de memoria (`packages/core/src/memory/mod.rs`) se construye a partir
de la configuración de entorno al arrancar el servidor (server.rs:95-97):
cuando `ARONA_MEMORY_URL` y `ARONA_MEMORY_TOKEN` están definidas, las
solicitudes de chat consultan el servicio de memoria Philia de entelecheia por
un WebSocket JSON-RPC y `recall_and_inject` antepone las memorias relevantes
como sección de sistema (`## Relevant Long-Term Memories`) al contexto saliente.
Los turnos completados se escriben de vuelta como episodios mediante
`writeback_dialogue` — trabajo fire-and-forget generado después de que la
respuesta del asistente se persiste, así que los fallos de memoria nunca
bloquean ni ralentizan la ruta de respuesta de chat. `ARONA_MEMORY_WRITEBACK`
(predeterminado activado) conmuta el writeback. Consulte
[gateway de memoria](memory-gateway.md) para el panorama completo.

Cada respuesta de chat lleva una cabecera `X-Arona-Memory` con uno de tres
estados: `enabled` (la recuperación se ejecutó e inyectó), `disabled` (no
configurado o la solicitud pasó `memory: false`) u `offline` (configurado pero
el servicio no estaba accesible).

## Sesiones y notificaciones

`AppState` mantiene un `SessionManager` de plana (`state.sessions`). Los RPC
streaming como `chat.send` crean un session id (`gateway/rpc.rs:1701`) y
empujan notificaciones — tokens de `chat.stream`, progreso de despliegue
`models.progress`, `realtime.event` — al canal de esa sesión. Los clientes las
consumen del sidecar SSE `GET /api/rpc/events?session=<id>` (server.rs:191-200);
consulte [eventos](../api/events.md) para el formato de notificación y la
advertencia de la ventana de pre-suscripción.

El canal de sesión también se usa para las llamadas RPC de solicitud/respuesta:
cuando un cliente envía una cabecera `x-session-id` en `POST /api/rpc`, el
servidor empuja también el resultado a ese canal de sesión (server.rs:184-188,
rpc.rs:128-144), de modo que un cliente puede multiplexar una respuesta RPC en
un stream SSE ya abierto.

## Límites y compensaciones de diseño

El diseño acepta deliberadamente una serie de límites; conózcalos antes del uso
en producción:

- **Límite de cuerpo de solicitud de 1 MB** — las capas rechazan los cuerpos más
  grandes; si necesita llamadas de contexto grande, esto es lo primero que hay
  que ajustar.
- **CORS `*`** — el gateway responde llamadas de origen cruzado desde cualquier
  lugar. Bien para una API, pero si lo expone más allá de clientes de confianza,
  póngalo delante de un proxy que aplique su propia política de CORS.
- **Billing fail-open** — la aplicación de cuotas/límites de tasa degrada a
  permitir la solicitud cuando la base de datos no está disponible. El billing
  es medición, no control de acceso.
- **Sin timeout total en los streams SSE** — las llamadas streaming no llevan
  plazo total (las generaciones largas son legales); la detección de cuelgues
  depende de un timeout de inactividad de 120 s por lectura
  (`backends/external.rs:24-31`). Las llamadas no streaming tienen un plazo
  total de 600 s.
- **Uso estimado con tokenizador** — los backends que nunca informan uso
  (ollama, ws_engine) se facturan a partir de una estimación local de un
  tokenizador consciente de CJK, registrada tal cual (consulte
  [billing y uso](billing-usage.md)).
- **Ventanas de límite de tasa y revocación en memoria** — la ventana
  deslizante por clave y el conjunto de claves revocadas viven en la memoria del
  proceso (`auth.rs`), así que un reinicio las restablece. El limitador a nivel
  de auth acota las solicitudes por clave por ventana; el limitador de billing
  tier está respaldado por la base de datos (consulte
  [auth y seguridad](auth-security.md) y [billing y uso](billing-usage.md)).
- **`/ws/agent` no está autenticado** — el plano de control de agentes acepta
  cualquier WebSocket que hable el protocolo register/heartbeat. Solo es seguro
  en una red que usted controle.
- **Sin TLS en el gateway** — el servidor se vincula a HTTP plano; termine el
  TLS delante (reverse proxy) en cualquier despliegue que cruce un límite de
  red. Consulte [despliegue](deployment.md).

En el lado amable, el servidor realiza un apagado correcto: instala handlers de
Ctrl+C y SIGTERM, registra "draining connections" y deja que las solicitudes en
curso terminen antes de que el proceso salga (`gateway/run.rs:14-38`, y el
cableado `with_graceful_shutdown` en run.rs:154-159).

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table); gateway/mod.rs:29-53; routing/mod.rs:102-200; health_checker.rs:14-37; run.rs:135-159; memory/mod.rs:207-241; rpc.rs:128-144,1701 -->
