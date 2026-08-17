---
title: "Konfiguration"
description: "Referenz für jede vom Arona-Server gelesene Umgebungsvariable, mit Standardwerten und Semantik."
---

# Konfiguration

Arona wird **vollständig über Umgebungsvariablen** konfiguriert, die einmal
beim Prozessstart gelesen werden (`Config::load` in
`packages/core/src/config.rs`, einige wenige erst bei der ersten Verwendung).
Es gibt keine Konfigurationsdatei: Ändern Sie eine Variable und starten Sie den
Server neu, damit sie wirksam wird.

Diese Seite ist die Referenz für alles, was der Servercode liest, gruppiert
nach Themenbereich. Nur-Mock- und Laufzeitvariablen sind der Vollständigkeit
halber enthalten.

## Referenztabelle

| Variable | Standard | Zweck |
| --- | --- | --- |
| `DATABASE_URL` | *(erforderlich)* | PostgreSQL-Verbindungs-URL. |
| `JWT_SECRET` | *(außerhalb des Mock-Modus erforderlich)* | Secret zum Signieren von JWTs. |
| `ARONA_HOST` | `0.0.0.0` | Bind-Adresse (fällt auf `SHITTIM_CHEST_HOST` zurück). |
| `ARONA_PORT` | `8420` | Bind-Port (fällt auf `SHITTIM_CHEST_PORT` zurück). |
| `ARONA_DATA_DIR` | unset | Lokales Datenverzeichnis. |
| `ARONA_ADMIN_TOKEN` | unset | Bearer-Token für `/api/admin/*` und die Admin-RPC-Methoden. |
| `ARONA_REGISTRATION_OPEN` | `0` | Truthy (`1`/`true`/`yes`/`on`) öffnet die Registrierung. |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | In-Memory-Anfragelimit pro Key und Minute; `0` blockiert alles. |
| `MOCK_MODE` | unset | Vorhandensein (beliebiger Wert) aktiviert den Dev-Mock-Modus. |
| `MOCK_SEED_PATH` | unset | Roh-SQL-Seed-Datei, die im Mock-Modus ausgeführt wird. |
| `ARONA_MEMORY_URL` | unset | WebSocket-URL des Philia-Memory-Gateways. |
| `ARONA_MEMORY_TOKEN` | unset | Token für das Memory-Gateway. |
| `ARONA_MEMORY_WRITEBACK` | `true` | Schreibt abgeschlossene Chat-Runden in den Speicher zurück; akzeptiert `true`/`false` (jeder andere Wert fällt auf den Standard zurück). |
| `ARONA_AGENT_NAME` | `arona-agent` | Agent-Identität des GPU-Knotens. |
| `ARONA_PANEL_URL` | `localhost:8080` | Wohin sich der Agent verbindet (`ws://<panel_url>/ws/agent`). |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | Lokaler evernight-Agent für `evernight://`-Backend-URLs. |
| `ARONA_MISTRALRS` | unset | Vorhandensein erzwingt die Mistral.rs-Engine für Gguf-Modellpläne. |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | Interface, an das ein gestarteter llama.cpp-Modellserver bindet. |
| `HF_ENDPOINT` | `https://huggingface.co` | Hugging-Face-Basis-URL für Modell-Downloads. |
| `GITHUB_TOKEN` | unset | Zugriffstoken für die GitHub-Modellregistrierung. |
| `RUST_LOG` | unset | Tracing-Filter, z. B. `info` oder `arona=debug,info`. |

## Erforderliche Variablen

### `DATABASE_URL`

PostgreSQL-Verbindungs-URL. **Erforderlich**: Der Server beendet sich mit
`FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.`, wenn sie
leer ist, und der CLI-Unterbefehl `migrate` weigert sich auszuführen. Das
Schema wird von `serve` beim Start automatisch erstellt bzw. aktualisiert.

### `JWT_SECRET`

Secret zum Signieren der Access-/Refresh-JWT-Paare, die von `auth.login` und
`auth.register` ausgestellt werden. **In der Produktion erforderlich**: Der
Code enthält einen Entwicklungs-Fallback
(`dev-secret-not-for-production-use-only-32chars`), aber der Server weigert
sich, damit zu starten, außer bei `MOCK_MODE=1`:

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

Verwenden Sie einen langen, zufälligen Wert (z. B. `openssl rand -hex 32`).

## Server

### `ARONA_HOST` / `ARONA_PORT`

Bind-Adresse und -Port für das Gateway. Aus Kompatibilitätsgründen fallen sie
auf `SHITTIM_CHEST_HOST` / `SHITTIM_CHEST_PORT` zurück; die endgültigen
Standardwerte sind `0.0.0.0:8420`.

### `ARONA_DATA_DIR`

Optionales lokales Datenverzeichnis, das im App-State für Komponenten
mitgeführt wird, die einen Ablageort für temporäre Daten benötigen.
Standardmäßig nicht gesetzt.

## Sicherheit & Zugriffskontrolle

### `ARONA_ADMIN_TOKEN`

Bearer-Token, das die `/api/admin/*`-HTTP-Routen schützt (`POST/GET/DELETE
/api/admin/backends`, `/api/admin/aliases`) sowie die RPC-Methoden
`billing.plan.set` / `billing.video.pricing.set`. Wenn es **nicht gesetzt**
ist, lehnt jede dieser Routen mit `Admin access required` (401) ab — es gibt
keinen Standardwert. Setzen Sie es vor dem Serverstart auf einen starken,
zufälligen Wert.

### `ARONA_REGISTRATION_OPEN`

Öffnet die Selbstregistrierung über `auth.register`. Truthy-Werte sind genau
`1`, `true`, `yes`, `on` (Groß-/Kleinschreibung egal); alles andere —
einschließlich `0`, `false`, `off` oder einer nicht gesetzten/leeren Variable
— bleibt geschlossen. Das Flag wird einmal beim Start gelesen. Der **erste
registrierte Benutzer ist immer erlaubt** (auch wenn die Registrierung
geschlossen ist) und wird zum Admin.

### `ARONA_API_RATE_LIMIT_RPM`

In-Memory-Ratenlimit mit gleitendem Fenster pro Key (Anfragen pro Minute),
angewendet auf jede authentifizierte `/v1/*`-Anfrage (Chat, Embeddings, Video,
Models), schlüsselt nach dem API-Key-Hash (bzw. dem `u:<email>`-Label für die
JWT-akzeptierende `GET /v1/models`). Die RPC-Plane wird von diesem Limiter
nicht ratenbegrenzt — die `/v1/*`-Auth-Extraktoren sind die einzigen Aufrufer.
Standard `60`. Der Wert wird einmal in einen prozessweiten `OnceLock` geparst.
**Ein Wert von `0` blockiert jede Anfrage** — die Prüfung ist
`entry.len() >= rpm`, sodass bei `0` keine Anfrage durchkommt. Dies ist das
gatewayweite Limit; die Billing-Tiers erlegen darüber hinaus ihre eigenen
Limits pro Key auf.

## Entwicklung

### `MOCK_MODE`

Dev-Mock-Modus, aktiviert durch **Vorhandensein** — die Prüfung ist
`std::env::var("MOCK_MODE").is_ok()`, sodass *jeder* Wert (einschließlich `0`
oder eines leeren, aber gesetzten Werts) ihn aktiviert. Er:

- hebt die `JWT_SECRET`-Anforderung auf (das integrierte Dev-Secret wird
  akzeptiert);
- seedet vier Demo-Konten (`demiurge@celestia.world`, `momoi@celestia.world`,
  `midori@celestia.world`, `yuzu@celestia.world`, Passwort `33550336`);
- wartet, bis der Seed abgeschlossen ist, bevor der Listener gebunden wird.

Verwenden Sie es niemals in der Produktion.

### `MOCK_SEED_PATH`

Nur im Mock-Modus: zeigt auf eine Roh-SQL-Datei, die anstelle des integrierten
Konto-Seeds ausgeführt wird (Anweisungen an `;` aufgeteilt, `--`-Kommentare
übersprungen). Wenn die Datei nicht gelesen werden kann, wird der integrierte
Seed als Fallback verwendet.

## Memory-Gateway

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

Konfiguration für das Langzeitspeicher-Gateway (entelecheia Philia). Memory ist
**vollständig deaktiviert**, solange nicht sowohl `ARONA_MEMORY_URL` als auch
`ARONA_MEMORY_TOKEN` gesetzt und nicht leer sind. Wenn aktiviert:

- werden abgeschlossene Chat-Runden abgerufen und als Kontext injiziert, und
- steuert `ARONA_MEMORY_WRITEBACK` (Standard `true`), ob abgeschlossene Runden
  in den Memory-Dienst zurückgeschrieben werden; `0` oder `false` deaktiviert
  das Writeback.

Memory-Fehler blockieren den Chat nie; der resultierende Zustand wird im
`X-Arona-Memory`-Antwort-Header widergespiegelt
(`enabled` / `disabled` / `offline`).

## Agent-Identität & Cluster

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

Identität der GPU-Knoten-Agent-Binärdatei (`_agent`): `ARONA_AGENT_NAME`
(Standard `arona-agent`) wird dem Panel als Name/ID des Agents gemeldet, und
`ARONA_PANEL_URL` (Standard `localhost:8080`) ist die Adresse, mit der sich
der Agent verbindet (`ws://<panel_url>/ws/agent`).

Die eigene HTTP-API des Agents ist **hartkodiert** auf die Bindung an
`0.0.0.0:5790` — dafür gibt es keine Bind-Adress-Umgebungsvariable.

### `ARONA_AGENT_BIND_ADDR`

Interface, an das ein **gestarteter llama.cpp-Modellserver** bindet, wenn der
Agent ein Gguf-Modell bereitstellt, damit die Engine von anderen Rechnern
erreichbar ist (z. B. `0.0.0.0`). Standard `127.0.0.1`. Beachten Sie: Dies ist
*nicht* der Bind der Agent-HTTP-API (der fest auf `0.0.0.0:5790` steht).

## Evernight-Bridge

### `ARONA_EVERNIGHT_URL`

WebSocket-URL des lokalen evernight-Agents, der verwendet wird, um
`evernight://`-Backend-URLs in lokale TCP-Forwards aufzulösen. Standard
`ws://127.0.0.1:3001/ws`.

## Modell-Laufzeit & Downloads

### `ARONA_MISTRALRS`

Vorhandensein (beliebiger Wert) erzwingt die Mistral.rs-Engine für
Gguf-Modellpläne, die sonst standardmäßig llama.cpp verwenden würden.
Vorhandenseins-Semantik wie bei `MOCK_MODE`.

### `HF_ENDPOINT`

Basis-URL für Hugging-Face-Modell-Downloads (`hf:`-Quellen), Standard
`https://huggingface.co`. Setzen Sie sie auf einen Mirror wie
`https://hf-mirror.com`, wenn huggingface.co nicht erreichbar ist. Wird vom
Modell-Downloader gelesen; ein abschließender Schrägstrich wird entfernt.

### `GITHUB_TOKEN`

Zugriffstoken, das von der GitHub-Modellregistrierung (`gh:`-Quellen) für den
API-Zugriff verwendet wird. Standardmäßig nicht gesetzt; ohne es gelten die
GitHub-API-Ratenlimits.

## Protokollierung

### `RUST_LOG`

Standard-Tracing-Filter, der von `tracing_subscriber` beim Start angewendet
wird, z. B. `info` oder `arona=debug,info`. Folgt der üblichen `RUST_LOG`-
Semantik (`error`/`warn`/`info`/`debug`/`trace`, Überschreibungen pro Target).

## Standardwerte auf einen Blick

| Einstellung | Standard |
| --- | --- |
| Bind-Adresse / -Port | `0.0.0.0:8420` |
| API-Ratenlimit pro Key | 60 RPM |
| Agent-Name | `arona-agent` |
| Panel-URL | `localhost:8080` |
| Memory-Writeback | an |
| Registrierung | geschlossen |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
