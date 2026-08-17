---
title: "Краткое руководство"
description: "Сквозное знакомство с Arona на встроенном mock upstream: migrate, serve, регистрация backend, создание API key и чат."
---

# Краткое руководство

Это руководство проведёт вас через полную сквозную настройку Arona на одной
машине с использованием **встроенного mock upstream** — без реальных весов моделей,
GPU и внешнего API-аккаунта. По итогам у вас будут:

- работающий шлюз Arona (`/v1/*` — OpenAI-совместимый REST API, плюс
  управляющая плоскость JSON-RPC `/api/rpc`),
- mock upstream, зарегистрированный как backend типа `external`,
- учётная запись пользователя и API key,
- работающий non-streaming **и** streaming чат против mock,
- записи usage, видимые через `usage.list`.

## Предварительные требования

- **Rust toolchain** (см. `rust-toolchain.toml` в корне репозитория).
- **Python 3** с `aiohttp` — нужен только для mock upstream
  (`pip install aiohttp`).
- **Работающий PostgreSQL** и URL подключения к нему.

## 1. Настройка окружения

Arona читает свою конфигурацию из переменных окружения **при запуске
процесса**. Две из них обязательны: `DATABASE_URL` и `JWT_SECRET` — без них
сервер отказывается запускаться (если только не задан `MOCK_MODE=1`).
`ARONA_ADMIN_TOKEN` настоятельно рекомендуется: без него каждый маршрут
`/api/admin/*` возвращает 401.

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

Эти переменные читаются один раз при старте процесса — если вы их меняете,
перезапустите сервер. Полный справочник переменных см. в
[Конфигурация](configuration.md).

## 2. Миграция и запуск сервера

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

Для свежей базы данных достаточно одного `serve`: он сам выполняет миграции
при старте. По умолчанию сервер слушает `0.0.0.0:8420` (переопределяется через
`ARONA_HOST` / `ARONA_PORT`).

## 3. Запуск mock upstream

Во втором терминале:

```bash
python3 scripts/mock/server.py
```

Mock — это aiohttp-сервер, который по умолчанию слушает `127.0.0.1:8429`
(порт переопределяется через `ARONA_MOCK_PORT`). При старте он печатает свой
API key, а также обслуживает `GET /api/test-key`, возвращающий
`{"api_key": ..., "base_url": ...}`. Он предоставляет несколько id моделей —
включая `gpt-5.5`, используемый ниже, — и отвечает как на обычные, так и на
streaming chat completions.

Сохраните напечатанный ключ:

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. Регистрация mock как external backend

Backends регистрируются через admin HTTP API:

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

Backend немедленно пробируется при регистрации и становится healthy в течение
~1–2 секунд; пока проба не завершена, он остаётся в fail-closed состоянии «ещё
не пробирован» (см. блок устранения неполадок ниже). Конфигурация сохраняется,
поэтому backend переживает перезапуск.

## 5. Регистрация аккаунта и вход

Аккаунты живут на плоскости JSON-RPC, `POST /api/rpc`. Так как задан
`ARONA_REGISTRATION_OPEN=1`, метод `auth.register` открыт; первый
зарегистрированный пользователь становится администратором.

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

Пароли должны быть не короче 8 символов **и** содержать как минимум 3 из 4
категорий символов (заглавные, строчные, цифры, спецсимволы). Затем войдите,
чтобы получить пару JWT:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

Сохраните `access_token` из ответа:

```bash
export JWT="<access_token from the login response>"
```

## 6. Создание API key

`keys.create` аутентифицируется через JWT и возвращает **полный** секрет
`arona-{uuid}` ровно один раз — в базе хранится только его SHA-256-хеш,
поэтому скопируйте его сейчас:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. Чат (non-streaming)

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

Вы получаете объект завершения в стиле OpenAI с `choices[0].message`
и блоком `usage`.

## 8. Чат (streaming)

Тот же endpoint с `"stream": true` отвечает server-sent events:
по одному чанку `data:` на token, завершающийся финальным чанком
`data: [DONE]`:

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. Проверка usage

Каждый ход чата записывает строку usage под префиксом ключа. Запросите её
с помощью JWT:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

Вы должны увидеть одну или несколько записей для запросов `gpt-5.5`,
сделанных выше.

## Устранение неполадок

- **`No backend available for model: gpt-5.5` (HTTP 404, `code:
  model_not_found`)** — ни один зарегистрированный backend не обслуживает этот
  id модели. Либо backend вообще не был зарегистрирован (или его список
  `models` не включает этот id), либо вызов регистрации завершился ошибкой.
  Проверьте через `GET /api/admin/backends` (admin token).
- **`all backends unhealthy` (HTTP 500, `backend_error`)** — backend *есть*
  для этой модели, но ни один кандидат не healthy. Свежезарегистрированный
  external backend начинает в fail-closed состоянии «ещё не пробирован»
  и становится healthy, как только завершится проба при регистрации, через
  ~1–2 секунды; если вы отправляете чат в этом окне, вы получите эту ошибку.
  Повторите через мгновение или проверьте, что mock действительно запущен
  на `127.0.0.1:8429`.
- **HTTP 401 на `/v1/*`** — отсутствующий заголовок `Authorization` даёт
  `Missing Authorization header. Use: Bearer <api_key>`; неизвестный ключ даёт
  `Invalid API key`. Перепроверьте `$AR_KEY` (полный секрет, не префикс).
- **HTTP 401 `Admin access required` на `/api/admin/*`** — bearer-token не
  совпадает с `ARONA_ADMIN_TOKEN`, либо переменная не задана (тогда маршрут
  всегда отклоняет запрос). Перезапустите сервер после её установки.
- **`auth.register` завершается ошибкой «Registration is closed»** —
  регистрация отключена, если `ARONA_REGISTRATION_OPEN` не имеет truthy-значения.
  Установите `ARONA_REGISTRATION_OPEN=1` **до** запуска сервера (переменная
  читается при старте), либо станьте самым первым пользователем — первому
  зарегистрированному пользователю разрешено всегда, и он становится
  администратором.
- **HTTP 429 rate limits** — могут сработать три независимых лимита:
  - лимит в памяти на ключ, по умолчанию 60 RPM
    (`ARONA_API_RATE_LIMIT_RPM`) → `Rate limit exceeded. Try again later.`;
  - лимит 10 RPM на ключ бесплатного billing tier → 429 с заголовком
    `Retry-After: 60`;
  - месячная квота бесплатного tier $1 / 100 тыс. tokens → 429
    с `Retry-After`, указывающим на следующий расчётный период.

## Дальнейшие шаги

- [Конфигурация](configuration.md) — все переменные окружения.
- [Backends](backends.md) — типы backends, семантика URL и probing.
- [Развёртывание](deployment.md) — bare metal, systemd, Docker.
- [OpenAI-совместимый REST API](../api/openai-rest.md) — полная поверхность `/v1/*`.
- [JSON-RPC API](../api/jsonrpc.md) — управляющая плоскость, использованная выше.

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
