---
title: "API HTTP de administración"
description: "Superficie de admin con bearer token — registrar/listar/eliminar backends y gestionar aliases de modelos mediante /api/admin/*."
---

# API HTTP de administración

La superficie `/api/admin/*` gestiona los **backends** del gateway (providers
de modelos upstream) y los **aliases** (redirección nombre-de-modelo →
id-de-modelo). Es la contraparte HTTP del plano de gestión RPC (consulte la
[API JSON-RPC](./jsonrpc.md)) y la usan principalmente los operadores y la
interfaz de admin.

## Autenticación

Cada ruta `/api/admin/*` requiere:

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN` se lee del entorno al iniciar el proceso
(`GatewayServer::new`). Si la variable está **sin definir**, o el token
presentado no coincide, la solicitud se rechaza con `401`:

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

El prefijo bearer se compara sin distinguir mayúsculas (`Bearer` o `bearer`).

> A diferencia de la superficie `/v1/*`, la auth de admin nunca recurre a API
> keys o JWTs, y se aplica con una comparación de token exacta — rote el token
> reiniciando el proceso con un valor nuevo.

## Backends

Los backends son los upstreams enrutables detrás del gateway. El registro hace
que un backend sea enrutable de inmediato, persiste su configuración para la
restauración en el reinicio, le hace un probe (pasa a saludable en ~1-2 s) y,
para las URLs de bridge, mantiene vivo el túnel. Los tipos de backend y la
semántica de URLs se detallan en [Backends](../guides/backends.md).

### POST /api/admin/backends — registrar un backend

Cuerpo de la solicitud (todos los campos opcionales salvo donde se indique):

| Campo | Tipo | Notas |
| --- | --- | --- |
| `type` | string | Tipo de backend. Uno de `external` (cualquier API HTTP compatible con OpenAI), `ollama` (servidor ollama local o remoto), `engine` (CEP engine sobre `ws://`/`wss://`), `minimax-cloud` (API de vídeo en la nube), `comfyui` (servidor ComfyUI autoalojado; requiere `workflow`, restaurado 2026-10-13). Los nombres de engines MDD (`llama_cpp`, `vllm`, `ollama`, `cloud`, `external_api`, `candle`, `native`, ...) se resuelven vía el planner — `vllm` y `llama_cpp` se registran en **modo attach** contra el endpoint compatible con OpenAI ya en ejecución. Cualquier otro valor → `400` `unknown_type`. Por omisión `ollama`. |
| `url` | string | URL base del backend. Las URLs de bridge `evernight://<node>/<service>` se resuelven a través del agente evernight local en un reenvío TCP local (fallo de resolución → `502` `evernight_unreachable`). Predeterminado `http://localhost:11434`. |
| `api_key` | string | API key opcional del upstream, enviada como `Authorization: Bearer` en las llamadas al upstream. |
| `name` | string | Nombre del backend. Predeterminado al valor de `type` cuando falta. Se usa como pista `provider` del routing y como identidad de la fila de configuración. |
| `models` | string[] | Lista estática de modelos. La fuente del routing cuando el probing no descubre ninguno. Para los backends `external`, los modelos descubiertos se fusionan después de la lista estática (los ids estáticos conservan la precedencia); los backends `engine` devuelven primero su caché de modelos descubierta y añaden los ids estáticos después; `minimax-cloud` no realiza descubrimiento de modelos (su probe solo hace health-ping de `/v1/query/available_models`) y sirve solo la lista estática. `ollama` la ignora, ya que descubre los modelos desde `/api/tags`. |
| `workflow` | object | Opcional. La plantilla de flujo de trabajo de ComfyUI — la lee y requiere el backend `comfyui`; otros tipos la ignoran (se conserva por compatibilidad de la columna `backend_configs`). |

Ejemplo:

```bash
curl -X POST http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "name": "mock-upstream",
    "url": "http://192.0.2.20:11434",
    "api_key": "sk-xxx",
    "models": ["Qwen/Qwen3-1.7B"]
  }'
```

Éxito → `200`:

```json
{ "status": "ok", "message": "backend registered" }
```

Efectos secundarios del registro:

- El backend se **registra y es enrutable de inmediato** (sin reinicio).
- La configuración se **persiste** en la tabla `backend_configs` y se restaura
  al arrancar (un fallo de base de datos se registra pero nunca bloquea la
  respuesta).
- Un **probe** fire-and-forget se ejecuta de inmediato para que el backend pase
  a saludable en ~1-2 s en lugar de permanecer en fail-closed hasta la siguiente
  ronda de 60 s del verificador de salud.
- Para las URLs `evernight://`, una **tarea keepalive** vigila el túnel: al
  reconectar con un puerto local nuevo, reconstruye y re-registra
  transparentemente el backend con el mismo nombre.

### GET /api/admin/backends — listar backends

```json
{
  "backends": {
    "count": 2,
    "health": [
      ["backend_0:ExternalApi", "Healthy"],
      ["backend_1:Ollama", { "Unhealthy": "connection refused" }]
    ],
    "rows": [
      {
        "index": 0,
        "label": "backend_0:ExternalApi",
        "name": "external",
        "kind": "external",
        "status": "healthy",
        "models": ["Qwen/Qwen3-1.7B"]
      }
    ]
  },
  "models": [
    { "id": "Qwen/Qwen3-1.7B", "object": "model", "owned_by": "mock-upstream" }
  ]
}
```

- `backends.count` — número de backends **saludables**.
- `backends.health` — etiqueta `backend_<index>:<kind>` por backend y estado de
  salud (`Healthy` / `Degraded` / `Unhealthy`). El `<index>` es el índice de
  registro del router usado por `DELETE /api/admin/backends`.
- `backends.rows` — filas de registro legibles por máquina (lectura principal del panel): `index` (la dirección del plano de eliminación), `label`, `name` configurado, `kind` en snake_case, `status` como palabra discreta en minúsculas (`healthy` / `degraded` / `unhealthy`), razón `detail` opcional y la instantánea de `models` servidos.
- `models` — cada id de modelo enrutable hoy (el mismo listado que
  `GET /v1/models`, sin la fusión de inicio rápido; consulte
  [REST compatible con OpenAI](./openai-rest.md#get-v1models)).

### DELETE /api/admin/backends — eliminar un backend

Se identifica por su **índice** de router en el cuerpo JSON — no por nombre:

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| Campo | Tipo | Obligatorio | Notas |
| --- | --- | --- | --- |
| `index` | integer | sí | Índice de registro del router, que coincide con la etiqueta `backend_<index>` del informe de salud de `GET /api/admin/backends`. |

- Falta `index` → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`.
- Índice fuera de rango → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`.
- Éxito → `200` `{ "status": "ok", "message": "backend removed" }`.
- La fila persistida de `backend_configs` se elimina de mejor esfuerzo: el
  nombre del backend se recupera del `owned_by` de su listado de modelos; un
  desajuste deja la fila en el almacén (los fallos de base de datos se
  registran, nunca son fatales).

## Aliases

Los aliases asignan un nombre de modelo a otro (`alias` → `target`) para que
las solicitudes de un id de modelo se enruten a un modelo de backend distinto.
Los aliases se resuelven antes del routing, así que se aplican de forma
uniforme a las búsquedas de chat, embeddings y vídeo.

> Los aliases son **solo estado del router en memoria** — no se persisten y se
> pierden al reiniciar. Regístrelos después del arranque o recréelos desde su
> propio estado de aprovisionamiento.

### POST /api/admin/aliases — añadir un alias

| Campo | Tipo | Obligatorio | Notas |
| --- | --- | --- | --- |
| `alias` | string | sí | El nombre de modelo que solicitarán los clientes. |
| `target` | string | sí | El id de modelo al que se enrutan las solicitudes. |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- Falta `alias` → `400` `missing_alias`; falta `target` → `400`
  `missing_target`.
- Éxito → `200` `{ "status": "ok", "message": "alias added" }`.
- Añadir un alias existente sustituye su target.

### GET /api/admin/aliases — listar aliases

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

Los pares se devuelven ordenados por alias.

### DELETE /api/admin/aliases — eliminar un alias

| Campo | Tipo | Obligatorio | Notas |
| --- | --- | --- | --- |
| `alias` | string | sí | El alias a eliminar. |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- Falta `alias` → `400` `missing_alias`.
- Eliminar un alias desconocido es un éxito no-op → `200`
  `{ "status": "ok", "message": "alias removed" }`.

## Resumen de persistencia

| Recurso | ¿Persistido? | Restauración en el reinicio |
| --- | --- | --- |
| Backends | Sí — tabla `backend_configs` (clave `name`, upsert al registrar, borrado al eliminar). | Sí: se restauran al arrancar; los backends external arrancan en fail-closed y pasan a saludables después de la primera ronda de probe. Las URLs `evernight://` se vuelven a resolver a través del bridge al arrancar. |
| Aliases | No — solo `Router.aliases` en memoria. | No. |

<!-- src: packages/core/src/gateway/server.rs (check_rbac_permission, add_backend, list_backends_admin, remove_backend, add_alias_handler, list_aliases, remove_alias_handler); packages/core/src/backends/mod.rs (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs (aliases); packages/core/src/gateway/run.rs (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs remove_backend. Also confirmed: aliases are in-memory only (routing/mod.rs aliases()); `workflow` is the ComfyUI workflow template, required and read by the `comfyui` backend (backends/mod.rs build_backend_from_config, comfyui.rs); GET also returns `backends.rows` machine-readable registry rows; `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (ollama.rs). -->
