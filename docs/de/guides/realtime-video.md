---
title: "Echtzeit & Video"
description: "Vollduplex-Echtzeit-Sitzungen (realtime.start/event/stop), der engine.invoke-Wahrnehmungs-/Steuerkanal und asynchrone Videogenerierungs-Jobs."
---

# Echtzeit & Video

Arona stellt über den reinen Text-Chat hinaus zwei multimodale Kanäle bereit:
**Vollduplex-Echtzeit-Sitzungen** (Sprache/Video in beide Richtungen über einen
einzigen bidirektionalen Kanal) und **asynchrone Videogenerierung** (Task-artige
Jobs, die im Hintergrund laufen und Fortschritt melden). Beide werden an das
Backend geroutet, das das angeforderte Modell bedient, und beide werden über die
Abrechnungsebene gemessen.

## Echtzeit-Sitzungen

Eine Echtzeit-Sitzung ist ein bidirektionaler Kanal zwischen **einem Client** und
**einem Upstream**: einer Cloud-Realtime-API (OpenAI-Realtime-/Qwen-Omni-
Realtime-WebSocket-Vokabular) oder einer lokalen CEP-Engine. Client-Ereignisse
treffen über JSON-RPC ein und werden an den Upstream weitergeleitet;
Server-Ereignisse werden als `realtime.event`-Benachrichtigungen über den
SSE-Kanal der Sitzung zurückgeschoben. Audio wird als base64-PCM16 übertragen
(16 kHz Client→Modell, 24 kHz Modell→Client), passend zum Drahtformat der
Cloud provider, sodass das Gateway die Bytes unverändert durchreicht
(`packages/core/src/backends/realtime.rs:1-19`).

### `realtime.start`

Öffnet eine Sitzung gegen das Backend, das `model` bedient (JWT; Parameter
`model`, `config?`, `conversation_id?` — `packages/core/src/gateway/rpc.rs:1890-1898,
1914-1984`). Das Backend **muss** die
`realtime`-Fähigkeit deklarieren (Audio-/Video-Modalitäten); andernfalls schlägt
der Aufruf explizit mit `model {model} does not support realtime sessions (no audio/video modality)` fehl —
es gibt keinen stillen Fallback auf Text-Chat
(`packages/core/src/gateway/realtime.rs:138-142`).

Zwei Upstream-Arten werden unterstützt
(`packages/core/src/gateway/realtime.rs:143-167`):

- **CEP-Engine-Upstream** — leitet Ereignisse über den Streaming-Kanal
  `Engine.InvokeStart` des Celestia Engine Protocol, sodass eine lokal
  bereitgestellte Omni-Engine ohne neues Drahtformat an derselben
  Client-Oberfläche teilnimmt.
- **Cloud-Upstream** — eine feste `wss://`-URL, die das Cloud-Realtime-
  Ereignisvokabular spricht (`session.update`, `input_audio_buffer.*`,
  `response.audio.delta`, ...). Die Cloud-Implementierung gibt `session.update`
  bei einer Wiederverbindung erneut aus.

Die Antwort ist `{ "session_id": ..., "stream_session": ... }` — abonnieren Sie
`/api/rpc/events?session=<stream_session>` vor (oder unmittelbar nach) dem
Aufruf, um Server-Ereignisse zu empfangen. Die optionale `conversation_id`
persistiert das Sprachtranskript als Assistenten-Nachrichten und zeichnet die
Token-Nutzung für die Abrechnung auf
(`packages/core/src/gateway/realtime.rs:32-85`).

### `realtime.event`

Sendet ein Client-Ereignis in die Sitzung (JWT; Parameter `session_id`, `event` —
`packages/core/src/gateway/rpc.rs:1989-2013`). Unterstützte Ereignisse umfassen
`session.update`, `input_audio_buffer.append` / `.commit` / `.clear`,
`input_image_buffer.append`, `response.create`, `response.cancel` und
`session.stop`. `send_event` ist **nicht blockierend**: Das Ereignis wird auf
einem mpsc-Kanal in die Warteschlange gestellt und der Forwarder-Task schreibt es
an den Upstream (`packages/core/src/gateway/realtime.rs:254-280`). Nur der
Besitzer der Sitzung darf Ereignisse senden.

### `realtime.stop`

Schließt und entfernt die Sitzung (JWT; Parameter `session_id` —
`packages/core/src/gateway/rpc.rs:2016-2034`). Jede Sitzung besitzt genau einen
**Forwarder-Task**, der den Upstream hält und beide Richtungen multiplexiert:
Client-Ereignisse werden aus der Warteschlange konsumiert und Upstream-Ereignisse
werden in derselben Schleife abgefragt. Der Forwarder beendet sich, wenn der
Upstream schließt oder die Sitzung gestoppt wird, und entfernt dabei den
Registry-Eintrag (`packages/core/src/gateway/realtime.rs:195-250`).

Server-Ereignisse werden als `realtime.event`-Benachrichtigungen mit den
Parametern `{ session_id, event }` über den Sitzungskanal geschoben — siehe
[Ereignisse & Benachrichtigungen](../api/events.md).

## `engine.invoke`

`engine.invoke` ist der generische **synchrone** Engine-Methodenkanal (ADMIN: JWT
+ `is_admin`; Parameter `model`, `method`, `params?` —
`packages/core/src/gateway/rpc.rs:261-264,2049-2079`). Er ruft eine beliebige
Methode auf dem Backend auf, das `model` bedient, und gibt das Ergebnis direkt
zurück, was ihn zum Hochfrequenz-Wahrnehmungs-/Steuerkanal macht:
`sensor.ingest`-, `control.setpoint`-artige Aufrufe in 20-30-Hz-Schleifen.
Backends ohne generischen Aufrufkanal (alle OpenAI-kompatiblen HTTP-Backends)
lehnen explizit mit `backend does not support generic invocation` ab
(`packages/core/src/backends/mod.rs:573-586`).

## Videogenerierung (REST)

Video-Jobs sind OpenAI-artige asynchrone Tasks über die REST-Oberfläche
(API-Key-Authentifizierung — `packages/core/src/gateway/server.rs:876-993`; siehe
[OpenAI-kompatible REST-API](../api/openai-rest.md)):

**`POST /v1/video/generations`**

| Feld | Typ | Anmerkungen |
| --- | --- | --- |
| `model` | string | erforderlich — wählt ein videofähiges Backend aus. |
| `prompt` | string | erforderlich. |
| `negative_prompt` | string? | |
| `images` | array? | Base64-Daten-URLs (`data:image/png;base64,...`), übertragen als `{ data_base64, mime_type }`. |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | Backend-Auswahlhinweis (gegen den Backend-Namen abgeglichen). |
| `extra` | object? | Backend-spezifische Überschreibungen (seed, steps, cfg, ...). |

Antwort:

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`** fragt den Job ab und gibt `id`, `object`,
`model`, `status`, `progress`, `result`, `error`, `cost`, `created_at` zurück.
Jobs sind auf den Aufrufer beschränkt: Ein Job, der einem anderen Benutzer
gehört, gibt 404 zurück. Die REST-Oberfläche erzwingt dieselben Abrechnungs-Gates
(Monatskontingent, Rate-Limit pro Minute) wie der Chat-Pfad.

## Videogenerierung (RPC)

Dieselbe Fähigkeit ist über JSON-RPC verfügbar (JWT —
`packages/core/src/gateway/rpc.rs:371-386,1296-1431`):

| Methode | Parameter | Rückgabe |
| --- | --- | --- |
| `video.create` | dieselben Felder wie der REST-Aufruf | `{ job_id, stream_id }`. |
| `video.get` | `job_id` | Die Job-Ansicht (status, progress, result, cost, ...). |
| `video.list` | `limit?` (Standard 20, auf 1-100 begrenzt) | `{ jobs: [...] }`, neueste zuerst. |
| `video.cancel` | `job_id` | `{ "ok": true }` — nur der Besitzer darf abbrechen. |

`video.create` gibt eine `stream_id` zurück; abonnieren Sie
`/api/rpc/events?session=<stream_id>`, um Job-Benachrichtigungen zu empfangen
(`video.progress` / `video.done` / `video.failed` — siehe
[Ereignisse & Benachrichtigungen](../api/events.md)).

## Backend

Videogenerierung ist **nur Cloud**: die MiniMax-H3-Open-Platform-API,
Backend-Art `minimax-cloud` (`BackendKind::CloudVideo` —
`packages/core/src/backends/mod.rs:502-504,720-727,759-761`). Der Ablauf ist
Task-artig:

1. `POST /v1/video_generation_v2` → `task_id`
2. pollt `GET /v1/query/video_generation_v2?task_id=...`, bis `Success` /
   `Fail` / weiterhin `Pending`
3. bei Erfolg das Artefakt über
   `GET /v1/files/{file_id}/content` → `{ file: { download_url } }` auflösen

(`packages/core/src/backends/minimax_cloud.rs:66-210`). Das MiniMax-Backend
bedient keinen Chat und keine Embeddings; seine Fähigkeiten deklarieren
`supports_video_generation` und `realtime: false` (siehe
[Backends](./backends.md) für das Fähigkeitsmodell). Das Routing löst
Video-Anfragen nur gegen Backends mit `supports_video_generation` auf und
berücksichtigt den optionalen `provider`-Hinweis
(`packages/core/src/routing/mod.rs:205-263`).

Das **ComfyUI-Backend wurde entfernt** während der Konvergenz der
Modellplattform: Die Konfiguration der Backend-Art `"comfyui"` schlägt mit
`comfyui backend removed` fehl (`packages/core/src/backends/mod.rs:756-757`).
Es gibt keinen selbstgehosteten Video-Pfad; Video läuft immer über ein
`minimax-cloud`-Backend.

## Job-Lebenszyklus & Preisgestaltung

Ein Video-Job durchläuft `queued → running → done | failed | cancelled`
(`packages/core/src/gateway/video.rs`):

- **create** — die Job-Zeile wird persistiert (`queued`, Fortschritt 0) und ein
  Poller-Task wird gespawnt (`video.rs:109-176`).
- **running** — der Poller reicht den Task ein (Fortschritt 5) und pollt dann
  alle 1,5 s, wobei der Fortschritt alle paar Iterationen um 5 bis auf **90**
  erhöht wird (`video.rs:178-275`). Poll-Fehler werden protokolliert und erneut
  versucht.
- **done** — Fortschritt 100, die Ergebnis-URL und die berechneten Kosten werden
  persistiert, die Nutzung wird aufgezeichnet und eine `video.done`-
  Benachrichtigung wird verteilt (`video.rs:332-368`).
- **failed** — Einreichungs- oder Poll-Fehler → `video.failed`; nach 900
  Poll-Iterationen (~22,5 Minuten) schlägt der Job mit `generation timed out`
  fehl.
- **cancelled** — `video.cancel` setzt ein Flag, das der Poller bei seinem
  nächsten Durchlauf beobachtet; der Job wird als `cancelled` markiert und
  `video.failed` feuert mit dem Fehler `cancelled` (`video.rs:389-400`).

Die Nutzung wird mit den videospezifischen Kosten aufgezeichnet: `record_video`
schreibt einen Nutzungsdatensatz pro Anfrage mit null Token und einem expliziten
Dollarbetrag (`packages/core/src/billing/mod.rs:496-531`).

**Preisgestaltung** ist modellspezifisch, in der Tabelle `video_pricing`
(`packages/core/src/billing/video.rs`):

| Modus | Formel |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (Standard) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` bildet den Pixel-Key der kurzen Seite (z. B. `"768"`) auf
einen Multiplikator ab, mit `"*"` als Fallback. Modelle ohne konfigurierte Zeile
fallen zurück auf: Modus `per_second_resolution`, `base_price` 0.0,
`price_per_second` 0.005, `price_per_frame` 0.0, `resolution_coeff {"*": 1.0}`,
Währung USD (`billing/video.rs:20-32`). Fragen Sie Zeilen mit
`billing.video.pricing.get` (JWT) ab und aktualisieren Sie sie mit
`billing.video.pricing.set` (Admin-Token) — siehe
[JSON-RPC-API](../api/jsonrpc.md). Siehe [Abrechnung & Nutzung](./billing-usage.md)
dafür, wie Nutzungsdatensätze in die monatliche Abrechnung einfließen.

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
