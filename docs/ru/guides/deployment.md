---
title: "Развёртывание"
description: "Установка arona-server из релизного артефакта, скрипта или исходников; запуск на bare metal с systemd, в Docker Compose или под supervision malkuth; безопасное обновление."
---

# Развёртывание

Arona поставляется как **один самодостаточный Rust-бинарник**, собранный из
крейта `_cli`. Релизный workflow публикует его как
`arona-server-<tag>-linux-x86_64`
(`.github/workflows/release.yml`), и один и тот же бинарник выполняет обе роли:

- **API-сервер** — `serve` (шлюз; `migrate` применяет миграции схемы
  явно);
- **Инструменты для моделей** — `install` (определение оборудования),
  `status`, `deploy <model>`, `download <repo>` и `connect <panel-url>`.

Серверу требуется PostgreSQL. Схема создаётся и мигрируется автоматически при
старте, поэтому развёртывание в основном сводится к «получите бинарник, укажите
ему базу данных, запустите». Начните с [краткого руководства](./quickstart.md)
для сквозного знакомства, затем вернитесь сюда за production-раскладкой.

## Требования

- **Linux x86_64** для готового релизного артефакта; любая платформа,
  поддерживаемая Rust 1.91+, может собрать из исходников.
- **PostgreSQL** — пример Docker Compose использует `postgres:16-alpine`;
  подойдёт любая свежая версия. Сервер отказывается запускаться без
  `DATABASE_URL`.
- **`curl`, `python3`, `ca-certificates`**, если вы используете скрипт
  установки (на Debian/RedHat скрипт сам устанавливает их через apt/dnf).
- Место, доступное для записи, под бинарник (например, `/usr/local/bin`)
  и, если вы запускаете скачивание моделей, дисковое пространство под кэш
  моделей в `ARONA_DATA_DIR`.

## Установка

### Из релизного артефакта

Скачайте артефакт для нужного тега и сделайте его исполняемым:

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### Из скрипта установки

Репозиторий содержит `scripts/install.sh`: он определяет последний тег
релиза из GitHub API (или учитывает переопределение `ARONA_VERSION`),
скачивает подходящий артефакт, устанавливает его как `arona-server`
в `~/.local/bin` по умолчанию (переопределите директорию через
`ARONA_BIN_DIR`) и печатает следующие шаги:

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### Из исходников

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` — это ровно то, что собирают релизный workflow
и Dockerfile.

## Конфигурация

Вся конфигурация задаётся через переменные окружения; [справочник по
конфигурации](./configuration.md) документирует каждую переменную, а
`.env.example` в корне репозитория — рабочая отправная точка. Минимальный
набор:

| Переменная | Обязательна? | Назначение |
| --- | --- | --- |
| `DATABASE_URL` | да | Строка подключения к PostgreSQL; сервер завершается, если она не задана. |
| `JWT_SECRET` | да | Секрет подписи tokens; сервер отказывается работать со встроенным development-секретом, если только не задан `MOCK_MODE=1`. |
| `ARONA_ADMIN_TOKEN` | настоятельно рекомендуется | Общий bearer-token для маршрутов `/api/admin/*`; без него эти маршруты всегда возвращают 401. |

Необязательные: `ARONA_HOST` (по умолчанию `0.0.0.0`), `ARONA_PORT`
(по умолчанию `8420`), `ARONA_REGISTRATION_OPEN` (truthy `1`/`true`/`yes`/`on` —
открывает регистрацию; первый зарегистрированный пользователь становится
администратором), `ARONA_DATA_DIR` (корень кэша моделей), `ARONA_MEMORY_URL` /
`ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK` (шлюз памяти),
`ARONA_API_RATE_LIMIT_RPM` (лимит запросов на ключ) и `RUST_LOG` (фильтр
трассировки).

## Миграции базы данных

`serve` подключается к базе данных и применяет все ожидающие миграции схемы
при старте (`init_database` → `Migrator::up`), поэтому отдельного шага
развёртывания нет. Чтобы применить миграции явно — например, проверить их
перед первым запуском — выполните:

```bash
arona-server migrate
```

Пользователю базы данных требуется привилегия **`CREATE`** на целевой схеме,
потому что стартовая миграция создаёт таблицы. Помимо схемы миграции данных
нет: база данных *и есть* состояние, поэтому делайте резервную копию перед
обновлениями (см. [Upgrade and backup](#upgrade-and-backup)).

## Bare metal с systemd

Пример unit-файла (`/etc/systemd/system/arona.service`). **Все секретные
значения ниже — заглушки: замените `CHANGE_ME` перед использованием:**

```ini
[Unit]
Description=Arona API server
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=arona
Environment=DATABASE_URL=postgres://arona:CHANGE_ME@127.0.0.1:5432/arona
Environment=JWT_SECRET=CHANGE_ME
Environment=ARONA_ADMIN_TOKEN=CHANGE_ME
Environment=ARONA_HOST=0.0.0.0
Environment=ARONA_PORT=8420
Environment=RUST_LOG=info
# Optional:
# Environment=ARONA_REGISTRATION_OPEN=1
# Environment=ARONA_MEMORY_URL=ws://127.0.0.1:8424/ws
# Environment=ARONA_MEMORY_TOKEN=CHANGE_ME
# Environment=ARONA_MEMORY_WRITEBACK=1
# Environment=ARONA_DATA_DIR=/var/lib/arona
ExecStart=/usr/local/bin/arona-server serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Затем включите и запустите его:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

В корне репозитория лежит `docker-compose.yml` с двумя сервисами:

- **`arona`** — собирает образ из `Dockerfile` (тег `arona:local`),
  публикует `${ARONA_PORT:-8420}:8420` и требует `DATABASE_URL`,
  `JWT_SECRET` и `ARONA_ADMIN_TOKEN` — Compose быстро завершается с ошибкой
  в стиле `:?`, если чего-то не хватает в `.env`. Его healthcheck выполняет
  `curl -fsS http://127.0.0.1:8420/readyz`.
- **`postgres`** — `postgres:16-alpine` только с placeholder-учётными данными
  (`POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`; пароль по умолчанию —
  `change-me` — переопределите его) и именованным томом `arona-pgdata`.

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

Встроенный сервис postgres доступен по хосту `postgres` внутри сети Compose.
`Dockerfile` собирает `_cli` и `_agent` в builder-образе
`rust:1.91-slim-bookworm`, устанавливает `ca-certificates`, `curl` и
`python3` в рантайм-образ `debian:bookworm-slim`, копирует оба бинарника,
открывает порты 8420 (сервер) и 5790 (локальный API встроенного `_agent`)
и запускает `arona-server serve` как entrypoint.

## Развёртывание под supervision с malkuth (конвенция workspace)

В этом workspace arona работает под supervision **malkuth** как
`arona-malkuth.service`. Паттерн применим к любому сервису, развёртываемому
здесь:

- malkuth наблюдает за процессом `arona-server serve` — он запускает его,
  пробирует, перезапускает при сбое и дренирует соединения перед
  выключением.
- Каждый supervised-сервис доступен только через **info-порт** сервиса
  и **proxy-порт** сервиса; сам сервис привязан к `127.0.0.1` и никогда
  не доступен напрямую из сети.
- Супервизор запускается с `--watch <deployment-path>`: когда бинарник по
  этому пути меняется — например, обновление копирует поверх новый файл —
  malkuth выполняет **rolling restart**, по одному инстансу за раз,
  сначала дренируя соединения.

Эксплуатационные следствия:

- Привязывайте `ARONA_HOST=127.0.0.1`, когда работаете за супервизорным
  proxy; proxy — единственная точка входа, обращённая к сети.
- Обновление — это «скопируйте новый бинарник на путь развёртывания»:
  watcher запускает rolling restart. Можно также перезапустить
  supervised-юнит явно.
- Направьте health check супервизора на `/readyz` (или `/api/health`);
  см. [Health probes](#health-probes).

## Health-пробы

Сервер открывает два семейства health-endpoint'ов без аутентификации (оба
также описаны в [руководстве по эксплуатации](./operations.md)):

- `GET /healthz`, `GET /readyz` (алиасы) и `GET /v1/health` возвращают
  `200` с телом `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`.
- `GET /api/health` возвращает форму plana `HealthResponse`: `status`,
  `version`, `kind`, `uptime` (секунды), `network`, `build_hash`
  и `engine_version`.

Направляйте балансировщики нагрузки, супервизоры и контейнерные healthcheck'и
на `/readyz`; используйте `/api/health`, когда нужны uptime и детали сети.

## Обновление и резервное копирование

- **Сохраните предыдущий бинарник.** Перед перезаписью `arona-server`
  скопируйте его в сторону — `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak` —
  чтобы можно было немедленно откатить бинарник, если новая версия не
  запустится.
- **Резервируйте PostgreSQL.** База данных хранит всё состояние — backends,
  узлы агентов, пользователей, ключи, диалоги и usage — а единственное
  автоматическое изменение схемы это стартовая миграция. Делайте `pg_dump`
  базы `arona` перед каждым обновлением.
- **Пользователю базы данных нужны права `CREATE`**, потому что миграции
  выполняются при старте; роль только для чтения не сможет запустить сервер.
- **Завершайте работу корректно.** Сервер дренирует активные соединения по
  SIGINT/SIGTERM, поэтому предпочитайте `systemctl restart arona` или
  перезапуск через супервизор вместо `kill -9`.

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
