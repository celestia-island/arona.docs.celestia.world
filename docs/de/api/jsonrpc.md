---
title: "JSON-RPC-API-Referenz"
description: "JSON-RPC-2.0-API der Arona-Verwaltungsebene unter /api/rpc — Chat-, Realtime-, Engine-, Auth-, Keys-, Provider-, Agents-, Memory-, Konversations-, Usage-, Billing-, Video- und System-Methoden über HTTP und WebSocket."
---

# JSON-RPC-API-Referenz

Arona stellt unter `/api/rpc` eine JSON-RPC-2.0-Oberfläche für die
Verwaltungsebene bereit: Auth, Keys, Providers, Agents, Memory, Konversationen,
Usage, Billing, Video, Realtime und Streaming-Chat. Sie ergänzt die
OpenAI-kompatible REST-Oberfläche (`/v1/*`, siehe
[OpenAI-kompatible REST-API](./openai-rest.md)); nutzen Sie REST für
key-authentifizierte Inferenz-Workloads und JSON-RPC für die
Sitzungs-/Kontoverwaltung und Streaming-Steuerung. Der
[Schnellstart](../guides/quickstart.md) führt durch die erste End-to-End-Runde.

Die Oberfläche verarbeitet **39 Anfrage-Methoden** plus eine anonyme, nur über
WebSocket verfügbare Liveness-Methode, `system.probe` (insgesamt 40 Methoden).
Jede Anfrage ist ein JSON-RPC-2.0-Objekt mit `jsonrpc: "2.0"`, einem
`method`-String, einem optionalen `params`-Objekt und einer optionalen `id`.

## Transport

- **HTTP-POST `/api/rpc`** — Anfrage/Antwort. Senden Sie `Content-Type:
  application/json`. Das JWT wird im `Authorization: Bearer <jwt>`-Header
  übertragen. Der Anfrage-Body ist auf 1 MiB begrenzt.
- **WebSocket `GET /api/rpc`** — langlebige Verbindung. Browser können beim
  WebSocket-Upgrade keine benutzerdefinierten Header setzen, daher wird das JWT
  als `?token=<jwt>`-Abfrageparameter übertragen; der Server faltet es intern in
  einen `Authorization: Bearer`-Header ein (siehe
  `packages/core/src/gateway/server.rs`). Authentifizierte Sockets können
  unbegrenzt verbunden bleiben.
- **Batch-Anfragen** — ein POST-Body, der ein JSON-Array ist, wird Element für
  Element ausgeführt und mit einem JSON-Array von Antworten in derselben
  Reihenfolge beantwortet.
- **Anonymer Zugriff** — über WebSocket ohne JWT bleiben die öffentlichen
  Methoden (`auth.register`/`auth.login`/`auth.refresh`, `providers.list`,
  `system.status`) aufrufbar, und `system.probe` wird mit einem einzigen Ack
  beantwortet, bevor der Socket geschlossen wird. Jede andere Methode erfordert
  ein gültiges JWT; die admin-gesperrten Methoden erfordern zusätzlich ein
  Admin-Konto (siehe Legende unten). Anonyme Sockets unterliegen außerdem einem
  10-Sekunden-Idle-Timeout.
- **Sitzungsanbindung** — ein `x-session-id`-Header bei `POST /api/rpc`
  schiebt zusätzlich die RPC-Antwort selbst auf diesen Sitzungskanal, neben den
  Streaming-Benachrichtigungen.

## IDs

Anfrage-`id`-Werte werden typgetreu zurückgespiegelt: `null` → `null`, Strings
→ Strings, Ganzzahlen → Zahlen, und alles andere (Fließkommazahlen, Objekte,
Ganzzahlen außerhalb des i64-Bereichs) → die JSON-String-Darstellung. Eine
ausgelassene `id` wird mit `null` beantwortet.

## Server → Client-Benachrichtigungen (SSE-Sidecar)

Tokens, Deploy-Fortschritt und Realtime-Events werden **nicht** über den
WebSocket-Socket zugestellt. Jeder Streaming-RPC erzeugt eine Sitzungs-ID und
schiebt Benachrichtigungen als Server-sent Events an
`GET /api/rpc/events?session=<session_id>`. Abonnieren Sie den SSE-Endpunkt
**vor oder unmittelbar nach** dem Zeitpunkt, an dem der RPC-Aufruf eine
Sitzungs-ID zurückgibt — Benachrichtigungen, die zwischen der Rückkehr des
Aufrufs und dem Aufbau des SSE-Abonnements emittiert werden, gehen verloren
(das Prä-Abonnement-Fenster). Das empfohlene Muster ist, zuerst den SSE-Stream
zu öffnen und dann den RPC auszulösen.

Benachrichtigungs-Methoden: `chat.stream` (ein Token pro Event aus
`chat.send`), `models.progress` (Download-Fortschritt von Agent-Modellen aus
`agents.deploy`), `realtime.event` (Server-Events für eine offene Realtime-
Sitzung) sowie `video.progress` / `video.done` / `video.failed` (asynchrone
Video-Jobs). Den vollständigen Katalog finden Sie in
[Events & Benachrichtigungen](./events.md).

## Fehlercodes

| Code | Name | Bedeutung |
| --- | --- | --- |
| `-32700` | Parse error | Der Anfrage-Body ist kein gültiges JSON. |
| `-32600` | Invalid request | Das Anfrage-Objekt ist fehlerhaft, z. B. ein fehlendes `method`. |
| `-32601` | Method not found | Unbekannter `method`-String; die Meldung spiegelt ihn wider. |
| `-32602` | Invalid params | `params` konnte für die Methode nicht deserialisiert werden. |
| `-32603` | Internal error | Unerwarteter Server-Fehler. |
| `-32000` | `APP_ERROR` | Allgemeiner Anwendungsfehler — z. B. Konversation/Provider/Agent nicht gefunden, kein Online-Agent für den Deploy verfügbar. |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` — fehlendes oder ungültiges JWT. Wird auch von Admin-Token-Methoden verwendet, wenn das Bearer-Token nicht mit `ARONA_ADMIN_TOKEN` übereinstimmt (`"Admin access required"`). |
| `-32006` | `QUOTA_ERROR` | Monatliches Billing-Quota für eine JWT-gesperrte RPC-Methode (`chat.send`) überschritten. |
| `-32007` | `ADMIN_REQUIRED` | Authentifizierter **Nicht-Admin** ruft eine admin-gesperrte Methode auf (`agents.*`, `engine.invoke`); die Meldung enthält einen methodenspezifischen Hinweis. |

> Die Methoden `agents.*` und `engine.invoke` sind nur für Admins: Sie erfordern
> ein JWT, dessen Konto `users.is_admin = true` hat. Ein authentifizierter
> Nicht-Admin wird mit `-32007` (`ADMIN_REQUIRED`) abgelehnt; ein nicht
> authentifizierter Aufrufer erhält den standardmäßigen `AUTH_ERROR`, damit der
> Server nicht preisgibt, dass die Methode privilegiert ist.

## Auth-Legende

| Legende | Anmeldedaten |
| --- | --- |
| **public** | Keine Anmeldedaten erforderlich. |
| **JWT** | `Authorization: Bearer <jwt>` bei HTTP oder `?token=<jwt>` bei WebSocket. |
| **admin (JWT + is_admin)** | Bearer-JWT eines Kontos mit `users.is_admin = true`. |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN` (per Umgebungsvariable konfiguriert; wenn nicht gesetzt, wird die Methode immer verweigert — Default-Deny). |

Alle Beispiel-Anmeldedaten und Adressen in diesem Dokument sind Platzhalter
(RFC-5737-Dokumentations-IPs, `sk-xxx`-Keys). Das vollständige Auth-Modell
hinter dieser Legende finden Sie unter
[Authentifizierung & Sicherheit](../guides/auth-security.md).

## Chat

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model` (string), `messages` (array von `{ role, content, images?, tool_calls? }`), `temperature?` (number), `max_tokens?` (integer), `conversation_id?` (string), `memory?` (bool), `extra?` (object), `tools?` (array von OpenAI-artigen Funktionsdefinitionen), `provider?` (string) | Sendet eine Streaming-Chat-Runde. Gibt `{ "stream_id", "memory" }` zurück — `memory` ist der Recall-Zustand (`enabled` / `disabled` / `offline`); Tokens kommen als `chat.stream`-Benachrichtigungen über den SSE-Sidecar. Mit einer `conversation_id` wird der abgeschlossene persistierte Verlauf serverseitig zusammengestellt und die Runde persistiert. Billing-gesperrt (Monatskontingent → `-32006`); Usage wird unter `jwt-<user-uuid>` aufgezeichnet. |

## Realtime (Vollduplex-Audio-/Videositzungen)

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model` (string), `config?` (Sitzungskonfigurationsobjekt), `conversation_id?` (string) | Öffnet eine Vollduplex-Sitzung gegen das Backend, das `model` bedient. Gibt `{ "session_id", "stream_session" }` zurück: Verwenden Sie `session_id` für `realtime.event` / `realtime.stop` und abonnieren Sie `stream_session` über den SSE-Sidecar, um `realtime.event`-Benachrichtigungen zu empfangen. |
| `realtime.event` | JWT | `session_id` (string), `event` (Client-Event — audio append/commit/clear, image frame, response create/cancel, session stop) | Sendet ein Client-Event in eine offene Sitzung; es wird an das Upstream-Backend weitergeleitet. Gibt `{ "ok": true }` zurück. |
| `realtime.stop` | JWT | `session_id` (string) | Schließt und entfernt eine Sitzung. Gibt `{ "removed": bool }` zurück. |

## Engine (generischer Wahrnehmungs-/Steuerkanal)

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `engine.invoke` | admin (JWT + is_admin) | `model` (string), `method` (string), `params?` (object) | Synchrone Anfrage-/Antwort-Invokation einer beliebigen Engine-Methode auf dem Backend, das `model` bedient — der Hochfrequenzkanal für Aufrufe im Stil von `sensor.ingest` / `control.setpoint` (20–30-Hz-Schleifen). Das Ergebnis ist die rohe Antwort des Backends. |

## Auth

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `auth.register` | public | `email`, `password`, `name?` | Registriert ein Konto. Nur erlaubt, solange die Registrierung geöffnet ist (`ARONA_REGISTRATION_OPEN`); der erste registrierte Benutzer wird zum Admin. Gibt dieselbe Token-Antwort wie `auth.login` zurück (`access_token`, `refresh_token`, `token_type`, `expires_in`, `user`). |
| `auth.login` | public | `email`, `password` | Meldet sich an. Gibt `access_token`, `refresh_token`, `token_type`, `expires_in`, `user` (`{ id, email, name, is_admin }`) zurück. Ratenbegrenzt pro IP und Konto. |
| `auth.refresh` | public | `refresh_token` | Tauscht ein Refresh-Token gegen ein frisches Access-Token (und ein neues Refresh-Token). Wiederverwendete oder abgelaufene Refresh-Tokens werden mit `AUTH_ERROR` abgelehnt. |
| `auth.me` | JWT | — | Aktuelles Benutzerprofil: `{ "id", "email", "name" }`. |

## Keys

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | Listet die API keys des Aufrufers (id, name, `key_prefix`, project, Zeitstempel, aktives Flag). |
| `keys.create` | JWT | `name`, `project?` | Erstellt einen API key. Gibt `{ id, name, key, key_prefix, project, created_at }` zurück — das vollständige `arona-<uuid>`-Geheimnis in `key` wird **einmal** angezeigt; speichern Sie es sofort. |
| `keys.revoke` | JWT | `key_id` | Widerruft einen API key. Gibt `{ "ok": true }` zurück. |

## Providers

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | Listet bekannte Provider: integrierte offizielle Einträge plus benutzerdefinierte, als Anzeige-Metadaten (`id`, `name`, `description`, `website_domain`, `is_official`, `is_operator`). Bewusst öffentlich — die Liste trägt keine Anmeldedaten; nur die unten stehenden Mutationen sind JWT-gesperrt. |
| `providers.add` | JWT | `id`, `name`, `description?`, `website_domain?` | Fügt einen benutzerdefinierten Provider-Eintrag hinzu. Gibt `{ "ok": true }` zurück. |
| `providers.update` | JWT | `provider_id`, `name?`, `description?`, `website_domain?` | Aktualisiert die Felder eines benutzerdefinierten Providers (nur die angegebenen). Gibt `{ "ok": true }` zurück. |
| `providers.remove` | JWT | `provider_id` | Entfernt einen benutzerdefinierten Provider. Gibt `{ "ok": true }` zurück. |
| `providers.test` | JWT | — | Testet eine Provider-Verbindung. Stub: gibt `{ "ok": true, "message": "Provider connection test not yet implemented" }` zurück. |

## Agents

Alle `agents.*`-Methoden sind nur für Admins (JWT + `is_admin`). Agent-Knoten
verbinden sich ausgehend über `GET /ws/agent`; diese RPC-Gruppe steuert das
Registry (siehe [Agent-Cluster](../guides/agent-cluster.md)).

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `agents.list` | admin (JWT + is_admin) | — | Listet registrierte Agent-Knoten: id, name, host, `online`/`offline`-Status (heartbeat-basiert), GPU-Zusammenfassung, deployte Modelle, Version, Zeitstempel. |
| `agents.register` | admin (JWT + is_admin) | `machine_name`, `version` | Registriert einen Agent-Knoten beim Tunnel-Manager. Gibt `{ "agent_id", "token" }` zurück (das Token ist die Control-Plane-Anmeldedaten des Agents). |
| `agents.deregister` | admin (JWT + is_admin) | `agent_id` | Hebt die Registrierung eines Agents auf (trennt die Verbindung). Gibt `{ "ok": true }` zurück. |
| `agents.status` | admin (JWT + is_admin) | `agent_id` | Status pro Agent: Online-Flag, host, GPU-Zusammenfassung, geladene Modelle, GPU-Auslastung, Heartbeat-/Verbindungszeitstempel. |
| `agents.deploy` | admin (JWT + is_admin) | `model_id`, `agent_id?` (leer/fehlend = am wenigsten ausgelasteter Knoten; Fehler, wenn keiner online ist) | Deployt ein Modell auf einem Agent. Gibt `{ "ok": true, "stream_id" }` zurück — abonnieren Sie `stream_id` über den SSE-Sidecar für `models.progress`-Download-Benachrichtigungen. |
| `agents.stop` | admin (JWT + is_admin) | `agent_id`, `model_id` | Stoppt ein deploytes Modell. Gibt `{ "ok": true, "stream_id": null }` zurück (kein Fortschritts-Stream). |

## Memory

Langzeit-Memory wird vom entelecheia-Philia-Dienst über einen WebSocket
bedient; Fehler blockieren den Chat nie (siehe
[Memory-Gateway](../guides/memory-gateway.md)).

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | Memory-Gateway-Zustand: `{ "enabled", "writeback", "events" }` — Flags plus bis zu 50 letzte Aktivitäts-Events (neueste zuerst). |
| `memory.delete` | JWT | `node_id` | Löscht einen gespeicherten Memory-Knoten. Gibt `{ "deleted": bool }` zurück. |

## Konversationen

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | Listet die Konversationen des Aufrufers mit Zeitstempeln für das relative Alter. |
| `conversations.create` | JWT | `title?` (Standard `New Conversation`) | Erstellt eine Konversation. Gibt das neue Konversationsobjekt zurück. |
| `conversations.get` | JWT | `conversation_id` (Legacy-Alias: `id`) | Ruft eine Konversation mit ihren Nachrichten ab. Besitzgeprüft; Zugriff über Benutzergrenzen hinweg wird abgelehnt. |
| `conversations.delete` | JWT | `conversation_id` (Legacy-Alias: `id`) | Löscht eine Konversation (nur der Besitzer). Gibt `{ "ok": true }` zurück. |

> `conversations.get` / `conversations.delete` akzeptieren außerdem den
> Legacy-`id`-Schlüssel von älteren Dashboard-Clients; `conversation_id` gewinnt,
> wenn beide vorhanden sind.

## Usage

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?` (integer, Standard 50, auf 1–200 begrenzt), `offset?` (integer, Standard 0), `project?` (string) | Paginiert Usage-Datensätze für den Aufrufer, neueste zuerst, sowohl API-key-Zeilen (`arona-XX`-Präfix) als auch JWT-zugeordnete Zeilen (`jwt-<user-uuid>`) umfassend. Gibt `{ "records", "total", "limit", "offset", "project" }` zurück; der `project`-Filter schränkt auf ausschließlich Key-getaggte Zeilen ein. |

## Billing

Tiers, Quotas und Nutzungsabrechnung werden in
[Abrechnung & Nutzung](../guides/billing-usage.md) beschrieben.

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | Aktueller Billing-Zustand: `{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` — monatliche Nutzung (`cost_usd`, Tokens, Anfrageanzahl) und verbleibendes Quota. |
| `billing.plan.set` | admin token | `user_email`, `tier` | Setzt das Billing-Tier eines Benutzers. Gibt `{ "ok": true }` zurück. Wird mit `AUTH_ERROR` verweigert, wenn das Bearer nicht mit `ARONA_ADMIN_TOKEN` übereinstimmt. |
| `billing.video.pricing.get` | JWT | — | Video-Preistabelle. Gibt `{ "pricing": [...] }` zurück. |
| `billing.video.pricing.set` | admin token | `model`, `mode?` (Standard `per_second_resolution`), `base_price?` (number, Standard 0), `price_per_second?` (number, Standard 0), `price_per_frame?` (number, Standard 0), `resolution_coeff?` (object), `currency?` (Standard `USD`), `enabled?` (bool, Standard `true`) | Fügt Video-Preise für ein Modell ein oder aktualisiert sie (Upsert). Gibt `{ "ok": true }` zurück. Wird mit `AUTH_ERROR` verweigert, wenn das Bearer nicht mit `ARONA_ADMIN_TOKEN` übereinstimmt. |

## Video

Asynchrone Video-Generierungsjobs (siehe
[Realtime & Video](../guides/realtime-video.md)). Job-Fortschritt wird als
`video.progress` / `video.done` / `video.failed`-Benachrichtigungen auf den
Sitzungskanal geschoben.

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`, `prompt`, `negative_prompt?`, `images?` (array von `{ data_base64, mime_type }`), `duration_seconds?` (integer), `width?` (integer), `height?` (integer), `provider?` (string), `extra?` (object) | Reicht einen asynchronen Video-Generierungsjob ein. Gibt `{ "job_id", "stream_id" }` zurück — abonnieren Sie `stream_id` für Fortschritts-Benachrichtigungen. |
| `video.get` | JWT | `job_id` (UUID) | Fragt den Status/das Ergebnis eines Jobs ab (status, progress, result, error, cost). |
| `video.list` | JWT | `limit?` (integer, Standard 20) | Listet die Jobs des Aufrufers. Gibt `{ "jobs": [...] }` zurück. |
| `video.cancel` | JWT | `job_id` (UUID) | Bricht einen laufenden Job ab. Gibt `{ "ok": true }` zurück. |

## System

| Methode | Auth | Parameter | Beschreibung |
| --- | --- | --- | --- |
| `system.status` | public | — | Aggregierter Gateway-Status: `{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`. |
| `system.probe` | anonymous (nur WS) | — | Einmaliger Liveness-Probe über den WebSocket-Transport. Der Server bestätigt `{ "ok": true, "status": "ok" }` und schließt dann den Socket — anonyme Besucher halten nie eine offene Verbindung. Jede andere Methode auf einem nicht authentifizierten Socket wird mit `AUTH_ERROR` abgelehnt. |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
