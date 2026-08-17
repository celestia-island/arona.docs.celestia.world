---
title: "Testen"
description: "Die Arona-Testpyramide — Unit-Tests, hermetische Integration, PostgreSQL-gesteuerte Integration, Live-Server-Smoke-Tests, der Mock-Server und die Disziplin für Real-Credential-Smokes."
---

# Testen

Aronas Tests sind in Schichten angeordnet, sodass der Standardlauf `cargo test`
schnell und hermetisch ist und weder eine Datenbank noch das Netzwerk benötigt,
während die schwereren Suiten explizite Opt-ins sind, die die echte
Drahtoberfläche und ein echtes PostgreSQL ausüben. Diese Seite ordnet die
Schichten zu, die Befehle, die sie ausführen, und die Workspace-Disziplin rund
um Real-Credential-Smoke-Läufe.

## Unit-Tests

Der Großteil der Abdeckung besteht aus einfachen Unit-Tests in
`packages/core/src`: 217 `#[test]`-/`#[tokio::test]`-Funktionen plus ~23 weitere
über `packages/agent` und `packages/cli`. Sie laufen mit:

```bash
cargo test --workspace
```

Kein Netzwerk, keine Datenbank. Wichtige Suiten:

- **auth.rs** — die Passwortrichtlinie (≥8 Zeichen UND ≥3 von 4
  Zeichenkategorien), `::uuid`-Casts im rohen INSERT/REVOKE-SQL,
  Anfrage-Standardwerte und Admin-Flag-Lesevorgänge, die auf `false`
  zurückfallen.
- **billing/mod.rs** — Quoten-Mathematik auf der Kosten- *oder*
  Token-Dimension, das Monatsfenster (`month_start`, `seconds_until_month_end`),
  die Rate-Limit-Obergrenze (löst nur *bei* dem RPM aus, `None` = unbegrenzt),
  SQL-Form-Schutz für die Monatsnutzungs-/Tier-/Key-Fenster-Abfragen und
  `estimate_usage`, das von Upstream gemeldete Zahlen bevorzugt.
- **routing/mod.rs** — Alias-Auflösung, `:latest`-Suffix-Abgleich,
  Provider-Hinweise, Least-loaded-Auswahl und Unterhaltungs-Pinning.
- **gateway/mod.rs** — Agent-Backend-Registrierung: Registrieren von
  `agent-{model_id}`, erneute Registrierung, die ersetzt (nicht dupliziert),
  und Abmeldung, die den Router wiederherstellt.

## Hermetische Integration (immer laufend, DB-frei)

`packages/core/tests/gateway_integration.rs` enthält drei immer laufende Tests,
die echte Serialisierungs-/Vertragslogik ausüben, ohne eine Datenbank zu
berühren:

- **A1** — der JSON-RPC-Id-Echo-Serialisierungsvertrag: numerische, String- und
  Null-Anfrage-Ids durchlaufen planas `Id`-Enum mit Typ-Treue.
- **A2** — der Admin-Gate-Fehlercode-Vertrag: `AUTH_ERROR` (-32005, anonym) und
  `ADMIN_REQUIRED` (-32007, authentifizierter Nicht-Admin) bleiben verschieden,
  leben im implementierungsdefinierten Bereich und kollidieren nie mit planas
  Codes oder dem Abrechnungs-`QUOTA_ERROR` (-32006).
- **A3** — `estimate_usage`: die von Upstream gemeldete Nutzung gewinnt
  wörtlich; ohne sie erzeugt die lokale Tokenizer-Schätzung von Null
  verschiedene Prompt-/Completion-Zähler, deren Summe ihre Summe ist.

`packages/core/tests/smoke.rs` fügt drei weitere immer laufende Tests hinzu:
Hardware-Erkennung, den Modell-Registry-Root-Pfad und
Konfigurations-Standardwerte unter `MOCK_MODE=1`.

## PG-gesteuerte Integration

Die vollständige In-Process-Gateway-Suite —
`packages/core/tests/gateway_integration.rs` — spinnt den kompletten axum-Router
auf einem zufälligen Loopback-Port, registriert wegwerfbare OpenAI-kompatible
Mock-Upstreams über die echte Admin-API und treibt die Drahtoberfläche mit
reqwest. Da `AuthManager` auf jedem Pfad mit PostgreSQL spricht (auch
`MOCK_MODE=1` seedet nur Konten *in die Datenbank*), ist diese Suite hinter
`ARONA_TEST_PG=1` gated und standardmäßig übersprungen. Die 10 Tests:

- **T1** register + login + `keys.create`/`keys.list` (roher Key in
  Auflistungen maskiert, `arona-`-Präfix).
- **T2** REST-Chat mit Nutzungsdatensatz-Persistenz in PostgreSQL.
- **T3** JSON-RPC-Id-Echo über die Leitung (Erfolgs- und Fehlerpfade).
- **T4** Admin-Gate auf `agents.list`: anonym → `AUTH_ERROR`, Nicht-Admin →
  `ADMIN_REQUIRED`.
- **T5** Upstream-401 → HTTP 502 `bad_gateway` mit dem Upstream-Detail.
- **T6** Registrierungs-Probe veröffentlicht Modelle (Modell erscheint in
  `GET /v1/models` innerhalb von 10s ohne statische Modellliste).
- **T7** Unterhaltungspersistenz über `chat.send` (beide Turns landen in
  `conversations.get`).
- **T8** Free-Tier-Abrechnungs-Gate: 10 RPM pro Key, die 11. Anfrage im Fenster
  ist 429 `rate_limit_exceeded`.
- **T9** SSE-Stream mit terminaler Nutzung, die vom Upstream aufgezeichnet
  wird.
- **T10** fehlerhaftes JSON → 400; unbekanntes Modell → 404 `model_not_found`.

Führen Sie sie mit dem Einzeiler für das wegwerfbare Postgres aus den
Modul-Dokumenten aus (gateway_integration.rs:18-26):

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

Dies sind Beispiel-Zugangsdaten nur für den wegwerfbaren Testcontainer —
richten Sie das niemals auf eine echte Datenbank.

## Live-Server-Smoke

`packages/core/tests/auth_flow.rs` durchläuft die vollständige Kette
`register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` gegen einen **Live**-Arona-Server und spiegelt die bereitgestellte
Auth-Schleife. Es ist standardmäßig `#[ignore]`d — der einfache `cargo test`-Lauf
berührt nie das Netzwerk. Führen Sie es explizit aus:

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

Stellschrauben:

- `ARONA_TEST_URL` — Basis-URL (Standard `http://127.0.0.1:8420`).
- `ARONA_TEST_EXPECT_CHAT=1` — assertet hart, dass `POST /v1/chat/completions`
  200 zurückgibt. Ohne sie assertet der Test nur, dass die Auth bestanden wurde
  (nicht 401/403), weil die Zielumgebung möglicherweise keinen
  Inference-Provider konfiguriert hat.

Die Suite enthält auch Negativtests: Eine nicht authentifizierte
Chat-Completion und ein nicht authentifiziertes `GET /v1/models` müssen beide
mit 401 abgelehnt werden.

## Mock-Server

`scripts/mock/server.py` ist eine aiohttp-basierte OpenAI-kompatible Fälschung,
die vom Schnellstart und von Smoke-Läufen verwendet wird. Sie bedient
`POST /v1/chat/completions` (non-stream und SSE), `GET /v1/models`,
`GET /api/health`, die JSON-RPC-WebSocket-/HTTP-Oberfläche unter `/api/rpc`,
einen SSE-Sidecar unter `/api/rpc/events` und `GET /api/test-key`, das den
Mock-API-Key zurückgibt, damit andere Dienste ihn entdecken können. Sie lauscht
standardmäßig auf Port 8429 (überschreibbar mit `ARONA_MOCK_PORT`, Host mit
`ARONA_MOCK_HOST`). Der [Schnellstart](quickstart.md) verwendet sie, um eine
End-to-End-Umgebung ohne echte Modell-Provider aufzubauen.

## Real-Credential-Smoke-Disziplin

Smoke-Läufe gegen echte Provider (DeepSeek / GLM) sind bewusst **keine**
Repository-Tests — sie erfordern echte Zugangsdaten und echtes Geld, daher
können sie weder in CI noch im Git-Baum leben. Die Workspace-Konvention,
dokumentiert in den gateway_integration-Modul-Dokumenten
(gateway_integration.rs:54-55), ist:

- Nachweisdateien liegen unter `/mnt/work/arona-pr*-smoke.md` — workspace-lokal,
  niemals in Git committet.
- Zugangsdaten kommen nur aus der Umgebung; Budgets werden klein gehalten.
- Jeder PR, der den Inference-Pfad berührt, erhält einen schriftlichen
  Nachweis.

Der Mock-Server ist der Stellvertreter für diese Läufe in CI und lokaler
Entwicklung; der Real-Credential-Smoke ist ein menschlicher Schritt zur
Release-Zeit.

## CI

`.github/workflows/ci.yml` führt `cargo fmt`, `cargo clippy`, `cargo test
--workspace` und `cargo-deny` auf den Self-Hosted-Runnern der Organisation aus
(`[self-hosted, linux, x64, local]`); `ci-hosted.yml` spiegelt dieselben
Prüfungen auf GitHub-gehosteten Runnern. `.github/workflows/docs.yml` baut
diese Docs-Site mit lagrange und stellt sie bei Pushes, die `docs/**`
berühren, auf GitHub Pages bereit.

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
