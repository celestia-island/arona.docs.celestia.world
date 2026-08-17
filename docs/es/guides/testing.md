---
title: "Pruebas"
description: "La pirámide de pruebas de Arona — pruebas unitarias, integración hermética, integración con puerta PostgreSQL, smoke tests con servidor en vivo, el servidor mock y la disciplina de smoke con credenciales reales."
---

# Pruebas

Las pruebas de Arona están organizadas en capas para que la ejecución
predeterminada de `cargo test` sea rápida, hermética y no necesite ni base de
datos ni red, mientras que las suites más pesadas son opt-ins explícitos que
ejercitan la superficie de vía real y un PostgreSQL real. Esta página mapea las
capas, los comandos que las ejecutan y la disciplina del workspace en torno a
las ejecuciones de smoke con credenciales reales.

## Pruebas unitarias

El grueso de la cobertura son pruebas unitarias normales dentro de
`packages/core/src`: 217 funciones `#[test]` / `#[tokio::test]`, más ~23 más en
`packages/agent` y `packages/cli`. Se ejecutan con:

```bash
cargo test --workspace
```

Sin red, sin base de datos. Suites clave:

- **auth.rs** — la política de contraseñas (≥8 caracteres Y ≥3 de 4 categorías
  de caracteres), los casts `::uuid` en el SQL crudo INSERT/REVOKE, los
  predeterminados de solicitud y las lecturas de la marca de admin que recurren
  a `false`.
- **billing/mod.rs** — el cálculo de cuotas en la dimensión de coste *o* de
  tokens, la ventana mensual (`month_start`, `seconds_until_month_end`), el
  techo de límite de tasa (solo dispara *en* el RPM, `None` = ilimitado),
  guardas de forma SQL para las consultas de uso mensual / tier / ventana de
  clave, y `estimate_usage` prefiriendo las cifras informadas por el upstream.
- **routing/mod.rs** — resolución de aliases, coincidencia del sufijo
  `:latest`, pistas de provider, selección por menor carga y fijación de
  conversaciones.
- **gateway/mod.rs** — registro de backends de agente: registrar
  `agent-{model_id}`, el re-registro que sustituye (no duplica) y el
  des-registro que restaura el router.

## Integración hermética (siempre se ejecuta, sin DB)

`packages/core/tests/gateway_integration.rs` contiene tres pruebas que siempre
se ejecutan y ejercitan la lógica real de serialización/contrato sin tocar una
base de datos:

- **A1** — el contrato de serialización del echo de id JSON-RPC: los ids de
  solicitud numéricos, de cadena y null hacen round trip por el enum `Id` de
  plana con fidelidad de tipo.
- **A2** — el contrato de códigos de error de la puerta de admin:
  `AUTH_ERROR` (-32005, anónimo) y `ADMIN_REQUIRED` (-32007, no-admin
  autenticado) se mantienen distintos, viven en el rango definido por la
  implementación y nunca chocan con los códigos de plana ni con el
  `QUOTA_ERROR` (-32006) de billing.
- **A3** — `estimate_usage`: el uso informado por el upstream gana tal cual;
  sin él, la estimación local del tokenizador produce recuentos de
  prompt/completion distintos de cero cuya suma es su total.

`packages/core/tests/smoke.rs` añade tres pruebas más que siempre se ejecutan:
detección de hardware, la ruta raíz del model-registry y los predeterminados de
configuración bajo `MOCK_MODE=1`.

## Integración con puerta PG

La suite completa del gateway en proceso — `packages/core/tests/gateway_integration.rs`
— hace girar el router axum completo en un puerto loopback aleatorio, registra
upstreams mock desechables compatibles con OpenAI a través de la API de admin
real y maneja la superficie de vía con reqwest. Como `AuthManager` habla con
PostgreSQL en cada ruta (incluso `MOCK_MODE=1` solo siembra cuentas *en la base
de datos*), esta suite está detrás de `ARONA_TEST_PG=1` y se omite por defecto.
Las 10 pruebas:

- **T1** registrar + login + `keys.create`/`keys.list` (clave cruda enmascarada
  en los listados, prefijo `arona-`).
- **T2** chat REST con persistencia del registro de uso en PostgreSQL.
- **T3** echo de id JSON-RPC por la vía (rutas de éxito y de error).
- **T4** puerta de admin en `agents.list`: anónimo → `AUTH_ERROR`, no-admin →
  `ADMIN_REQUIRED`.
- **T5** 401 del upstream → HTTP 502 `bad_gateway` con el detalle del upstream.
- **T6** el probe en el momento del registro publica modelos (el modelo aparece
  en `GET /v1/models` en 10 s sin lista estática de modelos).
- **T7** persistencia de conversación a través de `chat.send` (ambos turnos
  aterrizan en `conversations.get`).
- **T8** puerta de billing del tier free: 10 RPM por clave, la 11.ª solicitud
  de la ventana es 429 `rate_limit_exceeded`.
- **T9** stream SSE con uso terminal registrado desde el upstream.
- **T10** JSON malformado → 400; modelo desconocido → 404 `model_not_found`.

Ejecútela con el one-liner de Postgres desechable de los docs del módulo
(gateway_integration.rs:18-26):

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

Estas son credenciales de ejemplo solo para el contenedor de pruebas
desechable — nunca las apunte a una base de datos real.

## Smoke con servidor en vivo

`packages/core/tests/auth_flow.rs` recorre la cadena completa
`register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` contra un servidor Arona **en vivo**, reflejando el bucle de auth
desplegado. Está `#[ignore]`da por defecto — la ejecución normal de `cargo
test` nunca toca la red. Ejecútela explícitamente:

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

Parámetros:

- `ARONA_TEST_URL` — URL base (predeterminado `http://127.0.0.1:8420`).
- `ARONA_TEST_EXPECT_CHAT=1` — afirma de forma estricta que
  `POST /v1/chat/completions` devuelve 200. Sin él, la prueba solo afirma que la
  auth pasó (no 401/403), porque el entorno de destino puede no tener un
  provider de inferencia configurado.

La suite también incluye pruebas negativas: una chat completion sin autenticar
y un `GET /v1/models` sin autenticar deben rechazarse ambos con 401.

## Servidor mock

`scripts/mock/server.py` es un fake compatible con OpenAI basado en aiohttp
usado por el inicio rápido y las ejecuciones de smoke. Sirve
`POST /v1/chat/completions` (no-stream y SSE), `GET /v1/models`,
`GET /api/health`, la superficie WebSocket/HTTP JSON-RPC en `/api/rpc`, un
sidecar SSE en `/api/rpc/events` y `GET /api/test-key`, que devuelve la API key
del mock para que otros servicios puedan descubrirla. Escucha en el puerto 8429
por defecto (sobrescriba el puerto con `ARONA_MOCK_PORT`, el host con
`ARONA_MOCK_HOST`). El [inicio rápido](quickstart.md) lo usa para montar un
entorno de extremo a extremo sin providers de modelos reales.

## Disciplina de smoke con credenciales reales

Las ejecuciones de smoke contra providers reales (DeepSeek / GLM) **no** son
deliberadamente pruebas de repositorio — requieren credenciales reales y dinero
real, así que no pueden vivir en CI ni en el árbol de git. La convención del
workspace, documentada en los docs del módulo gateway_integration
(gateway_integration.rs:54-55), es:

- Los archivos de evidencia viven en `/mnt/work/arona-pr*-smoke.md` — locales al
  workspace, nunca se comprometen a git.
- Las credenciales provienen solo del entorno; los presupuestos se mantienen
  pequeños.
- Cada PR que toca la ruta de inferencia obtiene un registro de evidencia
  escrito.

El servidor mock es el sustituto de estas ejecuciones en CI y en el desarrollo
local; el smoke con credenciales reales es un paso humano en el momento del
release.

## CI

`.github/workflows/ci.yml` ejecuta `cargo fmt`, `cargo clippy`, `cargo test
--workspace` y `cargo-deny` en los runners autoalojados de la org
(`[self-hosted, linux, x64, local]`); `ci-hosted.yml` refleja las mismas
comprobaciones en runners alojados en GitHub. `.github/workflows/docs.yml`
compila este sitio de documentación con lagrange y lo despliega en GitHub
Pages en los pushes que tocan `docs/**`.

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
