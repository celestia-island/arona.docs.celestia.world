---
title: "Arona"
description: "Plattform zur Selbstbereitstellung und Fernverwaltung von KI-Modellen — Gateway, Backends, Abrechnung, Agents, Speicher."
---

# Arona

**Plattform zur Selbstbereitstellung und Fernverwaltung von KI-Modellen.**

Arona ist eine **reine Backend**-Plattform, geschrieben in Rust (axum): Sie ist
ein OpenAI-kompatibles Modell-Gateway *und* eine Management-Plane für die
Modelle, die Sie auf Ihrer eigenen Hardware betreiben. Sie stellt die
OpenAI-kompatible REST-API `/v1/*`, die JSON-RPC-2.0-Management-Plane
(`/api/rpc`), die Agent-Control-Plane (`/ws/agent`) sowie eine Swagger-UI unter
`/docs` bereit.

Es gibt **kein gebündeltes Web-Dashboard und keinen gebündelten CLI-Chat** —
die Chat- und Admin-Oberfläche lebt in
[shittim-chest](https://github.com/celestia-island/shittim-chest), das über die
RPC-Oberfläche mit Arona kommuniziert. Arona konzentriert sich auf die
Serverseite: Routing, Abrechnung, Authentifizierung, Modell-Bereitstellung,
Agents und Speicher.

## Funktionsmatrix

| Bereich | Was Arona bietet |
| --- | --- |
| **Konversationskern** | OpenAI-kompatible `chat.completions` (Stream + Non-Stream), `embeddings`, `models`-Auflistung; Streaming mit einem abschließenden `[DONE]`-Chunk und echter Nutzung (usage) im letzten Chunk. |
| **Backends** | Vom Admin registrierte Upstreams: `external` (beliebige OpenAI-kompatible HTTP-API), `ollama`, CEP `engine` (WebSocket), `minimax-cloud`-Video und `evernight://`-Bridge-URLs zu Industrie-/Edge-Diensten. |
| **Authentifizierung** | JWT-Access/Refresh-Paare (15 Min. / 7 Tage), API keys `arona-{uuid}` als SHA-256-Hashes gespeichert, drei Admin-Stufen, Passwortrichtlinie, zweigleisiges Rate Limiting. |
| **Abrechnung & Nutzung** | Vorseedierte Billing-Tiers (free / pro / enterprise), usage records pro Anfrage auf jedem Kanal, plana-Preistabelle, Quotenbegrenzung pro Projekt, 429 + `Retry-After`. |
| **Modellverwaltung** | Modell-Download (`hf:` / `ms:` / `gh:`-Quellen), `_agent`-Bereitstellung auf GPU-Knoten, automatische Registrierung bereitgestellter Modelle als routbare Backends. |
| **Realtime & multimodal** | Vollduplex-`realtime.*`-Sitzungen, `engine.invoke`-Wahrnehmungs-/Steuerungskanal, asynchrone Video-Generierungsaufträge (MiniMax-Cloud). |
| **Agent-Cluster** | GPU-Knoten verbinden sich über `/ws/agent`, Platzierung nach geringster Auslastung, Sitzungs-Affinität, Knoten-Persistenz über Neustarts hinweg. |
| **Memory-Gateway** | Langzeitspeicher über entelecheia Philia: Recall-Injektion, Writeback von Episoden, explizite Degradation. |
| **Betrieb** | Health-Probes, `RUST_LOG`-Tracing, Upstream-Fehlerzuordnung (502 vs. 500), Graceful Shutdown, automatische Migration beim Start. |

## Positionierung

Arona ist ein **Gateway + Plattform**: Es routet Modell-Traffic zu Ihren
Backends, stellt Modelle auf Ihren GPU-Agents bereit und misst die Nutzung von
allem.

- vs **pi** — pi ist ein CLI-Assistent, der mit Modellen spricht; arona hat
  keinen CLI-Chat. Arona ist die Plattform, *mit der* pi (und andere Tools)
  spricht.
- vs **one-api / new-api** — das sind API-Key-Gateways vor Modell-Providern;
  arona fügt **Modell-Bereitstellung** (Gewichte herunterladen, auf Ihren
  Agents ausführen), eine vollständige Management-RPC-Plane, Billing-Tiers und
  Speicher hinzu.
- vs **LiteLLM** — ein Gateway-Pendant; arona besitzt darüber hinaus den
  Bereitstellungs-Lebenszyklus der Modelle hinter dem Gateway.

## Erste Schritte

- [Schnellstart](quickstart.md) — End-to-End mit dem integrierten Mock-Upstream.
- [Konfiguration](configuration.md) — jede Umgebungsvariable.
- [Deployment](deployment.md) — Bare Metal, systemd, Docker, Supervision.
- [Backends](backends.md) — Backend-Typen, URL-Semantik und Probing.
- [OpenAI-kompatible REST-API](../api/openai-rest.md) — `/v1/*`.
- [JSON-RPC-API](../api/jsonrpc.md) — die vollständige Management-Plane.

## Repository-Aufbau

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

Das Web-Dashboard wurde aus diesem Repository entfernt und lebt nun in
[shittim-chest](https://github.com/celestia-island/shittim-chest) (chest #291).

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
