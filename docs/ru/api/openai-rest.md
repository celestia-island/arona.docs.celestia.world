---
title: "OpenAI-совместимый REST API"
description: "Справочник по /v1/* в стиле OpenAI — chat completions, embeddings, список моделей, асинхронная генерация видео, формы ошибок и rate limits."
---

# OpenAI-совместимый REST API

Arona открывает OpenAI-совместимую REST-поверхность под `/v1/*` для LLM-чата,
embeddings, списка моделей, health-проб и асинхронной генерации видео. Любой
OpenAI SDK, направленный на базовый URL, работает для чата и embeddings;
видео-endpoints следуют задачной конвенции submit/poll OpenAI.

Все тела запросов и ответов — JSON. Ошибки используют единую форму (см.
[Errors](#errors)); сбои аутентификации на слое middleware — единственное
исключение и возвращаются как обычный текст (см. [Authentication](#authentication)).

## Endpoints одним взглядом

| Метод | Путь | Описание |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | Ход чата, streaming или non-streaming. |
| `POST` | `/v1/embeddings` | Векторы embeddings для одного или многих входов. |
| `GET` | `/v1/models` | Модели роутера, объединённые с быстрыми стартовыми моделями. |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`. |
| `POST` | `/v1/video/generations` | Отправить асинхронную задачу генерации видео. |
| `GET` | `/v1/video/generations/{id}` | Опросить статус / результат видео-задачи. |

`/api/health`, `/healthz` и `/readyz` — дополнительные readiness-пробы
(алиасы `/v1/health` в стиле Kubernetes).

## Authentication

Chat, embeddings и видео-endpoints аутентифицируются с помощью **API key**
в заголовке `Authorization: Bearer`. API keys создаются через плоскость
управления (`keys.create`, см. [JSON-RPC API](./jsonrpc.md#keys)) и выглядят
как `arona-<uuid>`. На сервере они хранятся как SHA-256-хеши.

```
Authorization: Bearer arona-CHANGE_ME
```

- **Отсутствующий заголовок** → `401` обычным текстом: `Missing Authorization header. Use: Bearer <api_key>`.
- **Неверный или отозванный ключ** → `401` обычным текстом: `Invalid API key`.
- `GET /v1/models` дополнительно принимает **JWT**-access-token (выданный
  `auth.login` / `auth.register`), чтобы веб-панель могла перечислять модели
  тем же token, который использует для плоскости RPC. Для этого endpoint'а
  сообщения — `Missing Authorization header. Use: Bearer <api_key_or_jwt>`
  и `Invalid API key or JWT`.

Отклонения на уровне middleware — тела обычным текстом, а не JSON-форма
ошибок, описанная в [Errors](#errors) — JSON-форма производится только когда
запрос достигает обработчика.

Каждый аутентифицированный запрос `/v1` также проходит через **per-key
лимитер в памяти** (по умолчанию 60 RPM, окно 60 секунд, настраивается через
`ARONA_API_RATE_LIMIT_RPM`). Превышение возвращает `429` обычным текстом:
`Rate limit exceeded. Try again later.` Квоты и rate limits уровня tier
применяются отдельно и возвращают JSON-429 с заголовком `Retry-After`
(см. [429 and Retry-After](#429-and-retry-after)).

> Управление API keys, проектами и их охватом описано в
> [Аутентификация и безопасность](../guides/auth-security.md).

## POST /v1/chat/completions

Основной OpenAI-совместимый chat-endpoint с поддержкой streaming
и расширениями arona (`conversation_id`, `memory`, `extra`, `provider`).

### Тело запроса

| Поле | Тип | Обязательно | Примечания |
| --- | --- | --- | --- |
| `model` | string | да | Id модели, как перечислено в `GET /v1/models`. |
| `messages` | array | да | Ходы чата, см. ниже. |
| `stream` | boolean | нет | По умолчанию `false`. При `true` ответ — это SSE-поток (см. [Streaming](#streaming)). |
| `temperature` | number | нет | Температура сэмплирования, пересылается upstream. |
| `max_tokens` | integer | нет | Ограничение tokens завершения, пересылается upstream. |
| `conversation_id` | string | нет | Привязка сессии + сохранение. Диалог должен существовать и принадлежать пользователю API key (иначе `403` `conversation_forbidden`, `404` `conversation_not_found`, если его нет). Ход пользователя сохраняется при отправке, ответ ассистента — когда ход завершается; маршрутизация закрепляет диалог за backend, обслужившим его первым. |
| `memory` | boolean | нет | Переопределение memory gateway. По умолчанию `true` (инъекция воспоминаний выполняется, когда memory gateway включён); `false` отключает инъекцию recall для этого запроса. |
| `extra` | object | нет | Свободный passthrough, объединяемый в верхний уровень payload upstream (см. ниже). |
| `tools` | array | нет | Определения функций в стиле OpenAI, передаются upstream дословно. |
| `provider` | string | нет | Явная подсказка выбора backend, сопоставляемая с **именем** backend (или видом) без учёта регистра. Когда задана, кандидатами становятся только backends, совпадающие с подсказкой. |

**Записи `messages`** — это `{ "role": "user" | "assistant" | "system", "content": "..." }`.
Для мультимодальных / агентских нагрузок upstream пересылаются два расширения:

- `images` — прикреплённые изображения для vision-запросов (массив
  объектов `{ "media_type", "data", "position" }`; external backend
  отображает их как content-части `image_url` OpenAI).
- `tool_calls` — payload вызовов функций, произведённый моделью upstream,
  который нужно эхом вернуть на последующих ходах. External backend ОБЯЗАН
  пересылать их, иначе агентские пайплайны (например, цепочки навыков scepter)
  потеряют каждый вызов инструмента.

**Правила слияния `extra`**: каждый ключ `extra` объединяется в payload
запроса upstream на верхнем уровне с двумя жёсткими гарантиями —
зарезервированные ключи `model`, `messages`, `stream`, `temperature`,
`max_tokens` и `options` **никогда** не переопределяются, как и любой ключ,
который шлюз уже установил сам. Не-объектные значения `extra` игнорируются.

**Записи `tools`** — это `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }` и пересылаются дословно.

### Non-streaming ответ

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1720000000,
  "model": "Qwen/Qwen3-1.7B",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 12, "completion_tokens": 2, "total_tokens": 14 }
}
```

- `choices[].message` может нести `tool_calls` для ходов с вызовом функций.
- Состояние памяти запроса отражается в заголовке ответа
  **`X-Arona-Memory`**: `enabled` | `disabled` | `offline`.

### Streaming

Установите `"stream": true`. Ответ — это SSE-поток `text/event-stream` — одна
строка `data:` на чанк, каждая несёт один JSON-объект `ChatChunk`:

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta` несёт `content` (и дельты `tool_calls` с
  `index`/`id`/`type`/`function` для потоков с вызовом функций).
- На OpenAI-совместимых upstream **терминальный чанк** несёт поле `usage`
  с реальными счётчиками tokens; шлюз записывает его (откатываясь к локальной
  оценке tokenizer для upstream, которые никогда не сообщают usage,
  например ollama / ws_engine).
- Поток завершается `data: [DONE]`.
- Ошибка потока доставляется как событие `data:`, несущее
  `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}`;
  событие `[DONE]` всё равно следует, а запись usage и сохранение ответа
  ассистента для проваленного потока пропускаются.
- Заголовок `X-Arona-Memory` присутствует и на SSE-ответе.

### Пример

```bash
curl http://192.0.2.10:8080/v1/chat/completions \
  -H "Authorization: Bearer arona-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-1.7B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

## POST /v1/embeddings

| Поле | Тип | Обязательно | Примечания |
| --- | --- | --- | --- |
| `model` | string | да | Id модели embeddings (например, `nomic-embed-text` — голое имя также совпадает с тегом `:latest`). |
| `input` | string или string[] | да | Один вход или много. |

Ответ: `{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`.

## GET /v1/models

Перечисляет модели, маршрутизируемые сегодня: список моделей каждого healthy
зарегистрированного backend, объединённый со встроенными **быстрыми
стартовыми моделями** (рекламируются всегда, даже до регистрации backend):
`Qwen/Qwen3-0.6B`, `Qwen/Qwen3-1.7B`,
`HuggingFaceTB/SmolLM2-1.7B-Instruct`, `google/gemma-3-1b-it`,
`microsoft/Phi-4-mini-instruct`, `deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`.

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

Быстрые стартовые модели появляются с `owned_by`, установленным в их provider;
модели роутера несут имя владеющего backend.

## Генерация видео

Задачные видео-endpoints для backends, способных к видео (например,
`minimax-cloud`, см. [Backends](../guides/backends.md)). Задачи прогрессируют
асинхронно; опрашивайте endpoint статуса до `done`.

### POST /v1/video/generations

| Поле | Тип | Обязательно | Примечания |
| --- | --- | --- | --- |
| `model` | string | да | Id видео-модели, зарегистрированный на способном к видео backend. |
| `prompt` | string | да | Промпт генерации. |
| `negative_prompt` | string | нет | Негативный промпт. |
| `images` | array | нет | Кондиционирующие/референсные изображения как массив объектов `{ "data_base64": "...", "mime_type": "image/png" }`. |
| `duration_seconds` | integer | нет | Запрошенная длительность. |
| `width` / `height` | integer | нет | Разрешение вывода. |
| `provider` | string | нет | Явная подсказка выбора backend (имя backend). |
| `extra` | object | нет | Переопределения workflow, специфичные для backend (seed, steps, cfg, ...). |

Успех → `200`:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

Ошибки: `400` `missing_fields`, когда `model` или `prompt` отсутствует; `503`
`video_backend_error` / `no_backend`, когда ни один healthy способный к видео
backend не обслуживает модель; `429` `quota_error` / `quota_exceeded`, когда
месячная квота исчерпана.

### GET /v1/video/generations/{id}

Возвращает статус задачи:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "running",
  "progress": 40,
  "result": null,
  "error": null,
  "cost": 0.0,
  "created_at": 1720000000
}
```

- `status`: `queued` | `running` | `done` | `failed` | `cancelled`; `progress`
  растёт 0–90, пока задача выполняется, и достигает 100 на `done`.
- `result` (на `done`): `{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }` — `url` указывает на сгенерированный артефакт, обслуживаемый
  backend.
- `error` (на `failed` / `cancelled`) и `cost` заполняются, когда применимо.
- Ошибки: `400` `bad_id` для id не-UUID; `404` `no_job`, когда задача не
  существует или принадлежит другому API key.

Видео-задачи также рассылают прогресс через RPC SSE-sidecar
(`video.progress` / `video.done` / `video.failed`, см.
[Events & Notifications](./events.md#video-job-notifications)).

## Errors

Ошибки уровня шлюза используют одну форму (`json_error_response`):

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| Статус | `type` / `code` | Когда |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`, `missing_index`, `bad_id`, ... | Некорректные или отсутствующие поля запроса. |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id` принадлежит другому пользователю. |
| `404` | `invalid_request_error` / `model_not_found` | Ни один backend не обслуживает запрошенную модель. Сообщение: `No backend available for model: <model>`. |
| `404` | `invalid_request` / `conversation_not_found` | Диалог не найден. |
| `404` | `not_found` / `no_job` | Видео-задача не найдена. |
| `502` | `server_error` / `bad_gateway` | Статус upstream не из диапазона 2xx: сообщение `upstream <status>: <detail>` (detail из тела ошибки upstream, ограничено 4 КБ). Транспортные сбои (connect/read/timeout) тоже отображаются в 502 со строкой ошибки. |
| `500` | `server_error` / `backend_error` | Прочие сбои backend (например, backend не поддерживает операцию). |
| `500` | `server_error` / `internal_error` | Любая оставшаяся внутренняя ошибка шлюза. |
| `429` | см. ниже | Отклонения по квоте / rate limit с `Retry-After`. |

## 429 and Retry-After

Ответы 429 включают заголовок `Retry-After` (секунды), чтобы OpenAI-совместимые
клиенты отступали:

| Триггер | Тело статуса | `Retry-After` |
| --- | --- | --- |
| Месячная квота превышена | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | Секунды до следующего месяца. |
| Минутный rate limit tier | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`. |
| Per-key лимитер в памяти (по умолчанию 60 RPM) | обычный текст `Rate limit exceeded. Try again later.` | нет (отклонение middleware). |

Tiers, охват квот и учёт usage описаны в
[Billing и usage](../guides/billing-usage.md).

## Запись usage

Каждый запрос `/v1` записывает строку usage под префиксом API key (`arona-XX`)
при завершении (non-streaming чат, streaming-чат на терминальном чанке,
embeddings и видео-задачи при завершении с их вычисленной стоимостью). Модель
записи и то, как применяются квоты, см. в
[Billing и usage](../guides/billing-usage.md).

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
