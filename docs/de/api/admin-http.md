---
title: "Admin-HTTP-API"
description: "Admin-Oberfläche mit Bearer-Token — Backends registrieren/auflisten/entfernen und Modell-Aliasse über /api/admin/* verwalten."
---

# Admin-HTTP-API

Die `/api/admin/*`-Oberfläche verwaltet die **Backends** des Gateways
(Upstream-Modell-Provider) und **Aliasse** (Modellname → Modell-ID-Umleitung).
Sie ist das HTTP-Pendant zur RPC-Verwaltungsebene (siehe
[JSON-RPC-API](./jsonrpc.md)) und wird hauptsächlich von Operatoren und der
Admin-UI verwendet.

## Authentifizierung

Jede `/api/admin/*`-Route erfordert:

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN` wird beim Prozessstart aus der Umgebung gelesen
(`GatewayServer::new`). Wenn die Variable **nicht gesetzt** ist oder das
präsentierte Token nicht übereinstimmt, wird die Anfrage mit `401` abgelehnt:

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

Das Bearer-Präfix wird case-insensitiv abgeglichen (`Bearer` oder `bearer`).

> Anders als die `/v1/*`-Oberfläche fällt die Admin-Auth nie auf API keys oder
> JWTs zurück, und sie wird mit einem exakten Token-Vergleich durchgesetzt —
> rotieren Sie das Token, indem Sie den Prozess mit einem neuen Wert neu
> starten.

## Backends

Backends sind die routbaren Upstreams hinter dem Gateway. Die Registrierung
macht ein Backend sofort routbar, persistiert seine Konfiguration für die
Wiederherstellung nach Neustart, probt es (wechselt innerhalb von ~1–2 s auf
healthy) und hält bei Bridge-URLs den Tunnel am Leben. Backend-Typen und
URL-Semantik sind in [Backends](../guides/backends.md) detailliert beschrieben.

### POST /api/admin/backends — Backend registrieren

Anfrage-Body (alle Felder optional, außer wo angegeben):

| Feld | Typ | Hinweise |
| --- | --- | --- |
| `type` | string | Backend-Typ. Einer von `external` (eine beliebige OpenAI-kompatible HTTP-API), `ollama` (lokaler oder entfernter ollama-Server), `engine` (CEP-Engine über `ws://`/`wss://`), `minimax-cloud` (Cloud-Video-API). MDD-Engine-Namen (`llama_cpp`, `vllm`, `ollama`, `cloud`, `external_api`, `candle`, `native`, ...) werden über den Planner aufgelöst. `comfyui` wird **abgelehnt** (`comfyui backend removed`); alles andere → `400` `unknown_type`. Standard `ollama`, wenn fehlend. |
| `url` | string | Basis-URL des Backends. `evernight://<node>/<service>`-Bridge-URLs werden über den lokalen evernight-Agent in einen lokalen TCP-Forward aufgelöst (Auflösungsfehler → `502` `evernight_unreachable`). Standard `http://localhost:11434`. |
| `api_key` | string | Optionaler Upstream-API key, der bei Upstream-Aufrufen als `Authorization: Bearer` gesendet wird. |
| `name` | string | Backend-Name. Standard der `type`-Wert, wenn fehlend. Wird als Routing-`provider`-Hinweis und für die Identität der Konfigurationszeile verwendet. |
| `models` | string[] | Statische Modellliste. Die Routing-Quelle, wenn das Probing keine entdeckt. Bei `external`-Backends werden entdeckte Modelle nach der statischen Liste eingefügt (statische IDs behalten Vorrang); `engine`-Backends geben zuerst ihren Cache entdeckter Modelle zurück und hängen statische IDs an; `minimax-cloud` führt keine Modell-Erkennung durch (sein Probe macht nur einen Health-Ping auf `/v1/query/available_models`) und bedient ausschließlich die statische Liste. Von `ollama` ignoriert, das Modelle aus `/api/tags` entdeckt. |
| `workflow` | object | Optional. Legacy — historisch vom entfernten ComfyUI-Backend konsumiert; kein aktuelles Backend liest es (für die `backend_configs`-Spaltenkompatibilität beibehalten). |

Beispiel:

```bash
curl -X POST http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "name": "mock-upstream",
    "url": "http://192.0.2.20:11434",
    "api_key": "sk-xxx",
    "models": ["Qwen/Qwen3-1.7B"]
  }'
```

Erfolg → `200`:

```json
{ "status": "ok", "message": "backend registered" }
```

Nebenwirkungen der Registrierung:

- Das Backend ist **sofort registriert und routbar** (kein Neustart nötig).
- Die Konfiguration wird in der `backend_configs`-Tabelle **persistiert** und
  beim Start wiederhergestellt (ein DB-Fehler wird protokolliert, blockiert
  aber nie die Antwort).
- Ein **Probe** im Fire-and-Forget-Stil läuft sofort, sodass das Backend
  innerhalb von ~1–2 s auf healthy wechselt, statt bis zur nächsten 60-s-Runde
  des Health-Checkers fail-closed zu bleiben.
- Für `evernight://`-URLs überwacht eine **Keepalive-Aufgabe** den Tunnel: Bei
  erneuter Verbindung mit einem neuen lokalen Port baut sie das Backend
  transparent neu auf und registriert es unter demselben Namen neu.

### GET /api/admin/backends — Backends auflisten

```json
{
  "backends": {
    "count": 2,
    "health": [
      ["backend_0:ExternalApi", "Healthy"],
      ["backend_1:Ollama", { "Unhealthy": "connection refused" }]
    ]
  },
  "models": [
    { "id": "Qwen/Qwen3-1.7B", "object": "model", "owned_by": "mock-upstream" }
  ]
}
```

- `backends.count` — Anzahl **gesunder** Backends.
- `backends.health` — Pro-Backend-`backend_<index>:<kind>`-Label und
  Gesundheitszustand (`Healthy` / `Degraded` / `Unhealthy`). Der `<index>` ist
  der Router-Registrierungsindex, der von `DELETE /api/admin/backends`
  verwendet wird.
- `models` — jede heute routbare Modell-ID (dieselbe Auflistung wie
  `GET /v1/models`, ohne die Quick-Start-Zusammenführung; siehe
  [OpenAI-kompatible REST](./openai-rest.md#get-v1models)).

### DELETE /api/admin/backends — Backend entfernen

Identifiziert durch seinen Router-**Index** im JSON-Body — nicht durch den
Namen:

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| Feld | Typ | Erforderlich | Hinweise |
| --- | --- | --- | --- |
| `index` | integer | ja | Router-Registrierungsindex, passend zum `backend_<index>`-Label im Health-Report von `GET /api/admin/backends`. |

- Fehlender `index` → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`.
- Index außerhalb des Bereichs → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`.
- Erfolg → `200` `{ "status": "ok", "message": "backend removed" }`.
- Die persistierte `backend_configs`-Zeile wird nach dem Best-Effort-Prinzip
  gelöscht: Der Backend-Name wird aus dem `owned_by` seiner Modellauflistung
  wiederhergestellt; ein Mismatch lässt die Zeile im Store (DB-Fehler werden
  protokolliert, nie fatal).

## Aliasse

Aliasse bilden einen Modellnamen auf einen anderen ab (`alias` → `target`),
sodass Anfragen für eine Modell-ID zu einem anderen Backend-Modell geroutet
werden. Aliasse werden vor dem Routing aufgelöst und gelten daher einheitlich
für Chat-, Embedding- und Video-Lookups.

> Aliasse sind **nur In-Memory-Router-Zustand** — sie werden nicht persistiert
> und gehen beim Neustart verloren. Registrieren Sie sie nach dem Start oder
> erstellen Sie sie aus Ihrem eigenen Provisioning-Zustand neu.

### POST /api/admin/aliases — Alias hinzufügen

| Feld | Typ | Erforderlich | Hinweise |
| --- | --- | --- | --- |
| `alias` | string | ja | Der Modellname, den Clients anfragen werden. |
| `target` | string | ja | Die Modell-ID, zu der Anfragen geroutet werden. |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- Fehlender `alias` → `400` `missing_alias`; fehlendes `target` → `400`
  `missing_target`.
- Erfolg → `200` `{ "status": "ok", "message": "alias added" }`.
- Das Hinzufügen eines vorhandenen Alias ersetzt sein Target.

### GET /api/admin/aliases — Aliasse auflisten

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

Paare werden nach Alias sortiert zurückgegeben.

### DELETE /api/admin/aliases — Alias entfernen

| Feld | Typ | Erforderlich | Hinweise |
| --- | --- | --- | --- |
| `alias` | string | ja | Der zu entfernende Alias. |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- Fehlender `alias` → `400` `missing_alias`.
- Das Entfernen eines unbekannten Alias ist ein No-op und gilt als Erfolg →
  `200` `{ "status": "ok", "message": "alias removed" }`.

## Persistenz-Zusammenfassung

| Ressource | Persistiert? | Wiederherstellung beim Neustart |
| --- | --- | --- |
| Backends | Ja — `backend_configs`-Tabelle (`name`-Schlüssel, Upsert bei Registrierung, Löschung bei Entfernung). | Ja: beim Start wiederhergestellt; externe Backends starten fail-closed und wechseln nach der ersten Probe-Runde auf healthy. `evernight://`-URLs werden beim Start erneut über die Bridge aufgelöst. |
| Aliasse | Nein — nur In-Memory-`Router.aliases`. | Nein. |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
