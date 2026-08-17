---
title: "Opérations"
description: "Endpoints de santé, tracing RUST_LOG, timeouts upstream, mapping d'erreurs et dépannage pour un arona-server en fonctionnement."
---

# Opérations

Cette page s'adresse aux opérateurs qui exécutent `arona-server serve`. Elle
couvre les endpoints de santé que vous sondez, les lignes de log qui valent la
peine d'être recherchées, le modèle de timeout appliqué aux backends upstream,
comment les échecs de backend sont mappés en erreurs HTTP, et les pièges
opérationnels qui font trébucher les gens. Le déploiement lui-même est couvert
dans le [guide de déploiement](./deployment.md).

## Matrice de santé

Les trois endpoints de santé sont non authentifiés et renvoient `200 OK`
chaque fois que le processus sert — il n'y a pas de distinction
liveness/readiness :

| Endpoint | Réponse |
| --- | --- |
| `/healthz`, `/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | le même corps détaillé que ci-dessus |
| `/api/health` | plana `HealthResponse` : `status`, `version` (`CARGO_PKG_VERSION`), `kind` (`Dev`), `uptime` (secondes), `network` (transport / region / asn), `build_hash` (`BUILD_HASH`), `engine_version` (`"0.1.0"`) |

`/healthz` et `/readyz` sont des alias du même handler, et `/v1/health` le
partage, donc les sondes de style Kubernetes et la route de santé compatible
OpenAI sont interchangeables. `/api/health` ajoute uptime, réseau et version de
moteur. Utilisez `/readyz` pour les load balancers et superviseurs ; utilisez
`/api/health` quand vous avez besoin de la charge utile plus riche.

## Journalisation

Le serveur journalise via `tracing`, filtré avec la variable standard
`RUST_LOG` (`RUST_LOG=info` est le réglage courant ; `RUST_LOG=debug` révèle le
trafic des sondes). Événements bons à connaître, en ordre approximatif de
fréquence :

| Ligne de log | Niveau | Ce qu'elle vous dit |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | Une par requête de chat, avec `key_prefix`, `model`, `stream` et `request_id` — la piste d'audit par requête la plus simple. |
| `request completed` | info | Journalisée par le helper `logging_middleware` après chaque réponse **non-streaming** `/v1/chat/completions` et `/v1/embeddings` : `method`, `path`, `status`, `latency_ms`, `trace_id`. (Le chat streaming journalise `chat completions SSE request` au début à la place.) |
| `usage recorded` / `usage persisted` | info | Une ligne d'usage a été enregistrée (en mémoire, avec tokens/coût) puis écrite dans la table `usage_records`. |
| `external probe: sending` / `external probe: returned` | debug | Une sonde de santé du `/v1/models` d'un backend external ; `matched` indique si la sonde s'est terminée dans le timeout de 2 s. |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | Une requête `/v1/*` refusée par la porte de facturation — le client a reçu 429 plus `Retry-After`. |
| `rpc billing gate rejected: monthly quota exceeded` | warn | La porte de quota côté RPC pour les méthodes authentifiées par JWT (fenêtre de tout l'utilisateur ; réponse d'erreur JSON-RPC). |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | Restauration au démarrage : backends enregistrés par l'admin et nœuds d'agents chargés depuis la base de données et rendus à nouveau routables. |
| `Shutdown signal received, draining connections…` | info | L'arrêt gracieux a commencé (SIGINT/SIGTERM). |

## Modèle de timeout

Les timeouts sont appliqués sur le client upstream utilisé pour les backends
external (`packages/core/src/backends/external.rs`) :

| Timeout | Valeur | S'applique à |
| --- | --- | --- |
| Connexion | 10 s | L'établissement de la connexion TCP/TLS upstream. |
| Lecture idle | 120 s par lecture | Chaque appel upstream ; chaque octet reçu réinitialise l'horloge, donc un stream sain mais lent n'est jamais coupé. |
| Total non-streaming | 600 s | Appels non-streaming de chat/embeddings — un upstream lent mais vivant ne peut pas tenir une requête indéfiniment. |
| Streaming (SSE) | aucun | Les appels streaming ne portent **aucune échéance globale** ; les longues générations sont légales et la détection de blocage repose sur le timeout de lecture idle. |
| Sonde de santé | 2 s | La sonde `/v1/models`. |

## Mapping d'erreurs

Les échecs de backend sont mappés en statuts HTTP dans les handlers de
chat/embeddings (`packages/core/src/gateway/server.rs`) :

| Condition | HTTP | `type` / `code` | Message |
| --- | --- | --- | --- |
| Statut upstream non-2xx (`UpstreamStatus`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| Échec de transport upstream (`RequestFailed`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | la chaîne d'erreur de transport |
| Toute autre erreur de backend | **500** | `server_error` / `backend_error` | la chaîne d'erreur |
| Aucun backend pour le modèle (`NoBackend`) | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| API key invalide (`Unauthorized`) | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| Limite de débit (`RateLimited`) | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

L'intention de conception : les appelants peuvent distinguer « votre provider a
rejeté ou échoué » (502) de « le gateway lui-même est cassé » (500). Chaque
corps d'erreur a la même forme de style OpenAI —
`{"error":{"message":...,"type":...,"code":...}}` (`json_error_response`). Les
429 de la porte de facturation portent en plus un en-tête `Retry-After` et
utilisent `quota_error`/`quota_exceeded` (quota) et
`rate_limit_error`/`rate_limit_exceeded` (limite de débit de tier)
respectivement.

## Dépannage

### Un backend fraîchement enregistré reste fail-closed jusqu'à la sonde

Les backends external démarrent dans un état de santé inconnu et rapportent
`"<url> not probed yet"`. Ils passent à healthy quand (a) le premier tour du
vérificateur de santé s'exécute — immédiatement au démarrage, puis toutes les
60 s — ou (b) la sonde fire-and-forget lancée à l'enregistrement ou à la
restauration réussit, normalement en ~1-2 secondes. Jusque-là, les requêtes
routées vers le backend échouent en fail-closed par conception.

### Un 404 sur le `/models` de la sonde est normal pour certains backends

La sonde external frappe `GET {base}/v1/models` (ou `{base}/models` pour les
URL de base avec préfixe de chemin). Certains serveurs compatibles OpenAI
implémentent le chat mais n'exposent aucune liste de modèles — l'endpoint du
plan de codage Zhipu GLM en est un. Un **404 est toléré** : le backend est
marqué healthy et la liste de modèles configurée par l'admin reste
l'autorité du routage. Seules les sondes réellement échouées (timeout, erreur
réseau, autre non-2xx) marquent le backend unhealthy.

### Les streams SSE qui ne produisent rien ne sont pas facturés

Une réponse streaming n'est enregistrée comme usage que quand le stream a
produit du texte **ou** porté un usage terminal ; un stream qui s'est terminé
sans l'un ni l'autre n'est pas enregistré du tout. Si vous voyez une requête
sans ligne `usage recorded` correspondante, vérifiez si le stream a réellement
produit du contenu.

### Rapport de version

`version` dans les corps de santé est `CARGO_PKG_VERSION` ; `build_hash` est la
valeur `BUILD_HASH` au moment de la compilation émise par
`packages/core/build.rs`. Comparez `build_hash` entre les nœuds pour confirmer
qu'ils exécutent tous le même artefact.

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
