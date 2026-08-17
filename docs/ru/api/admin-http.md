---
title: "Admin HTTP API"
description: "Admin-поверхность на bearer-token — регистрация/список/удаление backends и управление алиасами моделей через /api/admin/*."
---

# Admin HTTP API

Поверхность `/api/admin/*` управляет **backends** шлюза (upstream providers
моделей) и **алиасами** (перенаправлением имя-модели → id-модели). Это
HTTP-аналог RPC-плоскости управления (см.
[JSON-RPC API](./jsonrpc.md)), используемый в первую очередь операторами
и admin UI.

## Аутентификация

Каждый маршрут `/api/admin/*` требует:

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN` читается из окружения при старте процесса
(`GatewayServer::new`). Если переменная **не задана** или предъявленный token
не совпадает, запрос отклоняется с `401`:

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

Префикс bearer сопоставляется без учёта регистра (`Bearer` или `bearer`).

> В отличие от поверхности `/v1/*`, admin-auth никогда не откатывается
> к API keys или JWT и применяется сравнением точного token — ротируйте
> token, перезапустив процесс с новым значением.

## Backends

Backends — это маршрутизируемые upstream за шлюзом. Регистрация делает backend
маршрутизируемым немедленно, сохраняет его конфигурацию для восстановления
при перезапуске, пробирует его (переводит в healthy в течение ~1–2 с)
и, для bridge-URL, держит туннель живым. Типы backends и семантика URL
подробно описаны в [Backends](../guides/backends.md).

### POST /api/admin/backends — регистрация backend

Тело запроса (все поля опциональны, кроме отмеченных):

| Поле | Тип | Примечания |
| --- | --- | --- |
| `type` | string | Вид backend. Один из `external` (любой OpenAI-совместимый HTTP API), `ollama` (локальный или удалённый сервер ollama), `engine` (CEP-движок через `ws://`/`wss://`), `minimax-cloud` (облачный видео-API). Имена MDD-движков (`llama_cpp`, `vllm`, `ollama`, `cloud`, `external_api`, `candle`, `native`, ...) разрешаются через planner. `comfyui` **отклоняется** (`comfyui backend removed`); всё остальное → `400` `unknown_type`. По умолчанию `ollama`, когда не задан. |
| `url` | string | Базовый URL backend. Bridge-URL `evernight://<node>/<service>` разрешаются через локального агента evernight в локальный TCP-форвард (сбой разрешения → `502` `evernight_unreachable`). По умолчанию `http://localhost:11434`. |
| `api_key` | string | Опциональный API key upstream, отправляется как `Authorization: Bearer` при вызовах upstream. |
| `name` | string | Имя backend. По умолчанию значение `type`, когда отсутствует. Используется как подсказка `provider` при маршрутизации и для идентичности строки конфигурации. |
| `models` | string[] | Статический список моделей. Источник маршрутизации, когда probing не обнаруживает ничего. Для `external` backends обнаруженные модели объединяются после статического списка (статичные id сохраняют приоритет); `engine` backends сначала возвращают свой кэш обнаруженных моделей и добавляют статичные id после; `minimax-cloud` не выполняет обнаружение моделей (его проба только health-пингует `/v1/query/available_models`) и обслуживает один статический список. Игнорируется для `ollama`, который обнаруживает модели из `/api/tags`. |
| `workflow` | object | Опционально. Legacy — исторически потреблялся удалённым backend ComfyUI; ни один текущий backend его не читает (сохранён для совместимости колонки `backend_configs`). |

Пример:

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

Успех → `200`:

```json
{ "status": "ok", "message": "backend registered" }
```

Побочные эффекты регистрации:

- Backend **регистрируется и становится маршрутизируемым немедленно**
  (перезапуск не нужен).
- Конфигурация **сохраняется** в таблицу `backend_configs` и восстанавливается
  при старте (сбой БД логируется, но никогда не блокирует ответ).
- Сразу же запускается fire-and-forget-**проба**, чтобы backend перешёл
  в healthy в течение ~1–2 с, а не оставался fail-closed до следующего
  60-секундного раунда health-checker'а.
- Для URL `evernight://` **keepalive-задача** наблюдает за туннелем: при
  переподключении с новым локальным портом она прозрачно пересобирает
  и перерегистрирует backend под тем же именем.

### GET /api/admin/backends — список backends

```json
{
  "backends": {
    "count": 2,
    "health": [
      ["backend_0:ExternalApi", "Healthy"],
      ["backend_1:Ollama", { "Unhealthy": "connection refused" }]
    ]
  },
  "models": [
    { "id": "Qwen/Qwen3-1.7B", "object": "model", "owned_by": "mock-upstream" }
  ]
}
```

- `backends.count` — число **healthy** backends.
- `backends.health` — метка `backend_<index>:<kind>` на каждый backend
  и состояние health (`Healthy` / `Degraded` / `Unhealthy`). `<index>` — это
  индекс регистрации в роутере, используемый `DELETE /api/admin/backends`.
- `models` — каждый id модели, маршрутизируемый сегодня (тот же список,
  что и `GET /v1/models`, без объединения быстрых стартовых; см.
  [OpenAI-совместимый REST](./openai-rest.md#get-v1models)).

### DELETE /api/admin/backends — удаление backend

Идентифицируется по **индексу** в роутере в JSON-теле — не по имени:

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| Поле | Тип | Обязательно | Примечания |
| --- | --- | --- | --- |
| `index` | integer | да | Индекс регистрации в роутере, совпадающий с меткой `backend_<index>` в health-отчёте `GET /api/admin/backends`. |

- Отсутствующий `index` → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`.
- Индекс вне диапазона → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`.
- Успех → `200` `{ "status": "ok", "message": "backend removed" }`.
- Сохранённая строка `backend_configs` удаляется best-effort: имя backend
  восстанавливается из `owned_by` его списка моделей; при несовпадении строка
  остаётся в хранилище (сбои БД логируются, никогда не фатальны).

## Aliases

Алиасы сопоставляют одно имя модели с другим (`alias` → `target`), чтобы
запросы на один id модели маршрутизировались на другую модель backend.
Алиасы разрешаются до маршрутизации, поэтому применяются единообразно
к поискам чата, embeddings и видео.

> Алиасы — **только in-memory состояние роутера** — они не сохраняются
> и теряются при перезапуске. Регистрируйте их после старта или
> воссоздавайте из собственного состояния провижининга.

### POST /api/admin/aliases — добавление алиаса

| Поле | Тип | Обязательно | Примечания |
| --- | --- | --- | --- |
| `alias` | string | да | Имя модели, которое будут запрашивать клиенты. |
| `target` | string | да | Id модели, на который маршрутизируются запросы. |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- Отсутствующий `alias` → `400` `missing_alias`; отсутствующий `target` → `400`
  `missing_target`.
- Успех → `200` `{ "status": "ok", "message": "alias added" }`.
- Добавление существующего алиаса заменяет его цель.

### GET /api/admin/aliases — список алиасов

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

Пары возвращаются отсортированными по алиасу.

### DELETE /api/admin/aliases — удаление алиаса

| Поле | Тип | Обязательно | Примечания |
| --- | --- | --- | --- |
| `alias` | string | да | Алиас для удаления. |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- Отсутствующий `alias` → `400` `missing_alias`.
- Удаление неизвестного алиаса — no-op успех → `200`
  `{ "status": "ok", "message": "alias removed" }`.

## Сводка по сохранению

| Ресурс | Сохраняется? | Восстановление при перезапуске |
| --- | --- | --- |
| Backends | Да — таблица `backend_configs` (ключ `name`, upsert при регистрации, удаление при удалении). | Да: восстанавливаются при старте; external backends стартуют fail-closed и переходят в healthy после первого раунда проб. URL `evernight://` заново разрешаются через bridge при старте. |
| Aliases | Нет — только in-memory `Router.aliases`. | Нет. |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
