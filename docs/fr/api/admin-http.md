---
title: "API HTTP d'administration"
description: "Surface admin à bearer token — enregistrer/lister/supprimer des backends et gérer les alias de modèles sur /api/admin/*."
---

# API HTTP d'administration

La surface `/api/admin/*` gère les **backends** du gateway (providers de
modèles upstream) et les **alias** (redirection nom-de-modèle → id-de-modèle).
C'est la contrepartie HTTP du plan de gestion RPC (voir l'[API
JSON-RPC](./jsonrpc.md)) et elle est principalement utilisée par les
opérateurs et l'interface d'administration.

## Authentification

Chaque route `/api/admin/*` requiert :

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN` est lu depuis l'environnement au démarrage du processus
(`GatewayServer::new`). Si la variable est **non définie**, ou que le token
présenté ne correspond pas, la requête est rejetée avec `401` :

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

Le préfixe bearer est comparé insensiblement à la casse (`Bearer` ou
`bearer`).

> Contrairement à la surface `/v1/*`, l'auth admin ne retombe jamais sur les
> API keys ou les JWT, et elle est appliquée avec une comparaison de token
> exacte — faites tourner le token en redémarrant le processus avec une
> nouvelle valeur.

## Backends

Les backends sont les upstreams routables derrière le gateway.
L'enregistrement rend un backend routable immédiatement, persiste sa
configuration pour la restauration au redémarrage, le sonde (passe à healthy
en ~1–2 s) et, pour les URL de pont, maintient le tunnel vivant. Les types de
backend et la sémantique des URL sont détaillés dans
[Backends](../guides/backends.md).

### POST /api/admin/backends — enregistrer un backend

Corps de requête (tous les champs optionnels sauf mention contraire) :

| Champ | Type | Notes |
| --- | --- | --- |
| `type` | string | Type de backend. Un de `external` (toute API HTTP compatible OpenAI), `ollama` (serveur ollama local ou distant), `engine` (moteur CEP sur `ws://`/`wss://`), `minimax-cloud` (API vidéo cloud). Les noms de moteur MDD (`llama_cpp`, `vllm`, `ollama`, `cloud`, `external_api`, `candle`, `native`, ...) se résolvent via le planner. `comfyui` est **rejeté** (`comfyui backend removed`) ; tout le reste → `400` `unknown_type`. Par défaut `ollama` quand absent. |
| `url` | string | URL de base du backend. Les URL de pont `evernight://<node>/<service>` sont résolues via l'agent evernight local en un forward TCP local (échec de résolution → `502` `evernight_unreachable`). Défaut `http://localhost:11434`. |
| `api_key` | string | API key upstream optionnelle, envoyée comme `Authorization: Bearer` sur les appels upstream. |
| `name` | string | Nom du backend. Par défaut la valeur `type` quand absent. Utilisé comme indice `provider` de routage et pour l'identité de la ligne de config. |
| `models` | string[] | Liste statique de modèles. La source de routage quand la sonde n'en découvre aucun. Pour les backends `external`, les modèles découverts sont fusionnés après la liste statique (les ids statiques gardent la précédence) ; les backends `engine` renvoient d'abord leur cache de modèles découverts et ajoutent les ids statiques après ; `minimax-cloud` n'effectue aucune découverte de modèles (sa sonde ne health-ping que `/v1/query/available_models`) et sert la liste statique seule. Ignorée par `ollama`, qui découvre les modèles depuis `/api/tags`. |
| `workflow` | object | Optionnel. Hérité — historiquement consommé par le backend ComfyUI supprimé ; aucun backend actuel ne le lit (conservé pour la compatibilité de colonne `backend_configs`). |

Exemple :

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

Succès → `200` :

```json
{ "status": "ok", "message": "backend registered" }
```

Effets de bord de l'enregistrement :

- Le backend est **enregistré et routable immédiatement** (pas de redémarrage
  nécessaire).
- La configuration est **persistée** dans la table `backend_configs` et
  restaurée au démarrage (un échec de base de données est journalisé mais ne
  bloque jamais la réponse).
- Une **sonde** fire-and-forget s'exécute immédiatement pour que le backend
  passe à healthy en ~1–2 s au lieu de rester fail-closed jusqu'au prochain
  tour du vérificateur de santé de 60 s.
- Pour les URL `evernight://`, une **tâche keepalive** surveille le tunnel :
  à la reconnexion avec un nouveau port local, elle reconstruit
  transparentment et ré-enregistre le backend sous le même nom.

### GET /api/admin/backends — lister les backends

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

- `backends.count` — nombre de backends **healthy**.
- `backends.health` — par backend, le libellé `backend_<index>:<kind>` et
  l'état de santé (`Healthy` / `Degraded` / `Unhealthy`). Le `<index>` est
  l'index d'enregistrement du routeur utilisé par `DELETE /api/admin/backends`.
- `models` — chaque id de modèle routable aujourd'hui (même liste que
  `GET /v1/models`, sans la fusion de démarrage rapide ; voir
  [REST compatible OpenAI](./openai-rest.md#get-v1models)).

### DELETE /api/admin/backends — supprimer un backend

Identifié par son **index** de routeur dans le corps JSON — pas par nom :

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| Champ | Type | Requis | Notes |
| --- | --- | --- | --- |
| `index` | integer | oui | Index d'enregistrement du routeur, correspondant au libellé `backend_<index>` dans le rapport de santé `GET /api/admin/backends`. |

- `index` manquant → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`.
- Index hors plage → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`.
- Succès → `200` `{ "status": "ok", "message": "backend removed" }`.
- La ligne persistée `backend_configs` est supprimée en best-effort : le nom
  du backend est récupéré depuis le `owned_by` de sa liste de modèles ; un
  écart laisse la ligne dans le stockage (les échecs de base de données sont
  journalisés, jamais fatals).

## Alias

Les alias font correspondre un nom de modèle à un autre (`alias` → `target`)
pour que les requêtes d'un id de modèle routent vers un modèle de backend
différent. Les alias sont résolus avant le routage, donc ils s'appliquent
uniformément aux recherches de chat, embeddings et vidéo.

> Les alias sont **uniquement de l'état de routeur en mémoire** — ils ne sont
> pas persistés et sont perdus au redémarrage. Enregistrez-les après le
> démarrage ou recréez-les depuis votre propre état de provisionnement.

### POST /api/admin/aliases — ajouter un alias

| Champ | Type | Requis | Notes |
| --- | --- | --- | --- |
| `alias` | string | oui | Le nom de modèle que les clients demanderont. |
| `target` | string | oui | L'id de modèle vers lequel les requêtes sont routées. |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- `alias` manquant → `400` `missing_alias` ; `target` manquant → `400`
  `missing_target`.
- Succès → `200` `{ "status": "ok", "message": "alias added" }`.
- Ajouter un alias existant remplace sa cible.

### GET /api/admin/aliases — lister les alias

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

Les paires sont renvoyées triées par alias.

### DELETE /api/admin/aliases — supprimer un alias

| Champ | Type | Requis | Notes |
| --- | --- | --- | --- |
| `alias` | string | oui | L'alias à supprimer. |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- `alias` manquant → `400` `missing_alias`.
- Supprimer un alias inconnu est un succès no-op → `200`
  `{ "status": "ok", "message": "alias removed" }`.

## Résumé de persistance

| Ressource | Persistée ? | Restauration au redémarrage |
| --- | --- | --- |
| Backends | Oui — table `backend_configs` (clé `name`, upsert à l'enregistrement, suppression à la suppression). | Oui : restaurés au démarrage ; les backends external démarrent fail-closed et passent à healthy après le premier tour de sonde. Les URL `evernight://` sont re-résolues via le pont au démarrage. |
| Alias | Non — uniquement `Router.aliases` en mémoire. | Non. |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
