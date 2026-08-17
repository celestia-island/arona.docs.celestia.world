---
title: "API REST compatible OpenAI"
description: "Référence /v1/* de style OpenAI — complétions de chat, embeddings, liste des modèles, générations vidéo asynchrones, formes d'erreur et limites de débit."
---

# API REST compatible OpenAI

Arona expose une surface REST compatible OpenAI sous `/v1/*` pour le chat LLM,
les embeddings, la liste des modèles, les sondes de santé et la génération
vidéo asynchrone. Tout SDK OpenAI pointé sur l'URL de base fonctionne pour le
chat et les embeddings ; les endpoints vidéo suivent la convention de tâche
soumission/sondage d'OpenAI.

Tous les corps de requête et de réponse sont en JSON. Les erreurs utilisent
une forme uniforme (voir [Errors](#errors)) ; les échecs d'authentification
au niveau de la couche middleware sont l'unique exception et sont renvoyés en
texte simple (voir [Authentication](#authentication)).

## Endpoints en un coup d'œil

| Méthode | Chemin | Description |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | Tour de chat, streaming ou non-streaming. |
| `POST` | `/v1/embeddings` | Vecteurs d'embedding pour une ou plusieurs entrées. |
| `GET` | `/v1/models` | Modèles du routeur fusionnés avec les modèles de démarrage rapide. |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`. |
| `POST` | `/v1/video/generations` | Soumettre une tâche asynchrone de génération vidéo. |
| `GET` | `/v1/video/generations/{id}` | Sonder le statut / résultat d'une tâche vidéo. |

`/api/health`, `/healthz` et `/readyz` sont des sondes de préparation
supplémentaires (alias de style Kubernetes de `/v1/health`).

## Authentification

Les endpoints de chat, embeddings et vidéo s'authentifient avec une **API
key** dans l'en-tête `Authorization: Bearer`. Les API keys sont créées via le
plan de gestion (`keys.create`, voir l'[API JSON-RPC](./jsonrpc.md#keys)) et
ressemblent à `arona-<uuid>`. Elles sont stockées côté serveur comme hash
SHA-256.

```
Authorization: Bearer arona-CHANGE_ME
```

- **En-tête manquant** → `401` texte simple : `Missing Authorization header. Use: Bearer <api_key>`.
- **Key invalide ou révoquée** → `401` texte simple : `Invalid API key`.
- `GET /v1/models` accepte en plus un token d'accès **JWT** (émis par
  `auth.login` / `auth.register`) pour que le tableau de bord web puisse lister
  les modèles avec le même token qu'il utilise pour le plan RPC. Pour cet
  endpoint, les messages sont
  `Missing Authorization header. Use: Bearer <api_key_or_jwt>` et
  `Invalid API key or JWT`.

Les rejets au niveau middleware sont des corps en texte simple, pas la forme
d'erreur JSON décrite dans [Errors](#errors) — la forme JSON n'est produite
qu'une fois qu'une requête atteint un handler.

Chaque requête `/v1` authentifiée passe aussi par un **limiteur de débit en
mémoire par key** (défaut 60 RPM, fenêtre de 60 secondes, configurable via
`ARONA_API_RATE_LIMIT_RPM`). Le dépasser renvoie `429` texte simple :
`Rate limit exceeded. Try again later.` Les quotas et limites de débit de
niveau tier sont appliqués séparément et renvoient des 429 JSON avec un
en-tête `Retry-After` (voir [429 and Retry-After](#429-and-retry-after)).

> La gestion des API keys, projets et de leur scopage est couverte dans
> [Authentification et sécurité](../guides/auth-security.md).

## POST /v1/chat/completions

L'endpoint de chat compatible OpenAI central, avec support du streaming et
extensions spécifiques à arona (`conversation_id`, `memory`, `extra`,
`provider`).

### Corps de requête

| Champ | Type | Requis | Notes |
| --- | --- | --- | --- |
| `model` | string | oui | Id de modèle tel que listé par `GET /v1/models`. |
| `messages` | array | oui | Tours de chat, voir ci-dessous. |
| `stream` | boolean | non | Défaut `false`. Quand `true`, la réponse est un stream SSE (voir [Streaming](#streaming)). |
| `temperature` | number | non | Température d'échantillonnage, transmise à l'upstream. |
| `max_tokens` | integer | non | Plafond de tokens de complétion, transmis à l'upstream. |
| `conversation_id` | string | non | Affinité de session + persistance. La conversation doit exister et appartenir à l'utilisateur de l'API key (`403` `conversation_forbidden` sinon, `404` `conversation_not_found` si elle n'existe pas). Le tour utilisateur est persisté à l'envoi et la réponse d'assistant quand le tour se termine ; le routage épingle la conversation au backend qui l'a servie en premier. |
| `memory` | boolean | non | Remplacement du gateway mémoire. Défaut `true` (le rappel mémoire est injecté quand le gateway mémoire est activé) ; `false` désactive l'injection de rappel pour cette requête. |
| `extra` | object | non | Passe-through libre fusionné au niveau supérieur de la charge utile upstream (voir ci-dessous). |
| `tools` | array | non | Définitions d'appels de fonction de style OpenAI, passées verbatim à l'upstream. |
| `provider` | string | non | Indice explicite de sélection de backend correspondant à un **nom** de backend (ou type) insensiblement à la casse. Quand défini, seuls les backends correspondant à l'indice sont candidats. |

**Les entrées `messages`** sont `{ "role": "user" | "assistant" | "system", "content": "..." }`.
Deux extensions sont transmises à l'upstream pour les
charges de travail multimodales / agents :

- `images` — images jointes pour les requêtes vision (un tableau d'objets
  `{ "media_type", "data", "position" }` ; le backend external les rend comme
  parties de contenu `image_url` OpenAI).
- `tool_calls` — charges utiles d'appels de fonction produites par le modèle
  upstream, à refléter sur les tours de suivi. Le backend external DOIT les
  transmettre ou les pipelines d'agents (p. ex. les chaînes de compétences
  scepter) perdent chaque invocation d'outil.

**Règles de fusion `extra`** : chaque clé `extra` est fusionnée dans la charge
utile de requête upstream au niveau supérieur, avec deux garanties dures — les
clés réservées `model`, `messages`, `stream`, `temperature`, `max_tokens` et
`options` ne sont **jamais** remplacées, et aucune clé que le gateway a déjà
définie ne l'est non plus. Les valeurs `extra` non-objets sont ignorées.

**Les entrées `tools`** sont `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }` et sont transmises verbatim.

### Réponse non-streaming

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1720000000,
  "model": "Qwen/Qwen3-1.7B",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 12, "completion_tokens": 2, "total_tokens": 14 }
}
```

- `choices[].message` peut porter `tool_calls` pour les tours d'appels de
  fonction.
- L'état mémoire de la requête est reflété dans l'en-tête de réponse
  **`X-Arona-Memory`** : `enabled` | `disabled` | `offline`.

### Streaming

Définissez `"stream": true`. La réponse est un stream SSE
`text/event-stream` — une ligne `data:` par chunk, chacune portant un seul
`ChatChunk` JSON :

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta` porte `content` (et les deltas `tool_calls` avec
  `index`/`id`/`type`/`function` pour les streams d'appels de fonction).
- Sur les upstreams compatibles OpenAI, le **chunk terminal** porte un champ
  `usage` avec les vrais comptes de tokens ; le gateway l'enregistre (retombant
  sur une estimation locale par tokenizer pour les upstreams qui ne rapportent
  jamais l'usage, p. ex. ollama / ws_engine).
- Le stream se termine avec `data: [DONE]`.
- Une erreur de stream est délivrée comme événement `data:` portant
  `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}` ;
  l'événement `[DONE]` suit quand même, et l'enregistrement d'usage plus la
  persistance d'assistant sont ignorés pour le stream échoué.
- L'en-tête `X-Arona-Memory` est présent aussi sur la réponse SSE.

### Exemple

```bash
curl http://192.0.2.10:8080/v1/chat/completions \
  -H "Authorization: Bearer arona-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-1.7B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

## POST /v1/embeddings

| Champ | Type | Requis | Notes |
| --- | --- | --- | --- |
| `model` | string | oui | Id de modèle d'embedding (p. ex. `nomic-embed-text` — un nom nu correspond aussi à un tag `:latest`). |
| `input` | string ou string[] | oui | Une entrée, ou plusieurs. |

Réponse : `{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`.

## GET /v1/models

Liste les modèles routables aujourd'hui : la liste de modèles de chaque
backend enregistré healthy, fusionnée avec les **modèles de démarrage rapide**
intégrés (toujours annoncés, même avant l'enregistrement d'un backend) :
`Qwen/Qwen3-0.6B`, `Qwen/Qwen3-1.7B`,
`HuggingFaceTB/SmolLM2-1.7B-Instruct`, `google/gemma-3-1b-it`,
`microsoft/Phi-4-mini-instruct`, `deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`.

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

Les modèles de démarrage rapide apparaissent avec `owned_by` défini sur leur
provider ; les modèles du routeur portent le nom du backend propriétaire.

## Génération vidéo

Endpoints vidéo de style tâche pour les backends capables de vidéo (p. ex.
`minimax-cloud`, voir [Backends](../guides/backends.md)). Les jobs progressent
de manière asynchrone ; sondez l'endpoint de statut jusqu'à `done`.

### POST /v1/video/generations

| Champ | Type | Requis | Notes |
| --- | --- | --- | --- |
| `model` | string | oui | Id de modèle vidéo enregistré sur un backend capable de vidéo. |
| `prompt` | string | oui | Prompt de génération. |
| `negative_prompt` | string | non | Prompt négatif. |
| `images` | array | non | Images de conditionnement/référence comme tableau d'objets `{ "data_base64": "...", "mime_type": "image/png" }`. |
| `duration_seconds` | integer | non | Durée demandée. |
| `width` / `height` | integer | non | Résolution de sortie. |
| `provider` | string | non | Indice explicite de sélection de backend (nom de backend). |
| `extra` | object | non | Remplacements de workflow spécifiques au backend (seed, steps, cfg, ...). |

Succès → `200` :

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

Erreurs : `400` `missing_fields` quand `model` ou `prompt` est absent ; `503`
`video_backend_error` / `no_backend` quand aucun backend healthy capable de
vidéo ne sert le modèle ; `429` `quota_error` / `quota_exceeded` quand le quota
mensuel est épuisé.

### GET /v1/video/generations/{id}

Renvoie le statut de la tâche :

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "running",
  "progress": 40,
  "result": null,
  "error": null,
  "cost": 0.0,
  "created_at": 1720000000
}
```

- `status` : `queued` | `running` | `done` | `failed` | `cancelled` ; `progress`
  avance de 0 à 90 pendant l'exécution et atteint 100 sur `done`.
- `result` (sur `done`) : `{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }` — `url` pointe vers l'artefact généré servi par le backend.
- `error` (sur `failed` / `cancelled`) et `cost` sont renseignés quand
  applicable.
- Erreurs : `400` `bad_id` pour un id non-UUID ; `404` `no_job` quand le job
  n'existe pas ou appartient à une autre API key.

Les jobs vidéo diffusent aussi la progression sur le sidecar SSE RPC
(`video.progress` / `video.done` / `video.failed`, voir
[Événements et notifications](./events.md#video-job-notifications)).

## Erreurs

Les erreurs au niveau gateway utilisent une forme unique (`json_error_response`) :

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| Statut | `type` / `code` | Quand |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`, `missing_index`, `bad_id`, ... | Champs de requête malformés ou manquants. |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id` appartient à un autre utilisateur. |
| `404` | `invalid_request_error` / `model_not_found` | Aucun backend ne sert le modèle demandé. Message : `No backend available for model: <model>`. |
| `404` | `invalid_request` / `conversation_not_found` | Conversation introuvable. |
| `404` | `not_found` / `no_job` | Job vidéo introuvable. |
| `502` | `server_error` / `bad_gateway` | Upstream non-2xx : message `upstream <status>: <detail>` (détail du corps d'erreur upstream, borné à 4 Ko). Les échecs de transport (connect/read/timeout) se mappent aussi en 502 avec la chaîne d'erreur. |
| `500` | `server_error` / `backend_error` | Autres échecs de backend (p. ex. backend ne supportant pas l'opération). |
| `500` | `server_error` / `internal_error` | Toute erreur interne restante du gateway. |
| `429` | voir ci-dessous | Rejets de quota / limite de débit avec `Retry-After`. |

## 429 et Retry-After

Les réponses 429 incluent un en-tête `Retry-After` (secondes) pour que les
clients compatibles OpenAI se retirent :

| Déclencheur | Corps de statut | `Retry-After` |
| --- | --- | --- |
| Quota mensuel dépassé | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | Secondes jusqu'au mois suivant. |
| Limite de débit par minute de tier | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`. |
| Limiteur en mémoire par key (60 RPM par défaut) | texte simple `Rate limit exceeded. Try again later.` | aucun (rejet middleware). |

Les tiers, le scopage de quota et la comptabilité d'usage sont décrits dans
[Facturation et usage](../guides/billing-usage.md).

## Enregistrement d'usage

Chaque requête `/v1` enregistre une ligne d'usage sous le préfixe d'API key
(`arona-XX`) à son achèvement (chat non-streaming, chat streaming au chunk
terminal, embeddings, et jobs vidéo à l'achèvement avec leur coût calculé).
Voir [Facturation et usage](../guides/billing-usage.md) pour le modèle
d'enregistrement et la manière dont le quota est appliqué.

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
