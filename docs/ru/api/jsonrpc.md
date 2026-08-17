---
title: "Справочник JSON-RPC API"
description: "JSON-RPC 2.0 API плоскости управления Arona на /api/rpc — методы chat, realtime, engine, auth, keys, providers, agents, memory, conversations, usage, billing, video и system по HTTP и WebSocket."
---

# Справочник JSON-RPC API

Arona открывает поверхность JSON-RPC 2.0 на `/api/rpc` для плоскости
управления: auth, keys, providers, agents, memory, conversations, usage,
billing, video, realtime и streaming-чат. Она дополняет OpenAI-совместимую
REST-поверхность (`/v1/*`, см. [OpenAI-совместимый REST API](./openai-rest.md));
используйте REST для аутентифицированных ключом inference-нагрузок
и JSON-RPC для управления сессиями/аккаунтами и streaming-управления.
[Краткое руководство](../guides/quickstart.md) проводит через первый
end-to-end ход.

Поверхность диспетчеризует **39 методов запросов** плюс один анонимный
WebSocket-only метод живости, `system.probe` (всего 40 методов). Каждый
запрос — это JSON-RPC 2.0 объект с `jsonrpc: "2.0"`, строкой `method`,
опциональным объектом `params` и опциональным `id`.

## Транспорт

- **HTTP POST `/api/rpc`** — запрос/ответ. Отправляйте `Content-Type:
  application/json`. JWT путешествует в заголовке `Authorization: Bearer <jwt>`.
  Тело запроса ограничено 1 МиБ.
- **WebSocket `GET /api/rpc`** — долгоживущее соединение. Браузеры не могут
  задавать кастомные заголовки при апгрейде WebSocket, поэтому JWT
  путешествует как query-параметр `?token=<jwt>`; сервер сворачивает его
  во внутренний заголовок `Authorization: Bearer`
  (см. `packages/core/src/gateway/server.rs`). Аутентифицированные сокеты
  могут оставаться подключёнными неограниченно.
- **Пакетные запросы** — POST-тело, являющееся JSON-массивом, выполняется
  элемент за элементом и отвечает JSON-массивом ответов в том же порядке.
- **Анонимный доступ** — по WebSocket без JWT публичные методы
  (`auth.register`/`auth.login`/`auth.refresh`, `providers.list`,
  `system.status`) остаются вызываемыми, а `system.probe` отвечает одиночным
  ack перед закрытием сокета. Каждый прочий метод требует валидный JWT;
  гейтимые администратором методы дополнительно требуют admin-аккаунт
  (см. легенду ниже). Анонимные сокеты также ограничены 10-секундным idle
  таймаутом.
- **Прикрепление сессии** — заголовок `x-session-id` на `POST /api/rpc`
  дополнительно выталкивает сам RPC-ответ в канал этой сессии,
  рядом со streaming-уведомлениями.

## Id

Значения `id` запроса эхом возвращаются с сохранением типа: `null` → `null`,
строки → строки, целые → числа, а всё остальное (числа с плавающей точкой,
объекты, целые вне диапазона i64) → строковое JSON-представление. Опущенный
`id` получает ответ `null`.

## Уведомления сервер → клиент (SSE sidecar)

Токены, прогресс развёртывания и realtime-события **не** доставляются по
WebSocket-сокету. Каждый streaming-RPC создаёт id сессии и выталкивает
уведомления на `GET /api/rpc/events?session=<session_id>` как server-sent
events. Подпишитесь на SSE-endpoint **до или сразу после** того, как RPC-вызов
вернёт id сессии — уведомления, выпущенные между возвратом вызова
и установлением SSE-подписки, теряются (окно до подписки). Рекомендуемый
паттерн — сначала открыть SSE-поток, затем запустить RPC.

Методы уведомлений: `chat.stream` (один token на событие от `chat.send`),
`models.progress` (прогресс скачивания модели агентом от `agents.deploy`),
`realtime.event` (серверные события открытой realtime-сессии) и
`video.progress` / `video.done` / `video.failed` (асинхронные видео-задачи).
Полный каталог см. в [Events & Notifications](./events.md).

## Коды ошибок

| Код | Имя | Значение |
| --- | --- | --- |
| `-32700` | Parse error | Тело запроса — не валидный JSON. |
| `-32600` | Invalid request | Объект запроса некорректен, например отсутствует `method`. |
| `-32601` | Method not found | Неизвестная строка `method`; сообщение её эхом повторяет. |
| `-32602` | Invalid params | `params` не прошли десериализацию для метода. |
| `-32603` | Internal error | Непредвиденный сбой сервера. |
| `-32000` | `APP_ERROR` | Общая ошибка приложения — например, диалог/provider/агент не найден, нет онлайн-агента для развёртывания. |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` — отсутствующий или неверный JWT. Также используется методами admin-token, когда bearer-token не совпадает с `ARONA_ADMIN_TOKEN` (`"Admin access required"`). |
| `-32006` | `QUOTA_ERROR` | Превышена месячная billing-квота для JWT-гейтимого RPC-метода (`chat.send`). |
| `-32007` | `ADMIN_REQUIRED` | Аутентифицированный **не-администратор** вызывает гейтимый администратором метод (`agents.*`, `engine.invoke`); сообщение включает подсказку, специфичную для метода. |

> Методы `agents.*` и `engine.invoke` — только для администраторов: они
> требуют JWT, у аккаунта которого `users.is_admin = true`. Аутентифицированный
> не-администратор отклоняется с `-32007` (`ADMIN_REQUIRED`);
> неаутентифицированный вызывающий получает стандартный `AUTH_ERROR`, чтобы
> сервер не раскрывал, что метод привилегированный.

## Легенда auth

| Легенда | Учётные данные |
| --- | --- |
| **public** | Учётные данные не требуются. |
| **JWT** | `Authorization: Bearer <jwt>` по HTTP или `?token=<jwt>` по WebSocket. |
| **admin (JWT + is_admin)** | Bearer-JWT аккаунта с `users.is_admin = true`. |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN` (настраивается через окружение; когда не задан, метод всегда отклоняется, default-deny). |

Все примерные учётные данные и адреса в этом документе — заглушки
(документационные IP RFC 5737, ключи `sk-xxx`). Полную модель auth за этой
легендой см. в [Аутентификация и безопасность](../guides/auth-security.md).

## Chat

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model` (string), `messages` (array из `{ role, content, images?, tool_calls? }`), `temperature?` (number), `max_tokens?` (integer), `conversation_id?` (string), `memory?` (bool), `extra?` (object), `tools?` (array определений функций в стиле OpenAI), `provider?` (string) | Отправить streaming-ход чата. Возвращает `{ "stream_id", "memory" }` — `memory` это состояние recall (`enabled` / `disabled` / `offline`); tokens приходят как уведомления `chat.stream` на SSE-sidecar. С `conversation_id` завершённая сохранённая история собирается на стороне сервера, и ход сохраняется. Гейтится billing (месячная квота → `-32006`); usage записывается под `jwt-<user-uuid>`. |

## Realtime (полнодуплексные аудио/видео-сессии)

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model` (string), `config?` (объект конфигурации сессии), `conversation_id?` (string) | Открыть полнодуплексную сессию против backend, обслуживающего `model`. Возвращает `{ "session_id", "stream_session" }`: используйте `session_id` для `realtime.event` / `realtime.stop` и подпишитесь на `stream_session` на SSE-sidecar, чтобы получать уведомления `realtime.event`. |
| `realtime.event` | JWT | `session_id` (string), `event` (событие клиента — append/commit/clear аудио, кадр изображения, create/cancel ответа, stop сессии) | Отправить одно событие клиента в открытую сессию; оно пересылается upstream-backend. Возвращает `{ "ok": true }`. |
| `realtime.stop` | JWT | `session_id` (string) | Закрыть и удалить сессию. Возвращает `{ "removed": bool }`. |

## Engine (общий канал восприятия/управления)

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `engine.invoke` | admin (JWT + is_admin) | `model` (string), `method` (string), `params?` (object) | Синхронный вызов запрос/ответ произвольного метода движка на backend, обслуживающем `model` — высокочастотный канал для вызовов в стиле `sensor.ingest` / `control.setpoint` (циклы 20–30 Гц). Результат — сырой ответ backend. |

## Auth

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `auth.register` | public | `email`, `password`, `name?` | Зарегистрировать аккаунт. Разрешено, только пока регистрация открыта (`ARONA_REGISTRATION_OPEN`); первый зарегистрированный пользователь становится администратором. Возвращает тот же token ответ, что и `auth.login` (`access_token`, `refresh_token`, `token_type`, `expires_in`, `user`). |
| `auth.login` | public | `email`, `password` | Войти. Возвращает `access_token`, `refresh_token`, `token_type`, `expires_in`, `user` (`{ id, email, name, is_admin }`). Rate limit по IP и аккаунту. |
| `auth.refresh` | public | `refresh_token` | Обменять refresh-token на свежий access-token (и новый refresh-token). Повторно использованные или истёкшие refresh-tokens отклоняются с `AUTH_ERROR`. |
| `auth.me` | JWT | — | Профиль текущего пользователя: `{ "id", "email", "name" }`. |

## Keys

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | Список API keys вызывающего (id, name, `key_prefix`, project, временные метки, флаг активности). |
| `keys.create` | JWT | `name`, `project?` | Создать API key. Возвращает `{ id, name, key, key_prefix, project, created_at }` — полный секрет `arona-<uuid>` в `key` показывается **один раз**; сохраните его немедленно. |
| `keys.revoke` | JWT | `key_id` | Отозвать API key. Возвращает `{ "ok": true }`. |

## Providers

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | Список известных provider'ов: встроенные официальные записи плюс кастомные, как display-метаданные (`id`, `name`, `description`, `website_domain`, `is_official`, `is_operator`). Публичен по дизайну — список не несёт учётных данных; только мутации ниже гейтятся JWT. |
| `providers.add` | JWT | `id`, `name`, `description?`, `website_domain?` | Добавить запись кастомного provider'а. Возвращает `{ "ok": true }`. |
| `providers.update` | JWT | `provider_id`, `name?`, `description?`, `website_domain?` | Обновить поля кастомного provider'а (только переданные). Возвращает `{ "ok": true }`. |
| `providers.remove` | JWT | `provider_id` | Удалить кастомного provider'а. Возвращает `{ "ok": true }`. |
| `providers.test` | JWT | — | Протестировать соединение с provider'ом. Stub: возвращает `{ "ok": true, "message": "Provider connection test not yet implemented" }`. |

## Agents

Все методы `agents.*` — только для администраторов (JWT + `is_admin`). Узлы
агентов подключаются исходяще через `GET /ws/agent`; эта RPC-группа управляет
реестром (см. [Кластер агентов](../guides/agent-cluster.md)).

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `agents.list` | admin (JWT + is_admin) | — | Список зарегистрированных узлов агентов: id, name, host, статус `online`/`offline` (по heartbeats), сводка GPU, развёрнутые модели, версия, временные метки. |
| `agents.register` | admin (JWT + is_admin) | `machine_name`, `version` | Зарегистрировать узел агента в менеджере туннелей. Возвращает `{ "agent_id", "token" }` (token — учётные данные плоскости управления агента). |
| `agents.deregister` | admin (JWT + is_admin) | `agent_id` | Дерегистрировать (отключить) агента. Возвращает `{ "ok": true }`. |
| `agents.status` | admin (JWT + is_admin) | `agent_id` | Статус конкретного агента: флаг online, host, сводка GPU, загруженные модели, загрузка GPU, временные метки heartbeat/подключения. |
| `agents.deploy` | admin (JWT + is_admin) | `model_id`, `agent_id?` (пусто/отсутствует = наименее загруженный узел; ошибка, если ни один не онлайн) | Развернуть модель на агенте. Возвращает `{ "ok": true, "stream_id" }` — подпишитесь на `stream_id` на SSE-sidecar для уведомлений о скачивании `models.progress`. |
| `agents.stop` | admin (JWT + is_admin) | `agent_id`, `model_id` | Остановить развёрнутую модель. Возвращает `{ "ok": true, "stream_id": null }` (потока прогресса нет). |

## Memory

Долговременная память обслуживается сервисом entelecheia Philia через
WebSocket; сбои никогда не блокируют чат (см.
[Memory Gateway](../guides/memory-gateway.md)).

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | Состояние memory gateway: `{ "enabled", "writeback", "events" }` — флаги плюс до 50 недавних событий активности (сначала новые). |
| `memory.delete` | JWT | `node_id` | Удалить сохранённый узел памяти. Возвращает `{ "deleted": bool }`. |

## Conversations

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | Список диалогов вызывающего с временными метками относительного возраста. |
| `conversations.create` | JWT | `title?` (по умолчанию `New Conversation`) | Создать диалог. Возвращает новый объект диалога. |
| `conversations.get` | JWT | `conversation_id` (legacy-алиас: `id`) | Получить диалог с его сообщениями. Проверка владения; кросс-пользовательский доступ отклоняется. |
| `conversations.delete` | JWT | `conversation_id` (legacy-алиас: `id`) | Удалить диалог (только владелец). Возвращает `{ "ok": true }`. |

> `conversations.get` / `conversations.delete` также принимают legacy-ключ
> `id` от старых dashboard-клиентов; `conversation_id` побеждает, когда
> присутствуют оба.

## Usage

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?` (integer, по умолчанию 50, ограничено 1–200), `offset?` (integer, по умолчанию 0), `project?` (string) | Пагинированные записи usage вызывающего, сначала новые, покрывающие и API-key строки (префикс `arona-XX`), и строки с атрибуцией через JWT (`jwt-<user-uuid>`). Возвращает `{ "records", "total", "limit", "offset", "project" }`; фильтр `project` сужает до строк только с ключами. |

## Billing

Tiers, квоты и учёт usage описаны в
[Billing и usage](../guides/billing-usage.md).

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | Текущее состояние billing: `{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` — месячный usage (`cost_usd`, tokens, число запросов) и оставшаяся квота. |
| `billing.plan.set` | admin token | `user_email`, `tier` | Установить billing tier пользователя. Возвращает `{ "ok": true }`. Отклоняется с `AUTH_ERROR`, когда bearer не совпадает с `ARONA_ADMIN_TOKEN`. |
| `billing.video.pricing.get` | JWT | — | Таблица цен на видео. Возвращает `{ "pricing": [...] }`. |
| `billing.video.pricing.set` | admin token | `model`, `mode?` (по умолчанию `per_second_resolution`), `base_price?` (number, по умолчанию 0), `price_per_second?` (number, по умолчанию 0), `price_per_frame?` (number, по умолчанию 0), `resolution_coeff?` (object), `currency?` (по умолчанию `USD`), `enabled?` (bool, по умолчанию `true`) | Upsert цен на видео для модели. Возвращает `{ "ok": true }`. Отклоняется с `AUTH_ERROR`, когда bearer не совпадает с `ARONA_ADMIN_TOKEN`. |

## Video

Асинхронные задачи генерации видео (см.
[Realtime и видео](../guides/realtime-video.md)). Прогресс задач выталкивается
как уведомления `video.progress` / `video.done` / `video.failed` в канал
сессии.

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`, `prompt`, `negative_prompt?`, `images?` (array из `{ data_base64, mime_type }`), `duration_seconds?` (integer), `width?` (integer), `height?` (integer), `provider?` (string), `extra?` (object) | Отправить асинхронную задачу генерации видео. Возвращает `{ "job_id", "stream_id" }` — подпишитесь на `stream_id` для уведомлений о прогрессе. |
| `video.get` | JWT | `job_id` (UUID) | Опросить статус/результат задачи (status, progress, result, error, cost). |
| `video.list` | JWT | `limit?` (integer, по умолчанию 20) | Список задач вызывающего. Возвращает `{ "jobs": [...] }`. |
| `video.cancel` | JWT | `job_id` (UUID) | Отменить выполняющуюся задачу. Возвращает `{ "ok": true }`. |

## System

| Метод | Auth | Params | Описание |
| --- | --- | --- | --- |
| `system.status` | public | — | Агрегированный статус шлюза: `{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`. |
| `system.probe` | anonymous (WS only) | — | Разовая проба живости по WebSocket-транспорту. Сервер подтверждает `{ "ok": true, "status": "ok" }` и затем закрывает сокет — анонимные посетители никогда не держат открытое соединение. Любой другой метод на неаутентифицированном сокете отклоняется с `AUTH_ERROR`. |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
