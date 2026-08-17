---
title: "Realtime и видео"
description: "Полнодуплексные realtime-сессии (realtime.start/event/stop), канал восприятия/управления engine.invoke и асинхронные задачи генерации видео."
---

# Realtime и видео

Помимо обычного текстового чата Arona открывает два мультимодальных канала:
**полнодуплексные realtime-сессии** (речь/видео в обе стороны по одному
двунаправленному каналу) и **асинхронную генерацию видео** (задачные задачи,
которые выполняются в фоне и сообщают прогресс). Оба маршрутизируются на
backend, обслуживающий запрошенную модель, и оба учитываются через слой
billing.

## Realtime-сессии

Realtime-сессия — это двунаправленный канал между **одним клиентом** и **одним
upstream**: облачным realtime API (словарь WebSocket OpenAI-Realtime /
Qwen-Omni-Realtime) или локальным CEP-движком. События клиента приходят
через JSON-RPC и пересылаются upstream; события сервера выталкиваются обратно
как уведомления `realtime.event` по SSE-каналу сессии. Аудио передаётся как
base64 PCM16 (16 кГц клиент→модель, 24 кГц модель→клиент), совпадая
с проводным форматом облачных вендоров, чтобы шлюз пропускал байты
нетронутыми (`packages/core/src/backends/realtime.rs:1-19`).

### `realtime.start`

Открывает сессию против backend, обслуживающего `model` (JWT; параметры
`model`, `config?`, `conversation_id?` —
`packages/core/src/gateway/rpc.rs:1890-1898, 1914-1984`). Backend **обязан**
объявить capability `realtime` (модальности audio/video); иначе вызов явно
завершается с `model {model} does not support realtime sessions (no audio/video
modality)` — молчаливого отката к текстовому чату нет
(`packages/core/src/gateway/realtime.rs:138-142`).

Поддерживаются два вида upstream
(`packages/core/src/gateway/realtime.rs:143-167`):

- **Upstream CEP-движок** — маршрутизирует события по streaming-каналу
  `Engine.InvokeStart` Celestia Engine Protocol, так что локально развёрнутый
  omni-движок присоединяется к той же клиентской поверхности без нового
  проводного формата.
- **Облачный upstream** — фиксированный `wss://`-URL, говорящий на облачном
  realtime-словаре событий (`session.update`, `input_audio_buffer.*`,
  `response.audio.delta`, ...). Облачная реализация повторно отправляет
  `session.update` при переподключении.

Ответ — `{ "session_id": ..., "stream_session": ... }` — подпишитесь на
`/api/rpc/events?session=<stream_session>` до (или сразу после) вызова,
чтобы получать события сервера. Опциональный `conversation_id` сохраняет
речевую расшифровку как сообщения ассистента и записывает usage tokens для
billing (`packages/core/src/gateway/realtime.rs:32-85`).

### `realtime.event`

Отправляет одно событие клиента в сессию (JWT; параметры `session_id`,
`event` — `packages/core/src/gateway/rpc.rs:1989-2013`). Поддерживаемые
события включают `session.update`, `input_audio_buffer.append` / `.commit` /
`.clear`, `input_image_buffer.append`, `response.create`, `response.cancel`
и `session.stop`. `send_event` **неблокирующий**: событие ставится в очередь
на mpsc-канале, и задача-форвардер пишет его upstream
(`packages/core/src/gateway/realtime.rs:254-280`). Отправлять события может
только владелец сессии.

### `realtime.stop`

Закрывает и удаляет сессию (JWT; параметр `session_id` —
`packages/core/src/gateway/rpc.rs:2016-2034`). Каждая сессия владеет ровно
одной **задачей-форвардером**, которая держит upstream и мультиплексирует
оба направления: события клиента потребляются из очереди, а события upstream
опрашиваются в том же цикле. Форвардер завершается, когда upstream
закрывается или сессия останавливается, удаляя запись из реестра
(`packages/core/src/gateway/realtime.rs:195-250`).

События сервера выталкиваются как уведомления `realtime.event` с параметрами
`{ session_id, event }` по каналу сессии — см.
[Events & Notifications](../api/events.md).

## `engine.invoke`

`engine.invoke` — это общий **синхронный** канал методов движка
(ADMIN: JWT + `is_admin`; параметры `model`, `method`, `params?` —
`packages/core/src/gateway/rpc.rs:261-264,2049-2079`). Он вызывает
произвольный метод на backend, обслуживающем `model`, и возвращает результат
напрямую, что делает его высокочастотным каналом восприятия/управления:
вызовы в стиле `sensor.ingest`, `control.setpoint` в циклах 20–30 Гц.
Backends без общего канала вызова (все OpenAI-совместимые HTTP-backends)
явно отклоняют вызов с `backend does not support generic invocation`
(`packages/core/src/backends/mod.rs:573-586`).

## Генерация видео (REST)

Видео-задачи — это асинхронные задачи в стиле OpenAI поверх REST-поверхности
(auth по API key — `packages/core/src/gateway/server.rs:876-993`; см.
[OpenAI-совместимый REST API](../api/openai-rest.md)):

**`POST /v1/video/generations`**

| Поле | Тип | Примечания |
| --- | --- | --- |
| `model` | string | обязательно — выбирает backend, способный к видео. |
| `prompt` | string | обязательно. |
| `negative_prompt` | string? | |
| `images` | array? | Base64 data-URL (`data:image/png;base64,...`), передаются как `{ data_base64, mime_type }`. |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | Подсказка выбора backend (сопоставляется с именем backend). |
| `extra` | object? | Переопределения, специфичные для backend (seed, steps, cfg, ...). |

Ответ:

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`** опрашивает задачу и возвращает `id`,
`object`, `model`, `status`, `progress`, `result`, `error`, `cost`,
`created_at`. Задачи ограничены вызывающим: задача, принадлежащая другому
пользователю, возвращает 404. REST-поверхность применяет те же billing-гейты
(месячная квота, минутный rate limit), что и путь чата.

## Генерация видео (RPC)

Та же возможность доступна через JSON-RPC (JWT —
`packages/core/src/gateway/rpc.rs:371-386,1296-1431`):

| Метод | Params | Возвращает |
| --- | --- | --- |
| `video.create` | те же поля, что у REST-вызова | `{ job_id, stream_id }`. |
| `video.get` | `job_id` | Представление задачи (status, progress, result, cost, ...). |
| `video.list` | `limit?` (по умолчанию 20, ограничено 1–100) | `{ jobs: [...] }`, сначала новые. |
| `video.cancel` | `job_id` | `{ "ok": true }` — отменить может только владелец. |

`video.create` возвращает `stream_id`; подпишитесь на
`/api/rpc/events?session=<stream_id>`, чтобы получать уведомления о задаче
(`video.progress` / `video.done` / `video.failed` — см.
[Events & Notifications](../api/events.md)).

## Backend

Генерация видео — **только облачная**: API открытой платформы MiniMax H3,
вид backend `minimax-cloud` (`BackendKind::CloudVideo` —
`packages/core/src/backends/mod.rs:502-504,720-727,759-761`). Поток задачный:

1. `POST /v1/video_generation_v2` → `task_id`
2. опрос `GET /v1/query/video_generation_v2?task_id=...` до `Success` /
   `Fail` или пока `Pending`
3. при успехе разрешите артефакт через
   `GET /v1/files/{file_id}/content` → `{ file: { download_url } }`

(`packages/core/src/backends/minimax_cloud.rs:66-210`). Backend MiniMax
не обслуживает чат/embeddings; его capabilities объявляют
`supports_video_generation` и `realtime: false` (модель capabilities см.
в [Backends](./backends.md)). Маршрутизация разрешает видео-запросы только
против backends с `supports_video_generation`, учитывая опциональную
подсказку `provider` (`packages/core/src/routing/mod.rs:205-263`).

**Backend ComfyUI был удалён** при конвергенции модельной платформы:
настройка вида backend `"comfyui"` завершается ошибкой `comfyui backend
removed` (`packages/core/src/backends/mod.rs:756-757`). Self-hosted пути
для видео нет; видео всегда идёт через backend `minimax-cloud`.

## Жизненный цикл задач и цены

Видео-задача проходит через `queued → running → done | failed | cancelled`
(`packages/core/src/gateway/video.rs`):

- **create** — строка задачи сохраняется (`queued`, progress 0),
  и порождается задача-опрашиватель (`video.rs:109-176`).
- **running** — опрашиватель отправляет задачу (progress 5), затем опрашивает
  каждые 1,5 с, поднимая progress на 5 каждые несколько итераций до **90**
  (`video.rs:178-275`). Ошибки опроса логируются и повторяются.
- **done** — progress 100, URL результата и вычисленная стоимость
  сохраняются, usage записывается, и рассылается уведомление `video.done`
  (`video.rs:332-368`).
- **failed** — сбой отправки или опроса → `video.failed`; после 900 итераций
  опроса (~22,5 минуты) задача завершается с `generation timed out`.
- **cancelled** — `video.cancel` ставит флаг, который опрашиватель замечает
  на следующем проходе; задача помечается `cancelled`, и срабатывает
  `video.failed` с ошибкой `cancelled` (`video.rs:389-400`).

Usage записывается со специфичной для видео стоимостью: `record_video` пишет
запись usage на запрос с нулевыми tokens и явной стоимостью в долларах
(`packages/core/src/billing/mod.rs:496-531`).

**Цены** специфичны для модели, в таблице `video_pricing`
(`packages/core/src/billing/video.rs`):

| Режим | Формула |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (по умолчанию) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` отображает ключ короткой стороны в пикселях (например,
`"768"`) в множитель, с `"*"` как запасным значением. Модели без настроенной
строки откатываются к: режим `per_second_resolution`, `base_price` 0.0,
`price_per_second` 0.005, `price_per_frame` 0.0, `resolution_coeff {"*": 1.0}`,
валюта USD (`billing/video.rs:20-32`). Запрашивайте строки через
`billing.video.pricing.get` (JWT) и обновляйте через
`billing.video.pricing.set` (admin token) — см.
[JSON-RPC API](../api/jsonrpc.md). О том, как записи usage агрегируются
в месячный billing, см. [Billing и usage](./billing-usage.md).

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
