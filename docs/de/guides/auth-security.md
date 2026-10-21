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

## Öffentlicher Modus & Relay-Login

Arona liefert zwei Authentifizierungshaltungen, gewählt über `auth.mode`
in der [TOML-Konfiguration](./configuration.md):

- **`local`** (Standard) — das historische Verhalten. `auth.login` /
  `auth.register` nehmen E-Mail+Passwort auf der JSON-RPC-Ebene an.
  In Ordnung hinter einem vertrauenswürdigen Netz oder einem strikten
  Reverse-Proxy.
- **`public`** — die internetseitige Haltung. Die Passwort-Arme schlagen
  geschlossen fehl (mit einem Fehler, der auf die Relay-Türen zeigt);
  interaktive Identität kommt **nur** über die Relays unten herein, und
  programmatische Aufrufer nutzen weiterhin unverändert API-Keys. Im
  Modus `public` verweigert der Server den Start, wenn nicht mindestens
  eine Relay-Tür konfiguriert ist — wer beide vergisst, sperrt jeden
  Menschen aus.

Es gibt zwei Türen, und arona ist nie sein eigener Identitätsanbieter:

### OAuth2-Relay (`[auth.oauth]`)

Ein standardmäßiger Authorization-Code-+PKCE(S256)-Client gegen ein
externes Panel (der Anbieter wird konfiguriert, nie selbst gehostet):

1. `GET /auth/oauth/begin` — arona erzeugt ein einmaliges `state` plus
   PKCE-Verifier (serverseitig gehalten, TTL-begrenzt) und leitet zum
   Authorize-Endpunkt des Panels um.
2. Das Panel authentifiziert den Nutzer und leitet zurück auf
   `GET /auth/oauth/callback?code=…&state=…`.
3. arona tauscht den Code am Token-Endpunkt, liest den
   Userinfo-Endpunkt und bildet die Panel-Identität auf ein lokales
   Konto ab. Die Antwort hat dieselbe Form wie `auth.login`
   (`access_token`, `refresh_token`, `user`) — Clients brauchen keine
   neue Token-Behandlung.

Der Identitätsanspruch ist `email` (OIDC-Standard), ersatzweise
`preferred_username`, wenn es wie eine E-Mail aussieht; ein nacktes
opakes `sub` wird nie akzeptiert. Unbekannte Identitäten werden
angelegt, solange `allow_signup` gilt (Standard `true`); die **erste
Relay-Anmeldung auf einer leeren Instanz bootstrapt in die
administrators-Gruppe** — exakt wie die erste Registrierung im
Local-Modus.

### Vertrauenswürdiges Aussteller-Relay (`[[auth.trusted_issuers]]`)

Das API-seitige Gegenstück (das Evernight-Gateway-Muster): Panels, deren
**kirino-ACCESS-Token** arona auf jeder Bearer-Fläche direkt akzeptiert
(`/v1/*`, wo JWTs akzeptiert werden, und die JSON-RPC-Ebene). Jede Zeile
benennt den Aussteller (`iss`, erzwungen), das gemeinsame
Verifikationsgeheimnis und ein optionales `audience`; die Verifikation
ist je Aussteller fail-closed — ein Token eines anderen (oder ohne)
Ausstellers wird abgelehnt, und Refresh-Tokens sind niemals
Sitzungs-Bearer. Der `username`-Anspruch des Panel-Nutzers muss eine
E-Mail sein (aronas Nutzerebene schlüsselt auf E-Mail); sie bildet auf
das LOKALE Konto ab, unbekannte Identitäten werden gemäß der
`allow_signup`-Zeile angelegt.

### Discovery & Haltungsbericht

Die unauthentifizierte JSON-RPC-Methode `auth.methods` meldet, welche
Türen existieren (`password`, `oauth {begin_url, provider}`,
`api_keys`, `trusted_issuers` — nur Boolesche Werte und Namen), und
`system.status` trägt einen additiven `auth`-Block mit denselben
Fakten für Operatoren.

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


4. **Management-Ebene-Identität (`group.*`, `group.credits.*`, `ledger.self`, `keys.*`, `billing.plan`-RPC-Methoden)** — diese Methoden akzeptieren ein Benutzer-JWT und zusätzlich denselben Bearer `ARONA_ADMIN_TOKEN` wie die Admin-Routen: Der Token fungiert dann als **Betreiber-Service-Identität** (die Claims des Instanz-Admins werden synthetisiert), damit Konsolen-Tools die Multi-Tenant-Ebene steuern können. Der Token autorisiert die Inferenz-/Chat-Ebene (`chat.send`, `/v1/*`) bewusst NICHT — er ist eine Verwaltungs-, keine Modell-Berechtigung.

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
  `providers.update` / `providers.remove` / `providers.test` sind admin-gesperrt
  (JWT + `users.is_admin`). Der öffentliche Lesepfad offenbart den Provider-Katalog, aber
  nichts Geheimes.
- **`/ws/agent` verwendet ein gemeinsames Operator-Token, keine Identität pro Agent**: Die Panel-Steuerung akzeptiert das Upgrade nur mit `Authorization: Bearer <Operator-Token>`, das zu `ARONA_ADMIN_TOKEN` passt; ist dieses Panel-Token nicht gesetzt, bleibt die Steuerungsebene geschlossen. GPU-Agenten senden `ARONA_AGENT_TOKEN` (mit Rückfall auf `ARONA_ADMIN_TOKEN`). Da die Anmeldedaten von allen Agenten geteilt werden, kann ein gestohlenes Token weiterhin einen Agenten imitieren; eigene Enrollment-Credentials pro Agent bleiben zukünftige Härtung. Betreiben Sie die Steuerungsebene hinter TLS, wenn sie Netzgrenzen überschreitet.
- **`memory.delete` ist JWT-only ohne Ownership-Prüfung**: Jeder
  authentifizierte Benutzer kann einen Memory-Knoten per `node_id` löschen.
  Das Löschen von Memory erfordert, eingeloggt zu sein, aber nicht, den Knoten
  zu besitzen.

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
