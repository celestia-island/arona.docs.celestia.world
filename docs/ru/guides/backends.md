---
title: "Backends"
description: "Типы backends (external, ollama, engine, minimax-cloud, evernight bridges), семантика URL, health probing, обнаружение моделей, алиасы и маршрутизация."
---

# Backends

**Backend** — это upstream, который обслуживает трафик моделей. Arona
маршрутизирует OpenAI-совместимые запросы (`/v1/chat/completions`,
`/v1/embeddings`, список моделей, видео-задачи) на один из зарегистрированных
backends, учитывает каждый запрос и поддерживает health и инвентарь моделей
каждого backend в актуальном состоянии.

Backends регистрируются администратором через
`POST /api/admin/backends` (см. [Admin HTTP API](../api/admin-http.md)),
сохраняются в таблице `backend_configs` и автоматически восстанавливаются при
старте. Каждая регистрация несёт `name`, `type`, `url`, опциональные
`api_key` и статический список `models`. Сохранённые backends переживают
перезапуски; восстановленные backends стартуют в fail-closed состоянии
и немедленно пробируются.

## Типы backends

| `type` | Транспорт | Протокол | Назначение |
| --- | --- | --- | --- |
| `external` | HTTP(S) | OpenAI-совместимый REST | Любой API чата/embeddings (облачный или self-hosted) |
| `ollama` | HTTP(S) | Нативный API Ollama (`/api/chat`, `/api/embed`, `/api/tags`) | Локальный или удалённый сервер Ollama; строится только из URL |
| `engine` | `ws://` / `wss://` | CEP (Celestia Engine Protocol), WebSocket + JSON-RPC | Любой движок, говорящий на стандарте обмена CEP (`plana::engine`) |
| `minimax-cloud` | HTTPS | Задачный API MiniMax H3 (submit + poll) | Облачная генерация видео |
| `evernight://<node>/<service>` | bridge-URL | Разрешается через локальный агент evernight в локальный TCP-форвард | Промышленные/edge-сервисы, доступные только через mesh evernight |
| `agent-{model}` | HTTP | OpenAI-совместимый (external) | Авторегистрируется, когда GPU-агент развёртывает модель |

### `external` — любой OpenAI-совместимый HTTP API

Универсальный backend: chat completions (streaming и non-streaming)
и embeddings против любого сервера, говорящего на REST-форме OpenAI.
Настройте его через базовый `url`, опциональные `api_key` и статический
список `models`:

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

Статический список `models` авторитетен: он объединяется перед любыми
моделями, обнаруженными при пробе (см. [Model discovery](#model-discovery)).

### `ollama` — строится только из URL

Backend Ollama конструируется только из URL — без API key, без списка
моделей. Он говорит на нативных протоколах Ollama: `/api/chat` для чата,
`/api/embed` для embeddings и `/api/tags` для health-проб и обнаружения
моделей.

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — CEP поверх WebSocket

Backend типа `engine` подключается к движку, открывающему `ws://` (или
`wss://`), и говорит на **Celestia Engine Protocol** (CEP): стандарте обмена
WebSocket + JSON-RPC 2.0, определённом в `plana::engine`. Любой движок,
написанный на любом языке и реализующий поток handshake → методы →
streaming-уведомления, регистрируется как первоклассный backend вообще без
адаптерного кода в Arona. Проводные методы: `Engine.Handshake` (первое
сообщение; идентичность + возможности), `Engine.Chat`, `Engine.ChatStart`
(streaming; чанки приходят как уведомления `Engine.ChatChunk`),
`Engine.Embeddings` и `Engine.Models`. Соединения устанавливаются лениво при
первом использовании и рвутся при любой ошибке; следующий вызов переподключается
и выполняет handshake заново.

### `minimax-cloud` — задачная генерация видео

Облачный видео-backend управляет API открытой платформы MiniMax H3: отправьте
задачу генерации, опрашивайте завершение, затем прочитайте URL артефакта из
результата. Именно он заменил удалённый backend ComfyUI (см. ниже); видео-задачи
отправляются через `/v1/video/generations` или RPC-методы `video.*`
и проходят через уведомления `video.progress` / `video.done` / `video.failed`
(см. [Realtime & Video](realtime-video.md)).

### Bridge-URL `evernight://`

URL backend вида `evernight://<node>/<service>` **не** контактируется
напрямую. Локальный агент evernight хоста разрешает его (вызов JSON-RPC
`Bridge.Connect` через WebSocket-endpoint агента) в локальный TCP-форвард,
и backend работает против `http://127.0.0.1:<local_port>` вместо
жёстко зашитого удалённого адреса. Это архитектура единой панели: панель Arona
достигает сервисов на других узлах (CEP-движки, scepter, ...) через mesh, нигде
не встраивая удалённый адрес в конфигурацию.

Keepalive-задача пробирует туннель каждые 15 секунд; когда удалённая сторона
перезапускается и туннель переустанавливается на новом локальном порту,
пострадавший backend **прозрачно пересобирается** с новым URL — сохранённая
конфигурация хранит URL `evernight://`, так что перезапуски разрешают его
заново. Для backends типа `engine` разрешённый форвард
`http://127.0.0.1:<port>` адаптируется в `ws://` для WebSocket-транспорта.

### Авторегистрация развёрнутых агентом моделей

Когда GPU-агент завершает развёртывание модели, шлюз регистрирует backend
с именем `agent-{model_id}` (объект `ExternalApiBackend` поверх
`http://{agent host}:{port}`), так что модель становится маршрутизируемой
немедленно; остановка развёртывания снова её дерегистрирует. Полный жизненный
цикл развёртывания см. в [Agent Cluster](agent-cluster.md).

### `comfyui` отклоняется

Тип backend `comfyui` явно отвергается с ошибкой
`comfyui backend removed`: backend ComfyUI был удалён при конвергенции
модельной платформы, и генерация видео теперь работает через
`minimax-cloud`. Регистрация backend `comfyui` возвращает HTTP 400.

## Семантика URL

То, как настроенный базовый URL отображается на фактические endpoints,
определяется тем, есть ли у URL компонент пути:

- **Root-style base** — URL, путь которого пуст или равен `/`, трактуется как
  корень хоста и сохраняет конвенцию OpenAI `/v1`: `{base}/v1/chat/completions`,
  `{base}/v1/models`. Примеры: `http://192.0.2.20:8429`,
  `https://api.deepseek.com`.
- **Path-style base** — URL с непустым путём трактуется как полный префикс
  API, который сервер фактически обслуживает, и endpoint добавляется
  напрямую: `{base}/chat/completions`, `{base}/models`. Это нужно
  OpenAI-совместимым серверам вне конвенции `/v1`. Канонический пример —
  coding-план Zhipu GLM: его API живёт на
  `https://open.bigmodel.cn/api/coding/paas/v4` с чатом напрямую на
  `{base}/chat/completions` и **вообще без endpoint `/models`** — стандартный
  корень `/api/paas/v4` возвращает ошибки баланса для ключей coding-плана.
- **Завершающий слэш** в настроенном базовом URL нормализуется (убирается),
  чтобы при склейке никогда не получалось двойного слэша.

## Probing и health

Фоновый health-checker пробирует каждый зарегистрированный backend каждые
**60 секунд**; список backends заново забирается на каждом раунде, поэтому
backends, зарегистрированные после старта, подхватываются без перезапуска.
Каждая admin-регистрация также запускает немедленную пробу, так что backend
становится healthy в течение ~1–2 секунд, не дожидаясь следующего раунда
checker'а.

- **External backends** пробируются через `GET {base}/models` (или
  `{base}/v1/models` для root-style base) с **таймаутом 2 секунды**.
  **404 допускается**: некоторые серверы реализуют чат, но не открывают
  список моделей (у coding-плана GLM нет endpoint `/models`), поэтому 404
  помечает backend как healthy, а источником маршрутизации становится
  заданный администратором список `models`. Таймауты, сетевые ошибки и другие
  ответы не из диапазона 2xx помечают backend как unhealthy.
- **Ollama backends** пробируются через `/api/tags` с тем же таймаутом
  2 секунды.
- Backends стартуют в **fail-closed** состоянии — отображаются как
  `not probed yet` — до первой успешной пробы, поэтому свежезарегистрированный
  (или восстановленный) backend никогда не получает трафик до проверки.

Состояние health кэшируется для каждого backend и учитывается роутером при
каждом запросе; unhealthy backends исключаются из выбора кандидатов (см.
[Routing](#routing)).

## Обнаружение моделей

Backend рекламирует id моделей, которые он обслуживает, и роутер сопоставляет
запросы с этой рекламой:

- **External** backends рекламируют модели, разобранные из ответа пробы
  (принимаются и массив `data`, и массив `models`), объединённые со
  статическим списком, заданным администратором — статические id сохраняют
  порядок и приоритет, динамические id дедуплицируются и добавляются в конец.
  Когда у сервера нет endpoint моделей, единственным источником
  маршрутизации становится статический список.
- **Ollama** backends рекламируют теги, возвращаемые `/api/tags`.
- **Развёрнутые агентом** модели рекламируют ровно развёрнутый `model_id`.

Публичная поверхность — это `GET /v1/models` (с аутентификацией), который
перечисляет маршрутизируемые модели каждого healthy backend (см.
[OpenAI-совместимый REST API](../api/openai-rest.md)).

## Алиасы и нормализация имён

Алиасы сопоставляют запрошенный id модели с целевым id. Алиас разрешается
первым при маршрутизации, поэтому запрос на алиас обслуживается тем backend,
который рекламирует цель:

```json
{ "alias": "fast-chat", "target": "deepseek-v4-flash" }
```

Алиасами управляют через admin-endpoints `/api/admin/aliases`; изменения
вступают в силу немедленно.

Сопоставление имён также нормализует теги в стиле Ollama: backend, перечисляющий
`nomic-embed-text:latest`, совпадает с «голым» запросом `nomic-embed-text`,
поэтому запросы embeddings/чата разрешаются без учёта суффикса `:latest`.
Явный тег (`qwen3:0.6b`) совпадает только с этим точным тегом.

## Маршрутизация

Каждый запрос разрешается через роутер, который выбирает один backend:

1. **Разрешение алиаса** — запрошенный id модели отображается через таблицу
   алиасов (если есть).
2. **Подсказка provider** — опциональное поле `provider` фильтрует кандидатов
   по имени backend (или имени вида, например `cloud` для видео-backends).
3. **Только healthy-кандидаты** — backend должен сообщать `Healthy` *и*
   пройти свой circuit breaker (3 последовательных сбоя открывают breaker
   на 30 секунд, с одним half-open тестовым вызовом), чтобы быть
   выбираемым.
4. **Выбор по наименьшей загрузке** — кандидаты, обслуживающие модель,
   сортируются по своему счётчику запросов на backend, и выбирается
   наименее загруженный. Это распределяет нагрузку между healthy backends,
   обслуживающими одну модель.
5. **Привязка сессии** — когда запрос несёт `conversation_id`, диалог
   закрепляется за backend, который обслужил его первым. Привязка живёт
   в map со ссылками `Weak`, поэтому удалённый backend исчезает из map без
   дрейфа индексов. Аффинитет best-effort: повторное использование одного
   backend на ходах диалога позволяет upstream переиспользовать
   per-диалоговое рантайм-состояние (тёплые контексты, KV-кэши). Если
   закреплённый backend стал unhealthy (или привязка умерла вместе с удалённым
   backend), роутер откатывается к свежему выбору по наименьшей загрузке
   и **перепривязывает** диалог.

Если ни один healthy backend не обслуживает модель, маршрутизация терпит
неудачу: неизвестная модель даёт `model not found` (HTTP 404), известная,
но недостижимая модель даёт `all backends unhealthy`, что выходит наружу как
ошибка 500 internal server error. HTTP 502 зарезервирован для сбоев,
сообщённых *достижимым* upstream (ответы upstream не из диапазона 2xx
и транспортные сбои после выбора). Полное отображение ошибок см. в
[Operations](operations.md).

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
