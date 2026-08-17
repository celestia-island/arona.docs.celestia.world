---
title: "Architektur"
description: "Wie Arona zusammengesetzt ist — Workspace-Layout, der Anfragepfad durch das Gateway, Routing, Health-Probing, Speicher, Sitzungen und die bewussten Design-Kompromisse."
---

# Architektur

Diese Seite führt durch, wie Arona strukturiert ist und wie eine Anfrage durch
es fließt: das Workspace-Layout, der Anfragepfad, das Gateway und der Router,
Health-Checks, der Speicher-Client, Sitzungen und Benachrichtigungen und
schließlich die bewussten Grenzen und Kompromisse, die das Design akzeptiert.
Siehe [Schnellstart](quickstart.md) für ein laufendes Beispiel und
[Betrieb](operations.md) für die täglichen Laufzeit-Belange.

## Workspace-Layout

Das Repository ist ein Cargo-Workspace mit drei Paketen:

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC,
│            # backends, migration, entities, providers, models, registry,
│            # orchestration
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

- `packages/core` ist die Bibliotheks-Crate (`_core`). Sie enthält alles, was
  der Server braucht: das axum-Gateway (`gateway/`), den Modell-Router
  (`routing/`), die Backend-Adapter (`backends/`), die Abrechnung (`billing/`),
  die Authentifizierung (`auth.rs`), den Speicher-Client (`memory/`), die
  JSON-RPC-Ebene (`gateway/rpc.rs`), das Schema (`migration/`, `entity/`),
  Modell-Metadaten (`models/`, `providers/`, `registry/`) und die
  Modell-Orchestrierung (`orchestration/`).
- `packages/agent` baut die `_agent`-Binärdatei, die auf GPU-Knoten läuft und
  sich über `/ws/agent` zurückverbindet (siehe [Agent-Cluster](agent-cluster.md)).
- `packages/cli` baut die `_cli`-Binärdatei, die für Install-, Deploy-, Serve-,
  Migrate- und Download-Operationen verwendet wird.

In diesem Repository gibt es kein Web-Dashboard mehr: Das Vue-Dashboard wurde
entfernt und lebt nun in
[shittim-chest](https://github.com/celestia-island/shittim-chest) (chest #291),
das über die JSON-RPC-Oberfläche mit Arona spricht. Arona selbst ist ein reines
Backend (siehe die [Übersicht](./README.md)).

## Anfragepfad

Der Einstiegspunkt ist der axum-Router, der in `GatewayServer::app`
zusammengesetzt wird (`packages/core/src/gateway/server.rs`). Seine
Routentabelle (server.rs:128-163) deckt die OpenAI-kompatible REST-Oberfläche ab
(`/v1/chat/completions`, `/v1/embeddings`, `/v1/models`, `/v1/health`), die
Videogenerierung, den JSON-RPC-Endpunkt `/api/rpc` (POST + WebSocket-Upgrade),
den SSE-Sidecar `/api/rpc/events`, die Agent-Steuerungsebene `/ws/agent`, die
Swagger-UI unter `/docs` und die Admin-Endpunkte für die Backend-/Alias-
Verwaltung.

Der Router ist in einen kleinen Stapel von Layern gewickelt (server.rs:158-162):

1. Auth-Manager als `Extension`s, sodass die Per-Handler-Extraktoren auf sie
   zugreifen können.
2. Eine Request-Id-Schicht, die einen eingehenden `X-Request-ID`-Header
   wiederverwendet oder einen erzeugt und ihn Handlern und Protokollen
   verfügbar macht (`gateway/request_id.rs`).
3. Ein 1-MB-Limit für den Anfragebody (`RequestBodyLimitLayer`).
4. Eine permissive CORS-Schicht (`*`-Origin, `*`-Header).

Da axum die Layer bottom-up anwendet, ist die CORS-Schicht die äußerste.

Jeder `/v1/*`-Handler läuft dann durch dasselbe Grundgerüst:

1. **Auth-Extraktion** — `ApiKeyAuth` für die reinen Key-Endpunkte
   (`/v1/chat/completions`, `/v1/embeddings`, Video) und `ApiKeyOrJwt` für
   `GET /v1/models`, das sowohl API-Keys als auch Sitzungs-JWTs akzeptieren
   muss (`gateway/middleware.rs`). Der Extraktor löst den Key/das JWT zu einer
   Benutzer-E-Mail, einem Key-Präfix, einem Rate-Limit-Key (dem SHA-256-Hash
   des API-Keys oder einem `u:<email>`-Label für JWTs, damit rotierende Token
   das Fenster nicht zurücksetzen) und einem optionalen Projekt-Scope auf.
2. **Abrechnungs-Gates** — `enforce_billing_gates` (server.rs:492-539) lehnt
   die Anfrage mit HTTP 429 + `Retry-After` ab, wenn das Monatskontingent der
   Tier-Stufe oder das Rate-Limit pro Minute des Benutzers überschritten ist.
   DB-Fehler schlagen fail-open fehl: Die Abrechnung ist Best-Effort, nie eine
   harte Abhängigkeit für das Bedienen eines Chats.
3. **Memory-Recall** (Chat-Pfade) — wenn der Speicher-Client konfiguriert ist
   und die Anfrage danach fragt, werden relevante Langzeiterinnerungen als
   System-Abschnitt injiziert (siehe [Speicher-Client](#memory-client) unten).
   Ein Fehler blockiert den Chat nie; der resultierende Zustand wird im
   `X-Arona-Memory`-Header gespiegelt.
4. **Unterhaltungspersistenz** — eine optionale `conversation_id` wird auf
   Besitz geprüft und der Benutzer-Turn wird zum Sendezeitpunkt persistiert.
5. **Gateway-Dispatch** — die Anfrage wird an das `Gateway` übergeben, das ein
   Backend auflöst, den Kontext trimmt und das Backend-Trait aufruft.
6. **Nutzungsaufzeichnung** — die zurückgegebene (oder stream-terminale)
   Nutzung wird über den `UsageTracker` unter dem Key-Präfix aufgezeichnet und
   persistiert.

Das `Gateway` selbst lebt in `AppState` als `Arc<Gateway>` — es gibt keine
äußere Mutex; innere Mutabilität hält gleichzeitige Chat-/Embeddings-/
Stream-Aufrufe davon ab, jemals eine Sperre über einen Upstream-HTTP-Roundtrip
zu halten (`gateway/mod.rs:29-53`).

## Gateway & Router

Das `Gateway` (`packages/core/src/gateway/mod.rs`) besitzt:

- **Router-Zustand** — die Backend-Liste und Aliase, geschützt durch ein
  `tokio::sync::RwLock`. Die Auflösung auf der Leseseite borgt über awaits;
  Mutationen (register/remove/alias) nehmen eine kurze Schreibsperre und halten
  sie nie über einen Upstream-Aufruf.
- **Einen Anfragezähler** (`AtomicU64`) und eine `start_time`, die von
  `system.status` und den Health-Endpunkten verwendet werden.
- **Eine Deployments-Map** (`model_id → backend name`) für agent-deployte
  Modelle. `register_agent_backend` baut ein `ExternalApiBackend` namens
  `agent-{model_id}` und fügt es in den Router ein; die erneute Registrierung
  desselben Modells ersetzt das vorherige Backend, und
  `unregister_agent_backend` entfernt es bei einem `stop_result`-Frame (siehe
  [Agent-Cluster](agent-cluster.md)).

Die Backend-Auflösung geschieht im `Router`
(`packages/core/src/routing/mod.rs`):

1. **Alias-Auflösung** — ein konfigurierter Alias wird auf sein Ziel
   umgeschrieben.
2. **Sitzungs-Affinität** — wenn eine `conversation_id` vorhanden ist, prüft
   der Router eine Weak-Reference-Map, die die Unterhaltung an das Backend
   pinnt, von dem sie zuerst bedient wurde. Weak-References halten die Map nur
   so lange am Leben, wie das Backend registriert oder in flight ist, sodass
   entfernte Backends ohne Index-Drift verschwinden. Ein ausgelöster Circuit
   Breaker oder ein unhealthy gepinntes Backend degradiert zu einer frischen
   Auswahl, die die Unterhaltung neu bindet.
3. **Kandidatenfilterung** — ein optionaler `provider`-Hinweis filtert nach
   Backend-Name/-Art; Kandidaten müssen healthy *und* einen offenen Circuit
   Breaker haben und das angeforderte Modell listen. Modell-Ids stimmen exakt
   oder über die `:latest`-Suffix-Konvention überein (eine nackte
   `nomic-embed-text`-Anfrage matcht ein gelistetes `nomic-embed-text:latest`).
4. **Least-loaded-Auswahl** — die verbliebenen Kandidaten werden nach ihrem
   Hit-Zähler sortiert und der am wenigsten belastete wird gewählt; der
   Unterhaltungs-Pin (falls vorhanden) wird gleichzeitig aufgezeichnet.

Bevor das Backend aufgerufen wird, trimmt `RequestPipeline::transform`
(`packages/core/src/pipeline.rs:422+`) die Nachrichtenliste auf
`max_context_length` des Backends: System-Nachrichten werden vollständig
beibehalten, Nicht-System-Nachrichten werden neueste zuerst beibehalten,
solange sie passen, und eine einzelne überdimensionierte Nachricht wird hart
nach Zeichen gekürzt (der heuristische Token-Zähler kann nicht token-genau
kürzen). Der Aufruf läuft dann durch das `InferenceBackend`-Trait; Erfolge und
Fehler werden in den Per-Backend-Circuit-Breaker des Routers zurückgeschrieben
(3 Fehler, 30s Erholung, 1 Half-Open-Aufruf — routing/mod.rs:57-64).

## Health-Checker & Probing

`run_health_checks` (`packages/core/src/gateway/health_checker.rs`) läuft als
Hintergrund-Task, der beim Start gespawnt wird (run.rs:135-150), und probt jedes
registrierte Backend einmal pro 60-Sekunden-Intervall. Zwei Details sind
wichtig:

- Die Backend-Liste wird **in jeder Runde frisch** über eine async-Fetcher-
  Closure geholt, sodass Backends, die nach dem Start registriert werden (z. B.
  über die Admin-API), ohne Neustart aufgenommen werden.
- Die erste Runde läuft sofort, bevor das erste Intervall verstreicht, sodass
  der Health-Zustand etabliert ist, sobald der Prozess startet.

`probe_backend` ist der einzige Probe-Codepfad. Er wird von den einmaligen
**Registrierungs-Probes** wiederverwendet: Nachdem ein Admin ein Backend
registriert (server.rs:688-693) oder ein persistiertes Backend beim Boot
wiederhergestellt wird (run.rs:122-127), dreht ein fire-and-forget-Probe das
Backend innerhalb von ~1–2s auf healthy, statt bis zur nächsten 60s-Runde
fail-closed zu bleiben. Genau das lässt die Modellliste eines frisch
registrierten externen Backends fast sofort in `GET /v1/models` erscheinen.

Der Probe selbst ist ein leichtgewichtiger Backend-Aufruf (z. B. trifft das
externe Backend `/v1/models` mit einem 2s-Probe-Timeout); das Ergebnis wird im
Backend gecacht, und das Routing wählt nur Backends aus, deren gecachter
Health-Zustand `Healthy` ist (plus ein offener Circuit Breaker).

## Speicher-Client

Der Speicher-Client (`packages/core/src/memory/mod.rs`) wird beim Serverstart
aus der Umgebungskonfiguration konstruiert (server.rs:95-97): Wenn
`ARONA_MEMORY_URL` und `ARONA_MEMORY_TOKEN` gesetzt sind, fragen Chat-Anfragen
den entelecheia-Philia-Speicherdienst über eine JSON-RPC-WebSocket ab, und
`recall_and_inject` stellt die relevanten Erinnerungen als System-Abschnitt
(`## Relevant Long-Term Memories`) dem ausgehenden Kontext voran. Abgeschlossene
Turns werden über `writeback_dialogue` als Episoden zurückgeschrieben —
fire-and-forget-Arbeit, die gespawnt wird, nachdem die Assistenten-Antwort
persistiert wurde, sodass Speicherfehler den Chat-Antwortpfad nie blockieren
oder verlangsamen. `ARONA_MEMORY_WRITEBACK` (Standard an) schaltet das
Writeback um. Siehe [Memory-Gateway](memory-gateway.md) für das vollständige
Bild.

Jede Chat-Antwort trägt einen `X-Arona-Memory`-Header mit einem von drei
Zuständen: `enabled` (Recall lief und injizierte), `disabled` (nicht
konfiguriert oder die Anfrage übergab `memory: false`) oder `offline`
(konfiguriert, aber der Dienst war unerreichbar).

## Sitzungen & Benachrichtigungen

`AppState` hält einen `plana`-`SessionManager` (`state.sessions`).
Streaming-RPCs wie `chat.send` erstellen eine Sitzungs-Id (`gateway/rpc.rs:1701`)
und schieben Benachrichtigungen — `chat.stream`-Tokens, `models.progress`-
Deploy-Fortschritt, `realtime.event` — auf den Kanal dieser Sitzung. Clients
konsumieren sie aus dem SSE-Sidecar `GET /api/rpc/events?session=<id>`
(server.rs:191-200); siehe [Ereignisse](../api/events.md) für das
Benachrichtigungsformat und den Hinweis zum Vorabonnement-Fenster.

Der Sitzungskanal wird auch für Request/Response-RPC-Aufrufe verwendet: Wenn
ein Client einen `x-session-id`-Header auf `POST /api/rpc` sendet, schiebt der
Server das Ergebnis ebenfalls auf diesen Sitzungskanal (server.rs:184-188,
rpc.rs:128-144), sodass ein Client eine RPC-Antwort auf einen bereits offenen
SSE-Stream multiplexen kann.

## Grenzen & Design-Kompromisse

Das Design akzeptiert bewusst eine Reihe von Grenzen; kennen Sie sie, bevor Sie
die Produktion verwenden:

- **1-MB-Limit für den Anfragebody** — größere Bodies werden von der Schicht
  abgelehnt; wenn Sie Big-Context-Aufrufe benötigen, ist dies das Erste, das
  Sie justieren sollten.
- **CORS `*`** — das Gateway beantwortet Cross-Origin-Aufrufe von überall. Für
  eine API in Ordnung, aber wenn Sie es über vertrauenswürdige Clients hinaus
  freigeben, stellen Sie einen Proxy davor, der Ihre eigene CORS-Richtlinie
  durchsetzt.
- **Fail-open-Abrechnung** — die Quota-/Rate-Limit-Durchsetzung degradiert
  dazu, die Anfrage zuzulassen, wenn die DB nicht verfügbar ist. Abrechnung ist
  Messung, keine Zugriffskontrolle.
- **Kein Gesamt-Timeout auf SSE-Streams** — Streaming-Aufrufe tragen keine
  Gesamtfrist (lange Generierungen sind zulässig); die Hängeerkennung stützt
  sich auf ein 120s-Read-Idle-Timeout pro Lesevorgang
  (`backends/external.rs:24-31`). Nicht-Streaming-Aufrufe erhalten eine
  600s-Gesamtfrist.
- **Tokenizer-Schätzung der Nutzung** — Backends, die niemals Nutzung melden
  (ollama, ws_engine), werden über eine lokale CJK-bewusste Tokenizer-Schätzung
  abgerechnet und unverändert aufgezeichnet (siehe
  [Abrechnung & Nutzung](billing-usage.md)).
- **In-Memory-Rate-Limit-Fenster und -Widerruf** — das gleitende Fenster pro
  Key und die Menge widerrufener Keys leben im Prozessspeicher (`auth.rs`),
  sodass ein Neustart sie zurücksetzt. Der Limiter auf Auth-Ebene begrenzt
  Anfragen pro Key pro Fenster; der Limiter der Abrechnungs-Tier ist
  DB-gestützt (siehe [Auth & Sicherheit](auth-security.md) und
  [Abrechnung & Nutzung](billing-usage.md)).
- **`/ws/agent` ist nicht authentifiziert** — die Agent-Steuerungsebene
  akzeptiert jede WebSocket, die das Register-/Heartbeat-Protokoll spricht.
  Sie ist nur in einem Netzwerk sicher, das Sie kontrollieren.
- **Kein TLS am Gateway** — der Server bindet an klarem HTTP; terminieren Sie
  TLS davor (Reverse Proxy) in jeder Bereitstellung, die eine Netzwerkgrenze
  überschreitet. Siehe [Bereitstellung](deployment.md).

Positiv: Der Server führt einen Graceful-Shutdown durch — er installiert
Ctrl+C- und SIGTERM-Handler, protokolliert „draining connections“ und lässt
laufende Anfragen beenden, bevor der Prozess endet (`gateway/run.rs:14-38` und
die `with_graceful_shutdown`-Verdrahtung bei run.rs:154-159).

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table); gateway/mod.rs:29-53; routing/mod.rs:102-200; health_checker.rs:14-37; run.rs:135-159; memory/mod.rs:207-241; rpc.rs:128-144,1701 -->
