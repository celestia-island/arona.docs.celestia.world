---
title: "Événements et notifications"
description: "Sidecar server-sent events (SSE) — notifications chat.stream, models.progress, realtime.event et vidéo."
---

# Événements et notifications

Les tokens streaming, la progression de déploiement et les événements
realtime ne sont **pas** délivrés sur le socket WebSocket JSON-RPC. Chaque RPC
streaming crée un **id de session** et pousse les notifications vers
l'endpoint SSE comme server-sent events :

```
GET /api/rpc/events?session=<session_id>
```

## Recette s'abonner-avant-d'envoyer

Les notifications émises entre le retour de l'appel RPC (qui renvoie un id de
session) et l'établissement de l'abonnement SSE sont **perdues** (la fenêtre de
pré-abonnement). Le schéma fiable est :

1. Ouvrez d'abord le stream SSE (il bloque jusqu'à ce qu'un id de session soit
   attaché).
2. Déclenchez le RPC qui renvoie l'id de session (p. ex. `chat.send`,
   `agents.deploy`, `realtime.start`, `video.create`).
3. Lisez les notifications du stream SSE à mesure qu'elles arrivent.

Chaque notification est un message de style JSON-RPC 2.0 avec `"jsonrpc": "2.0"`,
une `method` et un objet `params`.

## Catalogue des notifications

### `chat.stream`

Une notification par token, produite par `chat.send` (et tout chemin de chat
streaming qui utilise un canal de session) :

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — un delta de contenu.
- `is_complete` — `false` jusqu'au chunk final (quand l'upstream attache une
  raison de fin, le chunk de contenu final peut déjà porter `is_complete:
  true` avec un token non vide) ; la notification **terminale** suit toujours
  avec un `token` vide et `is_complete: true`.
- Une erreur de stream est délivrée comme notification terminale avec
  `token: "Stream error: ..."` et `is_complete: true` (voir
  `packages/core/src/gateway/rpc.rs`).

### `models.progress`

Progression du téléchargement de modèles pour `agents.deploy`, transmise
depuis l'agent. Le `stream_id` vient de la réponse de `agents.deploy`.

### `realtime.event`

Événements serveur pour une session realtime full-duplex ouverte, poussés sur
le canal de session (`packages/core/src/gateway/realtime.rs`). Les événements
client envoyés via le RPC `realtime.event` sont transmis à l'upstream ; les
événements serveur arrivent ici.

### Notifications de jobs vidéo

Les jobs `video.create` poussent la progression sur le canal de session
(`packages/core/src/gateway/video.rs`) :

| Méthode | Payload (params) | Signification |
| --- | --- | --- |
| `video.progress` | `job_id`, `stream_id`, `status: "running"`, `progress` (0–90) | Le job tourne. |
| `video.done` | `job_id`, `stream_id`, `result`, `cost` | Le job est terminé ; `result` porte l'URL de l'artefact. |
| `video.failed` | `job_id`, `stream_id`, `error` | Le job a échoué ou a été annulé. |

## Notes d'ordonnancement

- Le stream SSE est ordonné par session ; les tokens arrivent dans l'ordre de
  génération.
- Un id de session unique ne peut être consommé que par un seul abonné SSE ;
  ouvrez le stream avant (ou immédiatement après) le RPC qui renvoie l'id.
- L'en-tête `x-session-id` sur `POST /api/rpc` attache la **réponse** RPC
  elle-même à un canal de session aussi — utilisé par les clients qui veulent
  la réponse reflétée sur le même stream.

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
