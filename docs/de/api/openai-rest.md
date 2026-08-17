---
title: "OpenAI-kompatible REST-API"
description: "OpenAI-artige /v1/*-Referenz — Chat-Completions, Embeddings, Modellauflistung, asynchrone Video-Generierungen, Fehlerformen und Ratenlimits."
---

# OpenAI-kompatible REST-API

Arona stellt unter `/v1/*` eine OpenAI-kompatible REST-Oberfläche für LLM-Chat,
Embeddings, Modellauflistung, Health-Probing und asynchrone Video-Generierung
bereit. Jedes OpenAI-SDK, das auf die Basis-URL zeigt, funktioniert für Chat und
Embeddings; die Video-Endpunkte folgen der OpenAI-Konvention für Aufgaben im
Submit-/Poll-Stil.

Alle Anfrage- und Antwort-Bodies sind JSON. Fehler verwenden eine einheitliche
Form (siehe [Errors](#errors)); Authentifizierungsfehler auf der
Middleware-Ebene sind die einzige Ausnahme und werden als Klartext
zurückgegeben (siehe [Authentication](#authentication)).

## Endpunkte auf einen Blick

| Method | Path | Description |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | Chat-Runde, mit oder ohne Streaming. |
| `POST` | `/v1/embeddings` | Embedding-Vektoren für eine oder mehrere Eingaben. |
| `GET` | `/v1/models` | Router-Modelle, zusammengeführt mit den Quick-Start-Modellen. |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`. |
| `POST` | `/v1/video/generations` | Asynchrone Video-Generierungsaufgabe einreichen. |
| `GET` | `/v1/video/generations/{id}` | Status / Ergebnis einer Video-Aufgabe abfragen. |

`/api/health`, `/healthz` und `/readyz` sind zusätzliche Readiness-Probes
(Kubernetes-artige Aliasse von `/v1/health`).

## Authentifizierung

Chat-, Embedding- und Video-Endpunkte authentifizieren sich mit einem
**API key** im `Authorization: Bearer`-Header. API keys werden über die
Verwaltungsebene erstellt (`keys.create`, siehe
[JSON-RPC-API](./jsonrpc.md#keys)) und sehen wie `arona-<uuid>` aus. Sie werden
serverseitig als SHA-256-Hashes gespeichert.

```
Authorization: Bearer arona-CHANGE_ME
```

- **Fehlender Header** → `401` Klartext: `Missing Authorization header. Use: Bearer <api_key>`.
- **Ungültiger oder widerrufener API key** → `401` Klartext: `Invalid API key`.
- `GET /v1/models` akzeptiert zusätzlich ein **JWT**-Access-Token (ausgestellt
  von `auth.login` / `auth.register`), damit das Web-Dashboard Modelle mit
  demselben Token auflisten kann, das es für die RPC-Ebene verwendet. Für diesen
  Endpunkt lauten die Meldungen `Missing Authorization header. Use: Bearer <api_key_or_jwt>`
  und `Invalid API key or JWT`.

Ablehnungen auf Middleware-Ebene sind Klartext-Bodies, nicht die in
[Errors](#errors) beschriebene JSON-Fehlerform — die JSON-Form wird nur erzeugt,
sobald eine Anfrage einen Handler erreicht.

Jede authentifizierte `/v1`-Anfrage durchläuft außerdem einen
**In-Memory-Rate-Limiter pro Key** (Standard 60 RPM, 60-Sekunden-Fenster,
konfigurierbar über `ARONA_API_RATE_LIMIT_RPM`). Bei Überschreitung wird `429`
als Klartext zurückgegeben: `Rate limit exceeded. Try again later.` Quota- und
Ratenlimits auf Tier-Ebene werden separat durchgesetzt und geben JSON-429er mit
einem `Retry-After`-Header zurück (siehe [429 and Retry-After](#429-and-retry-after)).

> Die Verwaltung von API keys, Projekten und deren Scoping wird in
> [Authentifizierung & Sicherheit](../guides/auth-security.md) behandelt.

## POST /v1/chat/completions

Der zentrale OpenAI-kompatible Chat-Endpunkt mit Streaming-Unterstützung und
arona-spezifischen Erweiterungen (`conversation_id`, `memory`, `extra`,
`provider`).

### Anfrage-Body

| Feld | Typ | Erforderlich | Hinweise |
| --- | --- | --- | --- |
| `model` | string | ja | Modell-ID, wie von `GET /v1/models` aufgelistet. |
| `messages` | array | ja | Chat-Runden, siehe unten. |
| `stream` | boolean | nein | Standard `false`. Wenn `true`, ist die Antwort ein SSE-Stream (siehe [Streaming](#streaming)). |
| `temperature` | number | nein | Sampling-Temperatur, wird an den Upstream weitergeleitet. |
| `max_tokens` | integer | nein | Obergrenze für Completion-Tokens, wird an den Upstream weitergeleitet. |
| `conversation_id` | string | nein | Session-Affinität + Persistenz. Die Konversation muss existieren und dem API-key-Benutzer gehören (sonst `403` `conversation_forbidden`, `404` `conversation_not_found`, falls sie nicht existiert). Die Benutzer-Runde wird beim Senden persistiert und die Assistant-Antwort, wenn die Runde abgeschlossen ist; das Routing bindet die Konversation an das Backend, das sie zuerst bedient hat. |
| `memory` | boolean | nein | Memory-Gateway-Override. Standard `true` (Memory-Recall wird injiziert, wenn das Memory-Gateway aktiviert ist); `false` deaktiviert die Recall-Injektion für diese Anfrage. |
| `extra` | object | nein | Freiform-Durchreichung, die auf oberster Ebene in die Upstream-Payload eingefügt wird (siehe unten). |
| `tools` | array | nein | OpenAI-artige Funktionsaufruf-Definitionen, die unverändert an den Upstream durchgereicht werden. |
| `provider` | string | nein | Expliziter Backend-Auswahlhinweis, der case-insensitiv zu einem Backend-**Namen** (oder -Typ) passt. Wenn gesetzt, kommen nur Backends infrage, die zum Hinweis passen. |

**`messages`-Einträge** sind `{ "role": "user" | "assistant" | "system", "content": "..." }`.
Für multimodale / Agent-Workloads werden zwei Erweiterungen an den Upstream
weitergeleitet:

- `images` — angehängte Bilder für Vision-Anfragen (ein Array von
  `{ "media_type", "data", "position" }`-Objekten; das externe Backend rendert
  sie als OpenAI-`image_url`-Inhaltsteile).
- `tool_calls` — Funktionsaufruf-Payloads, die vom Upstream-Modell erzeugt
  werden und in Folge-Runden zurückgespiegelt werden sollen. Das externe Backend
  MUSS diese weiterleiten, sonst verlieren Agent-Pipelines (z. B.
  scepter-Skill-Ketten) jeden Tool-Aufruf.

**`extra`-Merge-Regeln**: Jeder `extra`-Schlüssel wird auf oberster Ebene in
die Upstream-Anfrage-Payload eingefügt, mit zwei harten Garantien — die
reservierten Schlüssel `model`, `messages`, `stream`, `temperature`,
`max_tokens` und `options` werden **niemals** überschrieben, ebenso wenig wie
jeder Schlüssel, den das Gateway selbst bereits gesetzt hat. Nicht-Objekt-
`extra`-Werte werden ignoriert.

**`tools`-Einträge** sind `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }` und werden unverändert weitergeleitet.

### Antwort ohne Streaming

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1720000000,
  "model": "Qwen/Qwen3-1.7B",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 12, "completion_tokens": 2, "total_tokens": 14 }
}
```

- `choices[].message` kann bei Funktionsaufruf-Runden `tool_calls` enthalten.
- Der Memory-Zustand der Anfrage wird im **`X-Arona-Memory`**-Antwort-Header
  widergespiegelt: `enabled` | `disabled` | `offline`.

### Streaming

Setzen Sie `"stream": true`. Die Antwort ist ein `text/event-stream`-SSE-Stream
— eine `data:`-Zeile pro Chunk, die jeweils ein einzelnes JSON-`ChatChunk`
enthält:

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta` enthält `content` (und `tool_calls`-Deltas mit
  `index`/`id`/`type`/`function` für Funktionsaufruf-Streams).
- Bei OpenAI-kompatiblen Upstreams trägt der **terminale Chunk** ein `usage`-
  Feld mit den realen Token-Zahlen; das Gateway zeichnet es auf (mit Rückfall
  auf eine lokale Tokenizer-Schätzung für Upstreams, die nie Usage melden,
  z. B. ollama / ws_engine).
- Der Stream endet mit `data: [DONE]`.
- Ein Stream-Fehler wird als `data:`-Event mit
  `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}`
  zugestellt; das `[DONE]`-Event folgt weiterhin, und die Usage-Aufzeichnung
  plus Assistant-Persistenz werden für den fehlgeschlagenen Stream übersprungen.
- Der `X-Arona-Memory`-Header ist auch in der SSE-Antwort vorhanden.

### Beispiel

```bash
curl http://192.0.2.10:8080/v1/chat/completions \
  -H "Authorization: Bearer arona-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-1.7B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

## POST /v1/embeddings

| Feld | Typ | Erforderlich | Hinweise |
| --- | --- | --- | --- |
| `model` | string | ja | Embedding-Modell-ID (z. B. `nomic-embed-text` — ein nackter Name matcht auch ein `:latest`-Tag). |
| `input` | string oder string[] | ja | Eine Eingabe oder mehrere. |

Antwort: `{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`.

## GET /v1/models

Listet die heute routbaren Modelle auf: die Modellauflistung jedes gesunden
registrierten Backends, zusammengeführt mit den integrierten
**Quick-Start-Modellen** (immer angekündigt, sogar bevor ein Backend
registriert ist): `Qwen/Qwen3-0.6B`, `Qwen/Qwen3-1.7B`,
`HuggingFaceTB/SmolLM2-1.7B-Instruct`, `google/gemma-3-1b-it`,
`microsoft/Phi-4-mini-instruct`, `deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`.

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

Quick-Start-Modelle erscheinen mit `owned_by` auf ihren Provider gesetzt;
Router-Modelle tragen den Namen des besitzenden Backends.

## Video-Generierung

Task-artige Video-Endpunkte für video-fähige Backends (z. B. `minimax-cloud`,
siehe [Backends](../guides/backends.md)). Jobs laufen asynchron; fragen Sie den
Status-Endpunkt ab, bis `done` erreicht ist.

### POST /v1/video/generations

| Feld | Typ | Erforderlich | Hinweise |
| --- | --- | --- | --- |
| `model` | string | ja | Video-Modell-ID, die auf einem video-fähigen Backend registriert ist. |
| `prompt` | string | ja | Generierungs-Prompt. |
| `negative_prompt` | string | nein | Negativer Prompt. |
| `images` | array | nein | Konditionierungs-/Referenzbilder als Array von `{ "data_base64": "...", "mime_type": "image/png" }`-Objekten. |
| `duration_seconds` | integer | nein | Gewünschte Dauer. |
| `width` / `height` | integer | nein | Ausgabeauflösung. |
| `provider` | string | nein | Expliziter Backend-Auswahlhinweis (Backend-Name). |
| `extra` | object | nein | Backend-spezifische Workflow-Overrides (seed, steps, cfg, ...). |

Erfolg → `200`:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

Fehler: `400` `missing_fields`, wenn `model` oder `prompt` fehlt; `503`
`video_backend_error` / `no_backend`, wenn kein gesundes video-fähiges Backend
das Modell bedient; `429` `quota_error` / `quota_exceeded`, wenn das
Monatskontingent erschöpft ist.

### GET /v1/video/generations/{id}

Gibt den Aufgabenstatus zurück:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "running",
  "progress": 40,
  "result": null,
  "error": null,
  "cost": 0.0,
  "created_at": 1720000000
}
```

- `status`: `queued` | `running` | `done` | `failed` | `cancelled`; `progress`
  schreitet während des Laufs von 0–90 fort und erreicht bei `done` 100.
- `result` (bei `done`): `{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }` — `url` zeigt auf das erzeugte Artefakt, das vom Backend
  ausgeliefert wird.
- `error` (bei `failed` / `cancelled`) und `cost` werden gesetzt, wenn
  zutreffend.
- Fehler: `400` `bad_id` für eine Nicht-UUID-ID; `404` `no_job`, wenn der Job
  nicht existiert oder zu einem anderen API key gehört.

Video-Jobs verteilen den Fortschritt außerdem über den RPC-SSE-Sidecar
(`video.progress` / `video.done` / `video.failed`, siehe
[Events & Benachrichtigungen](./events.md#video-job-notifications)).

## Fehler

Fehler auf Gateway-Ebene verwenden eine einheitliche Form
(`json_error_response`):

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| Status | `type` / `code` | Wann |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`, `missing_index`, `bad_id`, ... | Fehlerhafte oder fehlende Anfragefelder. |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id` gehört einem anderen Benutzer. |
| `404` | `invalid_request_error` / `model_not_found` | Kein Backend bedient das angefragte Modell. Meldung: `No backend available for model: <model>`. |
| `404` | `invalid_request` / `conversation_not_found` | Konversation nicht gefunden. |
| `404` | `not_found` / `no_job` | Video-Job nicht gefunden. |
| `502` | `server_error` / `bad_gateway` | Upstream non-2xx: Meldung `upstream <status>: <detail>` (Detail aus dem Upstream-Fehler-Body, begrenzt auf 4 KB). Transportfehler (connect/read/timeout) werden ebenfalls mit dem Fehlerstring auf 502 abgebildet. |
| `500` | `server_error` / `backend_error` | Andere Backend-Fehler (z. B. Backend unterstützt die Operation nicht). |
| `500` | `server_error` / `internal_error` | Jeder verbleibende interne Gateway-Fehler. |
| `429` | siehe unten | Quota- / Ratenlimit-Ablehnungen mit `Retry-After`. |

## 429 und Retry-After

429-Antworten enthalten einen `Retry-After`-Header (in Sekunden), damit
OpenAI-kompatible Clients zurückweichen:

| Auslöser | Status-Body | `Retry-After` |
| --- | --- | --- |
| Monatskontingent überschritten | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | Sekunden bis zum nächsten Monat. |
| Tier-Minuten-Ratenlimit | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`. |
| In-Memory-Rate-Limiter pro Key (Standard 60 RPM) | Klartext `Rate limit exceeded. Try again later.` | keiner (Middleware-Ablehnung). |

Tiers, Quota-Scoping und Nutzungsabrechnung werden in
[Abrechnung & Nutzung](../guides/billing-usage.md) beschrieben.

## Usage-Aufzeichnung

Jede `/v1`-Anfrage zeichnet beim Abschluss eine Usage-Zeile unter dem
API-key-Präfix (`arona-XX`) auf (Chat ohne Streaming, Streaming-Chat beim
terminalen Chunk, Embeddings sowie Video-Jobs bei Abschluss mit ihren
berechneten Kosten). Siehe [Abrechnung & Nutzung](../guides/billing-usage.md)
für das Aufzeichnungsmodell und wie Quota durchgesetzt wird.

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
