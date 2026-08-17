---
title: "Abrechnung & Nutzung"
description: "Usage-Datensätze, Kosten pro Modell, Billing-Tiers, Quoten- und Rate-Limit-Durchsetzung, projektspezifische Keys, Video-Preise und die usage.list-RPC."
---

# Abrechnung & Nutzung

Arona misst jede Modell-Anfrage und setzt gestufte Quoten und Rate Limits am
Gateway durch. Preise pro Modell stammen aus der gemeinsamen
plana-Preistabelle (in arona nie neu implementiert), Usage wird als
`usage_records`-Zeilen persistiert, und das gesamte Monatsbild wird über die
`usage.list`-RPC exponiert.

## Usage-Datensätze

Jede gemessene Anfrage landet als eine Zeile in der
`usage_records`-Tabelle (`m20250101_000006_create_usage_records`):

| Spalte | Typ | Bedeutung |
| --- | --- | --- |
| `id` | `UUID` | Primärschlüssel, generiert. |
| `api_key_id` | `VARCHAR(64)` | Das **Key-Präfix** — die ersten 8 Zeichen des API-Keys (Keys sehen aus wie `arona-{uuid}`) — oder eine synthetische `jwt-<user-uuid>`-ID für JWT-zugeordnete RPC-Kanäle. |
| `model` | `VARCHAR(128)` | Modell-ID, an die die Anfrage geroutet wurde. |
| `backend` | `VARCHAR(64)` | Backend-Art: `gateway`, `rpc`, `realtime` oder der Backend-Capability-Name. |
| `prompt_tokens` | `INTEGER` | Eingabetokens, vom Upstream gemeldet oder geschätzt. |
| `completion_tokens` | `INTEGER` | Ausgabetokens, vom Upstream gemeldet oder geschätzt. |
| `total_tokens` | `INTEGER` | Summe der beiden. |
| `cost` | `DOUBLE PRECISION` | Berechnete USD-Kosten; `NULL`, wenn das Modell keine Preiszeile hat. |
| `created_at` | `TIMESTAMPTZ` | Wann die Anfrage abgeschlossen wurde. |

Indizes existieren auf `api_key_id`, `model` und `created_at` (die Spalten,
die die monatliche Aggregation und die Rate-Limit-Fenster scannen).

## Aufzeichnungskanäle

Usage wird auf jedem gemessenen Kanal aufgezeichnet:

1. **REST ohne Streaming** — `POST /v1/chat/completions` und
   `POST /v1/embeddings` zeichnen die exakte, vom Upstream gemeldete Usage
   auf, sobald die Antwort erzeugt wurde.
2. **REST-Streaming (SSE)** — die vom Upstream gemeldete Usage gewinnt, wenn
   der Stream sie trug (OpenAI-kompatibles `usage`-Feld des Terminal-Chunks);
   andernfalls wird eine lokale CJK-bewusste Tokenizer-Schätzung
   (`estimate_usage`) unverändert aufgezeichnet. Streams, die weder Text noch
   Usage erzeugten, werden **gar nicht** aufgezeichnet.
3. **RPC `chat.send`** — dieselbe Schätzung-vs-Upstream-Logik gilt; Zeilen
   werden mit der synthetischen `jwt-<user-uuid>`-ID zugeordnet, sodass sie
   zum Benutzer zurückverknüpfen.
4. **Realtime-Sessions** — jedes abgeschlossene `response_done`-Transkript
   zeichnet seinen Token-Verbrauch (wenn ungleich null) unter
   `jwt-<user-uuid>` mit Backend `realtime` auf.
5. **Video-Jobs** — ein abgeschlossener Job zeichnet explizite Dollarkosten
   auf (siehe [Video-Preise](#video-pricing)); Token-Zählungen sind null.

Die Aufzeichnung ist best-effort: Ein fehlgeschlagener Insert wird geloggt und
schlägt die Anfrage niemals fehl.

## Kostenberechnung

Die Kosten werden aus der kanonischen Preisliste pro 1 Mio. Tokens berechnet
(`plana_llm_provider::metering::lookup_pricing`, über alle
celestia-island-Dienste geteilt):

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

Die Modell-Zuordnung in der Tabelle ist substring-basiert auf der
kleingeschriebenen Modell-ID (spezifischere Familien gewinnen). Wenn ein
Modell keine Preiszeile hat, ist `cost` `NULL`. **Implementieren Sie Preise
nicht in arona neu — aktualisieren Sie planas Tabelle.**

## Tiers

Tiers leben in der `billing_tiers`-Tabelle, geseedet bei der ersten Migration
(`m20250101_000007_create_billing_tiers`). Eine `NULL`-Quotenspalte bedeutet
„unbegrenzt“ für diese Dimension. Benutzer ohne `tier_id` fallen auf das
geseedete `free`-Tier zurück.

| Tier | Monatliches USD-Quotenlimit | Monatliches Token-Quotenlimit | Pro-Key-RPM |
| --- | --- | --- | --- |
| `free` | $1.00 | 100,000 | 10 |
| `pro` | $20.00 | 5,000,000 | 120 |
| `enterprise` | unbegrenzt (`NULL`) | unbegrenzt (`NULL`) | 1000 |

Die Tier-Zuweisung ist eine Admin-Operation (`billing.plan.set`-RPC); das
aktuelle Tier und die Usage werden über `billing.plan` exponiert.

## Quoten- und Rate-Limit-Durchsetzung

### REST (`/v1/*`)

Vor jedem **gemessenen** REST-Endpunkt — `/v1/chat/completions`,
`/v1/embeddings` und `/v1/video/generations` — setzt das Gateway für
key-authentifizierte Anfragen zwei Gates durch:

- **Monatliche Quote** — die `monthly_quota_usd`- und `quota_tokens`-Limits
  des Tiers werden gegen die seit dem ersten Augenblick des aktuellen
  Kalendermonats angesammelte Usage geprüft. Wenn eine der beiden Dimensionen
  ihr Limit erreicht, löst das das Gate aus.
- **Pro-Key-Rate-Limit** — das `rate_limit_rpm` des Tiers wird gegen die
  Anzahl der für dieses Key-Präfix im vergangenen 60-Sekunden-Fenster
  aufgezeichneten Anfragen geprüft. (`/v1/models` und die Health-Endpunkte
  sind nicht gemessen und nicht gegated.)

Eine Ablehnung ist ein HTTP **429 Too Many Requests** mit einem
`Retry-After`-Header und einem OpenAI-artigen Fehler-Body:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| Ablehnung | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| Monatliche Quote erschöpft | `quota_error` / `quota_exceeded` | Sekunden bis zum Beginn des **nächsten Kalendermonats** |
| Tier-Rate-Limit überschritten | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

JWT-authentifiziertes `chat.send` durchläuft dasselbe monatliche Quoten-Gate,
aber gegen das **Ganz-Benutzer**-Fenster (der Aufruf trägt keinen API-Key).
Eine Ablehnung ist ein JSON-RPC-Fehler mit dem implementierungsdefinierten Code
`-32006` (`QUOTA_ERROR`) und derselben Nachricht wie die REST-Quoten-Ablehnung.
Es gibt kein Pro-Key-Rate-Limit auf dem RPC-Pfad — Rate Limiting ist
key-bezogen, und RPC-Aufrufe haben keinen Key. Realtime- und Video-**RPC**-
Methoden sind nicht quoten-gegated.

## Fail-open-Abwägung

Billing ist **von Design her best-effort**. Wenn die Datenbankabfrage hinter
einer Quoten- oder Rate-Limit-Prüfung fehlschlägt, gibt die Prüfung `Unknown`
zurück, und die Anfrage wird **erlaubt** (nur geloggt), statt Chat zu
blockieren. Ein Operator kann sich auf 429er verlassen, um Kapazität zu
schützen, darf sie aber nicht als harte Garantie behandeln, wenn die Datenbank
unhealthy ist — die dokumentierte Abwägung ist Verfügbarkeit des Chat-Pfads
gegenüber striktem Messen.

## Projektspezifische Keys

API-Keys können mit einem `project`-Label erstellt werden (`api_keys.project`,
`default`, wenn nicht gesetzt). Die Quoten-Durchsetzung berücksichtigt es:

- Ein Key, der mit einem anderen Projekt als `default` getaggt ist, prüft
  seine Quote gegen die Usage, die **dem eigenen Bucket dieses Projekts**
  zugeordnet ist (`PROJECT_MONTHLY_USAGE_SQL`).
- `default`-/ungetaggte Keys behalten das **Ganz-Benutzer**-Fenster, passend
  zum Verhalten vor Einführung der Projekte.

JWT-zugeordnete RPC-Zeilen (`jwt-<user-uuid>`) tragen kein Projektlabel und
sind **absichtlich ausgeschlossen** von Projekt-Fenstern — sie zählen
weiterhin zum Ganz-Benutzer-Fenster, sodass ein Projekt nicht „versteckt“
werden kann, indem Traffic über den RPC-Kanal gesendet wird.

## Video-Preise

Videogenerierung verwendet modellspezifische, Task-basierte Preise
(Pro-Token-Preise ergeben für ein Video keinen Sinn). Preiszeilen leben in der
`video_pricing`-Tabelle; `compute_cost` fällt auf einen Platzhalter-Standard
zurück, wenn keine Zeile konfiguriert ist.

| Modus | Kosten |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (Standard) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` ist ein JSON-Objekt, das nach dem Pixelwert der kurzen
Seite geschlüsselt ist (z. B. `"768"`); der `"*"`-Key ist der Fallback. Die
Standard-Preiszeile ist Modus `per_second_resolution`, `base_price` 0.0,
`price_per_second` 0.005, `resolution_coeff {"*": 1.0}`. Zeilen werden über
`billing.video.pricing.get` (beliebiges JWT) und
`billing.video.pricing.set` (Bearer `ARONA_ADMIN_TOKEN`) verwaltet; die
berechneten Kosten werden beim Abschluss des Jobs gegen den Key des Jobs
aufgezeichnet.

## usage.list

`usage.list` (JWT) gibt die paginierten Usage-Datensätze des Aufrufers
zurück, die **sowohl** API-Key-Zeilen (per Key-Präfix verknüpft) als auch
JWT-zugeordnete Zeilen (per synthetischer `jwt-<user-uuid>`-ID verknüpft)
abdecken, neueste zuerst:

| Parameter | Standard | Hinweise |
| --- | --- | --- |
| `limit` | `50` | Begrenzt auf `1..=200`. |
| `offset` | `0` | Seiten-Offset. |
| `project` | nicht gesetzt | Wenn gesetzt, nur Datensätze, die Keys mit diesem Projektlabel zugeordnet sind (JWT-Zeilen sind ausgeschlossen). |

Die Antwort ist `{ "records": [...], "total", "limit", "offset", "project" }`,
wobei jeder Datensatz `id`, `model`, `backend`, `prompt_tokens`,
`completion_tokens`, `total_tokens`, `cost` und `created_at` trägt. Die
monatliche Quoten-Aggregation verwendet dieselbe Join-Form, sodass
Quoten-Rechnung und Usage-Ansicht im Umfang immer übereinstimmen.

## Verwandte Themen

- [Schnellstart](quickstart.md) — einen Key holen und die erste gemessene
  Anfrage stellen.
- [Konfiguration](configuration.md) — Umgebungsvariablen für das Gateway.
- [Authentifizierung & Sicherheit](auth-security.md) — API-Key-Erstellung und
  das `project`-Label.
- [Realtime & Video](realtime-video.md) — der Video-Job-Lebenszyklus hinter den
  Video-Preisen.
- [Operations](operations.md) — Health-Probes und Observability.
- [OpenAI-kompatible REST-API](../api/openai-rest.md) — die `/v1/*`-Oberfläche.
- [JSON-RPC-API](../api/jsonrpc.md) — `usage.list`, `billing.plan`,
  `billing.video.pricing.*`.
- [Übersicht](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
