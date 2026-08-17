---
title: "Deployment"
description: "Installieren Sie arona-server aus einem Release-Asset, per Skript oder aus dem Quellcode; betreiben Sie ihn auf Bare Metal mit systemd, in Docker Compose oder unter malkuth-Supervision; und aktualisieren Sie ihn sicher."
---

# Deployment

Arona wird als **einzelne eigenständige Rust-Binärdatei** ausgeliefert, die aus
der `_cli`-Crate gebaut wird. Der Release-Workflow veröffentlicht sie als
`arona-server-<tag>-linux-x86_64`
(`.github/workflows/release.yml`), und dieselbe Binärdatei übernimmt beide
Rollen:

- **API-Server** — `serve` (das Gateway; `migrate` wendet Schema-Migrationen
  explizit an);
- **Modell-Tooling** — `install` (Hardware-Erkennung), `status`, `deploy
  <model>`, `download <repo>` und `connect <panel-url>`.

Der Server benötigt PostgreSQL. Das Schema wird beim Start automatisch
erstellt und migriert, sodass die Bereitstellung im Wesentlichen darin
besteht, „die Binärdatei zu holen, sie auf eine Datenbank zu richten und
auszuführen“. Beginnen Sie mit dem [Schnellstart](./quickstart.md) für einen
End-to-End-Durchgang und kehren Sie dann für die Produktionsanordnung
hierher zurück.

## Voraussetzungen

- **Linux x86_64** für das vorgebaute Release-Asset; jede von Rust 1.91+
  unterstützte Plattform kann aus dem Quellcode bauen.
- **PostgreSQL** — das Docker-Compose-Beispiel verwendet `postgres:16-alpine`;
  jede aktuelle Version funktioniert. Ohne `DATABASE_URL` weigert sich der
  Server zu starten.
- **`curl`, `python3`, `ca-certificates`**, wenn Sie das Installationsskript
  verwenden (auf Debian/RedHat installiert das Skript sie selbst über
  apt/dnf).
- Ein beschreibbarer Ort für die Binärdatei (z. B. `/usr/local/bin`) und, wenn
  Sie Modell-Downloads ausführen, Speicherplatz für den Modell-Cache unter
  `ARONA_DATA_DIR`.

## Installation

### Aus einem Release-Asset

Laden Sie das Asset für den gewünschten Tag herunter und machen Sie es
ausführbar:

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### Aus dem Installationsskript

Das Repository enthält `scripts/install.sh`: Es ermittelt den neuesten
Release-Tag über die GitHub-API (oder respektiert den `ARONA_VERSION`-Override),
lädt das passende Asset herunter, installiert es standardmäßig als
`arona-server` in `~/.local/bin` (Verzeichnis mit `ARONA_BIN_DIR`
überschreibbar) und gibt die nächsten Schritte aus:

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### Aus dem Quellcode

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` ist genau das, was der Release-Workflow und
das Dockerfile bauen.

## Konfiguration

Die gesamte Konfiguration erfolgt über Umgebungsvariablen; die
[Konfigurationsreferenz](./configuration.md) dokumentiert jede Variable, und
`.env.example` im Repository-Wurzelverzeichnis ist ein funktionierender
Ausgangspunkt. Das Mindestmaß:

| Variable | Erforderlich? | Zweck |
| --- | --- | --- |
| `DATABASE_URL` | ja | PostgreSQL-Verbindungszeichenkette; der Server beendet sich, wenn sie nicht gesetzt ist. |
| `JWT_SECRET` | ja | Token-Signatur-Secret; der Server weigert sich, mit dem integrierten Entwicklungs-Secret zu laufen, außer bei `MOCK_MODE=1`. |
| `ARONA_ADMIN_TOKEN` | dringend empfohlen | Gemeinsames Bearer-Token für `/api/admin/*`-Routen; ohne es liefern diese Routen immer 401 zurück. |

Optional: `ARONA_HOST` (Standard `0.0.0.0`), `ARONA_PORT` (Standard `8420`),
`ARONA_REGISTRATION_OPEN` (truthy `1`/`true`/`yes`/`on` — öffnet die
Registrierung; der erste registrierte Benutzer wird Admin), `ARONA_DATA_DIR`
(Modell-Cache-Wurzel), `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` /
`ARONA_MEMORY_WRITEBACK` (Memory-Gateway), `ARONA_API_RATE_LIMIT_RPM`
(Ratenlimit pro Key) und `RUST_LOG` (Tracing-Filter).

## Datenbank-Migrationen

`serve` verbindet sich mit der Datenbank und wendet beim Start alle
ausstehenden Schema-Migrationen an (`init_database` → `Migrator::up`), sodass
es keinen separaten Deploy-Schritt gibt. Um Migrationen explizit anzuwenden —
beispielsweise um sie vor dem ersten Start zu verifizieren — führen Sie aus:

```bash
arona-server migrate
```

Der Datenbank-Benutzer benötigt das **`CREATE`-Privileg** auf dem Zielschema,
weil die Startmigration Tabellen erstellt. Es gibt keine Datenmigration über
das Schema hinaus: Die Datenbank *ist* der Zustand, sichern Sie sie daher vor
Upgrades (siehe [Upgrade und Backup](#upgrade-and-backup)).

## Bare Metal mit systemd

Beispiel-Unit-Datei (`/etc/systemd/system/arona.service`). **Alle
Secret-Werte unten sind Platzhalter — ersetzen Sie `CHANGE_ME` vor der
Verwendung:**

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

Aktivieren und starten Sie die Unit dann:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

Das Repository-Wurzelverzeichnis enthält eine `docker-compose.yml` mit zwei
Diensten:

- **`arona`** — baut das Image aus dem `Dockerfile` (Tag `arona:local`),
  veröffentlicht `${ARONA_PORT:-8420}:8420` und benötigt `DATABASE_URL`,
  `JWT_SECRET` und `ARONA_ADMIN_TOKEN` — Compose schlägt mit einem Fehler im
  `:?`-Stil fehl, wenn eine davon in `.env` fehlt. Sein Healthcheck führt
  `curl -fsS http://127.0.0.1:8420/readyz` aus.
- **`postgres`** — `postgres:16-alpine` mit ausschließlich
  Platzhalter-Anmeldedaten (`POSTGRES_USER` / `POSTGRES_PASSWORD` /
  `POSTGRES_DB`; das Standard-Passwort ist `change-me` — überschreiben Sie es)
  und einem benannten Volume `arona-pgdata`.

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

Der gebündelte postgres-Dienst ist im Compose-Netzwerk unter dem Host
`postgres` erreichbar. Das `Dockerfile` baut `_cli` und `_agent` in einem
`rust:1.91-slim-bookworm`-Builder, installiert `ca-certificates`, `curl` und
`python3` in einer `debian:bookworm-slim`-Runtime, kopiert beide Binärdateien
hinein, exponiert die Ports 8420 (Server) und 5790 (die lokale API des
gebündelten `_agent`) und führt `arona-server serve` als Entrypoint aus.

## Überwachte Bereitstellung mit malkuth (Workspace-Konvention)

In diesem Workspace läuft arona unter der **malkuth**-Supervision als
`arona-malkuth.service`. Das Muster gilt für jeden hier bereitgestellten
Dienst:

- malkuth überwacht den `arona-server serve`-Prozess — es startet ihn, probt
  ihn, startet ihn bei Fehlern neu und leert Verbindungen (drains) vor dem
  Herunterfahren.
- Jeder überwachte Dienst wird nur über einen **Info-Port** pro Dienst und
  einen **Proxy-Port** pro Dienst exponiert; der Dienst selbst bindet an
  `127.0.0.1` und ist nie direkt aus dem Netz erreichbar.
- Der Supervisor wird mit `--watch <deployment-path>` gestartet: Wenn sich die
  Binärdatei an diesem Pfad ändert — z. B. ein Upgrade eine neue Datei
  darüberkopiert — führt malkuth einen **Rolling Restart** durch, eine Instanz
  nach der anderen, wobei zuerst die Verbindungen geleert werden.

Betriebliche Konsequenzen:

- Binden Sie `ARONA_HOST=127.0.0.1`, wenn Sie hinter dem Supervisor-Proxy
  laufen; der Proxy ist der einzige netzseitige Einstiegspunkt.
- Ein Upgrade bedeutet „die neue Binärdatei auf den Bereitstellungspfad
  kopieren“: Der Watcher löst den Rolling Restart aus. Sie können die
  überwachte Unit auch explizit neu starten.
- Richten Sie den Health Check des Supervisors auf `/readyz` (oder
  `/api/health`) aus; siehe [Health-Probes](#health-probes).

## Health-Probes

Der Server stellt zwei nicht authentifizierte Health-Familien bereit (beide
werden auch im [Operations-Leitfaden](./operations.md) behandelt):

- `GET /healthz`, `GET /readyz` (Aliasse) und `GET /v1/health` liefern
  `200` mit `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`.
- `GET /api/health` liefert die plana-`HealthResponse`-Form: `status`,
  `version`, `kind`, `uptime` (Sekunden), `network`, `build_hash` und
  `engine_version`.

Richten Sie Load Balancer, Supervisoren und Container-Healthchecks auf
`/readyz` aus; verwenden Sie `/api/health`, wenn Sie Uptime- und
Netzwerkdetails benötigen.

## Upgrade und Backup

- **Heben Sie die vorherige Binärdatei auf.** Kopieren Sie `arona-server` vor
  dem Überschreiben zur Seite —
  `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak` —
  damit Sie die Binärdatei sofort zurückrollen können, falls die neue Version
  nicht startet.
- **Sichern Sie PostgreSQL.** Die Datenbank enthält den gesamten Zustand —
  Backends, Agent-Knoten, Benutzer, Keys, Konversationen und Nutzung — und die
  einzige automatische Schemaänderung ist die Startmigration. Führen Sie vor
  jedem Upgrade einen `pg_dump` der `arona`-Datenbank durch.
- **Der Datenbank-Benutzer benötigt `CREATE`-Rechte**, weil Migrationen beim
  Start laufen; eine Nur-Lese-Rolle kann den Server nicht starten.
- **Fahren Sie ordentlich herunter.** Der Server leert laufende Verbindungen
  bei SIGINT/SIGTERM, bevorzugen Sie daher `systemctl restart arona` oder
  einen Supervisor-Neustart gegenüber `kill -9`.

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
