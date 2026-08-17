---
title: "Agent-Cluster"
description: "GPU-Cluster mit mehreren Knoten — Herunterladen von Modellgewichten mit der CLI, Ausführen des _agent-Binärs auf GPU-Knoten und Steuern von Bereitstellungen über die agents.*-RPC-Oberfläche."
---

# Agent-Cluster

Aronas Bereitstellungs-Geschichte teilt sich in zwei Hälften. Das **Panel**
(das `arona`-Server-Binary) besitzt Routing, Billing, Auth und die
Verwaltungsebene. Jeder GPU-Knoten führt einen **`_agent`-Prozess** aus, der
die Modellgewichte und die lokalen Serving-Prozesse besitzt. Agenten öffnen
einen langlebigen WebSocket zurück zur Agent-Control-Plane des Panels
(`/ws/agent`); das Panel schiebt `deploy`- / `stop`-Befehle über diesen Socket
nach unten, und der Agent streamt Download-Fortschritt, Heartbeats und
Befehls-Ergebnisse nach oben. Sobald ein Modell auf einem Agenten läuft,
registriert das Panel es als routbares Backend, sodass `/v1/*`- und
RPC-Traffic es erreichen — die Control-Plane ist WebSocket, die Datenebene ist
einfaches HTTP zum lokalen Engine-Port des Agenten.

## Modellgewichte herunterladen (CLI)

Das `_cli`-Binary lädt Modellgewichte von HuggingFace, ModelScope oder
GitHub-Releases herunter:

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **Repo-Formen** — `hf:owner/repo` (Standard; ein nacktes `owner/repo` löst
  auch zu HuggingFace auf), `ms:owner/repo` (ModelScope),
  `gh:owner/repo[:tag]` (GitHub-Release, Tag optional). Die langen Präfixe
  `huggingface:`, `modelscope:` und `github:` werden ebenfalls akzeptiert;
  eine nackte ID ohne Slash löst zur Ollama-Registry auf
  (`packages/core/src/models/download.rs:21-28,55-86`).
- **`--filter <glob|prefix>`** — wiederholbar; nur Manifest-Dateien, die dem
  Glob (oder Präfix) entsprechen, werden heruntergeladen. Ohne Filter wird das
  **gesamte Repo** ausgewählt.
- **Bestätigung** — ein ungefilterter Download fragt immer `Continue? [y/N]`,
  bevor er startet, es sei denn, `--yes` wird übergeben. Ein gefilterter
  Download fragt nie; wenn die ausgewählte Gesamtmenge bei 2 GiB oder darüber
  liegt, gibt er stattdessen ein informatives Banner aus
  (`NO_CONFIRM_THRESHOLD`, `packages/cli/src/main.rs:12-15,
  439-464`).
- **`--out <dir>`** — überschreibt das Standardziel
  `~/.arona/models/<repo-id>`.
- **`--revision <rev>`** — überschreibt jedes Inline-`:rev`-Suffix
  (`hf:owner/repo:rev`).
- **Fortsetzen** — unterbrochene Downloads werden automatisch fortgesetzt:
  Eine `.part`-Datei wird behalten, und der Download fährt von ihrer aktuellen
  Länge über eine Range-Anfrage fort; vollständige Dateien werden nach Größe
  übersprungen und, wenn das Manifest einen Digest trägt, SHA-256-verifiziert
  (`packages/cli/src/main.rs` `verify_sha256`/`summarize`).
- **Wiederholungen** — Netzwerkfehler werden bis zu 3 Mal mit einer
  5-Sekunden-Verzögerung wiederholt; Nicht-Netzwerkfehler schlagen sofort
  fehl (`packages/cli/src/main.rs:277-302`).
- **`HF_ENDPOINT`** — wechselt die HuggingFace-Basis-URL, z. B. einen Mirror:

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

Die anderen CLI-Befehle (`packages/cli/src/main.rs:28-53`):

| Befehl | Zweck |
| --- | --- |
| `install` | One-Click-Umgebungseinrichtung: erkennt das Hardware-Profil und gibt Backend-/Quantisierungs-Empfehlungen aus. |
| `status` | Gibt das Hardware-Profil aus. |
| `deploy <model>` | Löst ein Modell lokal auf und meldet, ob es bereits gecacht ist. |
| `download` | Lädt Modellgewichte herunter (oben). |
| `serve` | Startet den API-Server (Panel). |
| `connect <url>` | Verbindet sich mit einem Verwaltungspanel. |
| `migrate` | Führt Datenbank-Migrationen aus. |

## Das `_agent`-Binary

`_agent` läuft auf jedem GPU-Knoten und wird rein über Umgebungsvariablen
konfiguriert (`packages/core/src/config.rs:37-40`):

| Variable | Standard | Bedeutung |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | Eindeutige Knoten-ID; das Panel verwendet sie als `agent_id`. |
| `ARONA_PANEL_URL` | `localhost:8080` | Panel-`host:port`; der Agent verbindet sich mit `ws://{ARONA_PANEL_URL}/ws/agent`. |

Siehe [Konfiguration](./configuration.md) für die vollständige
Umgebungsvariablen-Referenz (Panel-seitige Variablen, Datenbank, Secrets).

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

Verhalten:

- **Control-Verbindung** — der Agent verbindet sich zurück zu
  `ws://{ARONA_PANEL_URL}/ws/agent` (`packages/agent/src/panel.rs:23`). Bei
  der Verbindung sendet er einen `register`-Frame mit `agent_name`,
  `gpu_info` und der Liste der bereits bereitgestellten Modelle; das Panel
  zeichnet die TCP-Peer-Adresse des Agenten als dessen `host` auf.
- **Reconnect-Backoff** — startet bei 1 Sekunde und verdoppelt sich bis zu
  einem 60-Sekunden-Cap (`packages/agent/src/panel.rs:27,33-34,63`).
- **Heartbeats** — alle 30 Sekunden meldet der Agent GPU-Auslastung, Anzahl
  geladener Modelle und Uptime. Das Panel betrachtet einen Agenten als
  offline, wenn sein letzter Heartbeat älter als 30 Sekunden ist.
- **Lokale HTTP-API** — bindet die **feste** Adresse `0.0.0.0:5790`; es gibt
  keine Bind-Address-Umgebungsvariable (`packages/agent/src/main.rs:109`).
  Das Panel kombiniert diesen Port mit dem aufgezeichneten Host des Agenten,
  um die Datenebenen-URL für bereitgestellte Modelle zu bauen.
- **Befehle** — das Panel reiht `deploy`- / `stop`-Befehle über den Socket
  ein. Ein `deploy`-Befehl trägt `model_id` und eine `stream_id`;
  Download-Fortschritt wird als `deploy_progress`-Frames auf demselben Socket
  zurückgestreamt (das Panel leitet sie als `models.progress`-SSE-Notifications
  weiter, siehe [Events & Notifications](../api/events.md)), und der finale
  `deploy_result`-Frame meldet den lokalen Engine-`port` und das `backend`.
  `stop` wird mit `stop_result` beantwortet.

Führen Sie `_agent` unter einem Service-Supervisor (systemd, malkuth, ...)
aus, damit er sich automatisch neu verbindet; das Panel toleriert Neustarts
auf beiden Seiten (siehe [Knoten-Persistenz](#node-persistence) unten).

## Agent-Control-Plane-RPC

Die gesamte Agent-Oberfläche ist admin-gegated: Jede Methode erfordert ein
gültiges JWT **und** ein Admin-Konto (`validate_admin_jwt` prüft
`is_admin_email`; `packages/core/src/gateway/rpc.rs:106-118,301-337`).

| Methode | Parameter | Rückgabe |
| --- | --- | --- |
| `agents.list` | — | Cluster-Topologie: `id`, `name`, `host`, `status` (`online`/`offline`), GPU-Zusammenfassung, `models`, `last_heartbeat`, `version`, `connected_at`. |
| `agents.register` | `machine_name`, `version` | `agent_id`, `token`. |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — entfernt den Knoten. |
| `agents.status` | `agent_id` | `online`, GPU-Zusammenfassung, `gpu_utilization`, `models`, `host`, `connected_at`, `last_heartbeat`. |
| `agents.deploy` | `model_id`, `agent_id?` | `{ "ok": true, "stream_id" }` — eine leere `agent_id` zielt automatisch auf den am wenigsten ausgelasteten Knoten. |
| `agents.stop` | `agent_id`, `model_id` | `{ "ok": true }` — stoppt die Bereitstellung. |

`agents.deploy` gibt eine `stream_id` zurück; abonnieren Sie
`/api/rpc/events?session=<stream_id>` **vor** oder unmittelbar nach dem
Aufruf, um `models.progress`-Download-Notifications zu erhalten (siehe
[Events & Notifications](../api/events.md)). Siehe
[JSON-RPC-API](../api/jsonrpc.md) für die Transport- und Auth-Details.

## Automatische Registrierung bereitgestellter Modelle

Wenn ein `deploy_result`-Frame Erfolg meldet, registriert das Panel ein
`ExternalApiBackend` namens **`agent-{model_id}`** in den Gateway-Router, mit
Basis-URL `http://{agent-host}:{port}` — der aufgezeichnete Host des Agenten
plus der Engine-Port, den er gemeldet hat
(`packages/core/src/gateway/server.rs:310-366`,
`packages/core/src/gateway/mod.rs:253-270`). Das bereitgestellte Modell wird
ein normales routbares Backend: `/v1/chat/completions`, Embeddings und
RPC-Chat erreichen es alle, Aliase greifen, und der Health-Checker probt es
(siehe [Backends](./backends.md) für Backend-Typen und Probing-Semantik).

- Die erneute Bereitstellung desselben Modells (z. B. auf einem anderen
  Agenten) ersetzt das vorherige Backend.
- Ein erfolgreiches `stop_result` hebt die Registrierung wieder auf
  (`packages/core/src/gateway/mod.rs:274-287`); die Modell-ID wird nicht mehr
  aufgelöst.

## Platzierung

Bereitstellungen ohne explizite `agent_id` durchlaufen eine
Least-Loaded-Platzierung (`packages/core/src/gateway/tunnel.rs:214-229`):
Kandidaten sind Agenten, deren letzter Heartbeat unter 30 Sekunden liegt, und
der mit der **niedrigsten durchschnittlichen GPU-Auslastung** wird gewählt.
Agenten ohne Telemetrie sortieren zuletzt, bleiben aber auswählbar. Wenn kein
Agent online ist, schlägt die RPC mit `No online agents available for deployment` fehl.

Auf der Routing-Seite sind Konversationen **an ein Backend gepinnt**
(Session-Affinität): Das erste Backend, das eine Konversation bedient, wird
aufgezeichnet und für nachfolgende Turns wiederverwendet, sodass
Pro-Konversations-Zustand wie ein Runtime-KV-Cache warm bleibt
(`packages/core/src/routing/mod.rs:31-34,110-138`). Wenn das gepinnte Backend
unhealthy wird, degradiert das Routing auf eine frische Auswahl und pinnt neu.

## Knoten-Persistenz

Agent-Knoten persistieren in der `agent_nodes`-Tabelle (`agent_id`,
`machine_name`, `version`, `host`, `gpu_info`, `models`, `connected_at`,
`last_heartbeat`; `packages/core/src/gateway/tunnel.rs:8-12`). Beim
Panel-Start werden die persistierten Zeilen wiederhergestellt, sodass zuvor
registrierte Knoten über Neustarts hinweg sichtbar bleiben;
wiederhergestellte Einträge sind **sender-los**, bis sich jeder Agent über
WebSocket neu verbindet (`packages/core/src/gateway/run.rs:74-85`). Eine
Bereitstellung auf einem wiederhergestellten, aber getrennten Knoten schlägt
daher fehl, bis sich dessen `_agent` neu verbindet.

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
