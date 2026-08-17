---
title: "Конфигурация"
description: "Справочник по каждой переменной окружения, которую читает сервер Arona, с значениями по умолчанию и семантикой."
---

# Конфигурация

Arona настраивается **целиком через переменные окружения**, которые читаются
один раз при запуске процесса (`Config::load` в
`packages/core/src/config.rs`, плюс несколько переменных, читаемых при первом
использовании). Файла конфигурации нет: измените переменную и перезапустите
сервер, чтобы она вступила в силу.

Эта страница — справочник по всему, что читает код сервера, сгруппированный
по областям. Mock-only и рантайм-переменные включены для полноты.

## Справочная таблица

| Переменная | По умолчанию | Назначение |
| --- | --- | --- |
| `DATABASE_URL` | *(обязательна)* | URL подключения к PostgreSQL. |
| `JWT_SECRET` | *(обязательна вне mock mode)* | Секрет для подписи JWT. |
| `ARONA_HOST` | `0.0.0.0` | Адрес привязки (fallback на `SHITTIM_CHEST_HOST`). |
| `ARONA_PORT` | `8420` | Порт привязки (fallback на `SHITTIM_CHEST_PORT`). |
| `ARONA_DATA_DIR` | не задана | Локальная директория данных. |
| `ARONA_ADMIN_TOKEN` | не задан | Bearer-token для `/api/admin/*` и admin-методов RPC. |
| `ARONA_REGISTRATION_OPEN` | `0` | Truthy-значение (`1`/`true`/`yes`/`on`) открывает регистрацию. |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | Лимит запросов в минуту на ключ в памяти; `0` блокирует всё. |
| `MOCK_MODE` | не задан | Наличие (любое значение) включает dev mock mode. |
| `MOCK_SEED_PATH` | не задан | Файл с сырым SQL-сидом, выполняемый в mock mode. |
| `ARONA_MEMORY_URL` | не задан | WebSocket-URL шлюза памяти Philia. |
| `ARONA_MEMORY_TOKEN` | не задан | Токен для шлюза памяти. |
| `ARONA_MEMORY_WRITEBACK` | `true` | Записывать завершённые ходы чата обратно в память; принимает `true`/`false` (любое другое значение откатывается к значению по умолчанию). |
| `ARONA_AGENT_NAME` | `arona-agent` | Идентичность агента на GPU-узле. |
| `ARONA_PANEL_URL` | `localhost:8080` | Куда подключается агент (`ws://<panel_url>/ws/agent`). |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | Локальный агент evernight для bridge-URL `evernight://`. |
| `ARONA_MISTRALRS` | не задан | Наличие принудительно включает движок Mistral.rs для планов моделей Gguf. |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | Интерфейс, к которому привязывается запущенный сервер модели llama.cpp. |
| `HF_ENDPOINT` | `https://huggingface.co` | Базовый URL Hugging Face для скачивания моделей. |
| `GITHUB_TOKEN` | не задан | Access token для реестра моделей GitHub. |
| `RUST_LOG` | не задан | Фильтр трассировки, например `info` или `arona=debug,info`. |

## Обязательные переменные

### `DATABASE_URL`

URL подключения к PostgreSQL. **Обязательна**: сервер завершается с ошибкой
`FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.` когда она
пуста, а подкоманда CLI `migrate` отказывается запускаться. Схема
создаётся/обновляется автоматически командой `serve` при старте.

### `JWT_SECRET`

Секрет для подписи пар JWT access/refresh, выдаваемых `auth.login` и
`auth.register`. **Обязателен в production**: в коде встроен development-запасной
вариант (`dev-secret-not-for-production-use-only-32chars`), но сервер
отказывается запускаться с ним, если только не задан `MOCK_MODE=1`:

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

Используйте длинное случайное значение (например, `openssl rand -hex 32`).

## Сервер

### `ARONA_HOST` / `ARONA_PORT`

Адрес и порт привязки шлюза. Для обратной совместимости они откатываются
к `SHITTIM_CHEST_HOST` / `SHITTIM_CHEST_PORT`; конечные значения по умолчанию —
`0.0.0.0:8420`.

### `ARONA_DATA_DIR`

Необязательная локальная директория данных, передаваемая в состоянии приложения
компонентам, которым нужно место для временных файлов. По умолчанию не задана.

## Безопасность и контроль доступа

### `ARONA_ADMIN_TOKEN`

Bearer-token, защищающий HTTP-маршруты `/api/admin/*` (`POST/GET/DELETE
/api/admin/backends`, `/api/admin/aliases`) и RPC-методы
`billing.plan.set` / `billing.video.pricing.set`. Когда он **не задан**, каждый
из этих маршрутов отклоняет запрос с `Admin access required` (401) — значения
по умолчанию нет. Установите надёжное случайное значение перед запуском сервера.

### `ARONA_REGISTRATION_OPEN`

Открывает самостоятельную регистрацию через `auth.register`. Truthy-значения —
ровно `1`, `true`, `yes`, `on` (без учёта регистра); всё остальное — включая
`0`, `false`, `off` или незаданную/пустую переменную — остаётся закрытым.
Флаг читается один раз при старте. **Первому зарегистрированному пользователю
разрешено всегда** (даже когда регистрация закрыта), и он становится
администратором.

### `ARONA_API_RATE_LIMIT_RPM`

Лимит запросов на ключ в памяти со скользящим окном (запросов в минуту),
применяемый к каждому аутентифицированному запросу `/v1/*` (chat, embeddings,
video, models), по ключу — хеш API key (или метка `u:<email>` для
принимающего JWT `GET /v1/models`). Плоскость RPC этим лимитером не
ограничивается — экстракторы auth `/v1/*` являются единственными
вызывающими. По умолчанию `60`. Значение парсится один раз в
процесс-глобальный `OnceLock`. **Значение `0` блокирует каждый
запрос** — проверка это `entry.len() >= rpm`, так что при `0` ни один запрос
не проходит. Это общешлюзовый лимит; billing tiers накладывают поверх свои
собственные лимиты на ключ.

## Разработка

### `MOCK_MODE`

Dev mock mode, включаемый **по наличию** — проверка это
`std::env::var("MOCK_MODE").is_ok()`, так что *любое* значение (включая `0`
или пустое, но заданное) включает его. Он:

- снимает требование `JWT_SECRET` (встроенный dev-секрет становится
  допустимым);
- засеивает четыре демо-аккаунта (`demiurge@celestia.world`,
  `momoi@celestia.world`, `midori@celestia.world`, `yuzu@celestia.world`,
  пароль `33550336`);
- ждёт завершения сида, прежде чем привязать listener.

Никогда не используйте его в production.

### `MOCK_SEED_PATH`

Только в mock mode: указывает на файл с сырым SQL, выполняемый вместо
встроенного сида аккаунтов (операторы разделяются `;`, комментарии `--`
пропускаются). Если файл не читается, используется встроенный сид как запасной
вариант.

## Memory gateway

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

Конфигурация шлюза долговременной памяти (entelecheia Philia). Память
**полностью отключена**, если только обе переменные `ARONA_MEMORY_URL`
и `ARONA_MEMORY_TOKEN` не заданы и не пусты. Когда включено:

- завершённые ходы чата извлекаются и инъецируются как контекст, и
- `ARONA_MEMORY_WRITEBACK` (по умолчанию `true`) управляет тем, записываются
  ли завершённые ходы обратно в сервис памяти; `0` или `false` отключает
  writeback.

Сбои памяти никогда не блокируют чат; итоговое состояние отражается
в заголовке ответа `X-Arona-Memory` (`enabled` / `disabled` / `offline`).

## Идентичность агента и кластер

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

Идентичность бинарника агента GPU-узла (`_agent`): `ARONA_AGENT_NAME`
(по умолчанию `arona-agent`) сообщается панели как имя/id агента, а
`ARONA_PANEL_URL` (по умолчанию `localhost:8080`) — это адрес, куда
подключается агент (`ws://<panel_url>/ws/agent`).

Собственный HTTP API агента **жёстко** привязан к `0.0.0.0:5790` — для него
не существует переменной окружения с адресом привязки.

### `ARONA_AGENT_BIND_ADDR`

Интерфейс, к которому привязывается **запущенный сервер модели llama.cpp**,
когда агент развёртывает модель Gguf, чтобы до движка можно было достучаться
с других машин (например, `0.0.0.0`). По умолчанию `127.0.0.1`. Обратите
внимание: это *не* адрес привязки HTTP API агента (он фиксирован на
`0.0.0.0:5790`).

## Evernight bridge

### `ARONA_EVERNIGHT_URL`

WebSocket-URL локального агента evernight, используемый для преобразования
bridge-URL `evernight://` в локальные TCP-форварды. По умолчанию
`ws://127.0.0.1:3001/ws`.

## Рантайм моделей и скачивание

### `ARONA_MISTRALRS`

Наличие (любое значение) принудительно включает движок Mistral.rs для планов
моделей Gguf, которые иначе по умолчанию использовали бы llama.cpp. Семантика
наличия как у `MOCK_MODE`.

### `HF_ENDPOINT`

Базовый URL для скачивания моделей Hugging Face (источники `hf:`), по
умолчанию `https://huggingface.co`. Установите зеркало, например
`https://hf-mirror.com`, когда huggingface.co недоступен. Читается
загрузчиком моделей; завершающий слэш обрезается.

### `GITHUB_TOKEN`

Access token, используемый реестром моделей GitHub (источники `gh:`) для
доступа к API. По умолчанию не задан; без него действуют лимиты частоты
запросов GitHub API.

## Логирование

### `RUST_LOG`

Стандартный фильтр трассировки, применяемый `tracing_subscriber` при старте,
например `info` или `arona=debug,info`. Следует обычной семантике `RUST_LOG`
(`error`/`warn`/`info`/`debug`/`trace`, переопределения по целям).

## Значения по умолчанию одним взглядом

| Настройка | По умолчанию |
| --- | --- |
| Адрес / порт привязки | `0.0.0.0:8420` |
| Лимит API-запросов на ключ | 60 RPM |
| Имя агента | `arona-agent` |
| Panel URL | `localhost:8080` |
| Writeback памяти | включён |
| Регистрация | закрыта |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
