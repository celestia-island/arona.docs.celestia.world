---
title: "Guide de démarrage rapide"
description: "Parcours Arona de bout en bout avec le mock upstream intégré : migrer, servir, enregistrer un backend, créer une API key et chatter."
---

# Guide de démarrage rapide

Ce guide vous fait parcourir une configuration Arona complète de bout en bout
sur une seule machine en utilisant le **mock upstream intégré** — aucun poids
de modèle réel, aucun GPU ni compte API externe requis. À la fin, vous aurez :

- un gateway Arona en fonctionnement (`/v1/*` API REST compatible OpenAI plus
  le plan de gestion JSON-RPC `/api/rpc`),
- le mock upstream enregistré comme backend `external`,
- un compte utilisateur et une API key,
- un tour de chat non-streaming **et** streaming fonctionnel contre le mock,
- des enregistrements d'usage visibles via `usage.list`.

## Prérequis

- **Chaîne d'outils Rust** (voir `rust-toolchain.toml` à la racine du dépôt).
- **Python 3** avec `aiohttp` — nécessaire uniquement pour le mock upstream
  (`pip install aiohttp`).
- Une instance **PostgreSQL en cours d'exécution** et son URL de connexion.

## 1. Configurer l'environnement

Arona lit sa configuration à partir des variables d'environnement **au
démarrage du processus**. Deux sont obligatoires : `DATABASE_URL` et
`JWT_SECRET` — le serveur refuse de démarrer sans elles (sauf si
`MOCK_MODE=1`). `ARONA_ADMIN_TOKEN` est fortement recommandé : sans lui,
chaque route `/api/admin/*` renvoie 401.

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

Ces variables sont lues une seule fois au démarrage du processus — si vous les
modifiez, redémarrez le serveur. Voir [Configuration](configuration.md) pour la
référence complète des variables.

## 2. Migrer et démarrer le serveur

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

`serve` seul suffit pour une base de données fraîche : il auto-migre au
démarrage. Le serveur écoute sur `0.0.0.0:8420` par défaut (remplaçable avec
`ARONA_HOST` / `ARONA_PORT`).

## 3. Démarrer le mock upstream

Dans un deuxième terminal :

```bash
python3 scripts/mock/server.py
```

Le mock est un serveur aiohttp qui écoute sur `127.0.0.1:8429` par défaut
(`ARONA_MOCK_PORT` remplace le port). Il affiche son API key au démarrage et
sert aussi `GET /api/test-key`, qui renvoie
`{"api_key": ..., "base_url": ...}`. Il expose une poignée d'ids de modèles —
dont `gpt-5.5`, utilisé ci-dessous — et répond aux complétions de chat simples
et en streaming.

Capturez la key affichée :

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. Enregistrer le mock comme backend external

Les backends sont enregistrés via l'API HTTP d'administration :

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

Le backend est sondé immédiatement à l'enregistrement et passe à l'état healthy
en ~1-2 secondes ; jusqu'à la fin de cette sonde, il reste dans un état
fail-closed « not probed yet » (voir l'encadré de dépannage ci-dessous). La
configuration est persistée, donc le backend survit à un redémarrage.

## 5. Enregistrer un compte et se connecter

Les comptes vivent sur le plan JSON-RPC, `POST /api/rpc`. Comme
`ARONA_REGISTRATION_OPEN=1` est défini, `auth.register` est ouvert ; le premier
utilisateur enregistré devient l'admin.

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

Les mots de passe doivent contenir au moins 8 caractères **et** au moins 3 des
4 catégories de caractères (majuscules, minuscules, chiffres, spéciaux).
Ensuite, connectez-vous pour obtenir la paire JWT :

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

Exportez l'`access_token` de la réponse :

```bash
export JWT="<access_token from the login response>"
```

## 6. Créer une API key

`keys.create` est authentifié par JWT et renvoie le secret **complet**
`arona-{uuid}` une seule fois — la base de données ne stocke que son hash
SHA-256, donc copiez-le maintenant :

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. Chat (non-streaming)

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

Vous obtenez un objet de complétion de style OpenAI avec un `choices[0].message`
et un bloc `usage`.

## 8. Chat (streaming)

Le même endpoint avec `"stream": true` répond avec des server-sent events :
un chunk `data:` par token, terminé par un chunk final `data: [DONE]` :

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. Vérifier l'usage

Chaque tour de chat enregistre une ligne d'usage sous le préfixe de la key.
Interrogez-la avec le JWT :

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

Vous devriez voir un ou plusieurs enregistrements pour les requêtes `gpt-5.5`
effectuées ci-dessus.

## Dépannage

- **`No backend available for model: gpt-5.5` (HTTP 404, `code:
  model_not_found`)** — aucun backend enregistré ne sert cet id de modèle. Soit
  le backend n'a jamais été enregistré (ou sa liste `models` n'inclut pas
  l'id), soit l'appel d'enregistrement a échoué. Vérifiez avec
  `GET /api/admin/backends` (token admin).
- **`all backends unhealthy` (HTTP 500, `backend_error`)** — un backend *est*
  enregistré pour le modèle mais aucun candidat n'est healthy. Un backend
  external fraîchement enregistré démarre dans un état fail-closed « not probed
  yet » et passe à healthy une fois la sonde d'enregistrement terminée, ~1-2 s
  plus tard ; si vous chattez dans cette fenêtre, vous obtenez cette erreur.
  Réessayez après un moment, ou vérifiez que le mock tourne bien sur
  `127.0.0.1:8429`.
- **HTTP 401 sur `/v1/*`** — un en-tête `Authorization` manquant produit
  `Missing Authorization header. Use: Bearer <api_key>` ; une key inconnue
  produit `Invalid API key`. Revérifiez `$AR_KEY` (secret complet, pas le
  préfixe).
- **HTTP 401 `Admin access required` sur `/api/admin/*`** — le bearer token ne
  correspond pas à `ARONA_ADMIN_TOKEN`, ou la variable n'est pas définie (la
  route rejette alors toujours). Redémarrez le serveur après l'avoir définie.
- **`auth.register` échoue avec « Registration is closed »** — l'inscription
  est désactivée quand `ARONA_REGISTRATION_OPEN` n'est pas truthy. Définissez
  `ARONA_REGISTRATION_OPEN=1` **avant** de démarrer le serveur (elle est lue au
  démarrage), ou soyez le tout premier utilisateur — le premier utilisateur
  enregistré est toujours autorisé et devient l'admin.
- **Limites de débit HTTP 429** — trois limites indépendantes peuvent se
  déclencher :
  - la limite en mémoire par key, 60 RPM par défaut
    (`ARONA_API_RATE_LIMIT_RPM`) → `Rate limit exceeded. Try again later.` ;
  - la limite de 10 RPM par key du tier de facturation free → 429 avec un
    en-tête `Retry-After: 60` ;
  - le quota mensuel de 1 $ / 100 000 tokens du tier free → 429 avec
    `Retry-After` pointant vers la prochaine période de facturation.

## Prochaines étapes

- [Configuration](configuration.md) — toutes les variables d'environnement.
- [Backends](backends.md) — types de backend, sémantique des URL et sondes.
- [Déploiement](deployment.md) — bare metal, systemd, Docker.
- [API REST compatible OpenAI](../api/openai-rest.md) — toute la surface `/v1/*`.
- [API JSON-RPC](../api/jsonrpc.md) — le plan de gestion utilisé ci-dessus.

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
