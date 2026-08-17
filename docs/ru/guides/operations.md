---
title: "Эксплуатация"
description: "Health-endpoints, трассировка RUST_LOG, таймауты upstream, отображение ошибок и устранение неполадок для работающего arona-server."
---

# Эксплуатация

Эта страница для операторов, запускающих `arona-server serve`. Она покрывает
health-endpoints, которые вы пробируете, строки логов, которые стоит грепать,
модель таймаутов, применяемую к upstream-backends, то, как сбои backends
отображаются в HTTP-ошибки, и эксплуатационные грабли, о которые спотыкаются
люди. Само развёртывание описано в [руководстве по развёртыванию](./deployment.md).

## Матрица health

Все три health-endpoint'а не требуют аутентификации и возвращают `200 OK`
всегда, когда процесс обслуживает запросы — различия liveness/readiness нет:

| Endpoint | Ответ |
| --- | --- |
| `/healthz`, `/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | то же подробное тело, что выше |
| `/api/health` | plana `HealthResponse`: `status`, `version` (`CARGO_PKG_VERSION`), `kind` (`Dev`), `uptime` (секунды), `network` (transport / region / asn), `build_hash` (`BUILD_HASH`), `engine_version` (`"0.1.0"`) |

`/healthz` и `/readyz` — алиасы одного обработчика, и `/v1/health` разделяет
его, поэтому Kubernetes-пробы и OpenAI-совместимый health-маршрут
взаимозаменяемы. `/api/health` добавляет uptime, network и версию движка.
Используйте `/readyz` для балансировщиков нагрузки и супервизоров; используйте
`/api/health`, когда нужен более насыщенный payload.

## Логирование

Сервер логирует через `tracing`, фильтруемый стандартной переменной
`RUST_LOG` (`RUST_LOG=info` — обычная настройка; `RUST_LOG=debug` раскрывает
пробный трафик). События, о которых стоит знать, в примерном порядке частоты:

| Строка лога | Уровень | Что она вам говорит |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | По одному на чат-запрос, с `key_prefix`, `model`, `stream` и `request_id` — простейший аудит-след на запрос. |
| `request completed` | info | Логируется помощником `logging_middleware` после каждого **non-streaming** ответа `/v1/chat/completions` и `/v1/embeddings`: `method`, `path`, `status`, `latency_ms`, `trace_id`. (Streaming-чат вместо этого логирует `chat completions SSE request` в начале.) |
| `usage recorded` / `usage persisted` | info | Запись usage записана (в памяти, с tokens/стоимостью) и затем записана в таблицу `usage_records`. |
| `external probe: sending` / `external probe: returned` | debug | Health-проба `/v1/models` external backend; `matched` говорит, уложилась ли проба в 2-секундный таймаут пробы. |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | Запрос `/v1/*` отклонён billing-гейтом — клиент получил 429 плюс `Retry-After`. |
| `rpc billing gate rejected: monthly quota exceeded` | warn | Квотный гейт RPC-стороны для JWT-аутентифицированных методов (общее пользовательское окно; JSON-RPC-ответ с ошибкой). |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | Восстановление при старте: зарегистрированные администратором backends и узлы агентов загружены из базы данных и снова маршрутизируемы. |
| `Shutdown signal received, draining connections…` | info | Начался graceful shutdown (SIGINT/SIGTERM). |

## Модель таймаутов

Таймауты применяются на клиенте upstream, используемом для external backends
(`packages/core/src/backends/external.rs`):

| Таймаут | Значение | Применяется к |
| --- | --- | --- |
| Connect | 10 с | Установление TCP/TLS-соединения с upstream. |
| Read idle | 120 с на чтение | Каждый вызов upstream; каждый полученный байт сбрасывает таймер, поэтому здоровый, но медленный поток никогда не режется. |
| Non-streaming, общий | 600 с | Non-streaming вызовы чата/embeddings — медленный, но живой upstream не может держать запрос вечно. |
| Streaming (SSE) | нет | Streaming-вызовы не несут **общего дедлайна**; длинные генерации допустимы, а обнаружение зависаний полагается на read-idle таймаут. |
| Health-проба | 2 с | Проба `/v1/models`. |

## Отображение ошибок

Сбои backends отображаются в HTTP-статусы в обработчиках чата/embeddings
(`packages/core/src/gateway/server.rs`):

| Условие | HTTP | `type` / `code` | Сообщение |
| --- | --- | --- | --- |
| Статус upstream не из диапазона 2xx (`UpstreamStatus`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| Транспортный сбой upstream (`RequestFailed`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | строка транспортной ошибки |
| Любая другая ошибка backend | **500** | `server_error` / `backend_error` | строка ошибки |
| Нет backend для модели (`NoBackend`) | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| Неверный API key (`Unauthorized`) | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| Rate limit (`RateLimited`) | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

Замысел дизайна: вызывающие могут отличить «ваш provider отклонил или
сломался» (502) от «сломан сам шлюз» (500). Каждое тело ошибки имеет одну
и ту же форму в стиле OpenAI —
`{"error":{"message":...,"type":...,"code":...}}`
(`json_error_response`). 429 billing-гейта дополнительно несут заголовок
`Retry-After` и используют `quota_error`/`quota_exceeded` (квота)
и `rate_limit_error`/`rate_limit_exceeded` (лимит tier) соответственно.

## Устранение неполадок

### Свежезарегистрированный backend остаётся fail-closed, пока не пробирован

External backends стартуют в неизвестном состоянии health и сообщают
`"<url> not probed yet"`. Они становятся healthy, когда (а) отрабатывает
первый раунд health-checker'а — сразу при старте, затем каждые 60 с — или (б)
успевает fire-and-forget-проба, запущенная при регистрации или восстановлении,
обычно в течение ~1–2 секунд. До этого запросы, маршрутизированные на backend,
по дизайну валятся fail-closed.

### 404 на `/models` пробы нормальна для некоторых backends

External-проба бьёт в `GET {base}/v1/models` (или `{base}/models` для базовых
URL с префиксом пути). Некоторые OpenAI-совместимые серверы реализуют чат,
но не открывают список моделей — например, coding-plan endpoint Zhipu GLM.
**404 допускается**: backend помечается healthy, и заданный администратором
список моделей остаётся авторитетным для маршрутизации. Только реально
проваленные пробы (таймаут, сетевая ошибка, другой не-2xx) помечают backend
unhealthy.

### SSE-потоки, не производящие ничего, не тарифицируются

Streaming-ответ записывается в usage только когда поток произвёл текст
**или** принёс терминальный usage; поток, завершившийся без того и другого,
не записывается вовсе. Если вы видите запрос без соответствующей строки
`usage recorded`, проверьте, произвёл ли поток вообще контент.

### Сообщение версии

`version` в health-телах — это `CARGO_PKG_VERSION`; `build_hash` — значение
`BUILD_HASH` времени сборки, выдаваемое `packages/core/build.rs`. Сравнивайте
`build_hash` между узлами, чтобы подтвердить, что все они запускают один
и тот же артефакт.

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
