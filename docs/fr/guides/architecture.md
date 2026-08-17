---
title: "Architecture"
description: "Comment Arona est assemblé — disposition de l'espace de travail, chemin de requête à travers le gateway, routage, sondes de santé, mémoire, sessions et les compromis de conception délibérés."
---

# Architecture

Cette page parcourt la structure d'Arona et le flux d'une requête à travers
lui : la disposition de l'espace de travail, le chemin de requête, le gateway
et le routeur, les vérifications de santé, le client mémoire, les sessions et
notifications, et enfin les limites et compromis délibérés que la conception
accepte. Voir le [guide de démarrage rapide](quickstart.md) pour un exemple en
fonctionnement et [opérations](operations.md) pour les préoccupations
d'exécution quotidiennes.

## Disposition de l'espace de travail

Le dépôt est un espace de travail Cargo avec trois packages :

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC,
│            # backends, migration, entities, providers, models, registry,
│            # orchestration
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

- `packages/core` est le crate bibliothèque (`_core`). Il contient tout ce dont
  le serveur a besoin : le gateway axum (`gateway/`), le routeur de modèles
  (`routing/`), les adaptateurs de backend (`backends/`), la facturation
  (`billing/`), l'auth (`auth.rs`), le client mémoire (`memory/`), le plan
  JSON-RPC (`gateway/rpc.rs`), le schéma (`migration/`, `entity/`), les
  métadonnées de modèles (`models/`, `providers/`, `registry/`) et
  l'orchestration de modèles (`orchestration/`).
- `packages/agent` construit le binaire `_agent` qui tourne sur les nœuds GPU et
  se reconnecte via `/ws/agent` (voir [cluster d'agents](agent-cluster.md)).
- `packages/cli` construit le binaire `_cli` utilisé pour les opérations
  install, deploy, serve, migrate et download.

Il n'y a plus de tableau de bord web dans ce dépôt : le tableau de bord Vue a
été supprimé et vit désormais dans
[shittim-chest](https://github.com/celestia-island/shittim-chest) (chest #291),
qui communique avec Arona via la surface JSON-RPC. Arona lui-même est un
backend pur (voir la [vue d'ensemble](./README.md)).

## Chemin de requête

Le point d'entrée est le routeur axum assemblé dans `GatewayServer::app`
(`packages/core/src/gateway/server.rs`). Sa table de routes (server.rs:128-163)
couvre la surface REST compatible OpenAI (`/v1/chat/completions`,
`/v1/embeddings`, `/v1/models`, `/v1/health`), la génération vidéo, l'endpoint
JSON-RPC `/api/rpc` (POST + mise à niveau WebSocket), le sidecar SSE
`/api/rpc/events`, le plan de contrôle d'agents `/ws/agent`, l'interface
Swagger à `/docs`, et les endpoints admin de gestion des backends/alias.

Le routeur est enveloppé dans une petite pile de couches (server.rs:158-162) :

1. Des gestionnaires d'auth comme `Extension`s pour que les extracteurs par
   handler puissent les atteindre.
2. Une couche d'id de requête qui réutilise un en-tête entrant `X-Request-ID`
   ou en génère un, l'exposant aux handlers et aux logs
   (`gateway/request_id.rs`).
3. Une limite de corps de requête de 1 Mo (`RequestBodyLimitLayer`).
4. Une couche CORS permissive (origine `*`, en-têtes `*`).

Parce qu'axum applique les couches de bas en haut, la couche CORS est la plus
externe.

Chaque handler `/v1/*` passe ensuite par le même squelette :

1. **Extraction d'auth** — `ApiKeyAuth` pour les endpoints réservés aux keys
   (`/v1/chat/completions`, `/v1/embeddings`, vidéo) et `ApiKeyOrJwt` pour
   `GET /v1/models`, qui doit accepter à la fois les API keys et les JWT de
   session (`gateway/middleware.rs`). L'extracteur résout la key/JWT en email
   d'utilisateur, préfixe de key, clé de limite de débit (le hash SHA-256 de
   l'API key, ou un libellé `u:<email>` pour les JWT pour que les tokens qui
   tournent ne réinitialisent pas la fenêtre) et un périmètre de projet
   optionnel.
2. **Portes de facturation** — `enforce_billing_gates` (server.rs:492-539)
   rejette la requête avec HTTP 429 + `Retry-After` quand le quota mensuel de
   tier ou la limite de débit par minute de l'utilisateur est dépassée. Les
   échecs de base de données fail-open : la facturation est best-effort, jamais
   une dépendance dure du service du chat.
3. **Rappel mémoire** (chemins de chat) — si le client mémoire est configuré et
   que la requête le demande, les mémoires à long terme pertinentes sont
   injectées comme section système (voir [Memory client](#memory-client)
   ci-dessous). Un échec ne bloque jamais le chat ; l'état résultant est reflété
   dans l'en-tête `X-Arona-Memory`.
4. **Persistance de conversation** — un `conversation_id` optionnel est
   vérifié pour la propriété, et le tour utilisateur est persisté à l'envoi.
5. **Dispatch du gateway** — la requête est remise au `Gateway`, qui résout un
   backend, rogne le contexte, et appelle le trait de backend.
6. **Enregistrement d'usage** — l'usage renvoyé (ou terminal de stream) est
   enregistré et persisté via le `UsageTracker` sous le préfixe de key.

Le `Gateway` lui-même vit dans `AppState` comme `Arc<Gateway>` — il n'y a pas
de mutex externe ; la mutabilité intérieure empêche les appels concurrents de
chat/embeddings/stream de jamais tenir un verrou à travers un aller-retour HTTP
upstream (`gateway/mod.rs:29-53`).

## Gateway et routeur

Le `Gateway` (`packages/core/src/gateway/mod.rs`) possède :

- **L'état du routeur** — la liste des backends et les alias, gardés par un
  `tokio::sync::RwLock`. La résolution côté lecture emprunte à travers les
  awaits ; les mutations (register/remove/alias) prennent un verrou
  d'écriture court et ne le tiennent jamais à travers un appel upstream.
- **Un compteur de requêtes** (`AtomicU64`) et un `start_time` utilisés par
  `system.status` et les endpoints de santé.
- **Une map des déploiements** (`model_id → backend name`) pour les modèles
  déployés par agents. `register_agent_backend` construit un
  `ExternalApiBackend` nommé `agent-{model_id}` et l'insère dans le routeur ;
  ré-enregistrer le même modèle remplace le backend précédent, et
  `unregister_agent_backend` le retire sur une frame `stop_result` (voir
  [cluster d'agents](agent-cluster.md)).

La résolution de backend a lieu dans le `Router` (`packages/core/src/routing/mod.rs`) :

1. **Résolution d'alias** — un alias configuré est réécrit vers sa cible.
2. **Affinité de session** — quand un `conversation_id` est présent, le routeur
   vérifie une map de références faibles qui épingle la conversation au backend
   qui l'a servie en premier. Les références faibles gardent la map vivante
   uniquement tant que le backend est enregistré ou en vol, donc les backends
   supprimés disparaissent sans dérive d'index. Un circuit breaker déclenché ou
   un backend épinglé unhealthy dégrade vers une nouvelle sélection, qui
   ré-épingle la conversation.
3. **Filtrage des candidats** — un indice `provider` optionnel filtre par
   nom/type de backend ; les candidats doivent être healthy *et* avoir un
   circuit breaker ouvert, et doivent lister le modèle demandé. Les ids de
   modèles correspondent exactement ou via la convention de suffixe `:latest`
   (une requête nue `nomic-embed-text` correspond à un `nomic-embed-text:latest`
   listé).
4. **Choix le moins chargé** — les candidats survivants sont triés par leur
   compteur de hits et le moins chargé est choisi ; l'épingle de conversation
   (le cas échéant) est enregistrée en même temps.

Avant que le backend ne soit appelé, `RequestPipeline::transform`
(`packages/core/src/pipeline.rs:422+`) rogne la liste de messages à la
`max_context_length` du backend : les messages système sont conservés en
entier, les messages non système sont conservés du plus récent au plus ancien
tant qu'ils tiennent, et un seul message surdimensionné est tronqué durement
par caractères (le compteur de tokens heuristique ne peut pas tronquer avec une
précision de token). L'appel passe ensuite par le trait `InferenceBackend` ;
les succès et échecs sont enregistrés dans le circuit breaker par backend du
routeur (3 échecs, récupération 30 s, 1 appel half-open — routing/mod.rs:57-64).

## Vérificateur de santé et sondes

`run_health_checks` (`packages/core/src/gateway/health_checker.rs`) s'exécute
comme tâche d'arrière-plan lancée au démarrage (run.rs:135-150) et sonde chaque
backend enregistré une fois par intervalle de 60 secondes. Deux détails
comptent :

- La liste des backends est **récupérée à nouveau à chaque tour** via une
  closure de récupération asynchrone, donc les backends enregistrés après le
  démarrage (p. ex. via l'API admin) sont pris en compte sans redémarrage.
- Le premier tour s'exécute immédiatement, avant que le premier intervalle ne
  s'écoule, donc l'état de santé est établi dès que le processus démarre.

`probe_backend` est le chemin de code de sonde unique. Il est réutilisé par les
**sondes au moment de l'enregistrement** ponctuelles : après qu'un admin
enregistre un backend (server.rs:688-693) ou qu'un backend persisté est
restauré au boot (run.rs:122-127), une sonde fire-and-forget fait passer le
backend à healthy en ~1–2 s au lieu de rester fail-closed jusqu'au prochain
tour de 60 s. C'est ce qui fait apparaître la liste de modèles d'un backend
external fraîchement enregistré dans `GET /v1/models` presque immédiatement.

La sonde elle-même est un appel de backend léger (p. ex. le backend external
frappe `/v1/models` avec un timeout de sonde de 2 s) ; le résultat est mis en
cache dans le backend et le routage ne sélectionne jamais que les backends dont
la santé en cache est `Healthy` (plus un circuit breaker ouvert).

## Client mémoire

Le client mémoire (`packages/core/src/memory/mod.rs`) est construit depuis la
configuration d'environnement au démarrage du serveur (server.rs:95-97) : quand
`ARONA_MEMORY_URL` et `ARONA_MEMORY_TOKEN` sont définies, les requêtes de chat
interrogent le service mémoire entelecheia Philia via un WebSocket JSON-RPC et
`recall_and_inject` préfixe les mémoires pertinentes comme section système
(`## Relevant Long-Term Memories`) dans le contexte sortant. Les tours terminés
sont réécrits comme épisodes via `writeback_dialogue` — du travail
fire-and-forget lancé après que la réponse d'assistant est persistée, donc les
échecs de mémoire ne bloquent ni ne ralentissent jamais le chemin de réponse
du chat. `ARONA_MEMORY_WRITEBACK` (défaut activé) bascule le writeback. Voir
[gateway mémoire](memory-gateway.md) pour le tableau complet.

Chaque réponse de chat porte un en-tête `X-Arona-Memory` avec l'un des trois
états : `enabled` (le rappel a tourné et injecté), `disabled` (non configuré ou
la requête a passé `memory: false`), ou `offline` (configuré mais le service
était injoignable).

## Sessions et notifications

`AppState` détient un `plana` `SessionManager` (`state.sessions`). Les RPC
streaming tels que `chat.send` créent un id de session (`gateway/rpc.rs:1701`)
et poussent les notifications — tokens `chat.stream`, progression de
déploiement `models.progress`, `realtime.event` — sur le canal de cette
session. Les clients les consomment depuis le sidecar SSE
`GET /api/rpc/events?session=<id>` (server.rs:191-200) ; voir
[événements](../api/events.md) pour le format de notification et la mise en
garde de fenêtre de pré-abonnement.

Le canal de session est aussi utilisé pour les appels RPC requête/réponse :
quand un client envoie un en-tête `x-session-id` sur `POST /api/rpc`, le
serveur pousse aussi le résultat sur ce canal de session (server.rs:184-188,
rpc.rs:128-144), donc un client peut multiplexer une réponse RPC sur un stream
SSE déjà ouvert.

## Limites et compromis de conception

La conception accepte délibérément un certain nombre de limites ; connaissez-les
avant l'utilisation en production :

- **Limite de corps de requête de 1 Mo** — les corps plus grands sont rejetés
  par la couche ; si vous avez besoin d'appels à grand contexte, c'est la
  première chose à régler.
- **CORS `*`** — le gateway répond aux appels cross-origin de n'importe où.
  Bien pour une API, mais si vous l'exposez au-delà de clients de confiance,
  placez devant un proxy qui applique votre propre politique CORS.
- **Facturation fail-open** — l'application des quotas/limites de débit dégrade
  en autorisant la requête quand la base de données est indisponible. La
  facturation est de la mesure, pas du contrôle d'accès.
- **Pas de timeout global sur les streams SSE** — les appels streaming ne
  portent aucune échéance totale (les longues générations sont légales) ; la
  détection de blocage repose sur un timeout idle de 120 s par lecture
  (`backends/external.rs:24-31`). Les appels non-streaming reçoivent une
  échéance globale de 600 s.
- **Usage estimé par tokenizer** — les backends qui ne rapportent jamais
  l'usage (ollama, ws_engine) sont facturés depuis une estimation locale par
  tokenizer conscient du CJK, enregistrée telle quelle (voir
  [facturation-usage](billing-usage.md)).
- **Fenêtres de limite de débit et révocation en mémoire** — la fenêtre
  glissante par key et l'ensemble des keys révoquées vivent dans la mémoire du
  processus (`auth.rs`), donc un redémarrage les réinitialise. Le limiteur au
  niveau auth borne les requêtes par key par fenêtre ; le limiteur de tier est
  adossé à la base de données (voir [auth-sécurité](auth-security.md) et
  [facturation-usage](billing-usage.md)).
- **`/ws/agent` est non authentifié** — le plan de contrôle d'agents accepte
  tout WebSocket qui parle le protocole register/heartbeat. Il n'est sûr que
  sur un réseau que vous contrôlez.
- **Pas de TLS au gateway** — le serveur écoute en HTTP simple ; terminez le
  TLS devant (reverse proxy) dans tout déploiement qui traverse une frontière
  réseau. Voir [déploiement](deployment.md).

Côté gracieux, le serveur effectue un arrêt gracieux : il installe des handlers
Ctrl+C et SIGTERM, journalise « draining connections », et laisse les requêtes
en vol se terminer avant la sortie du processus (`gateway/run.rs:14-38`, et le
câblage `with_graceful_shutdown` à run.rs:154-159).

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table); gateway/mod.rs:29-53; routing/mod.rs:102-200; health_checker.rs:14-37; run.rs:135-159; memory/mod.rs:207-241; rpc.rs:128-144,1701 -->
