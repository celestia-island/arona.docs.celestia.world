---
title: "Despliegue"
description: "Instale arona-server desde un asset de release, un script o el código fuente; ejecútelo en bare metal con systemd, en Docker Compose o bajo supervisión de malkuth; y actualícelo de forma segura."
---

# Despliegue

Arona se distribuye como un **único binario Rust autocontenido** construido a
partir del crate `_cli`. El workflow de release lo publica como
`arona-server-<tag>-linux-x86_64` (`.github/workflows/release.yml`), y el mismo
binario desempeña ambos roles:

- **API server** — `serve` (el gateway; `migrate` aplica las migraciones de
  esquema explícitamente);
- **Model tooling** — `install` (detección de hardware), `status`, `deploy
  <model>`, `download <repo>` y `connect <panel-url>`.

El servidor requiere PostgreSQL. El esquema se crea y migra automáticamente al
arrancar, así que el despliegue consiste sobre todo en "obtenga el binario,
apúntelo a una base de datos y ejecútelo". Empiece con el
[inicio rápido](./quickstart.md) para un recorrido de extremo a extremo y
vuelva aquí para la disposición de producción.

## Requisitos

- **Linux x86_64** para el asset de release precompilado; cualquier plataforma
  soportada por Rust 1.91+ puede compilar desde el código fuente.
- **PostgreSQL** — el ejemplo de Docker Compose usa `postgres:16-alpine`;
  cualquier versión reciente funciona. El servidor se niega a arrancar sin
  `DATABASE_URL`.
- **`curl`, `python3`, `ca-certificates`** si usa el script de instalación
  (en Debian/RedHat el script los instala él mismo vía apt/dnf).
- Una ubicación escribible para el binario (p. ej. `/usr/local/bin`) y, si
  ejecuta descargas de modelos, espacio en disco para la caché de modelos bajo
  `ARONA_DATA_DIR`.

## Instalación

### Desde un asset de release

Descargue el asset de la tag que quiera y hágalo ejecutable:

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### Desde el script de instalación

El repositorio incluye `scripts/install.sh`: resuelve la tag de release más
reciente desde la API de GitHub (o respeta la sobrescritura `ARONA_VERSION`),
descarga el asset correspondiente, lo instala como `arona-server` en
`~/.local/bin` por defecto (sobrescriba el directorio con `ARONA_BIN_DIR`) e
imprime los siguientes pasos:

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### Desde el código fuente

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` es exactamente lo que compilan el workflow de
release y el Dockerfile.

## Configuración

Toda la configuración se hace mediante variables de entorno; la
[referencia de configuración](./configuration.md) documenta todas las
variables, y `.env.example` en la raíz del repositorio es un punto de partida
funcional. El conjunto mínimo:

| Variable | ¿Obligatoria? | Propósito |
| --- | --- | --- |
| `DATABASE_URL` | sí | Cadena de conexión a PostgreSQL; el servidor sale si no está definida. |
| `JWT_SECRET` | sí | Secreto de firma de tokens; el servidor se niega a ejecutarse con el secreto de desarrollo integrado salvo que `MOCK_MODE=1`. |
| `ARONA_ADMIN_TOKEN` | muy recomendada | Bearer token compartido para las rutas `/api/admin/*`; sin él, esas rutas siempre devuelven 401. |

Opcionales: `ARONA_HOST` (predeterminado `0.0.0.0`), `ARONA_PORT`
(predeterminado `8420`), `ARONA_REGISTRATION_OPEN` (truthy `1`/`true`/`yes`/`on`
— abre el registro; el primer usuario registrado se convierte en admin),
`ARONA_DATA_DIR` (raíz de la caché de modelos), `ARONA_MEMORY_URL` /
`ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK` (gateway de memoria),
`ARONA_API_RATE_LIMIT_RPM` (límite de tasa por clave) y `RUST_LOG` (filtro de
tracing).

## Migraciones de base de datos

`serve` se conecta a la base de datos y aplica todas las migraciones de esquema
pendientes al arrancar (`init_database` → `Migrator::up`), así que no hay un
paso de despliegue separado. Para aplicar las migraciones explícitamente — por
ejemplo, para verificarlas antes del primer arranque — ejecute:

```bash
arona-server migrate
```

El usuario de la base de datos necesita el privilegio **`CREATE`** en el
esquema de destino, porque la migración de arranque crea tablas. No hay
migración de datos más allá del esquema: la base de datos *es* el estado, así
que haga una copia de seguridad antes de las actualizaciones (consulte
[Actualización y copia de seguridad](#upgrade-and-backup)).

## Bare metal con systemd

Ejemplo de archivo de unidad (`/etc/systemd/system/arona.service`). **Todos los
valores secretos siguientes son placeholders — sustituya `CHANGE_ME` antes de
usarlos:**

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

Después actívelo e inícielo:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

La raíz del repositorio incluye un `docker-compose.yml` con dos servicios:

- **`arona`** — construye la imagen desde el `Dockerfile` (tag `arona:local`),
  publica `${ARONA_PORT:-8420}:8420` y requiere `DATABASE_URL`, `JWT_SECRET` y
  `ARONA_ADMIN_TOKEN` — Compose falla rápido con un error estilo `:?` si falta
  alguno en `.env`. Su healthcheck ejecuta `curl -fsS http://127.0.0.1:8420/readyz`.
- **`postgres`** — `postgres:16-alpine` con credenciales solo placeholder
  (`POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`; la contraseña
  predeterminada es `change-me` — sobrescríbala) y un volumen con nombre
  `arona-pgdata`.

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

El servicio postgres incluido es alcanzable en el host `postgres` dentro de la
red de Compose. El `Dockerfile` compila `_cli` y `_agent` en un builder
`rust:1.91-slim-bookworm`, instala `ca-certificates`, `curl` y `python3` en un
runtime `debian:bookworm-slim`, copia ambos binarios, expone los puertos 8420
(server) y 5790 (la API local del `_agent` incluido) y ejecuta
`arona-server serve` como entrypoint.

## Despliegue supervisado con malkuth (convención del workspace)

En este workspace arona se ejecuta bajo supervisión de **malkuth** como
`arona-malkuth.service`. El patrón se aplica a cualquier servicio desplegado
aquí:

- malkuth supervisa el proceso `arona-server serve` — lo genera, le hace
  probes, lo reinicia ante fallos y drena las conexiones antes del apagado.
- Cada servicio supervisado solo se expone mediante un **puerto de info** por
  servicio y un **puerto de proxy** por servicio; el propio servicio se vincula
  a `127.0.0.1` y nunca es alcanzable directamente desde la red.
- El supervisor se inicia con `--watch <deployment-path>`: cuando el binario de
  esa ruta cambia — p. ej. una actualización copia un archivo nuevo encima —
  malkuth realiza un **reinicio gradual (rolling restart)**, una instancia a la
  vez, drenando primero las conexiones.

Consecuencias operativas:

- Vincule `ARONA_HOST=127.0.0.1` cuando se ejecute detrás del proxy del
  supervisor; el proxy es el único punto de entrada expuesto a la red.
- Actualizar es "copiar el nuevo binario sobre la ruta de despliegue": el
  watcher dispara el reinicio gradual. También puede reiniciar la unidad
  supervisada explícitamente.
- Apunte el health check del supervisor a `/readyz` (o `/api/health`); consulte
  [Probes de salud](#health-probes).

## Probes de salud

El servidor expone dos familias de endpoints de salud sin autenticación (ambas
también se tratan en la [guía de operaciones](./operations.md)):

- `GET /healthz`, `GET /readyz` (alias) y `GET /v1/health` devuelven `200` con
  `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`.
- `GET /api/health` devuelve la forma `HealthResponse` de plana: `status`,
  `version`, `kind`, `uptime` (segundos), `network`, `build_hash` y
  `engine_version`.

Apunte los load balancers, supervisores y healthchecks de contenedores a
`/readyz`; use `/api/health` cuando necesite el detalle de uptime y red.

## Actualización y copia de seguridad

- **Conserve el binario anterior.** Antes de sobrescribir `arona-server`,
  cópielo aparte — `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak` —
  para poder revertir el binario de inmediato si la nueva versión no arranca.
- **Haga una copia de seguridad de PostgreSQL.** La base de datos guarda todo
  el estado — backends, nodos agente, usuarios, claves, conversaciones y uso —
  y el único cambio automático de esquema es la migración de arranque. Ejecute
  `pg_dump` de la base de datos `arona` antes de cada actualización.
- **El usuario de la base de datos necesita derechos `CREATE`**, porque las
  migraciones se ejecutan al arrancar; un rol de solo lectura no puede arrancar
  el servidor.
- **Apague correctamente.** El servidor drena las conexiones en curso ante
  SIGINT/SIGTERM, así que prefiera `systemctl restart arona` o un reinicio del
  supervisor a `kill -9`.

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
