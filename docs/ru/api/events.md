---
title: "События и уведомления"
description: "SSE-sidecar server-sent events — chat.stream, models.progress, realtime.event и видео-уведомления."
---

# События и уведомления

Streaming-tokens, прогресс развёртывания и realtime-события **не** доставляются
по WebSocket-сокету JSON-RPC. Каждый streaming-RPC создаёт **id сессии**
и выталкивает уведомления на SSE-endpoint как server-sent events:

```
GET /api/rpc/events?session=<session_id>
```

## Рецепт «сначала подпишись, потом отправляй»

Уведомления, выпущенные между возвратом RPC-вызовом id сессии и установлением
SSE-подписки, **теряются** (окно до подписки). Надёжный паттерн таков:

1. Сначала откройте SSE-поток (он блокируется, пока не будет прикреплён id
   сессии).
2. Запустите RPC, возвращающий id сессии (например, `chat.send`,
   `agents.deploy`, `realtime.start`, `video.create`).
3. Читайте уведомления из SSE-потока по мере их поступления.

Каждое уведомление — сообщение в стиле JSON-RPC 2.0 с `"jsonrpc": "2.0"`,
`method` и объектом `params`.

## Каталог уведомлений

### `chat.stream`

Одно уведомление на token, производимое `chat.send` (и любым streaming-путём
чата, использующим канал сессии):

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — одна дельта контента.
- `is_complete` — `false` до финального чанка (когда upstream прикрепляет
  finish reason, финальный контент-чанк может уже нести `is_complete:
  true` с непустым token); **терминальное** уведомление всегда следует
  дальше с пустым `token` и `is_complete: true`.
- Ошибка потока доставляется как терминальное уведомление с
  `token: "Stream error: ..."` и `is_complete: true` (см.
  `packages/core/src/gateway/rpc.rs`).

### `models.progress`

Прогресс скачивания модели для `agents.deploy`, пересылаемый от агента.
`stream_id` приходит из ответа `agents.deploy`.

### `realtime.event`

Серверные события открытой полнодуплексной realtime-сессии, выталкиваемые
в канал сессии (`packages/core/src/gateway/realtime.rs`). События клиента,
отправленные через RPC `realtime.event`, пересылаются upstream; серверные
события приходят сюда.

### Уведомления видео-задач

Задачи `video.create` выталкивают прогресс в канал сессии
(`packages/core/src/gateway/video.rs`):

| Метод | Payload (params) | Значение |
| --- | --- | --- |
| `video.progress` | `job_id`, `stream_id`, `status: "running"`, `progress` (0–90) | Задача выполняется. |
| `video.done` | `job_id`, `stream_id`, `result`, `cost` | Задача завершена; `result` несёт URL артефакта. |
| `video.failed` | `job_id`, `stream_id`, `error` | Задача провалилась или была отменена. |

## Заметки о порядке

- SSE-поток упорядочен по сессии; tokens приходят в порядке генерации.
- Один id сессии может потребляться только одним SSE-подписчиком; открывайте
  поток до (или сразу после) RPC, возвращающего id.
- Заголовок `x-session-id` на `POST /api/rpc` прикрепляет сам RPC-**ответ**
  к каналу сессии тоже — используется клиентами, которые хотят, чтобы ответ
  эхом повторялся по тому же потоку.

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
