---
title: "Realtime et vidéo"
description: "Sessions realtime full-duplex (realtime.start/event/stop), le canal de perception/contrôle engine.invoke, et les jobs asynchrones de génération vidéo."
---

# Realtime et vidéo

Arona expose deux canaux multimodaux au-delà du chat texte simple : **les
sessions realtime full-duplex** (parole/vidéo entrante et sortante sur un
canal bidirectionnel unique) et **la génération vidéo asynchrone** (jobs de
style tâche qui s'exécutent en arrière-plan et rapportent leur progression).
Les deux sont routés vers le backend qui sert le modèle demandé et les deux
sont mesurés à travers la couche de facturation.

## Sessions realtime

Une session realtime est un canal bidirectionnel entre **un client** et **un
upstream** : une API realtime cloud (vocabulaire WebSocket
OpenAI-Realtime / Qwen-Omni-Realtime) ou un moteur CEP local. Les événements
client arrivent via JSON-RPC et sont transmis à l'upstream ; les événements
serveur sont repoussés comme notifications `realtime.event` sur le canal SSE de
la session. L'audio voyage en PCM16 base64 (16 kHz client→modèle, 24 kHz
modèle→client), correspondant au format filaire des providers cloud pour que
le gateway passe les octets inchangés
(`packages/core/src/backends/realtime.rs:1-19`).

### `realtime.start`

Ouvre une session contre le backend servant `model` (JWT ; params `model`,
`config?`, `conversation_id?` — `packages/core/src/gateway/rpc.rs:1890-1898,
1914-1984`). Le backend **doit** déclarer la capacité `realtime` (modalités
audio/vidéo) ; sinon l'appel échoue explicitement avec
`model {model} does not support realtime sessions (no audio/video modality)` —
il n'y a pas de repli silencieux vers le chat texte
(`packages/core/src/gateway/realtime.rs:138-142`).

Deux types d'upstream sont pris en charge (`packages/core/src/gateway/realtime.rs:143-167`) :

- **Upstream moteur CEP** — route les événements sur le canal streaming
  `Engine.InvokeStart` du Celestia Engine Protocol, donc un moteur omni
  déployé localement rejoint la même surface client sans nouveau format
  filaire.
- **Upstream cloud** — une URL fixe `wss://` parlant le vocabulaire
  d'événements realtime cloud (`session.update`, `input_audio_buffer.*`,
  `response.audio.delta`, ...). L'implémentation cloud ré-émet `session.update`
  à la reconnexion.

La réponse est `{ "session_id": ..., "stream_session": ... }` — abonnez-vous à
`/api/rpc/events?session=<stream_session>` avant (ou immédiatement après)
l'appel pour recevoir les événements serveur. Le `conversation_id` optionnel
persiste la transcription de parole comme messages d'assistant et enregistre
l'usage de tokens pour la facturation (`packages/core/src/gateway/realtime.rs:32-85`).

### `realtime.event`

Envoie un événement client dans la session (JWT ; params `session_id`, `event`
— `packages/core/src/gateway/rpc.rs:1989-2013`). Les événements pris en charge
incluent `session.update`, `input_audio_buffer.append` / `.commit` / `.clear`,
`input_image_buffer.append`, `response.create`, `response.cancel` et
`session.stop`. `send_event` est **non bloquant** : l'événement est mis en file
sur un canal mpsc et la tâche forwarder l'écrit vers l'upstream
(`packages/core/src/gateway/realtime.rs:254-280`). Seul le propriétaire de la
session peut envoyer des événements.

### `realtime.stop`

Ferme et supprime la session (JWT ; params `session_id` —
`packages/core/src/gateway/rpc.rs:2016-2034`). Chaque session possède
exactement une **tâche forwarder** qui détient l'upstream et multiplexe les
deux directions : les événements client sont consommés depuis la file et les
événements upstream sont sondés dans la même boucle. Le forwarder se termine
quand l'upstream se ferme ou que la session est arrêtée, supprimant l'entrée du
registre (`packages/core/src/gateway/realtime.rs:195-250`).

Les événements serveur sont poussés comme notifications `realtime.event` avec
les params `{ session_id, event }` sur le canal de session — voir
[Événements et notifications](../api/events.md).

## `engine.invoke`

`engine.invoke` est le canal générique **synchrone** de méthode de moteur
(ADMIN : JWT + `is_admin` ; params `model`, `method`, `params?` —
`packages/core/src/gateway/rpc.rs:261-264,2049-2079`). Il invoque une méthode
arbitraire sur le backend servant `model` et renvoie le résultat directement, ce
qui en fait le canal de perception/contrôle haute fréquence : appels de style
`sensor.ingest`, `control.setpoint` en boucles de 20-30 Hz. Les backends sans
canal d'invocation générique (tous les backends HTTP compatibles OpenAI)
rejettent explicitement avec `backend does not support generic invocation`
(`packages/core/src/backends/mod.rs:573-586`).

## Génération vidéo (REST)

Les jobs vidéo sont des tâches asynchrones de style OpenAI sur la surface REST
(auth par API key — `packages/core/src/gateway/server.rs:876-993` ; voir
[API REST compatible OpenAI](../api/openai-rest.md)) :

**`POST /v1/video/generations`**

| Champ | Type | Notes |
| --- | --- | --- |
| `model` | string | requis — sélectionne un backend capable de vidéo. |
| `prompt` | string | requis. |
| `negative_prompt` | string? | |
| `images` | array? | Data URLs Base64 (`data:image/png;base64,...`), portées comme `{ data_base64, mime_type }`. |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | Indice de sélection de backend (comparé au nom du backend). |
| `extra` | object? | Remplacements spécifiques au backend (seed, steps, cfg, ...). |

Réponse :

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`** sonde le job et renvoie `id`, `object`,
`model`, `status`, `progress`, `result`, `error`, `cost`, `created_at`. Les
jobs sont scopés à l'appelant : un job possédé par un autre utilisateur renvoie
404. La surface REST applique les mêmes portes de facturation (quota mensuel,
limite de débit par minute) que le chemin de chat.

## Génération vidéo (RPC)

La même capacité est disponible via JSON-RPC (JWT —
`packages/core/src/gateway/rpc.rs:371-386,1296-1431`) :

| Méthode | Params | Retours |
| --- | --- | --- |
| `video.create` | mêmes champs que l'appel REST | `{ job_id, stream_id }`. |
| `video.get` | `job_id` | La vue du job (status, progress, result, cost, ...). |
| `video.list` | `limit?` (défaut 20, borné à 1-100) | `{ jobs: [...] }`, les plus récents d'abord. |
| `video.cancel` | `job_id` | `{ "ok": true }` — seul le propriétaire peut annuler. |

`video.create` renvoie un `stream_id` ; abonnez-vous à
`/api/rpc/events?session=<stream_id>` pour recevoir les notifications de job
(`video.progress` / `video.done` / `video.failed` — voir
[Événements et notifications](../api/events.md)).

## Backend

La génération vidéo est **cloud uniquement** : l'API MiniMax H3 Open Platform,
type de backend `minimax-cloud` (`BackendKind::CloudVideo` —
`packages/core/src/backends/mod.rs:502-504,720-727,759-761`). Le flux est de
style tâche :

1. `POST /v1/video_generation_v2` → `task_id`
2. sondez `GET /v1/query/video_generation_v2?task_id=...` jusqu'à `Success` /
   `Fail` ou encore `Pending`
3. en cas de succès, résolvez l'artefact via
   `GET /v1/files/{file_id}/content` → `{ file: { download_url } }`

(`packages/core/src/backends/minimax_cloud.rs:66-210`). Le backend MiniMax ne
sert pas le chat/les embeddings ; ses capacités déclarent
`supports_video_generation` et `realtime: false` (voir
[Backends](./backends.md) pour le modèle de capacités). Le routage résout les
requêtes vidéo uniquement contre les backends avec `supports_video_generation`,
en honorant l'indice `provider` optionnel
(`packages/core/src/routing/mod.rs:205-263`).

Le **backend ComfyUI a été supprimé** lors de la convergence de la plateforme
de modèles : configurer le type de backend `"comfyui"` échoue avec
`comfyui backend removed` (`packages/core/src/backends/mod.rs:756-757`). Il
n'y a pas de chemin vidéo auto-hébergé ; la vidéo passe toujours par un
backend `minimax-cloud`.

## Cycle de vie et tarification des jobs

Un job vidéo traverse `queued → running → done | failed | cancelled`
(`packages/core/src/gateway/video.rs`) :

- **create** — la ligne du job est persistée (`queued`, progress 0) et une
  tâche poller est lancée (`video.rs:109-176`).
- **running** — le poller soumet la tâche (progress 5), puis sonde toutes les
  1,5 s, augmentant progress de 5 toutes les quelques itérations jusqu'à **90**
  (`video.rs:178-275`). Les erreurs de sondage sont journalisées et
  réessayées.
- **done** — progress 100, l'URL du résultat et le coût calculé sont persistés,
  l'usage est enregistré, et une notification `video.done` est diffusée
  (`video.rs:332-368`).
- **failed** — échec de soumission ou de sondage → `video.failed` ; après 900
  itérations de sondage (~22,5 minutes), le job échoue avec
  `generation timed out`.
- **cancelled** — `video.cancel` définit un drapeau que le poller observe à son
  passage suivant ; le job est marqué `cancelled` et `video.failed` se déclenche
  avec l'erreur `cancelled` (`video.rs:389-400`).

L'usage est enregistré avec le coût spécifique à la vidéo : `record_video`
écrit un enregistrement d'usage par requête avec zéro token et un coût en
dollars explicite (`packages/core/src/billing/mod.rs:496-531`).

**La tarification** est spécifique au modèle, dans la table `video_pricing`
(`packages/core/src/billing/video.rs`) :

| Mode | Formule |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (défaut) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` fait correspondre la clé de pixel du petit côté (p. ex.
`"768"`) à un multiplicateur, avec `"*"` comme repli. Les modèles sans ligne
configurée retombent sur : mode `per_second_resolution`, `base_price` 0.0,
`price_per_second` 0.005, `price_per_frame` 0.0,
`resolution_coeff {"*": 1.0}`, devise USD (`billing/video.rs:20-32`).
Interrogez les lignes avec `billing.video.pricing.get` (JWT) et upsertez-les
avec `billing.video.pricing.set` (token admin) — voir l'[API
JSON-RPC](../api/jsonrpc.md). Voir [Facturation et usage](./billing-usage.md)
pour la manière dont les enregistrements d'usage s'agrègent dans la
facturation mensuelle.

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
