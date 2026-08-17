---
title: "Events & Benachrichtigungen"
description: "Server-sent-Event-(SSE)-Sidecar — chat.stream-, models.progress-, realtime.event- und Video-Benachrichtigungen."
---

# Events & Benachrichtigungen

Streaming-Tokens, Deploy-Fortschritt und Realtime-Events werden **nicht** über
den JSON-RPC-WebSocket-Socket zugestellt. Jeder Streaming-RPC erzeugt eine
**Sitzungs-ID** und schiebt Benachrichtigungen als Server-sent Events an den
SSE-Endpunkt:

```
GET /api/rpc/events?session=<session_id>
```

## Erst abonnieren, dann senden

Benachrichtigungen, die zwischen der Rückkehr des RPC-Aufrufs mit einer
Sitzungs-ID und dem Aufbau des SSE-Abonnements emittiert werden, werden
**verworfen** (das Prä-Abonnement-Fenster). Das zuverlässige Muster ist:

1. Öffnen Sie zuerst den SSE-Stream (er blockiert, bis eine Sitzungs-ID
   angehängt wird).
2. Lösen Sie den RPC aus, der die Sitzungs-ID zurückgibt (z. B. `chat.send`,
   `agents.deploy`, `realtime.start`, `video.create`).
3. Lesen Sie Benachrichtigungen vom SSE-Stream, sobald sie eintreffen.

Jede Benachrichtigung ist eine Nachricht im JSON-RPC-2.0-Stil mit
`"jsonrpc": "2.0"`, einem `method`- und einem `params`-Objekt.

## Benachrichtigungskatalog

### `chat.stream`

Eine Benachrichtigung pro Token, erzeugt von `chat.send` (und jedem
Streaming-Chat-Pfad, der einen Sitzungskanal verwendet):

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — ein Inhalts-Delta.
- `is_complete` — `false` bis zum finalen Chunk (wenn der Upstream einen
  Finish-Grund anhängt, kann der finale Inhalts-Chunk bereits `is_complete:
  true` mit einem nicht-leeren Token tragen); die **terminale** Benachrichtigung
  folgt immer mit leerem `token` und `is_complete: true`.
- Ein Stream-Fehler wird als terminale Benachrichtigung mit
  `token: "Stream error: ..."` und `is_complete: true` zugestellt (siehe
  `packages/core/src/gateway/rpc.rs`).

### `models.progress`

Download-Fortschritt von Modellen für `agents.deploy`, vom Agent
weitergeleitet. Die `stream_id` stammt aus der `agents.deploy`-Antwort.

### `realtime.event`

Server-Events für eine offene Vollduplex-Realtime-Sitzung, auf den
Sitzungskanal geschoben (`packages/core/src/gateway/realtime.rs`).
Client-Events, die über den `realtime.event`-RPC gesendet werden, werden an den
Upstream weitergeleitet; Server-Events treffen hier ein.

### Video-Job-Benachrichtigungen

`video.create`-Jobs schieben den Fortschritt über den Sitzungskanal
(`packages/core/src/gateway/video.rs`):

| Methode | Payload (params) | Bedeutung |
| --- | --- | --- |
| `video.progress` | `job_id`, `stream_id`, `status: "running"`, `progress` (0–90) | Job läuft. |
| `video.done` | `job_id`, `stream_id`, `result`, `cost` | Job abgeschlossen; `result` trägt die Artefakt-URL. |
| `video.failed` | `job_id`, `stream_id`, `error` | Job fehlgeschlagen oder abgebrochen. |

## Hinweise zur Reihenfolge

- Der SSE-Stream ist pro Sitzung geordnet; Tokens treffen in
  Generierungsreihenfolge ein.
- Eine einzelne Sitzungs-ID darf nur von einem SSE-Abonnenten konsumiert
  werden; öffnen Sie den Stream vor (oder unmittelbar nach) dem RPC, der die ID
  zurückgibt.
- Der `x-session-id`-Header bei `POST /api/rpc` hängt auch die RPC-**Antwort**
  selbst an einen Sitzungskanal — verwendet von Clients, die die Antwort über
  denselben Stream zurückgespiegelt haben möchten.

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
