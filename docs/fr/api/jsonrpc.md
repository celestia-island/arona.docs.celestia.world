---
title: "Référence de l'API JSON-RPC"
description: "API JSON-RPC 2.0 du plan de gestion Arona à /api/rpc — méthodes chat, realtime, engine, auth, keys, providers, agents, memory, conversations, usage, billing, video et system sur HTTP et WebSocket."
---

# Référence de l'API JSON-RPC

Arona expose une surface JSON-RPC 2.0 à `/api/rpc` pour le plan de gestion :
auth, keys, providers, agents, memory, conversations, usage, billing, video,
realtime et chat streaming. Elle complète la surface REST compatible OpenAI
(`/v1/*`, voir [API REST compatible OpenAI](./openai-rest.md)) ; utilisez REST
pour les charges de travail d'inférence authentifiées par key et JSON-RPC pour
la gestion de session/compte et le contrôle du streaming. Le
[guide de démarrage rapide](../guides/quickstart.md) parcourt le premier tour
de bout en bout.

La surface dispatch **39 méthodes de requête** plus une méthode de liveness
anonyme réservée au WebSocket, `system.probe` (40 méthodes au total). Chaque
requête est un objet JSON-RPC 2.0 avec `jsonrpc: "2.0"`, une chaîne `method`,
un objet `params` optionnel et un `id` optionnel.

## Transport

- **HTTP POST `/api/rpc`** — requête/réponse. Envoyez `Content-Type:
  application/json`. Le JWT voyage dans l'en-tête `Authorization: Bearer <jwt>`.
  Le corps de requête est plafonné à 1 Mio.
- **WebSocket `GET /api/rpc`** — connexion longue durée. Les navigateurs ne
  peuvent pas définir d'en-têtes personnalisés sur la mise à niveau WebSocket,
  donc le JWT voyage comme paramètre de requête `?token=<jwt>` ; le serveur le
  replie dans un en-tête `Authorization: Bearer` en interne (voir
  `packages/core/src/gateway/server.rs`). Les sockets authentifiés peuvent
  rester connectés indéfiniment.
- **Requêtes par lots** — un corps POST qui est un tableau JSON est exécuté
  élément par élément et reçoit une réponse en tableau JSON dans le même ordre.
- **Accès anonyme** — sur WebSocket sans JWT, les méthodes publiques
  (`auth.register`/`auth.login`/`auth.refresh`, `providers.list`,
  `system.status`) restent appelables, et `system.probe` reçoit un unique ack
  avant la fermeture du socket. Toute autre méthode requiert un JWT valide ;
  les méthodes gardées par admin requièrent en plus un compte admin (voir la
  légende ci-dessous). Les sockets anonymes sont aussi soumis à un timeout
  d'inactivité de 10 secondes.
- **Attachement de session** — un en-tête `x-session-id` sur `POST /api/rpc`
  pousse en plus la réponse RPC elle-même sur ce canal de session, aux côtés
  des notifications de streaming.

## Ids

Les valeurs `id` de requête sont reflétées avec fidélité de type : `null` →
`null`, chaînes → chaînes, entiers → nombres, et tout le reste (flottants,
objets, entiers hors plage i64) → le rendu en chaîne JSON. Un `id` omis reçoit
une réponse `null`.

## Notifications serveur → client (sidecar SSE)

Les tokens, la progression de déploiement et les événements realtime ne sont
**pas** délivrés sur le socket WebSocket. Chaque RPC streaming crée un id de
session et pousse les notifications vers
`GET /api/rpc/events?session=<session_id>` comme server-sent events.
Abonnez-vous à l'endpoint SSE **avant ou immédiatement après** que l'appel RPC
renvoie un id de session — les notifications émises entre le retour de l'appel
et l'établissement de l'abonnement SSE sont perdues (la fenêtre de
pré-abonnement). Le schéma recommandé est d'ouvrir d'abord le stream SSE, puis
de déclencher le RPC.

Méthodes de notification : `chat.stream` (un token par événement depuis
`chat.send`), `models.progress` (progression de téléchargement de modèles
d'agents depuis `agents.deploy`), `realtime.event` (événements serveur pour
une session realtime ouverte), et `video.progress` / `video.done` /
`video.failed` (jobs vidéo asynchrones). Voir le catalogue complet dans
[Événements et notifications](./events.md).

## Codes d'erreur

| Code | Nom | Signification |
| --- | --- | --- |
| `-32700` | Parse error | Le corps de requête n'est pas du JSON valide. |
| `-32600` | Invalid request | L'objet de requête est malformé, p. ex. un `method` manquant. |
| `-32601` | Method not found | Chaîne `method` inconnue ; le message la reflète. |
| `-32602` | Invalid params | `params` a échoué à la désérialisation pour la méthode. |
| `-32603` | Internal error | Échec serveur inattendu. |
| `-32000` | `APP_ERROR` | Erreur d'application générique — p. ex. conversation/provider/agent introuvable, aucun agent en ligne disponible pour le déploiement. |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` — JWT manquant ou invalide. Aussi utilisé par les méthodes à token admin quand le bearer token ne correspond pas à `ARONA_ADMIN_TOKEN` (`"Admin access required"`). |
| `-32006` | `QUOTA_ERROR` | Quota mensuel de facturation dépassé pour une méthode RPC gardée par JWT (`chat.send`). |
| `-32007` | `ADMIN_REQUIRED` | Un **non-admin** authentifié appelle une méthode gardée par admin (`agents.*`, `engine.invoke`) ; le message inclut un indice spécifique à la méthode. |

> Les méthodes `agents.*` et `engine.invoke` sont réservées à l'admin : elles
> requièrent un JWT dont le compte a `users.is_admin = true`. Un non-admin
> authentifié est rejeté avec `-32007` (`ADMIN_REQUIRED`) ; un appelant non
> authentifié obtient le standard `AUTH_ERROR` pour que le serveur ne révèle
> pas que la méthode est privilégiée.

## Légende d'auth

| Légende | Identifiants |
| --- | --- |
| **public** | Aucun identifiant requis. |
| **JWT** | `Authorization: Bearer <jwt>` sur HTTP, ou `?token=<jwt>` sur WebSocket. |
| **admin (JWT + is_admin)** | JWT bearer d'un compte avec `users.is_admin = true`. |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN` (configuré par env ; quand non défini, la méthode est toujours refusée, défaut-refus). |

Tous les identifiants et adresses d'exemple de ce document sont des espaces
réservés (IP de documentation RFC 5737, keys `sk-xxx`). Voir
[Authentification et sécurité](../guides/auth-security.md) pour le modèle
d'auth complet derrière cette légende.

## Chat

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model` (string), `messages` (tableau de `{ role, content, images?, tool_calls? }`), `temperature?` (number), `max_tokens?` (integer), `conversation_id?` (string), `memory?` (bool), `extra?` (object), `tools?` (tableau de définitions de fonctions de style OpenAI), `provider?` (string) | Envoie un tour de chat streaming. Renvoie `{ "stream_id", "memory" }` — `memory` est l'état de rappel (`enabled` / `disabled` / `offline`) ; les tokens arrivent comme notifications `chat.stream` sur le sidecar SSE. Avec un `conversation_id`, l'historique persisté terminé est assemblé côté serveur et le tour est persisté. Gardé par facturation (quota mensuel → `-32006`) ; l'usage est enregistré sous `jwt-<user-uuid>`. |

## Realtime (sessions audio/vidéo full-duplex)

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model` (string), `config?` (objet de config de session), `conversation_id?` (string) | Ouvre une session full-duplex contre le backend servant `model`. Renvoie `{ "session_id", "stream_session" }` : utilisez `session_id` pour `realtime.event` / `realtime.stop`, et abonnez-vous à `stream_session` sur le sidecar SSE pour recevoir les notifications `realtime.event`. |
| `realtime.event` | JWT | `session_id` (string), `event` (événement client — append/commit/clear audio, frame image, response create/cancel, session stop) | Envoie un événement client dans une session ouverte ; il est transmis au backend upstream. Renvoie `{ "ok": true }`. |
| `realtime.stop` | JWT | `session_id` (string) | Ferme et supprime une session. Renvoie `{ "removed": bool }`. |

## Engine (canal générique de perception/contrôle)

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `engine.invoke` | admin (JWT + is_admin) | `model` (string), `method` (string), `params?` (object) | Invocation synchrone requête/réponse d'une méthode de moteur arbitraire sur le backend servant `model` — le canal haute fréquence pour les appels de style `sensor.ingest` / `control.setpoint` (boucles de 20–30 Hz). Le résultat est la réponse brute du backend. |

## Auth

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `auth.register` | public | `email`, `password`, `name?` | Enregistre un compte. Uniquement autorisé tant que l'inscription est ouverte (`ARONA_REGISTRATION_OPEN`) ; le premier utilisateur enregistré devient l'admin. Renvoie la même réponse de token que `auth.login` (`access_token`, `refresh_token`, `token_type`, `expires_in`, `user`). |
| `auth.login` | public | `email`, `password` | Se connecter. Renvoie `access_token`, `refresh_token`, `token_type`, `expires_in`, `user` (`{ id, email, name, is_admin }`). Limité en débit par IP et par compte. |
| `auth.refresh` | public | `refresh_token` | Échange un token refresh contre un nouveau token access (et un nouveau token refresh). Les tokens refresh réutilisés ou expirés sont rejetés avec `AUTH_ERROR`. |
| `auth.me` | JWT | — | Profil de l'utilisateur courant : `{ "id", "email", "name" }`. |

## Keys

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | Liste les API keys de l'appelant (id, name, `key_prefix`, project, horodatages, drapeau actif). |
| `keys.create` | JWT | `name`, `project?` | Crée une API key. Renvoie `{ id, name, key, key_prefix, project, created_at }` — le secret complet `arona-<uuid>` dans `key` est montré **une fois** ; stockez-le immédiatement. |
| `keys.revoke` | JWT | `key_id` | Révoque une API key. Renvoie `{ "ok": true }`. |

## Providers

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | Liste les providers connus : entrées officielles intégrées plus les entrées personnalisées, comme métadonnées d'affichage (`id`, `name`, `description`, `website_domain`, `is_official`, `is_operator`). Public par conception — la liste ne porte aucun identifiant ; seules les mutations ci-dessous sont gardées par JWT. |
| `providers.add` | JWT | `id`, `name`, `description?`, `website_domain?` | Ajoute une entrée de provider personnalisée. Renvoie `{ "ok": true }`. |
| `providers.update` | JWT | `provider_id`, `name?`, `description?`, `website_domain?` | Met à jour les champs d'un provider personnalisé (uniquement ceux fournis). Renvoie `{ "ok": true }`. |
| `providers.remove` | JWT | `provider_id` | Supprime un provider personnalisé. Renvoie `{ "ok": true }`. |
| `providers.test` | JWT | — | Teste une connexion de provider. Stub : renvoie `{ "ok": true, "message": "Provider connection test not yet implemented" }`. |

## Agents

Toutes les méthodes `agents.*` sont réservées à l'admin (JWT + `is_admin`).
Les nœuds d'agents se connectent en sortie via `GET /ws/agent` ; ce groupe RPC
contrôle le registre (voir [Cluster d'agents](../guides/agent-cluster.md)).

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `agents.list` | admin (JWT + is_admin) | — | Liste les nœuds d'agents enregistrés : id, name, host, statut `online`/`offline` (basé sur heartbeat), résumé GPU, modèles déployés, version, horodatages. |
| `agents.register` | admin (JWT + is_admin) | `machine_name`, `version` | Enregistre un nœud d'agent auprès du gestionnaire de tunnel. Renvoie `{ "agent_id", "token" }` (le token est l'identifiant du plan de contrôle de l'agent). |
| `agents.deregister` | admin (JWT + is_admin) | `agent_id` | Désenregistre (déconnecte) un agent. Renvoie `{ "ok": true }`. |
| `agents.status` | admin (JWT + is_admin) | `agent_id` | Statut par agent : drapeau online, host, résumé GPU, modèles chargés, utilisation GPU, horodatages heartbeat/connexion. |
| `agents.deploy` | admin (JWT + is_admin) | `model_id`, `agent_id?` (vide/absent = nœud le moins chargé ; erreur si aucun en ligne) | Déploie un modèle sur un agent. Renvoie `{ "ok": true, "stream_id" }` — abonnez-vous à `stream_id` sur le sidecar SSE pour les notifications de téléchargement `models.progress`. |
| `agents.stop` | admin (JWT + is_admin) | `agent_id`, `model_id` | Arrête un modèle déployé. Renvoie `{ "ok": true, "stream_id": null }` (pas de stream de progression). |

## Memory

La mémoire à long terme est servie par le service entelecheia Philia sur un
WebSocket ; les échecs ne bloquent jamais le chat (voir
[Gateway mémoire](../guides/memory-gateway.md)).

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | État du gateway mémoire : `{ "enabled", "writeback", "events" }` — drapeaux plus jusqu'à 50 événements d'activité récents (les plus récents d'abord). |
| `memory.delete` | JWT | `node_id` | Supprime un nœud mémoire stocké. Renvoie `{ "deleted": bool }`. |

## Conversations

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | Liste les conversations de l'appelant avec horodatages d'âge relatif. |
| `conversations.create` | JWT | `title?` (défaut `New Conversation`) | Crée une conversation. Renvoie le nouvel objet de conversation. |
| `conversations.get` | JWT | `conversation_id` (alias hérité : `id`) | Récupère une conversation avec ses messages. Vérifié pour la propriété ; l'accès inter-utilisateurs est rejeté. |
| `conversations.delete` | JWT | `conversation_id` (alias hérité : `id`) | Supprime une conversation (propriétaire uniquement). Renvoie `{ "ok": true }`. |

> `conversations.get` / `conversations.delete` acceptent aussi la clé `id`
> héritée des anciens clients du tableau de bord ; `conversation_id` gagne
> quand les deux sont présents.

## Usage

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?` (integer, défaut 50, borné à 1–200), `offset?` (integer, défaut 0), `project?` (string) | Enregistrements d'usage paginés pour l'appelant, les plus récents d'abord, couvrant à la fois les lignes par API key (préfixe `arona-XX`) et les lignes attribuées par JWT (`jwt-<user-uuid>`). Renvoie `{ "records", "total", "limit", "offset", "project" }` ; le filtre `project` réduit aux seules lignes marquées par key. |

## Billing

Les tiers, quotas et la comptabilité d'usage sont décrits dans
[Facturation et usage](../guides/billing-usage.md).

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | État de facturation courant : `{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` — usage mensuel (`cost_usd`, tokens, nombre de requêtes) et quota restant. |
| `billing.plan.set` | admin token | `user_email`, `tier` | Définit le tier de facturation d'un utilisateur. Renvoie `{ "ok": true }`. Refusé avec `AUTH_ERROR` quand le bearer ne correspond pas à `ARONA_ADMIN_TOKEN`. |
| `billing.video.pricing.get` | JWT | — | Table de tarification vidéo. Renvoie `{ "pricing": [...] }`. |
| `billing.video.pricing.set` | admin token | `model`, `mode?` (défaut `per_second_resolution`), `base_price?` (number, défaut 0), `price_per_second?` (number, défaut 0), `price_per_frame?` (number, défaut 0), `resolution_coeff?` (object), `currency?` (défaut `USD`), `enabled?` (bool, défaut `true`) | Upsert de la tarification vidéo pour un modèle. Renvoie `{ "ok": true }`. Refusé avec `AUTH_ERROR` quand le bearer ne correspond pas à `ARONA_ADMIN_TOKEN`. |

## Video

Jobs asynchrones de génération vidéo (voir
[Realtime et vidéo](../guides/realtime-video.md)). La progression des jobs est
poussée comme notifications `video.progress` / `video.done` / `video.failed`
sur le canal de session.

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`, `prompt`, `negative_prompt?`, `images?` (tableau de `{ data_base64, mime_type }`), `duration_seconds?` (integer), `width?` (integer), `height?` (integer), `provider?` (string), `extra?` (object) | Soumet un job asynchrone de génération vidéo. Renvoie `{ "job_id", "stream_id" }` — abonnez-vous à `stream_id` pour les notifications de progression. |
| `video.get` | JWT | `job_id` (UUID) | Sonde le statut/résultat d'un job (status, progress, result, error, cost). |
| `video.list` | JWT | `limit?` (integer, défaut 20) | Liste les jobs de l'appelant. Renvoie `{ "jobs": [...] }`. |
| `video.cancel` | JWT | `job_id` (UUID) | Annule un job en cours. Renvoie `{ "ok": true }`. |

## System

| Méthode | Auth | Params | Description |
| --- | --- | --- | --- |
| `system.status` | public | — | Statut agrégé du gateway : `{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`. |
| `system.probe` | anonyme (WS uniquement) | — | Sonde de liveness ponctuelle sur le transport WebSocket. Le serveur accuse `{ "ok": true, "status": "ok" }` puis ferme le socket — les visiteurs anonymes ne tiennent jamais de connexion ouverte. Toute autre méthode sur un socket non authentifié est rejetée avec `AUTH_ERROR`. |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
