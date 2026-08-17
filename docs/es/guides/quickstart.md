---
title: "Inicio rápido"
description: "Recorrido completo de Arona de extremo a extremo con el upstream mock integrado: migrar, servir, registrar un backend, crear una API key y chatear."
---

# Inicio rápido

Esta guía le acompaña a lo largo de una configuración completa de extremo a
extremo de Arona en una sola máquina con el **upstream mock integrado** — no se
necesitan pesos de modelos reales, GPU ni cuenta de API externa. Al final
tendrá:

- un gateway de Arona en funcionamiento (la API REST compatible con OpenAI
  `/v1/*` más el plano de gestión JSON-RPC `/api/rpc`),
- el upstream mock registrado como backend `external`,
- una cuenta de usuario y una API key,
- un turno de chat no streaming **y** streaming que funcione contra el mock,
- registros de uso visibles mediante `usage.list`.

## Requisitos previos

- **Toolchain de Rust** (consulte `rust-toolchain.toml` en la raíz del
  repositorio).
- **Python 3** con `aiohttp` — solo es necesario para el upstream mock
  (`pip install aiohttp`).
- Una instancia de **PostgreSQL en ejecución** y su URL de conexión.

## 1. Configure el entorno

Arona lee su configuración de las variables de entorno **al arrancar el
proceso**. Dos son obligatorias: `DATABASE_URL` y `JWT_SECRET` — el servidor se
niega a arrancar sin ellas (salvo con `MOCK_MODE=1`). Se recomienda
encarecidamente `ARONA_ADMIN_TOKEN`: sin él, todas las rutas `/api/admin/*`
devuelven 401.

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

Estas variables se leen una sola vez al iniciar el proceso — si las cambia,
reinicie el servidor. Consulte [Configuración](configuration.md) para la
referencia completa de variables.

## 2. Migre e inicie el servidor

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

`serve` por sí solo basta para una base de datos nueva: auto-migra al arrancar.
El servidor se vincula a `0.0.0.0:8420` por defecto (se puede sobrescribir con
`ARONA_HOST` / `ARONA_PORT`).

## 3. Inicie el upstream mock

En una segunda terminal:

```bash
python3 scripts/mock/server.py
```

El mock es un servidor aiohttp que escucha en `127.0.0.1:8429` por defecto
(`ARONA_MOCK_PORT` sobrescribe el puerto). Imprime su API key al arrancar y
también sirve `GET /api/test-key`, que devuelve
`{"api_key": ..., "base_url": ...}`. Expone un puñado de ids de modelos —
entre ellos `gpt-5.5`, que se usa más abajo — y responde a las chat
completions tanto normales como en streaming.

Capture la clave impresa:

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. Registre el mock como backend external

Los backends se registran a través de la API HTTP de administración:

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

El backend se somete a un probe inmediatamente al registrarse y pasa a
saludable en ~1-2 segundos; hasta que ese probe termina permanece en un estado
fail-closed de "aún no probado" (consulte el recuadro de solución de problemas
más abajo). La configuración se persiste, así que el backend sobrevive a un
reinicio.

## 5. Registre una cuenta e inicie sesión

Las cuentas viven en el plano JSON-RPC, `POST /api/rpc`. Como
`ARONA_REGISTRATION_OPEN=1` está definida, `auth.register` está abierto; el
primer usuario registrado se convierte en el admin.

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

Las contraseñas deben tener al menos 8 caracteres **y** contener al menos 3 de
las 4 categorías de caracteres (mayúsculas, minúsculas, dígitos, especiales).
Después inicie sesión para obtener el par de JWTs:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

Exporte el `access_token` de la respuesta:

```bash
export JWT="<access_token from the login response>"
```

## 6. Cree una API key

`keys.create` se autentica con JWT y devuelve el secreto **completo**
`arona-{uuid}` exactamente una vez — la base de datos solo guarda su hash
SHA-256, así que cópielo ahora:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. Chat (no streaming)

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

Recibe un objeto de completion con estilo OpenAI con `choices[0].message` y un
bloque `usage`.

## 8. Chat (streaming)

El mismo endpoint con `"stream": true` responde con server-sent events: un
chunk `data:` por token, terminado por un chunk final `data: [DONE]`:

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. Verifique el uso

Cada turno de chat registra una fila de uso bajo el prefijo de la clave.
Consúltela con el JWT:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

Debería ver uno o más registros de las solicitudes `gpt-5.5` realizadas más
arriba.

## Solución de problemas

- **`No backend available for model: gpt-5.5` (HTTP 404, `code:
  model_not_found`)** — ningún backend registrado sirve ese id de modelo. O
  bien el backend nunca se registró (o su lista `models` no incluye el id), o
  la llamada de registro falló. Compruébelo con `GET /api/admin/backends`
  (token de admin).
- **`all backends unhealthy` (HTTP 500, `backend_error`)** — hay un backend *sí*
  registrado para el modelo, pero ningún candidato está saludable. Un backend
  external recién registrado comienza en un estado fail-closed de "aún no
  probado" y pasa a saludable cuando el probe del registro termina, ~1-2 s
  después; si chatea dentro de esa ventana, obtendrá este error. Reintente al
  cabo de un momento, o compruebe que el mock está realmente en ejecución en
  `127.0.0.1:8429`.
- **HTTP 401 en `/v1/*`** — una cabecera `Authorization` ausente produce
  `Missing Authorization header. Use: Bearer <api_key>`; una clave desconocida
  produce `Invalid API key`. Vuelva a comprobar `$AR_KEY` (el secreto completo,
  no el prefijo).
- **HTTP 401 `Admin access required` en `/api/admin/*`** — el bearer token no
  coincide con `ARONA_ADMIN_TOKEN`, o la variable no está definida (entonces la
  ruta siempre rechaza). Reinicie el servidor después de definirla.
- **`auth.register` falla con "Registration is closed"** — el registro está
  deshabilitado cuando `ARONA_REGISTRATION_OPEN` no es truthy. Defina
  `ARONA_REGISTRATION_OPEN=1` **antes** de iniciar el servidor (se lee al
  arrancar), o sea el primer usuario — al primer usuario registrado siempre se
  le permite y se convierte en el admin.
- **Límites de tasa HTTP 429** — pueden activarse tres límites independientes:
  - el límite en memoria por clave, 60 RPM por defecto
    (`ARONA_API_RATE_LIMIT_RPM`) → `Rate limit exceeded. Try again later.`;
  - el límite de 10 RPM por clave del billing tier free → 429 con la cabecera
    `Retry-After: 60`;
  - la cuota mensual de $1 / 100k tokens del tier free → 429 con `Retry-After`
    apuntando al siguiente período de facturación.

## Siguientes pasos

- [Configuración](configuration.md) — todas las variables de entorno.
- [Backends](backends.md) — tipos de backend, semántica de URLs y probing.
- [Despliegue](deployment.md) — bare metal, systemd, Docker.
- [API REST compatible con OpenAI](../api/openai-rest.md) — toda la superficie `/v1/*`.
- [API JSON-RPC](../api/jsonrpc.md) — el plano de gestión usado arriba.

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
