---
title: "Кластер агентов"
description: "Многоузловые GPU-кластеры — скачивание весов моделей через CLI, запуск бинарника _agent на GPU-узлах и управление развёртываниями через RPC-поверхность agents.*."
---

# Кластер агентов

История развёртывания Arona делится на две половины. **Панель** (бинарник
сервера `arona`) владеет маршрутизацией, billing, auth и плоскостью
управления. На каждом GPU-узле работает один **процесс `_agent`**, который
владеет весами моделей и локальными обслуживающими процессами. Агенты открывают
долгоживущий WebSocket обратно к плоскости управления агентами панели
(`/ws/agent`); панель отправляет по этому сокету команды `deploy` / `stop`,
а агент стримит обратно прогресс скачивания, heartbeats и результаты команд.
Когда модель запущена на агенте, панель регистрирует её как маршрутизируемый
backend, чтобы трафик `/v1/*` и RPC до неё доходил — плоскость управления
это WebSocket, плоскость данных это обычный HTTP к локальному порту движка
агента.

## Скачивание весов моделей (CLI)

Бинарник `_cli` скачивает веса моделей из HuggingFace, ModelScope или
релизов GitHub:

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **Формы repo** — `hf:owner/repo` (по умолчанию; голый `owner/repo` тоже
  разрешается в HuggingFace), `ms:owner/repo` (ModelScope), `gh:owner/repo[:tag]`
  (релиз GitHub, тег опционален). Длинные префиксы `huggingface:`,
  `modelscope:` и `github:` тоже принимаются; голый id без слэша разрешается
  в реестр Ollama
  (`packages/core/src/models/download.rs:21-28,55-86`).
- **`--filter <glob|prefix>`** — повторяемый; скачиваются только файлы
  манифеста, совпадающие с glob (или префиксом). Без фильтра выбирается
  **весь repo**.
- **Подтверждение** — нефильтрованное скачивание всегда спрашивает
  `Continue? [y/N]` перед стартом, если не передан `--yes`. Фильтрованное
  скачивание никогда не запрашивает подтверждение; когда выбранный итог
  достигает 2 GiB или превышает его, вместо этого печатается информационный
  баннер (`NO_CONFIRM_THRESHOLD`, `packages/cli/src/main.rs:12-15,
  439-464`).
- **`--out <dir>`** — переопределяет пункт назначения по умолчанию
  `~/.arona/models/<repo-id>`.
- **`--revision <rev>`** — переопределяет любой встроенный суффикс `:rev`
  (`hf:owner/repo:rev`).
- **Resume** — прерванные скачивания возобновляются автоматически: файл
  `.part` сохраняется, и скачивание продолжается с его текущей длины через
  Range-запрос; завершённые файлы пропускаются по размеру и, когда манифест
  несёт digest, проверяются по SHA-256 (`verify_sha256`/`summarize`
  в `packages/cli/src/main.rs`).
- **Повторные попытки** — сетевые ошибки повторяются до 3 раз с задержкой
  5 секунд; несетевые ошибки завершаются сразу
  (`packages/cli/src/main.rs:277-302`).
- **`HF_ENDPOINT`** — переключает базовый URL HuggingFace, например на зеркало:

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

Остальные команды CLI (`packages/cli/src/main.rs:28-53`):

| Команда | Назначение |
| --- | --- |
| `install` | Настройка окружения в один клик: определяет профиль оборудования и печатает рекомендации по backend / квантованию. |
| `status` | Печатает профиль оборудования. |
| `deploy <model>` | Разрешает модель локально и сообщает, закэширована ли она уже. |
| `download` | Скачивание весов моделей (выше). |
| `serve` | Запускает API-сервер (панель). |
| `connect <url>` | Подключается к панели управления. |
| `migrate` | Запускает миграции базы данных. |

## Бинарник `_agent`

`_agent` работает на каждом GPU-узле и настраивается исключительно
переменными окружения (`packages/core/src/config.rs:37-40`):

| Переменная | По умолчанию | Значение |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | Уникальный id узла; панель использует его как `agent_id`. |
| `ARONA_PANEL_URL` | `localhost:8080` | `host:port` панели; агент подключается к `ws://{ARONA_PANEL_URL}/ws/agent`. |

Полный справочник переменных окружения (переменные панели, база данных,
секреты) см. в [Конфигурация](./configuration.md).

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

Поведение:

- **Управляющее соединение** — агент подключается обратно к
  `ws://{ARONA_PANEL_URL}/ws/agent` (`packages/agent/src/panel.rs:23`). При
  подключении он отправляет кадр `register` с `agent_name`, `gpu_info`
  и списком уже развёрнутых моделей; панель записывает TCP-адрес агента
  как его `host`.
- **Backoff переподключения** — начинается с 1 секунды и удваивается до
  потолка 60 секунд (`packages/agent/src/panel.rs:27,33-34,63`).
- **Heartbeats** — каждые 30 секунд агент сообщает загрузку GPU, число
  загруженных моделей и uptime. Панель считает агента офлайн, когда его
  последний heartbeat старше 30 секунд.
- **Локальный HTTP API** — привязан к **фиксированному** адресу
  `0.0.0.0:5790`; переменной окружения с адресом привязки нет
  (`packages/agent/src/main.rs:109`). Панель объединяет этот порт
  с записанным host агента, чтобы построить URL плоскости данных для
  развёрнутых моделей.
- **Команды** — панель ставит команды `deploy` / `stop` в очередь по сокету.
  Команда `deploy` несёт `model_id` и `stream_id`; прогресс скачивания
  стримится обратно кадрами `deploy_progress` по тому же сокету (панель
  пересылает их как SSE-уведомления `models.progress`, см.
  [Events & Notifications](../api/events.md)), а финальный кадр
  `deploy_result` сообщает локальный `port` и `backend` движка. `stop`
  подтверждается кадром `stop_result`.

Запускайте `_agent` под супервизором сервиса (systemd, malkuth, ...), чтобы
он переподключался автоматически; панель переживает перезапуски любой из
сторон (см. [node persistence](#node-persistence) ниже).

## RPC плоскости управления агентами

Вся поверхность агентов гейтится администратором: каждый метод требует
валидный JWT **и** admin-аккаунт (`validate_admin_jwt` проверяет
`is_admin_email`; `packages/core/src/gateway/rpc.rs:106-118,301-337`).

| Метод | Params | Возвращает |
| --- | --- | --- |
| `agents.list` | — | Топология кластера: `id`, `name`, `host`, `status` (`online`/`offline`), сводка GPU, `models`, `last_heartbeat`, `version`, `connected_at`. |
| `agents.register` | `machine_name`, `version` | `agent_id`, `token`. |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — удаляет узел. |
| `agents.status` | `agent_id` | `online`, сводка GPU, `gpu_utilization`, `models`, `host`, `connected_at`, `last_heartbeat`. |
| `agents.deploy` | `model_id`, `agent_id?` | `{ "ok": true, "stream_id" }` — пустой `agent_id` автоматически целится в наименее загруженный узел. |
| `agents.stop` | `agent_id`, `model_id` | `{ "ok": true }` — останавливает развёртывание. |

`agents.deploy` возвращает `stream_id`; подпишитесь на
`/api/rpc/events?session=<stream_id>` **до** вызова или сразу после него,
чтобы получать уведомления о скачивании `models.progress` (см.
[Events & Notifications](../api/events.md)). Детали транспорта и auth см.
в [JSON-RPC API](../api/jsonrpc.md).

## Авторегистрация развёрнутых моделей

Когда кадр `deploy_result` сообщает об успехе, панель регистрирует
`ExternalApiBackend` с именем **`agent-{model_id}`** в роутере шлюза,
с базовым URL `http://{agent-host}:{port}` — записанный host агента плюс
порт движка, который он сообщил (`packages/core/src/gateway/server.rs:310-366`,
`packages/core/src/gateway/mod.rs:253-270`). Развёрнутая модель становится
обычным маршрутизируемым backend: `/v1/chat/completions`, embeddings и RPC-чат
достигают её, алиасы применяются, и health-checker её пробирует (типы backends
и семантику проб см. в [Backends](./backends.md)).

- Повторное развёртывание той же модели (например, на другом агенте) заменяет
  предыдущий backend.
- Успешный `stop_result` снова её дерегистрирует
  (`packages/core/src/gateway/mod.rs:274-287`); id модели перестаёт
  разрешаться.

## Размещение

Развёртывания без явного `agent_id` проходят через размещение по наименьшей
загрузке (`packages/core/src/gateway/tunnel.rs:214-229`): кандидаты — агенты,
чей последний heartbeat моложе 30 секунд, и выбирается тот, у кого **самая
низкая средняя загрузка GPU**. Агенты без телеметрии сортируются последними,
но остаются выбираемыми. Если ни один агент не онлайн, RPC завершается
с ошибкой `No online agents available for deployment`.

На стороне маршрутизации диалоги **закрепляются за одним backend**
(привязка сессии): первый backend, обслуживший диалог, запоминается
и переиспользуется для последующих ходов, поэтому per-диалоговое состояние
вроде рантайм-кэша KV остаётся тёплым (`packages/core/src/routing/mod.rs:31-34,110-138`).
Если закреплённый backend становится unhealthy, маршрутизация деградирует
до свежего выбора и перепривязывается.

## Сохранение узлов

Узлы агентов сохраняются в таблице `agent_nodes` (`agent_id`, `machine_name`,
`version`, `host`, `gpu_info`, `models`, `connected_at`, `last_heartbeat`;
`packages/core/src/gateway/tunnel.rs:8-12`). При старте панели сохранённые
строки восстанавливаются, поэтому ранее зарегистрированные узлы остаются
видимыми между перезапусками; восстановленные записи **без sender'а**, пока
каждый агент не переподключится по WebSocket
(`packages/core/src/gateway/run.rs:74-85`). Развёртывание на восстановленный,
но отключённый узел поэтому завершается неудачей, пока его `_agent`
не переподключится.

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
