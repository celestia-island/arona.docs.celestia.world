---
title: "Billing и usage"
description: "Записи usage, стоимость по моделям, billing tiers, принудительное применение квот и rate limit, ключи с привязкой к проекту, цены на видео и RPC usage.list."
---

# Billing и usage

Arona учитывает каждый запрос модели и принудительно применяет tier-квоты
и rate limits на шлюзе. Цены по моделям берутся из общей таблицы цен plana
(никогда не перереализуются в arona), usage сохраняется как строки
`usage_records`, а вся месячная картина открывается через RPC `usage.list`.

## Записи usage

Каждый учитываемый запрос оканчивается одной строкой в таблице
`usage_records` (`m20250101_000006_create_usage_records`):

| Колонка | Тип | Значение |
| --- | --- | --- |
| `id` | `UUID` | Первичный ключ, генерируется. |
| `api_key_id` | `VARCHAR(64)` | **Префикс ключа** — первые 8 символов API key (ключи выглядят как `arona-{uuid}`) — или синтетический id `jwt-<user-uuid>` для RPC-каналов с атрибуцией через JWT. |
| `model` | `VARCHAR(128)` | Id модели, на которую маршрутизирован запрос. |
| `backend` | `VARCHAR(64)` | Вид backend: `gateway`, `rpc`, `realtime` или имя capability backend. |
| `prompt_tokens` | `INTEGER` | Входные tokens, сообщённые upstream или оценённые. |
| `completion_tokens` | `INTEGER` | Выходные tokens, сообщённые upstream или оценённые. |
| `total_tokens` | `INTEGER` | Сумма первых двух. |
| `cost` | `DOUBLE PRECISION` | Вычисленная стоимость в USD; `NULL`, когда у модели нет строки цены. |
| `created_at` | `TIMESTAMPTZ` | Когда завершился запрос. |

Индексы есть по `api_key_id`, `model` и `created_at` (колонкам, которые
сканируют месячная агрегация и окна rate limit).

## Каналы записи

Usage записывается на каждом учитываемом канале:

1. **REST non-streaming** — `POST /v1/chat/completions` и
   `POST /v1/embeddings` записывают точный usage, сообщённый upstream,
   после того как ответ был произведён.
2. **REST streaming (SSE)** — usage, сообщённый upstream, побеждает, когда
   поток его принёс (поле `usage` терминального чанка, совместимого с
   OpenAI); иначе как есть записывается оценка локального CJK-осведомлённого
   tokenizer (`estimate_usage`). Потоки, не произведшие ни текста, ни
   usage, **не** записываются вовсе.
3. **RPC `chat.send`** — применяется та же логика «оценка против upstream»;
   строки атрибутируются синтетическим id `jwt-<user-uuid>`, чтобы их можно
   было соединить обратно с пользователем.
4. **Realtime-сессии** — каждый завершённый транскрипт `response_done`
   записывает свой usage tokens (когда он ненулевой) под `jwt-<user-uuid>`
   с backend `realtime`.
5. **Видео-задачи** — завершённая задача записывает явную стоимость
   в долларах (см. [Video pricing](#video-pricing)); счётчики tokens
   равны нулю.

Запись best-effort: неудачный INSERT логируется и никогда не валит запрос.

## Расчёт стоимости

Стоимость вычисляется из канонической таблицы цен за 1 млн tokens
(`plana_llm_provider::metering::lookup_pricing`, общей для всех сервисов
celestia-island):

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

Сопоставление моделей в таблице — подстрочное по id модели в нижнем регистре
(более специфичные семейства побеждают). Когда у модели нет строки цены,
`cost` равен `NULL`. **Не перереализуйте цены в arona — обновляйте таблицу
plana.**

## Tiers

Tiers живут в таблице `billing_tiers`, засеянной при первой миграции
(`m20250101_000007_create_billing_tiers`). Колонка квоты `NULL` означает
«безлимитно» для этого измерения. Пользователи без `tier_id` откатываются
к засеянному tier `free`.

| Tier | Месячная квота USD | Месячная квота tokens | Per-key RPM |
| --- | --- | --- | --- |
| `free` | $1.00 | 100 000 | 10 |
| `pro` | $20.00 | 5 000 000 | 120 |
| `enterprise` | безлимитно (`NULL`) | безлимитно (`NULL`) | 1000 |

Назначение tier — административная операция (RPC `billing.plan.set`);
текущий tier и usage открываются через `billing.plan`.

## Принудительное применение квот и rate limit

### REST (`/v1/*`)

Перед каждым **учитываемым** REST-endpoint — `/v1/chat/completions`,
`/v1/embeddings` и `/v1/video/generations` — шлюз применяет два
гейта для запросов, аутентифицированных ключом:

- **Месячная квота** — лимиты `monthly_quota_usd` и `quota_tokens` tier
  проверяются против usage, накопленного с первой секунды текущего
  календарного месяца. Достижение лимита по любому измерению срабатывает
  как отклонение гейта.
- **Per-key rate limit** — `rate_limit_rpm` tier проверяется против числа
  запросов, записанных для этого префикса ключа в окне последних 60 секунд.
  (`/v1/models` и health-endpoints не учитываются и не гейтятся.)

Отклонение — это HTTP **429 Too Many Requests** с заголовком `Retry-After`
и телом ошибки в стиле OpenAI:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| Отклонение | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| Месячная квота исчерпана | `quota_error` / `quota_exceeded` | Секунды до начала **следующего календарного месяца** |
| Превышен rate limit tier | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

Аутентифицированный через JWT `chat.send` проходит через тот же гейт месячной
квоты, но против **общего пользовательского** окна (у вызова нет API key).
Отклонение — это JSON-RPC-ошибка с определяемым реализацией кодом `-32006`
(`QUOTA_ERROR`) и тем же сообщением, что и при отклонении квоты REST.
Per-key rate limit на пути RPC нет — rate limit привязан к ключу, а у RPC-вызовов
ключа нет. RPC-методы realtime и video **не гейтятся** квотой.

## Компромисс fail-open

Billing **best-effort by design**. Если запрос к базе данных за проверкой
квоты или rate limit не удаётся, проверка возвращает `Unknown`, и запрос
**разрешается** (только логируется) вместо блокировки чата. Оператор может
полагаться на 429 для защиты ёмкости, но не должен считать их жёсткой
гарантией, когда база данных нездорова — задокументированный компромисс
в пользу доступности пути чата, а не строгого учёта.

## Ключи с привязкой к проекту

API keys можно создавать с меткой `project` (`api_keys.project`,
`default`, когда не задана). Принудительное применение квоты её учитывает:

- Ключ с меткой проекта, отличной от `default`, проверяет свою квоту против
  usage, атрибутированного **собственному бакету этого проекта**
  (`PROJECT_MONTHLY_USAGE_SQL`).
- Ключи `default` / без метки сохраняют **общее пользовательское** окно,
  как до появления проектов.

RPC-строки с атрибуцией через JWT (`jwt-<user-uuid>`) не несут метку проекта
и **намеренно исключены** из проектных окон — они по-прежнему считаются
в общем пользовательском окне, поэтому проект нельзя «спрятать», отправив
трафик через RPC-канал.

## Video pricing

Генерация видео использует моделе-специфичные, задачные цены (цена за token
для видео бессмысленна). Строки цен живут в таблице `video_pricing`;
`compute_cost` откатывается к placeholder-значению по умолчанию, когда ни одна
строка не настроена.

| Режим | Стоимость |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (по умолчанию) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` — это JSON-объект, индексированный значением короткой
стороны в пикселях (например, `"768"`); ключ `"*"` — запасной. Строка цены
по умолчанию: режим `per_second_resolution`, `base_price` 0.0,
`price_per_second` 0.005, `resolution_coeff {"*": 1.0}`. Строками управляют
через `billing.video.pricing.get` (любой JWT) и `billing.video.pricing.set`
(Bearer `ARONA_ADMIN_TOKEN`); вычисленная стоимость записывается на ключ
задачи, когда задача завершается.

## usage.list

`usage.list` (JWT) возвращает пагинированные записи usage вызывающего,
покрывающие **оба** типа строк — API-key строки (соединённые через префикс
ключа) и строки с атрибуцией через JWT (соединённые через синтетический id
`jwt-<user-uuid>`) — сначала новые:

| Параметр | По умолчанию | Примечания |
| --- | --- | --- |
| `limit` | `50` | Ограничивается диапазоном `1..=200`. |
| `offset` | `0` | Смещение страницы. |
| `project` | не задан | Если задан — только записи, атрибутированные ключам с этой меткой проекта (JWT-строки исключаются). |

Ответ — `{ "records": [...], "total", "limit", "offset", "project" }`,
где каждая запись несёт `id`, `model`, `backend`, `prompt_tokens`,
`completion_tokens`, `total_tokens`, `cost` и `created_at`. Месячная агрегация
квот использует ту же форму соединения, поэтому математика квот и представление
usage всегда сходятся по охвату.

## Связанные разделы

- [Краткое руководство](quickstart.md) — получите ключ и сделайте первый
  учитываемый запрос.
- [Конфигурация](configuration.md) — переменные окружения шлюза.
- [Аутентификация и безопасность](auth-security.md) — создание API key и метка `project`.
- [Realtime & Video](realtime-video.md) — жизненный цикл видео-задач за ценами на видео.
- [Operations](operations.md) — health-пробы и наблюдаемость.
- [OpenAI-совместимый REST API](../api/openai-rest.md) — поверхность `/v1/*`.
- [JSON-RPC API](../api/jsonrpc.md) — `usage.list`, `billing.plan`, `billing.video.pricing.*`.
- [Обзор](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
