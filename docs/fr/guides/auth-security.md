---
title: "Authentification et sécurité"
description: "Sessions JWT, API keys, les trois portes admin, politique de mot de passe, limitation de débit à double voie et le modèle de sécurité."
---

# Authentification et sécurité

Arona authentifie les appelants sur deux voies : **les tokens de session JWT**
pour les clients interactifs (l'interface de chat + administration, les appels
RPC) et **les API keys** (`arona-…`) pour le trafic programmatique compatible
OpenAI. Un token admin séparé garde les surfaces administratives. Cette page
documente la mécanique, le modèle de sécurité, et les reliquats connus à faible
risque d'un audit de sécurité.

## Sessions JWT

Les sessions utilisent des paires de tokens JWT access/refresh émises par le
gestionnaire de tokens `kirino_session` :

- **TTL du token access : 900 secondes (15 minutes).**
- **TTL du token refresh : 604 800 secondes (7 jours).**

Les tokens access authentifient le plan JSON-RPC (`/api/rpc`) et
`GET /v1/models` ; le sidecar SSE (`/api/rpc/events`) est indexé par son id de
session, une capacité frappée lors des appels RPC authentifiés plutôt qu'une
identité bearer. Les endpoints `/v1/chat/completions`, `/v1/embeddings` et
`/v1/video/*` requièrent une **API key** (un JWT n'y est pas accepté).
Les tokens access sont de courte durée pour qu'un token volé ne soit
utilisable que brièvement. Les tokens refresh sont échangés contre de nouvelles
paires via `auth.refresh`.

Le refresh utilise la **rotation de famille de tokens** : consommer un token
refresh l'invalide et en émet un nouveau, et réutiliser un token refresh
consommé révoque toute la famille — `auth.refresh` répond avec `AUTH_ERROR` et
le message `Refresh token reused` (l'erreur sous-jacente est `TokenReused`,
« refresh token has been reused — token family revoked »), et le compte doit se
reconnecter. La révocation de famille est **en mémoire** (un ensemble
`revoked_families`) : un redémarrage du serveur la vide, donc la protection est
best-effort à travers les redémarrages (l'état de session par utilisateur ne
survit pas à un redémarrage).

Le secret de signature vient de la variable d'environnement `JWT_SECRET`. Hors
`MOCK_MODE=1`, le serveur **refuse de démarrer** si `JWT_SECRET` est non défini
ou vaut encore le secret de développement intégré, donc une instance de
production ne peut jamais servir par accident des tokens signés avec une
constante publique. Utilisez un secret fort et aléatoire et ne le committez
jamais.

## API keys

Les API keys sont l'identité machine pour la surface compatible OpenAI :

- **Format :** `arona-{uuid}`.
- **Stockage :** seul le **hash SHA-256** de la key est stocké dans la table
  `api_keys` — le texte en clair n'est renvoyé qu'une seule fois, dans la
  réponse de `keys.create`, et ne peut jamais être récupéré plus tard.
- **Préfixe de key :** les 8 premiers caractères (`key_prefix`) sont stockés en
  clair pour l'affichage et l'attribution d'usage ; l'interface affiche une
  forme masquée telle que `arona-XXXX…abcd`.
- **Révocation :** la recherche de key joint sur `api_keys.is_active = TRUE`,
  donc une key révoquée cesse immédiatement de valider — aucun TTL de cache à
  attendre.

## Niveaux admin

Il y a trois portes admin distinctes, chacune avec son identifiant :

1. **Les routes `/api/admin/*`** — la gestion des backends et alias
   (`POST/GET/DELETE /api/admin/backends`, `POST/GET/DELETE /api/admin/aliases`)
   requiert l'en-tête `Authorization: Bearer ARONA_ADMIN_TOKEN`. Quand
   `ARONA_ADMIN_TOKEN` est non défini, `check_admin` échoue toujours et chaque
   route admin renvoie **401 « Admin access required »** — toute la surface de
   gestion est désactivée plutôt qu'ouverte.

2. **Les méthodes RPC `agents.*` et `engine.invoke`** — le cluster d'agents et
   le plan de contrôle des moteurs requièrent un JWT dont le compte a
   `users.is_admin = true`. Un non-admin authentifié est rejeté avec le code
   défini par l'implémentation **-32007 (`ADMIN_REQUIRED`)** plus un indice
   spécifique à la méthode (p. ex. `agents.deploy starts model deployments on GPU nodes`) ;
   un appelant **non authentifié** obtient le standard **-32005
   (`AUTH_ERROR`)** pour que le serveur ne révèle pas que la méthode est
   privilégiée.

3. **Les méthodes RPC `billing.plan.set` et `billing.video.pricing.set`** —
   les mutations de facturation requièrent le même Bearer `ARONA_ADMIN_TOKEN`
   que les routes HTTP admin ; sans lui, elles renvoient `AUTH_ERROR` « Admin
   access required ».

Le **premier utilisateur enregistré devient l'admin** (`users.is_admin = true`).
Chaque inscription ultérieure est un utilisateur ordinaire, et
l'inscription n'est ouverte que tant que `ARONA_REGISTRATION_OPEN` est défini
sur une valeur truthy.

## Politique de mot de passe

Les mots de passe doivent satisfaire **les deux** règles (imposées à
l'inscription et sur tout chemin de changement de mot de passe) :

- au moins **8 caractères**, et
- au moins **3 des 4 catégories de caractères** : majuscules, minuscules,
  chiffres, spéciaux.

## Limitation de débit

La limitation de débit fonctionne sur deux voies indépendantes ; chacune peut
rejeter une requête avec **429** :

### 1. Fenêtre glissante en mémoire (par identité)

Chaque requête `/v1` authentifiée passe par un limiteur en mémoire à fenêtre
glissante indexé par l'identité de l'appelant :

- **Les appels par API key** sont indexés par le **hash SHA-256** de la key ;
- **Les appels par JWT** sont indexés par `u:<email>` — les JWT tournent toutes
  les 15 minutes, donc indexer la fenêtre par l'instance de token la
  réinitialiserait silencieusement à chaque refresh.

Le budget par défaut est de **60 requêtes par minute**, remplaçable avec
`ARONA_API_RATE_LIMIT_RPM` (à augmenter pour les pipelines d'agents qui
déploient beaucoup d'appels LLM parallèles). Le définir à **0 bloque chaque
requête**.

### 2. Limite de débit de tier (par key, depuis la base de données)

Les tiers de facturation portent un `rate_limit_rpm` par key. Le contrôle
compte les lignes `usage_records` pour le préfixe de la key dans les
**60 dernières secondes** (l'usage est persisté après chaque réponse, donc la
fenêtre retarde d'au plus une requête en vol ; les échecs de base de données
fail-open). Le **tier free seedé est à 10 RPM** ; les tiers pro/enterprise
relèvent le plafond. L'application du quota mensuel partage le même chemin de
rejet.

### Limitation de débit de connexion

La devinette d'identifiants est étranglée sur l'endpoint de connexion :
**5 tentatives échouées par fenêtre de 5 minutes par email** et **20 par
fenêtre de 5 minutes par IP**, chacune suivie d'un verrouillage de
15 minutes.

### `Retry-After`

Chaque réponse 429 porte un en-tête `Retry-After` pour que les clients
compatibles OpenAI se retirent au lieu de marteler l'endpoint : les rejets de
quota le règlent sur **les secondes jusqu'à la fin du mois** ; les rejets de
limite de débit le règlent sur **60**. Voir [Facturation et usage](billing-usage.md)
pour le modèle de quota.

## Notes sur le modèle de sécurité

- **CORS autorise toute origine** (`allow_origin(Any)`) — Arona est un backend
  consommé par de nombreux clients first-party et third-party ; si votre
  déploiement doit restreindre les origines, placez devant lui un reverse proxy
  qui applique CORS.
- **Les corps de requête sont limités à 1 Mo** (`RequestBodyLimitLayer`),
  bornant l'utilisation mémoire du gateway.
- **Le gateway ne termine pas de TLS** — il écoute en HTTP simple. Placez-le
  derrière un reverse proxy (voir [Déploiement](deployment.md)) qui termine le
  HTTPS.
- **Les secrets ne viennent que de l'environnement** : `ARONA_ADMIN_TOKEN` et
  `JWT_SECRET` sont lus depuis les variables d'environnement, et doivent être
  des valeurs aléatoires fortes jamais commitées dans le dépôt.
- L'adresse d'écoute par défaut du serveur est `0.0.0.0` ; restreignez
  l'exposition au niveau réseau.

## Reliquats connus à faible risque (de l'audit)

Les points suivants sont documentés tels quels ; ils sont intentionnels ou
acceptés pour l'instant, mais bons à connaître quand vous exposez une instance
au-delà d'un réseau de confiance :

- **`providers.list` est public**, tandis que `providers.add` /
  `providers.update` / `providers.remove` / `providers.test` requièrent un JWT.
  Le chemin de lecture public révèle le catalogue de providers mais rien de
  secret.
- **`/ws/agent` est un plan de contrôle non authentifié** : les agents GPU se
  connectent sans identifiant et s'auto-enregistrent (frames `register` /
  `heartbeat` / résultats de commandes). Quiconque peut atteindre le port
  WebSocket peut enregistrer un faux agent. Voir
  [Cluster d'agents](agent-cluster.md) pour les compromis opérationnels.
- **`memory.delete` est réservé au JWT sans contrôle de propriété** : tout
  utilisateur authentifié peut supprimer un nœud mémoire par `node_id`.
  Supprimer de la mémoire requiert d'être connecté, mais pas de posséder le
  nœud.

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
