---
title: "Memory Gateway"
description: "Долговременная память для чата — инъекция воспоминаний, запись эпизодов, управление на запрос, состояния заголовка и RPC memory.status / memory.delete."
---

# Memory Gateway

Memory Gateway даёт ходам чата доступ к **долговременной памяти**, хранящейся
в сервисе памяти entelecheia scepter / Philia. На каждом ходе чата Arona
запрашивает у сервиса воспоминания, релевантные диалогу, инъецирует их в
промпт как системную секцию и — после завершённого ответа — записывает ход
обратно как эпизод, чтобы будущие диалоги могли его вспомнить.

Это WebSocket-JSON-RPC-клиент к Philia (`Sync.ConnectHandshake`,
`Sync.MemoryQueryRequest`, `Sync.MemoryStoreRequest`,
`Sync.MemoryDeleteRequest`). Соединения устанавливаются лениво, рвутся при
любой ошибке и переустанавливаются при следующем вызове; каждый сбой деградирует
молча и **никогда не блокирует путь чата**.

## Конфигурация

Шлюзом управляют три переменные окружения:

| Переменная | Значение |
| --- | --- |
| `ARONA_MEMORY_URL` | WebSocket-URL сервиса scepter / Philia, например `ws://192.0.2.10:8424/ws`. |
| `ARONA_MEMORY_TOKEN` | Токен для сервиса памяти. |
| `ARONA_MEMORY_WRITEBACK` | Записываются ли завершённые ходы обратно. По умолчанию **включено**; установите `false`, чтобы отключить (парсится как строгий boolean — `0` не принимается). |

Обе переменные `ARONA_MEMORY_URL` **и** `ARONA_MEMORY_TOKEN` должны быть
заданы и непусты, иначе шлюз **отключён**: recall и writeback полностью
пропускаются, и каждый запрос сообщает `disabled`. Токен отправляется
и как query-параметр `?token=` при апгрейде WebSocket, и внутри запроса
`Sync.ConnectHandshake`.

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

Полный справочник окружения см. в [Конфигурация](configuration.md).

## Инъекция воспоминаний

Когда шлюз включён, **каждый ход чата** — REST non-streaming
`/v1/chat/completions`, REST streaming (SSE) и RPC `chat.send` — запрашивает
сервис памяти до того, как запрос будет переслан:

- Запрос — это **последнее пользовательское сообщение** собранного контекста.
- Запрашивается до **5** воспоминаний (`limit = 5`).
- Результаты оформляются как markdown-секция system с заголовком
  `## Relevant Long-Term Memories`, по одному пункту `- [score] text`
  на воспоминание (счёт с двумя десятичными знаками, пустые записи
  пропускаются), и добавляются в начало списка сообщений как сообщение
  `system`. Инъекция идемпотентна: контекст, который уже несёт эту секцию,
  не инъецируется повторно.
- Если релевантных воспоминаний нет, ничего не инъецируется, и ход
  продолжается без изменений.

Recall выполняется до сохранения диалога и пересылки upstream; медленный или
падающий сервис памяти не даёт **никакой гарантии задержки** сверх
собственного 10-секундного RPC-таймаута и не может провалить запрос.

## Writeback

После завершённого ответа ассистента ход записывается обратно в сервис памяти
как узел-**эпизод**. Текст эпизода — эвристическая расшифровка хода —
`User: <user content>\nAssistant: <assistant content>` (любая из сторон
опускается, когда пуста; обе пустые — writeback пропускается). Writeback
**fire-and-forget**: он выполняется в порождённой задаче, никогда не блокирует
ответ чата, а его сбои только логируются внутри клиента памяти. (На пути
REST streaming writeback дополнительно требует, чтобы к запросу был
прикреплён диалог; пути REST non-streaming и RPC записывают обратно в любом
случае.)

## Управление на запрос

И тело REST-запроса чата, и параметры RPC `chat.send` принимают опциональное
поле `memory` для переопределения серверной конфигурации **на каждый вызов**:

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` — принудительно включает/выключает recall
  для этого хода.
- опущено (`null`) — следовать серверной конфигурации (`req.memory.unwrap_or(true)`),
  т. е. включено тогда и только тогда, когда шлюз настроен.

Переопределение влияет на recall; writeback следует только
`ARONA_MEMORY_WRITEBACK` плюс включённости шлюза.

## Состояния заголовка

REST-ответы несут состояние памяти хода в заголовке ответа
**`X-Arona-Memory`**; ответ RPC `chat.send` отражает то же значение в поле
`memory` своего результата. Возможные состояния:

| Значение | Значение |
| --- | --- |
| `enabled` | Память запрошена, шлюз настроен, recall прошёл успешно и инъецировано хотя бы одно воспоминание. |
| `disabled` | Шлюз не настроен, или `memory: false` в запросе, или нет пользовательского сообщения для запроса, или recall прошёл успешно, но вернул **ни одного** релевантного воспоминания (нечего инъецировать). |
| `offline` | Память запрошена и шлюз настроен, но вызов recall завершился сбоем (отказ соединения, RPC-ошибка или таймаут). |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## Семантика сбоев

Всё деградирует явно, в одном направлении — чат всегда работает:

- **Сбой recall** — логируется на уровне `warn`; запрос продолжается без
  инъецированных воспоминаний и сообщает `offline` в заголовке.
- **Сбой writeback** — логируется внутри клиента памяти; ответ чата
  не затрагивается.
- **Сервис памяти не настроен** — recall и writeback являются no-op;
  каждый запрос сообщает `disabled`.

Не существует режима, в котором отключение памяти валит или задерживает
запрос чата сверх собственных ограниченных таймаутов клиента.

## RPC-поверхность

На поверхности JSON-RPC открыты два метода управления (оба требуют JWT;
см. [JSON-RPC API](../api/jsonrpc.md)):

**`memory.status`** — снимок состояния шлюза:

```json
{
  "enabled": true,
  "writeback": true,
  "events": [
    { "kind": "recall", "detail": "…query…", "at": "2026-01-01T00:00:00.000Z" },
    { "kind": "writeback", "detail": "…node id…", "at": "…" },
    { "kind": "error", "detail": "…", "at": "…" }
  ]
}
```

`events` — это кольцевой буфер в памяти недавней активности — события recall,
writeback, delete и error, сначала новые, до запрошенного количества
(обработчик status запрашивает последние 50; сам буфер ограничен 100). Это
**не** долговечный аудит-лог — он сбрасывается при перезапуске.

**`memory.delete`** — удалить сохранённый узел по id:

```json
{ "node_id": "…" }
```

Возвращает `{ "deleted": true | false }`. Завершается ошибкой, когда
`node_id` отсутствует или сервис памяти не настроен.

## Связанные разделы

- [Конфигурация](configuration.md) — переменные `ARONA_MEMORY_*`.
- [Краткое руководство](quickstart.md) — сквозная настройка шлюза.
- [Backends](backends.md) — как чат-запросы маршрутизируются до запуска recall.
- [Billing и usage](billing-usage.md) — как учитываются те же ходы чата.
- [Operations](operations.md) — логи и health для соединения с памятью.
- [JSON-RPC API](../api/jsonrpc.md) — `memory.status`, `memory.delete`, `chat.send`.
- [Обзор](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
