---
title: "Тестирование"
description: "Пирамида тестов Arona — unit-тесты, герметичная интеграция, интеграция под гейтом PostgreSQL, smoke-тесты на живом сервере, mock server и дисциплина smoke-тестов с реальными учётными данными."
---

# Тестирование

Тесты Arona выстроены слоями так, чтобы обычный запуск `cargo test` был
быстрым, герметичным и не требовал ни базы данных, ни сети, а более тяжёлые
наборы были явными opt-in, проверяющими реальную проводную поверхность
и реальный PostgreSQL. Эта страница описывает слои, команды, которые их
запускают, и дисциплину workspace вокруг smoke-прогонов с реальными учётными
данными.

## Unit-тесты

Основная масса покрытия — обычные unit-тесты внутри `packages/core/src`:
217 функций `#[test]` / `#[tokio::test]`, плюс ещё ~23 в
`packages/agent` и `packages/cli`. Они запускаются так:

```bash
cargo test --workspace
```

Ни сети, ни базы данных. Ключевые наборы:

- **auth.rs** — политика паролей (≥8 символов И ≥3 из 4 категорий
  символов), приведения `::uuid` в сыром SQL INSERT/REVOKE, значения по
  умолчанию запросов и чтения admin-флага, откатывающиеся к `false`.
- **billing/mod.rs** — математика квот по измерениям стоимости *или* tokens,
  месячное окно (`month_start`, `seconds_until_month_end`), потолок rate limit
  (срабатывает только *на* RPM, `None` = безлимит), SQL-shape-стражи для
  запросов месячного usage / tier / окна ключа и `estimate_usage`,
  предпочитающий числа, сообщённые upstream.
- **routing/mod.rs** — разрешение алиасов, сопоставление суффикса `:latest`,
  подсказки provider, выбор наименее загруженного и закрепление диалога.
- **gateway/mod.rs** — регистрация agent-backend: регистрация
  `agent-{model_id}`, повторная регистрация, заменяющая (не дублирующая),
  и дерегистрация, восстанавливающая роутер.

## Герметичная интеграция (всегда запускается, без БД)

`packages/core/tests/gateway_integration.rs` содержит три всегда запускаемых
теста, которые проверяют реальную логику сериализации/контрактов, не касаясь
базы данных:

- **A1** — контракт сериализации эха id JSON-RPC: числовые, строковые
  и null id запросов проходят кругооборот через enum `Id` plana с сохранением
  типа.
- **A2** — контракт кода ошибки admin-гейта: `AUTH_ERROR` (-32005, аноним)
  и `ADMIN_REQUIRED` (-32007, аутентифицированный не-администратор) остаются
  различными, живут в определяемом реализацией диапазоне и никогда не
  сталкиваются с кодами plana или billing `QUOTA_ERROR` (-32006).
- **A3** — `estimate_usage`: usage, сообщённый upstream, побеждает дословно;
  без него локальная оценка tokenizer выдаёт ненулевые счётчики
  prompt/completion, чья сумма равна их итогу.

`packages/core/tests/smoke.rs` добавляет ещё три всегда запускаемых теста:
определение оборудования, корневой путь реестра моделей и значения
конфигурации по умолчанию при `MOCK_MODE=1`.

## Интеграция под гейтом PG

Полный in-process набор шлюза — `packages/core/tests/gateway_integration.rs` —
поднимает полный axum-роутер на случайном loopback-порту, регистрирует
одноразовые OpenAI-совместимые mock upstream через реальный admin API
и гоняет проводную поверхность через reqwest. Поскольку `AuthManager` говорит
с PostgreSQL на каждом пути (даже `MOCK_MODE=1` только засеивает аккаунты
*в базу данных*), этот набор гейтится за `ARONA_TEST_PG=1` и по умолчанию
пропускается. 10 тестов:

- **T1** регистрация + вход + `keys.create`/`keys.list` (сырой ключ
  маскируется в списках, префикс `arona-`).
- **T2** REST-чат с сохранением записей usage в PostgreSQL.
- **T3** эхо id JSON-RPC по проводу (пути успеха и ошибки).
- **T4** admin-гейт на `agents.list`: аноним → `AUTH_ERROR`, не-администратор →
  `ADMIN_REQUIRED`.
- **T5** upstream 401 → HTTP 502 `bad_gateway` с деталями upstream.
- **T6** проба при регистрации публикует модели (модель появляется
  в `GET /v1/models` в течение 10 с без статического списка моделей).
- **T7** сохранение диалога через `chat.send` (оба хода попадают
  в `conversations.get`).
- **T8** billing-гейт free tier: 10 RPM на ключ, 11-й запрос в окне
  даёт 429 `rate_limit_exceeded`.
- **T9** SSE-поток с терминальным usage, записанным от upstream.
- **T10** некорректный JSON → 400; неизвестная модель → 404 `model_not_found`.

Запустите его с одноразовым-Postgres однострочником из доков модуля
(gateway_integration.rs:18-26):

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

Это примерные учётные данные только для одноразового тестового контейнера —
никогда не указывайте этим на реальную базу данных.

## Smoke-тесты на живом сервере

`packages/core/tests/auth_flow.rs` проходит по полной цепочке
`register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` против **живого** сервера Arona, зеркаля развёрнутый
auth-цикл. Он `#[ignore]`-нут по умолчанию — обычный запуск `cargo test`
никогда не касается сети. Запустите его явно:

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

Ручки:

- `ARONA_TEST_URL` — базовый URL (по умолчанию `http://127.0.0.1:8420`).
- `ARONA_TEST_EXPECT_CHAT=1` — жёстко проверяет, что `POST
  /v1/chat/completions` возвращает 200. Без него тест проверяет только, что
  auth прошёл (не 401/403), потому что в целевом окружении может не быть
  настроенного inference-provider'а.

Набор также включает негативные тесты: неаутентифицированный chat completion
и неаутентифицированный `GET /v1/models` должны оба отклоняться с 401.

## Mock server

`scripts/mock/server.py` — это aiohttp-основанный OpenAI-совместимый фейк,
используемый кратким руководством и smoke-прогонами. Он обслуживает
`POST /v1/chat/completions` (non-stream и SSE), `GET /v1/models`,
`GET /api/health`, JSON-RPC WebSocket/HTTP-поверхность на `/api/rpc`,
SSE-sidecar на `/api/rpc/events` и `GET /api/test-key`, который возвращает
mock API key, чтобы другие сервисы могли его обнаружить. По умолчанию он
слушает порт 8429 (переопределяется через `ARONA_MOCK_PORT`, хост — через
`ARONA_MOCK_HOST`). [Краткое руководство](quickstart.md) использует его,
чтобы поднять end-to-end окружение без реальных model providers.

## Дисциплина smoke-тестов с реальными учётными данными

Smoke-прогоны против реальных providers (DeepSeek / GLM) намеренно
**не** являются тестами репозитория — им нужны реальные учётные данные
и реальные деньги, поэтому они не могут жить в CI или в git-дереве.
Конвенция workspace, задокументированная в доках модуля gateway_integration
(gateway_integration.rs:54-55), такова:

- Файлы-доказательства живут в `/mnt/work/arona-pr*-smoke.md` — локально
  в workspace, никогда не коммитятся в git.
- Учётные данные приходят только из окружения; бюджеты держатся маленькими.
- Каждый PR, затрагивающий путь инференса, получает письменную запись-
  доказательство.

Mock server — замена этих прогонов в CI и локальной разработке;
smoke с реальными учётными данными — человеческий шаг на момент релиза.

## CI

`.github/workflows/ci.yml` запускает `cargo fmt`, `cargo clippy`, `cargo test
--workspace` и `cargo-deny` на self-hosted раннерах организации
(`[self-hosted, linux, x64, local]`); `ci-hosted.yml` зеркалит те же проверки
на GitHub-hosted раннерах. `.github/workflows/docs.yml` собирает этот
документационный сайт через lagrange и деплоит его на GitHub Pages при пушах,
затрагивающих `docs/**`.

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
