---
title: "Authentifizierung & Sicherheit"
description: "JWT-Sessions, API-Keys, die drei Admin-Gates, Passwortrichtlinie, zweigleisiges Rate Limiting und das Sicherheitsmodell."
---

# Authentifizierung & Sicherheit

Arona authentifiziert Aufrufer auf zwei Gleisen: **JWT-Session-Tokens** für
interaktive Clients (die Chat- + Admin-UI, RPC-Aufrufe) und **API-Keys**
(`arona-…`) für programmatischen OpenAI-kompatiblen Traffic. Ein separates
Admin-Token schützt die administrativen Oberflächen. Diese Seite dokumentiert
die Mechanismen, das Sicherheitsmodell und die bekannten risikoarmen Restposten
aus einem Sicherheits-Audit.

## JWT-Sessions

Sessions verwenden JWT-Access-/Refresh-Token-Paare, die vom
`kirino_session`-Token-Manager ausgestellt werden:

- **Access-Token-TTL: 900 Sekunden (15 Minuten).**
- **Refresh-Token-TTL: 604.800 Sekunden (7 Tage).**

Access-Tokens authentifizieren die JSON-RPC-Ebene (`/api/rpc`) und
`GET /v1/models`; der SSE-Sidecar (`/api/rpc/events`) ist über seine
Session-ID geschlüsselt, eine Fähigkeit, die während authentifizierter
RPC-Aufrufe geprägt wird, statt einer Bearer-Credential. Die Endpunkte
`/v1/chat/completions`, `/v1/embeddings` und `/v1/video/*` erfordern einen
**API-Key** (ein JWT wird dort nicht akzeptiert). Access-Tokens sind
kurzlebig, sodass ein gestohlenes Token nur kurz nutzbar ist. Refresh-Tokens
werden über `auth.refresh` gegen frische Paare eingetauscht.

Refresh verwendet **Token-Familien-Rotation**: Das Verbrauchen eines
Refresh-Tokens invalidiert es und stellt ein neues aus, und die
Wiederverwendung eines verbrauchten Refresh-Tokens widerruft die gesamte
Familie — `auth.refresh` antwortet mit `AUTH_ERROR` und der Nachricht
`Refresh token reused` (der zugrunde liegende Fehler ist `TokenReused`,
„refresh token has been reused — token family revoked“), und das Konto muss
sich erneut anmelden. Der Familien-Widerruf ist **in-memory** (ein
`revoked_families`-Set): Ein Server-Neustart leert es, sodass der Schutz über
Neustarts hinweg best-effort ist (Pro-Benutzer-Session-Zustand überlebt keinen
Neustart).

Das Signing-Secret stammt aus der Umgebungsvariable `JWT_SECRET`. Außerhalb
von `MOCK_MODE=1` **verweigert der Server den Start**, wenn `JWT_SECRET` nicht
gesetzt ist oder noch dem eingebauten Entwicklungs-Secret entspricht, sodass
eine Produktionsinstanz niemals versehentlich Tokens ausliefern kann, die mit
einer öffentlichen Konstante signiert sind. Verwenden Sie ein starkes,
zufälliges Secret und committen Sie es niemals.

## API-Keys

API-Keys sind die Maschinen-Credential für die OpenAI-kompatible Oberfläche:

- **Format:** `arona-{uuid}`.
- **Speicherung:** Nur der **SHA-256-Hash** des Keys wird in der
  `api_keys`-Tabelle gespeichert — der Klartext wird genau einmal
  zurückgegeben, in der `keys.create`-Antwort, und kann später nie
  wiederhergestellt werden.
- **Key-Präfix:** Die ersten 8 Zeichen (`key_prefix`) werden im Klartext für
  Anzeige und Usage-Zuordnung gespeichert; die UI zeigt eine maskierte Form
  wie `arona-XXXX…abcd`.
- **Widerruf:** Die Key-Suche verknüpft `api_keys.is_active = TRUE`, sodass ein
  widerrufener Key sofort nicht mehr validiert — es gibt keine Cache-TTL, die
  man abwarten müsste.

## Admin-Stufen

Es gibt drei getrennte Admin-Gates, jedes mit eigener Credential:

1. **`/api/admin/*`-Routen** — Backend- und Alias-Verwaltung
   (`POST/GET/DELETE /api/admin/backends`, `POST/GET/DELETE /api/admin/aliases`)
   erfordern den `Authorization: Bearer ARONA_ADMIN_TOKEN`-Header. Wenn
   `ARONA_ADMIN_TOKEN` nicht gesetzt ist, schlägt `check_admin` immer fehl, und
   jede Admin-Route gibt **401 „Admin access required“** zurück — die gesamte
   Verwaltungsoberfläche ist deaktiviert statt geöffnet.

2. **`agents.*`- und `engine.invoke`-RPC-Methoden** — der Agent-Cluster und die
   Engine-Control-Plane erfordern ein JWT, dessen Konto `users.is_admin =
   true` hat. Ein authentifizierter Nicht-Admin wird mit dem
   implementierungsdefinierten Code **-32007 (`ADMIN_REQUIRED`)** abgelehnt,
   plus einem methodenspezifischen Hinweis (z. B. `agents.deploy starts model deployments on GPU nodes`); ein **nicht authentifizierter** Aufrufer erhält
   den Standard **-32005 (`AUTH_ERROR`)**, sodass der Server nicht verrät, dass
   die Methode überhaupt privilegiert ist.

3. **`billing.plan.set`- und `billing.video.pricing.set`-RPC-Methoden** —
   Billing-Mutationen erfordern dasselbe Bearer-`ARONA_ADMIN_TOKEN` wie die
   Admin-HTTP-Routen; ohne es geben sie `AUTH_ERROR` „Admin access required“
   zurück.

Der **erste registrierte Benutzer wird zum Admin** (`users.is_admin = true`).
Jede spätere Registrierung ist ein regulärer Benutzer, und die Registrierung
ist nur offen, solange `ARONA_REGISTRATION_OPEN` auf einen truthy Wert gesetzt
ist.

## Passwortrichtlinie

Passwörter müssen **beide** Regeln erfüllen (durchgesetzt bei der Registrierung
und auf jedem Passwortänderungs-Pfad):

- mindestens **8 Zeichen**, und
- mindestens **3 der 4 Zeichenkategorien**: Großbuchstaben, Kleinbuchstaben,
  Ziffer, Sonderzeichen.

## Rate Limiting

Rate Limiting läuft auf zwei unabhängigen Gleisen; jedes von beiden kann eine
Anfrage mit **429** ablehnen:

### 1. In-Memory-Sliding-Window (pro Identität)

Jede authentifizierte `/v1`-Anfrage durchläuft einen
In-Memory-Sliding-Window-Limiter, der nach der Identität des Aufrufers
geschlüsselt ist:

- **API-Key-Aufrufe** sind nach dem **SHA-256-Hash** des Keys geschlüsselt;
- **JWT-Aufrufe** sind nach `u:<email>` geschlüsselt — JWTs rotieren alle
  15 Minuten, sodass das Schlüsseln des Fensters nach der Token-Instanz es bei
  jedem Refresh still zurücksetzen würde.

Das Standardbudget ist **60 Anfragen pro Minute**, überschreibbar mit
`ARONA_API_RATE_LIMIT_RPM` (höher setzen für Agent-Pipelines, die viele
parallele LLM-Aufrufe aufteilen). Es auf **0** zu setzen **blockiert jede
Anfrage**.

### 2. Tier-Rate-Limit (pro Key, aus der Datenbank)

Billing-Tiers tragen ein Pro-Key-`rate_limit_rpm`. Die Prüfung zählt
`usage_records`-Zeilen für das Präfix des Keys im **vergangenen
60-Sekunden-Fenster** (Usage wird nach jeder Antwort persistiert, sodass das
Fenster um höchstens eine laufende Anfrage nachhinkt; DB-Fehler schlagen fail
open). Das geseedete **Free-Tier ist 10 RPM**; Pro-/Enterprise-Tiers heben die
Obergrenze an. Die monatliche Quoten-Durchsetzung teilt denselben
Ablehnungspfad.

### Login-Rate-Limiting

Credential-Raten wird am Login-Endpunkt gedrosselt: **5 fehlgeschlagene
Versuche pro 5-Minuten-Fenster pro E-Mail** und **20 pro 5-Minuten-Fenster pro
IP**, jeweils gefolgt von einer 15-minütigen Sperre.

### `Retry-After`

Jede 429-Antwort trägt einen `Retry-After`-Header, damit OpenAI-kompatible
Clients zurückweichen, statt den Endpunkt zu hämmern: Quoten-Ablehnungen setzen
ihn auf **Sekunden bis zum Monatsende**; Rate-Limit-Ablehnungen setzen ihn auf
**60**. Siehe [Abrechnung & Nutzung](billing-usage.md) für das Quotenmodell.

## Hinweise zum Sicherheitsmodell

- **CORS erlaubt jede Origin** (`allow_origin(Any)`) — Arona ist ein Backend,
  das von vielen First-Party- und Third-Party-Clients konsumiert wird; wenn
  Ihre Bereitstellung Origins einschränken muss, setzen Sie einen
  Reverse-Proxy davor, der CORS durchsetzt.
- **Anfrage-Bodies sind auf 1 MB begrenzt** (`RequestBodyLimitLayer`), was den
  Speicherverbrauch auf dem Gateway begrenzt.
- **Das Gateway terminiert kein TLS** — es lauscht auf einfachem HTTP. Stellen
  Sie es hinter einen Reverse-Proxy (siehe [Bereitstellung](deployment.md)),
  der HTTPS terminiert.
- **Secrets kommen nur aus der Umgebung**: `ARONA_ADMIN_TOKEN` und `JWT_SECRET`
  werden aus Umgebungsvariablen gelesen und müssen starke Zufallswerte sein,
  die niemals ins Repository committet werden.
- Die Standard-Bind-Adresse des Servers ist `0.0.0.0`; schränken Sie die
  Exposition auf der Netzwerkebene ein.

## Bekannte risikoarme Restposten (aus dem Audit)

Die folgenden Punkte sind so dokumentiert, wie sie sind; sie sind beabsichtigt
oder vorerst akzeptiert, aber wissenswert, wenn Sie eine Instanz über ein
vertrauenswürdiges Netz hinaus exponieren:

- **`providers.list` ist öffentlich**, während `providers.add` /
  `providers.update` / `providers.remove` / `providers.test` ein JWT
  erfordern. Der öffentliche Lesepfad offenbart den Provider-Katalog, aber
  nichts Geheimes.
- **`/ws/agent` ist eine nicht authentifizierte Control-Plane**: GPU-Agenten
  verbinden sich ohne Credential und registrieren sich selbst (`register` /
  `heartbeat` / Command-Result-Frames). Jeder, der den WebSocket-Port erreichen
  kann, kann einen Fake-Agenten registrieren. Siehe
  [Agent-Cluster](agent-cluster.md) für die operativen Abwägungen.
- **`memory.delete` ist JWT-only ohne Ownership-Prüfung**: Jeder
  authentifizierte Benutzer kann einen Memory-Knoten per `node_id` löschen.
  Das Löschen von Memory erfordert, eingeloggt zu sein, aber nicht, den Knoten
  zu besitzen.

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
