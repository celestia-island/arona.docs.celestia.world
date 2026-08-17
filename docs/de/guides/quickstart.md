---
title: "Schnellstart"
description: "End-to-End-Durchgang durch Arona mit dem integrierten Mock-Upstream: migrieren, bereitstellen (serve), ein Backend registrieren, einen API key erstellen und chatten."
---

# Schnellstart

Diese Anleitung führt Sie durch eine vollständige End-to-End-Einrichtung von
Arona auf einem einzelnen Rechner mit dem **integrierten Mock-Upstream** —
keine echten Modell-Gewichte, keine GPU und kein externes API-Konto
erforderlich. Am Ende haben Sie:

- ein laufendes Arona-Gateway (`/v1/*`-OpenAI-kompatible REST-API plus die
  `/api/rpc`-JSON-RPC-Management-Plane),
- den Mock-Upstream, der als `external`-Backend registriert ist,
- ein Benutzerkonto und einen API key,
- eine funktionierende Non-Streaming- **und** Streaming-Chat-Runde gegen den
  Mock,
- usage records, die über `usage.list` sichtbar sind.

## Voraussetzungen

- **Rust-Toolchain** (siehe `rust-toolchain.toml` im Repository-Wurzelverzeichnis).
- **Python 3** mit `aiohttp` — nur für den Mock-Upstream erforderlich
  (`pip install aiohttp`).
- Eine **laufende PostgreSQL**-Instanz und die dazugehörige Verbindungs-URL.

## 1. Umgebung einrichten

Arona liest seine Konfiguration **beim Prozessstart** aus Umgebungsvariablen.
Zwei sind Pflicht: `DATABASE_URL` und `JWT_SECRET` — ohne sie weigert sich der
Server zu starten (außer bei `MOCK_MODE=1`). `ARONA_ADMIN_TOKEN` wird dringend
empfohlen: Ohne ihn liefert jede `/api/admin/*`-Route 401 zurück.

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

Diese Variablen werden einmal beim Prozessstart gelesen — wenn Sie sie ändern,
starten Sie den Server neu. Die vollständige Variablenreferenz finden Sie unter
[Konfiguration](configuration.md).

## 2. Migrieren und Server starten

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

`serve` allein genügt für eine frische Datenbank: Es migriert beim Start
automatisch. Der Server bindet standardmäßig an `0.0.0.0:8420` (überschreibbar
mit `ARONA_HOST` / `ARONA_PORT`).

## 3. Mock-Upstream starten

In einem zweiten Terminal:

```bash
python3 scripts/mock/server.py
```

Der Mock ist ein aiohttp-Server, der standardmäßig auf `127.0.0.1:8429`
lauscht (`ARONA_MOCK_PORT` überschreibt den Port). Er gibt seinen API key beim
Start aus und bedient außerdem `GET /api/test-key`, das
`{"api_key": ..., "base_url": ...}` zurückgibt. Er stellt eine Handvoll
Modell-IDs bereit — darunter `gpt-5.5`, das weiter unten verwendet wird — und
beantwortet sowohl einfache als auch Streaming-Chat-Completions.

Übernehmen Sie den ausgegebenen Key:

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. Mock als externes Backend registrieren

Backends werden über die Admin-HTTP-API registriert:

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

Das Backend wird sofort bei der Registrierung geprobt und wechselt innerhalb
von ~1-2 Sekunden auf healthy; bis dieser Probe abgeschlossen ist, verbleibt es
in einem fail-closed-Zustand „not probed yet“ (siehe das Troubleshooting-Feld
unten). Die Konfiguration wird persistiert, sodass das Backend einen Neustart
überlebt.

## 5. Konto registrieren und anmelden

Konten liegen auf der JSON-RPC-Plane, `POST /api/rpc`. Da
`ARONA_REGISTRATION_OPEN=1` gesetzt ist, ist `auth.register` offen; der erste
registrierte Benutzer wird zum Admin.

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

Passwörter müssen mindestens 8 Zeichen lang sein **und** mindestens 3 der 4
Zeichenkategorien enthalten (Großbuchstaben, Kleinbuchstaben, Ziffern,
Sonderzeichen). Melden Sie sich dann an, um das JWT-Paar zu erhalten:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

Exportieren Sie das `access_token` aus der Antwort:

```bash
export JWT="<access_token from the login response>"
```

## 6. API key erstellen

`keys.create` ist JWT-authentifiziert und gibt das **vollständige**
`arona-{uuid}`-Secret genau einmal zurück — die Datenbank speichert nur dessen
SHA-256-Hash, also kopieren Sie es jetzt:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. Chat (Non-Streaming)

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

Sie erhalten ein Completion-Objekt im OpenAI-Stil mit einem
`choices[0].message`-Feld und einem `usage`-Block.

## 8. Chat (Streaming)

Derselbe Endpunkt antwortet mit `"stream": true` über Server-Sent-Events: ein
`data:`-Chunk pro Token, abgeschlossen durch einen finalen
`data: [DONE]`-Chunk:

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. Nutzung prüfen

Jede Chat-Runde zeichnet eine Nutzungszeile unter dem Präfix des Keys auf.
Fragen Sie sie mit dem JWT ab:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

Sie sollten einen oder mehrere Datensätze für die oben gemachten
`gpt-5.5`-Anfragen sehen.

## Fehlerbehebung

- **`No backend available for model: gpt-5.5` (HTTP 404, `code:
  model_not_found`)** — kein registriertes Backend bedient diese Modell-ID.
  Entweder wurde das Backend nie registriert (oder seine `models`-Liste enthält
  die ID nicht), oder der Registrierungsaufruf ist fehlgeschlagen. Prüfen Sie
  mit `GET /api/admin/backends` (Admin-Token).
- **`all backends unhealthy` (HTTP 500, `backend_error`)** — für das Modell
  *ist* ein Backend registriert, aber kein Kandidat ist healthy. Ein frisch
  registriertes externes Backend startet in einem fail-closed-Zustand
  „not probed yet“ und wechselt auf healthy, sobald der Probe bei der
  Registrierung abgeschlossen ist, etwa 1-2 s später; chatten Sie innerhalb
  dieses Fensters, stoßen Sie auf diesen Fehler. Wiederholen Sie es nach einem
  Moment oder prüfen Sie, ob der Mock tatsächlich auf `127.0.0.1:8429` läuft.
- **HTTP 401 bei `/v1/*`** — ein fehlender `Authorization`-Header ergibt
  `Missing Authorization header. Use: Bearer <api_key>`; ein unbekannter Key
  ergibt `Invalid API key`. Prüfen Sie `$AR_KEY` noch einmal (vollständiges
  Secret, nicht das Präfix).
- **HTTP 401 `Admin access required` bei `/api/admin/*`** — das Bearer-Token
  stimmt nicht mit `ARONA_ADMIN_TOKEN` überein, oder die Variable ist nicht
  gesetzt (dann lehnt die Route immer ab). Starten Sie den Server nach dem
  Setzen neu.
- **`auth.register` schlägt mit „Registration is closed“ fehl** — die
  Registrierung ist deaktiviert, wenn `ARONA_REGISTRATION_OPEN` nicht truthy
  ist. Setzen Sie `ARONA_REGISTRATION_OPEN=1` **vor** dem Start des Servers (es
  wird beim Start gelesen), oder seien Sie der allererste Benutzer — der erste
  registrierte Benutzer ist immer erlaubt und wird zum Admin.
- **HTTP-429-Ratenlimits** — drei unabhängige Limits können greifen:
  - das In-Memory-Limit pro Key, standardmäßig 60 RPM
    (`ARONA_API_RATE_LIMIT_RPM`) → `Rate limit exceeded. Try again later.`;
  - das 10-RPM-Limit pro Key des kostenlosen Billing-Tiers → 429 mit einem
    `Retry-After: 60`-Header;
  - das monatliche Kontingent von $1 / 100k Token des kostenlosen Tiers → 429
    mit `Retry-After`, das auf den nächsten Abrechnungszeitraum verweist.

## Nächste Schritte

- [Konfiguration](configuration.md) — jede Umgebungsvariable.
- [Backends](backends.md) — Backend-Typen, URL-Semantik und Probing.
- [Deployment](deployment.md) — Bare Metal, systemd, Docker.
- [OpenAI-kompatible REST-API](../api/openai-rest.md) — die vollständige `/v1/*`-Oberfläche.
- [JSON-RPC-API](../api/jsonrpc.md) — die oben verwendete Management-Plane.

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
