---
title: "Betrieb"
description: "Health-Endpunkte, RUST_LOG-Tracing, Upstream-Timeouts, Fehlerzuordnung und Fehlerbehebung für einen laufenden arona-server."
---

# Betrieb

Diese Seite richtet sich an Betreiber, die `arona-server serve` ausführen. Sie
behandelt die Health-Endpunkte, die Sie überwachen, die Protokollzeilen, die sich
zu durchsuchen lohnen, das Timeout-Modell für Upstream-Backends, wie
Backend-Fehler auf HTTP-Fehler abgebildet werden, und die betrieblichen
Stolperfallen, über die Menschen stolpern. Die Bereitstellung selbst wird im
[Bereitstellungsleitfaden](./deployment.md) behandelt.

## Health-Matrix

Alle drei Health-Endpunkte sind nicht authentifiziert und geben `200 OK` zurück,
wann immer der Prozess bedient — es gibt keine Liveness-/Readiness-
Unterscheidung:

| Endpunkt | Antwort |
| --- | --- |
| `/healthz`, `/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | derselbe detaillierte Body wie oben |
| `/api/health` | plana `HealthResponse`: `status`, `version` (`CARGO_PKG_VERSION`), `kind` (`Dev`), `uptime` (Sekunden), `network` (transport / region / asn), `build_hash` (`BUILD_HASH`), `engine_version` (`"0.1.0"`) |

`/healthz` und `/readyz` sind Aliase desselben Handlers, und `/v1/health` teilt
ihn, sodass die Kubernetes-artigen Probes und die OpenAI-kompatible
Health-Route austauschbar sind. `/api/health` fügt uptime, network und
engine_version hinzu. Verwenden Sie `/readyz` für Load Balancer und Supervisor;
verwenden Sie `/api/health`, wenn Sie die umfangreichere Nutzlast benötigen.

## Protokollierung

Der Server protokolliert über `tracing`, gefiltert mit der Standardvariablen
`RUST_LOG` (`RUST_LOG=info` ist die übliche Einstellung; `RUST_LOG=debug`
offenbart Probe-Datenverkehr). Wissenswerte Ereignisse, in grober Reihenfolge
der Häufigkeit:

| Protokollzeile | Ebene | Was sie Ihnen sagt |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | Eine pro Chat-Anfrage, mit `key_prefix`, `model`, `stream` und `request_id` — die einfachste Audit-Spur pro Anfrage. |
| `request completed` | info | Wird vom `logging_middleware`-Helfer nach jeder **Nicht-Streaming**-Antwort von `/v1/chat/completions` und `/v1/embeddings` protokolliert: `method`, `path`, `status`, `latency_ms`, `trace_id`. (Streaming-Chat protokolliert stattdessen `chat completions SSE request` beim Start.) |
| `usage recorded` / `usage persisted` | info | Eine Nutzungszeile wurde aufgezeichnet (in-memory, mit Token/Kosten) und anschließend in die Tabelle `usage_records` geschrieben. |
| `external probe: sending` / `external probe: returned` | debug | Ein Health-Probe des `/v1/models`-Endpunkts eines externen Backends; `matched` sagt aus, ob der Probe innerhalb des 2s-Probe-Timeouts abgeschlossen wurde. |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | Eine `/v1/*`-Anfrage, die vom Abrechnungs-Gate abgelehnt wurde — der Client erhielt 429 plus `Retry-After`. |
| `rpc billing gate rejected: monthly quota exceeded` | warn | Das Quoten-Gate auf der RPC-Seite für JWT-authentifizierte Methoden (Fenster über den gesamten Benutzer; JSON-RPC-Fehlerantwort). |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | Wiederherstellung beim Start: von Admins registrierte Backends und Agent-Knoten werden aus der Datenbank geladen und wieder routbar gemacht. |
| `Shutdown signal received, draining connections…` | info | Der Graceful-Shutdown hat begonnen (SIGINT/SIGTERM). |

## Timeout-Modell

Timeouts werden auf dem Upstream-Client für externe Backends durchgesetzt
(`packages/core/src/backends/external.rs`):

| Timeout | Wert | Gilt für |
| --- | --- | --- |
| Connect | 10s | Herstellen der Upstream-TCP-/TLS-Verbindung. |
| Read idle | 120s pro Lesevorgang | Jeder Upstream-Aufruf; jedes empfangene Byte setzt die Uhr zurück, sodass ein gesunder, aber langsamer Stream nie abgeschnitten wird. |
| Non-streaming gesamt | 600s | Nicht-Streaming-Chat-/Embeddings-Aufrufe — ein langsamer, aber lebendiger Upstream kann eine Anfrage nicht ewig halten. |
| Streaming (SSE) | keines | Streaming-Aufrufe tragen **keine Gesamtfrist**; lange Generierungen sind zulässig, und die Hängeerkennung stützt sich auf das Read-Idle-Timeout. |
| Health-Probe | 2s | Der `/v1/models`-Probe. |

## Fehlerzuordnung

Backend-Fehler werden in den Chat-/Embeddings-Handlern auf HTTP-Status abgebildet
(`packages/core/src/gateway/server.rs`):

| Bedingung | HTTP | `type` / `code` | Meldung |
| --- | --- | --- | --- |
| Upstream-Nicht-2xx-Status (`UpstreamStatus`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| Upstream-Transportfehler (`RequestFailed`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | die Transportfehler-Meldung |
| Jeder andere Backend-Fehler | **500** | `server_error` / `backend_error` | die Fehlermeldung |
| Kein Backend für das Modell (`NoBackend`) | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| Ungültiger API-Key (`Unauthorized`) | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| Rate Limit (`RateLimited`) | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

Die Designabsicht: Aufrufer können „Ihr Provider hat abgelehnt oder
fehlgeschlagen“ (502) von „das Gateway selbst ist kaputt“ (500) unterscheiden.
Jeder Fehlerbody hat dieselbe OpenAI-artige Form —
`{"error":{"message":...,"type":...,"code":...}}` (`json_error_response`). Die
429er des Abrechnungs-Gates tragen zusätzlich einen `Retry-After`-Header und
verwenden `quota_error`/`quota_exceeded` (Quota) bzw.
`rate_limit_error`/`rate_limit_exceeded` (Tier-Rate-Limit).

## Fehlerbehebung

### Ein neu registriertes Backend bleibt bis zum Probe fail-closed

Externe Backends starten in einem unbekannten Health-Zustand und melden
`"<url> not probed yet"`. Sie wechseln auf healthy, wenn (a) die erste Runde des
Health-Checkers läuft — sofort beim Start, danach alle 60s — oder (b) der
fire-and-forget-Probe, der bei der Registrierung oder Wiederherstellung
gestartet wird, erfolgreich ist, normalerweise innerhalb von ~1-2 Sekunden. Bis
dahin schlagen Anfragen, die an das Backend geroutet werden,
konstruktionsbedingt fail-closed fehl.

### Ein 404 beim `/models`-Probe ist für manche Backends normal

Der externe Probe trifft `GET {base}/v1/models` (oder `{base}/models` für
Basis-URLs mit Pfadpräfix). Einige OpenAI-kompatible Server implementieren Chat,
legen aber keine Modellliste offen — der Zhipu-GLM-Coding-Plan-Endpunkt ist
einer davon. Ein **404 wird toleriert**: Das Backend wird als healthy markiert
und die vom Admin konfigurierte Modellliste bleibt für das Routing maßgeblich.
Nur echte fehlgeschlagene Probes (Timeout, Netzwerkfehler, andere Nicht-2xx)
markieren das Backend als unhealthy.

### SSE-Streams, die nichts erzeugen, werden nicht abgerechnet

Eine Streaming-Antwort wird der Nutzung nur dann zugeschrieben, wenn der Stream
Text erzeugt **oder** terminale Nutzung trug; ein Stream, der mit keinem von
beidem endete, wird überhaupt nicht aufgezeichnet. Wenn Sie eine Anfrage ohne
passende `usage recorded`-Zeile sehen, prüfen Sie, ob der Stream tatsächlich
Inhalt erzeugt hat.

### Versionsmeldung

`version` in den Health-Bodies ist `CARGO_PKG_VERSION`; `build_hash` ist der zur
Build-Zeit erzeugte `BUILD_HASH`-Wert, der von `packages/core/build.rs`
ausgegeben wird. Vergleichen Sie `build_hash` über die Knoten hinweg, um zu
bestätigen, dass alle dasselbe Artefakt ausführen.

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
