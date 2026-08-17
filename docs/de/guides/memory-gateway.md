---
title: "Memory Gateway"
description: "Langzeitgedächtnis für den Chat — Recall-Injektion, Episode-Writeback, Steuerung pro Anfrage, Header-Zustände und die RPCs memory.status / memory.delete."
---

# Memory Gateway

Das Memory-Gateway gibt Chat-Turns Zugriff auf das **Langzeitgedächtnis**, das im
entelecheia-scepter-/Philia-Speicherdienst abgelegt ist. Bei jedem Chat-Turn fragt
Arona den Dienst nach für die Unterhaltung relevanten Erinnerungen ab, injiziert
sie als System-Abschnitt in den Prompt und schreibt — nach einer abgeschlossenen
Antwort — den Turn als Episode zurück, damit zukünftige Unterhaltungen ihn
abrufen können.

Es ist ein WebSocket-JSON-RPC-Client für Philia (`Sync.ConnectHandshake`,
`Sync.MemoryQueryRequest`, `Sync.MemoryStoreRequest`,
`Sync.MemoryDeleteRequest`). Verbindungen werden lazy aufgebaut, bei jedem Fehler
verworfen und beim nächsten Aufruf neu hergestellt; jede Störung degradiert still
und **blockiert niemals den Chat-Pfad**.

## Konfiguration

Das Gateway wird über drei Umgebungsvariablen gesteuert:

| Variable | Bedeutung |
| --- | --- |
| `ARONA_MEMORY_URL` | WebSocket-URL des scepter-/Philia-Dienstes, z. B. `ws://192.0.2.10:8424/ws`. |
| `ARONA_MEMORY_TOKEN` | Token für den Speicherdienst. |
| `ARONA_MEMORY_WRITEBACK` | Legt fest, ob abgeschlossene Turns zurückgeschrieben werden. Standard **an**; setzen Sie `false`, um es zu deaktivieren (wird als strikter Boolean geparst — `0` wird nicht akzeptiert). |

Sowohl `ARONA_MEMORY_URL` **als auch** `ARONA_MEMORY_TOKEN` müssen gesetzt und
nicht leer sein, andernfalls ist das Gateway **deaktiviert**: Recall und Writeback
werden vollständig übersprungen und jede Anfrage meldet `disabled`. Das Token wird
sowohl als `?token=`-Query-Parameter beim WebSocket-Upgrade als auch innerhalb der
`Sync.ConnectHandshake`-Anfrage gesendet.

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

Siehe [Konfiguration](configuration.md) für die vollständige Umgebungsreferenz.

## Recall-Injektion

Mit aktiviertem Gateway fragt **jeder Chat-Turn** — REST ohne Streaming
`/v1/chat/completions`, REST-Streaming (SSE) und RPC `chat.send` — den
Speicherdienst ab, bevor die Anfrage weitergeleitet wird:

- Die Abfrage ist die **letzte Benutzernachricht** des zusammengesetzten Kontexts.
- Bis zu **5** Erinnerungen werden angefordert (`limit = 5`).
- Die Ergebnisse werden als Markdown-System-Abschnitt mit dem Titel
  `## Relevant Long-Term Memories` gerendert, ein `- [score] text`-Bullet pro
  Erinnerung (Scores mit zwei Dezimalstellen, leere Einträge übersprungen), und
  der Nachrichtenliste als `system`-Nachricht vorangestellt. Die Injektion ist
  idempotent: Ein Kontext, der den Abschnitt bereits enthält, wird nicht erneut
  injiziert.
- Wenn keine relevanten Erinnerungen zurückgegeben werden, wird nichts injiziert
  und der Turn läuft unverändert weiter.

Der Recall läuft vor der Unterhaltungspersistenz und der Upstream-Weiterleitung;
ein langsamer oder fehlerhafter Speicherdienst fügt **keine Latenzgarantie** über
seinen eigenen 10-Sekunden-RPC-Timeout hinaus hinzu und kann die Anfrage nicht zum
Scheitern bringen.

## Writeback

Nach einer abgeschlossenen Assistenten-Antwort wird der Turn als **Episode**-Knoten
an den Speicherdienst zurückgeschrieben. Der Episodentext ist ein heuristisches
Transkript des Turns — `User: <user content>\nAssistant: <assistant content>`
(jede Seite bei Leere weggelassen; sind beide leer, wird das Writeback
übersprungen). Das Writeback ist **fire-and-forget**: Es läuft in einem
gespawnten Task, blockiert niemals die Chat-Antwort, und seine Fehler werden nur
innerhalb des Speicher-Clients protokolliert. (Auf dem REST-Streaming-Pfad
erfordert das Writeback zusätzlich, dass der Anfrage eine Unterhaltung zugeordnet
ist; die Nicht-Streaming-REST- und RPC-Pfade schreiben unabhängig davon zurück.)

## Steuerung pro Anfrage

Sowohl der REST-Chat-Anfragebody als auch die Parameter von RPC `chat.send`
akzeptieren ein optionales `memory`-Feld, um die Server-Konfiguration **pro
Aufruf** zu überschreiben:

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` — erzwingt den Recall an/aus für diesen Turn.
- weggelassen (`null`) — folgt der Server-Konfiguration
  (`req.memory.unwrap_or(true)`), d. h. aktiv genau dann, wenn das Gateway
  konfiguriert ist.

Die Überschreibung betrifft den Recall; das Writeback folgt ausschließlich
`ARONA_MEMORY_WRITEBACK` plus dem aktivierten Gateway.

## Header-Zustände

REST-Antworten tragen den Speicherzustand des Turns im **`X-Arona-Memory`**-
Antwort-Header; die RPC-`chat.send`-Antwort spiegelt denselben Wert in einem
`memory`-Feld ihres Ergebnisses wider. Mögliche Zustände:

| Wert | Bedeutung |
| --- | --- |
| `enabled` | Speicher wurde angefordert, das Gateway ist konfiguriert, der Recall war erfolgreich und mindestens eine Erinnerung wurde injiziert. |
| `disabled` | Gateway nicht konfiguriert, oder `memory: false` in der Anfrage, oder keine Benutzernachricht zum Abfragen, oder der Recall war erfolgreich, lieferte aber **keine** relevanten Erinnerungen (nichts zu injizieren). |
| `offline` | Speicher wurde angefordert und das Gateway ist konfiguriert, aber der Recall-Aufruf schlug fehl (Verbindung abgelehnt, RPC-Fehler oder Timeout). |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## Fehlersemantik

Alles degradiert explizit in dieselbe Richtung — der Chat läuft immer:

- **Recall-Fehler** — auf `warn`-Ebene protokolliert; die Anfrage läuft ohne
  injizierte Erinnerungen weiter und meldet `offline` im Header.
- **Writeback-Fehler** — innerhalb des Speicher-Clients protokolliert; die
  Chat-Antwort bleibt unberührt.
- **Speicherdienst nicht konfiguriert** — Recall und Writeback sind No-Ops; jede
  Anfrage meldet `disabled`.

Es gibt keinen Modus, in dem ein Speicherausfall eine Chat-Anfrage über die
eigenen begrenzten Timeouts des Clients hinaus scheitern lässt oder verzögert.

## RPC-Oberfläche

Auf der JSON-RPC-Oberfläche sind zwei Verwaltungsmethoden verfügbar (beide
erfordern ein JWT; siehe [JSON-RPC-API](../api/jsonrpc.md)):

**`memory.status`** — Momentaufnahme des Gateways:

```json
{
  "enabled": true,
  "writeback": true,
  "events": [
    { "kind": "recall", "detail": "…query…", "at": "2026-01-01T00:00:00.000Z" },
    { "kind": "writeback", "detail": "…node id…", "at": "…" },
    { "kind": "error", "detail": "…", "at": "…" }
  ]
}
```

`events` ist ein In-Memory-Ringpuffer der letzten Aktivität — Recall-, Writeback-,
Delete- und Fehlerereignisse, neueste zuerst, bis zur angeforderten Anzahl (der
Status-Handler fordert die letzten 50 an; der Puffer selbst ist auf 100 begrenzt).
Er ist **kein** dauerhaftes Audit-Protokoll — er wird bei einem Neustart
zurückgesetzt.

**`memory.delete`** — entfernt einen gespeicherten Knoten anhand der Id:

```json
{ "node_id": "…" }
```

Gibt `{ "deleted": true | false }` zurück. Schlägt mit einem Fehler fehl, wenn
`node_id` fehlt oder der Speicherdienst nicht konfiguriert ist.

## Verwandtes

- [Konfiguration](configuration.md) — `ARONA_MEMORY_*`-Variablen.
- [Schnellstart](quickstart.md) — End-to-End-Einrichtung des Gateways.
- [Backends](backends.md) — wie Chat-Anfragen geroutet werden, bevor der Recall läuft.
- [Abrechnung & Nutzung](billing-usage.md) — wie dieselben Chat-Turns erfasst werden.
- [Betrieb](operations.md) — Protokolle und Health für die Speicherverbindung.
- [JSON-RPC-API](../api/jsonrpc.md) — `memory.status`, `memory.delete`, `chat.send`.
- [Übersicht](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
