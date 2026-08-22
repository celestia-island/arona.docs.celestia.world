---
title: "Backends"
description: "Backend-Typen (external, ollama, engine, minimax-cloud, evernight-Brücken), URL-Semantik, Health-Probing, Modell-Erkennung, Aliase und Routing."
---

# Backends

Ein **backend** ist ein Upstream, der den Modell-Traffic bedient. Arona leitet
OpenAI-kompatible Anfragen (`/v1/chat/completions`, `/v1/embeddings`,
Modell-Auflistung, Video-Jobs) an eines der registrierten Backends weiter,
misst jede Anfrage und hält den Health-Status und das Modell-Inventar jedes
Backends aktuell.

Backends werden von einem Admin über
`POST /api/admin/backends` registriert (siehe die [Admin-HTTP-API](../api/admin-http.md)),
in der `backend_configs`-Tabelle persistiert und beim Start automatisch
wiederhergestellt. Jede Registrierung trägt einen `name`, einen `type`, eine
`url`, eine optionale `api_key` und eine optionale statische `models`-Liste.
Persistierte Backends überleben Neustarts; wiederhergestellte Backends starten
fail-closed und werden sofort geprobt.

## Backend-Typen

| `type` | Transport | Protokoll | Zweck |
| --- | --- | --- | --- |
| `external` | HTTP(S) | OpenAI-kompatible REST | Beliebige Chat-/Embeddings-API (Cloud oder selbst gehostet) |
| `ollama` | HTTP(S) | Native Ollama-API (`/api/chat`, `/api/embed`, `/api/tags`) | Ein lokaler oder entfernter Ollama-Server; allein aus der URL aufgebaut |
| `engine` | `ws://` / `wss://` | CEP (Celestia Engine Protocol), WebSocket + JSON-RPC | Beliebige Engine, die den CEP-Austauschstandard spricht (`plana::engine`) |
| `minimax-cloud` | HTTPS | MiniMax-H3-Task-API (submit + poll) | Cloud-Videogenerierung |
| `evernight://<node>/<service>` | Bridge-URL | Über den lokalen evernight-Agenten in einen lokalen TCP-Forward aufgelöst | Industrielle/Edge-Dienste, die nur über das evernight-Mesh erreichbar sind |
| `agent-{model}` | HTTP | OpenAI-kompatibel (external) | Automatisch registriert, wenn ein GPU-Agent ein Modell bereitstellt |

### `external` — jede beliebige OpenAI-kompatible HTTP-API

Das Allzweck-Backend: Chat-Completions (Streaming und Nicht-Streaming) und
Embeddings gegen jeden Server, der die OpenAI-REST-Form spricht. Konfigurieren
Sie es mit einer Basis-`url`, einer `api_key` (optional) und einer optionalen
statischen `models`-Liste:

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

Die statische `models`-Liste ist maßgeblich: Sie wird allen beim Probing
entdeckten Modellen vorangestellt (siehe [Modell-Erkennung](#model-discovery)).

### `ollama` — allein aus der URL aufgebaut

Ein Ollama-Backend wird allein aus der URL konstruiert — keine API key, keine
Modellliste. Es spricht die nativen Ollama-Protokolle: `/api/chat` für Chat,
`/api/embed` für Embeddings und `/api/tags` für Health-Probing und
Modell-Erkennung.

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — CEP über WebSocket

Ein `engine`-Backend verbindet sich mit einer Engine, die `ws://` (oder
`wss://`) exponiert, und spricht das **Celestia Engine Protocol** (CEP): einen
WebSocket- + JSON-RPC-2.0-Austauschstandard, der in `plana::engine` definiert
ist. Jede Engine in beliebiger Sprache, die den Ablauf Handshake → Methoden →
Streaming-Notifications implementiert, registriert sich als erstklassiges
Backend mit null Adaptercode in Arona. Wire-Methoden: `Engine.Handshake`
(erste Nachricht; Identität + Fähigkeiten), `Engine.Chat`, `Engine.ChatStart`
(Streaming; Chunks kommen als `Engine.ChatChunk`-Notifications an),
`Engine.Embeddings` und `Engine.Models`. Verbindungen werden lazy bei der
ersten Nutzung aufgebaut und bei jedem Fehler verworfen; der nächste Aufruf
verbindet sich neu und macht erneut den Handshake.

### `minimax-cloud` — Task-basierte Videogenerierung

Das Cloud-Video-Backend steuert die MiniMax-H3-Open-Platform-API: einen
Generierungs-Task einreichen, auf Abschluss pollen und dann die Artefakt-URL
aus dem Ergebnis lesen. Das ist es, was das entfernte ComfyUI-Backend ersetzt
hat (siehe unten); Video-Jobs werden über `/v1/video/generations` oder die
`video.*`-RPC-Methoden eingereicht und schreiten über
`video.progress` / `video.done` / `video.failed`-Notifications fort (siehe
[Realtime & Video](realtime-video.md)).

### `evernight://`-Bridge-URLs

Eine Backend-URL der Form `evernight://<node>/<service>` wird **nicht** direkt
kontaktiert. Der lokale evernight-Agent des Hosts löst sie (ein
`Bridge.Connect`-JSON-RPC-Aufruf über den WebSocket-Endpunkt des Agenten) in
einen lokalen TCP-Forward auf, und das Backend läuft gegen
`http://127.0.0.1:<local_port>` statt gegen eine hartkodierte Remote-Adresse.
Das ist die Single-Panel-Architektur: Das Arona-Panel erreicht Dienste auf
anderen Knoten (CEP-Engines, scepter, ...) über das Mesh, ohne jemals eine
Remote-Adresse in eine Konfiguration einzubetten.

Ein Keepalive-Task probt den Tunnel alle 15 Sekunden; wenn die entfernte Seite
neu startet und der Tunnel auf einem neuen lokalen Port wiederhergestellt wird,
wird das betroffene Backend mit der neuen URL **transparent neu aufgebaut** —
die persistierte Konfiguration behält die `evernight://`-URL, sodass Neustarts
sie erneut auflösen. Bei `engine`-Backends wird der aufgelöste
`http://127.0.0.1:<port>`-Forward für den WebSocket-Transport an `ws://`
angepasst.

### Von Agenten bereitgestellte Modelle registrieren sich automatisch

Wenn ein GPU-Agent die Bereitstellung eines Modells abschließt, registriert
das Gateway ein Backend namens `agent-{model_id}` (ein `ExternalApiBackend`
über `http://{agent host}:{port}`), sodass das Modell sofort routbar wird; das
Stoppen der Bereitstellung hebt die Registrierung wieder auf. Siehe
[Agent-Cluster](agent-cluster.md) für den vollständigen
Bereitstellungs-Lebenszyklus.

### `comfyui` wird abgelehnt

Der Backend-Typ `comfyui` wird explizit mit dem Fehler `comfyui backend removed` verweigert: Das ComfyUI-Backend wurde während der
Modellplattform-Konvergenz entfernt, und Videogenerierung läuft jetzt über
`minimax-cloud`. Die Registrierung eines `comfyui`-Backends liefert ein
HTTP 400.

## URL-Semantik

Wie eine konfigurierte Basis-URL auf tatsächliche Endpunkte abgebildet wird,
entscheidet sich daran, ob die URL eine Pfadkomponente hat:

- **Root-artige Basis** — eine URL, deren Pfad leer oder `/` ist, wird als
  Host-Root behandelt und behält die OpenAI-`/v1`-Konvention bei:
  `{base}/v1/chat/completions`, `{base}/v1/models`. Beispiele:
  `http://192.0.2.20:8429`, `https://api.deepseek.com`.
- **Pfad-artige Basis** — eine URL mit nicht-leerem Pfad wird als das
  vollständige API-Präfix behandelt, das der Server tatsächlich bedient, und
  der Endpunkt wird direkt angehängt: `{base}/chat/completions`,
  `{base}/models`. Das ist es, was OpenAI-kompatible Server außerhalb der
  `/v1`-Konvention benötigen. Der Zhipu-GLM-Coding-Plan ist das kanonische
  Beispiel: Seine API liegt unter
  `https://open.bigmodel.cn/api/coding/paas/v4` mit Chat direkt unter
  `{base}/chat/completions` und **überhaupt keinem `/models`-Endpunkt** — die
  Standard-`/api/paas/v4`-Root liefert Balance-Fehler für Coding-Plan-Keys.
- Ein **trailing slash** auf der konfigurierten Basis-URL wird normalisiert
  entfernt, sodass die Verkettung niemals einen doppelten Slash erzeugt.

## Probing & Health

Ein Hintergrund-Health-Checker probt jedes registrierte Backend alle
**60 Sekunden**; die Backend-Liste wird in jeder Runde frisch abgerufen, sodass
nach dem Start registrierte Backends ohne Neustart übernommen werden. Jede
Admin-Registrierung löst auch sofort einen Probe aus, sodass das Backend
innerhalb von ~1–2 Sekunden auf healthy wechselt, statt auf die nächste
Checker-Runde zu warten.

- **Externe Backends** proben `GET {base}/models` (oder `{base}/v1/models` bei
  Root-artigen Basen) mit einem **2-Sekunden-Timeout**. Ein **404 wird
  toleriert**: Manche Server implementieren Chat, exponieren aber keine
  Modell-Auflistung (der GLM-Coding-Plan hat keinen `/models`-Endpunkt),
  sodass ein 404 das Backend als healthy markiert und die admin-konfigurierte
  `models`-Liste zur Routing-Quelle wird. Timeouts, Netzwerkfehler und andere
  Nicht-2xx-Antworten markieren das Backend als unhealthy.
- **Ollama-Backends** proben `/api/tags` mit demselben 2-Sekunden-Timeout.
- Backends starten **fail-closed** — gemeldet als `not probed yet` — bis zum
  ersten erfolgreichen Probe, sodass ein frisch registriertes (oder
  wiederhergestelltes) Backend niemals Traffic erhält, bevor es verifiziert
  wurde.

Der Health-Zustand wird pro Backend gecacht und vom Router bei jeder Anfrage
herangezogen; unhealthy Backends werden von der Kandidatenauswahl
ausgeschlossen (siehe [Routing](#routing)).

## Modell-Erkennung

Ein Backend kündigt die Modell-IDs an, die es bedient, und der Router gleicht
Anfragen gegen diese Ankündigung ab:

- **Externe** Backends kündigen die Modelle an, die aus der Probe-Antwort
  geparst wurden (sowohl ein `data`- als auch ein `models`-Array werden
  akzeptiert), zusammengeführt mit der admin-konfigurierten statischen Liste —
  statische IDs behalten Reihenfolge und Vorrang, dynamische IDs werden
  dedupliziert und angehängt. Wenn ein Server keinen Modell-Endpunkt hat, ist
  die statische Liste allein die Routing-Quelle.
- **Ollama**-Backends kündigen die von `/api/tags` zurückgegebenen Tags an.
- **Von Agenten bereitgestellte** Modelle kündigen exakt die bereitgestellte
  `model_id` an.

Die öffentliche Oberfläche ist `GET /v1/models` (authentifiziert), die die
routbaren Modelle jedes healthy Backends auflistet (siehe die
[OpenAI-kompatible REST-API](../api/openai-rest.md)).

## Aliase & Namensnormalisierung

Aliase bilden eine angeforderte Modell-ID auf eine Ziel-ID ab. Der Alias wird
im Routing zuerst aufgelöst, sodass eine Anfrage für den Alias von dem Backend
bedient wird, das das Ziel kündigt:

```json
{ "alias": "fast-chat", "target": "deepseek-chat" }
```

Aliase werden über die Admin-Endpunkte `/api/admin/aliases` verwaltet und
wirken sofort.

Die Namenszuordnung normalisiert auch Ollama-artige Tags: Ein Backend, das
`nomic-embed-text:latest` auflistet, passt auf eine nackte Anfrage für
`nomic-embed-text`, sodass Embedding-/Chat-Anfragen ohne
`:latest`-Suffix-Buchhaltung aufgelöst werden. Ein expliziter Tag
(`qwen3:0.6b`) passt nur auf genau diesen Tag.

## Routing

Jede Anfrage wird über den Router aufgelöst, der ein Backend auswählt:

1. **Alias-Auflösung** — die angeforderte Modell-ID wird über die Alias-Tabelle
   abgebildet (falls vorhanden).
2. **Provider-Hinweis** — ein optionales `provider`-Feld filtert Kandidaten
   nach Backend-Name (oder Kind-Name, z. B. `cloud` für Video-Backends).
3. **Nur healthy Kandidaten** — ein Backend muss `Healthy` melden *und* seinen
   Circuit Breaker bestehen (3 aufeinanderfolgende Fehlschläge öffnen den
   Breaker für 30 Sekunden, mit einem half-open Testaufruf), um auswählbar zu
   sein.
4. **Least-Count-Auswahl** — Kandidaten, die das Modell bedienen, werden nach
   ihrem Pro-Backend-Anfragezähler sortiert, und das am wenigsten ausgelastete
   wird gewählt. Das verteilt die Last über healthy Backends, die dasselbe
   Modell bedienen.
5. **Session-Affinität** — wenn eine Anfrage eine `conversation_id` trägt, wird
   die Konversation an das Backend gepinnt, das sie zuerst benutzt hat. Der Pin
   lebt in einer `Weak`-Referenz-Map, sodass ein entferntes Backend ohne
   Index-Drift aus der Map verschwindet. Affinität ist best-effort: Die
   Wiederverwendung desselben Backends über die Turns einer Konversation
   erlaubt dem Upstream, Pro-Konversations-Runtime-Zustand wiederzuverwenden
   (warme Kontexte, KV-Caches). Wenn das gepinnte Backend unhealthy geworden
   ist (oder der Pin mit einem entfernten Backend gestorben ist), fällt der
   Router auf eine frische Least-Count-Auswahl zurück und bindet die
   Konversation **neu**.

Wenn kein healthy Backend das Modell bedient, schlägt das Routing fehl: Ein
unbekanntes Modell ergibt `model not found` (HTTP 404), ein bekanntes, aber
unerreichbares Modell ergibt `all
backends unhealthy`, das als 500 Internal Server Error erscheint. HTTP 502 ist für Fehler reserviert, die von einem
*erreichbaren* Upstream gemeldet werden (Nicht-2xx-Upstream-Antworten und
Transportfehler nach der Auswahl). Siehe [Operations](operations.md) für die
vollständige Fehlerzuordnung.

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
