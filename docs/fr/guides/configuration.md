---
title: "Configuration"
description: "Référence du fichier de configuration TOML fixe et de chaque variable d'environnement lue par le serveur Arona, avec valeurs par défaut et sémantique."
---

# Configuration
Arona lit sa configuration dans un **fichier TOML fixe**, les variables d'environnement servant de surcharges par clé. Chaque clé se résout ainsi : **variable d'environnement explicite → valeur TOML → défaut intégré** ; les déploiements existants pilotés par env (unités systemd, docker-compose, `.env.example`) fonctionnent inchangés, les nouveaux déploiements peuvent tout fixer dans un seul fichier.

## Le fichier de configuration

Le fichier est découvert dans cet ordre — le premier trouvé l'emporte :
1. le chemin dans `ARONA_CONFIG` (il doit exister ; un fichier manquant ou illisible est une erreur fatale au démarrage) ;
2. `./arona.toml` dans le répertoire de travail ;
3. `/etc/arona/arona.toml` ;
4. aucun fichier → mode historique variables seules.

Il n'y a **aucun repli silencieux** : un fichier découvert qui échoue au parsing, contient une clé inconnue ou une interpolation `${VAR}` non résolue interrompt le démarrage avec une erreur fatale. L'interpolation `${VAR}` / `${VAR:-default}` permet de garder les secrets dans l'environnement ou un `LoadCredential` systemd tandis que le fichier ne contient que des références. Un modèle commenté est fourni sous `arona.toml.example` à la racine du dépôt.

Les clés booléennes (`registration_open`, `writeback`, `backend_local`, `mistralrs_force`, `mock`) doivent être des booléens TOML dans le fichier ; leurs formes env conservent la sémantique de chaîne truthy ou de présence documentée ci-dessous. Les variantes `ARONA_RUSTFS_ACCESS_KEY_FILE` / `ARONA_RUSTFS_SECRET_KEY_FILE` restent réservées aux variables d'environnement.

La section auth porte en outre deux blocs **à valeur structurée** sans forme env : `[auth.oauth]` (le fournisseur de relais OAuth2 — URL des points, identifiants client, scopes, politique d'inscription) et `[[auth.trusted_issuers]]` (panneaux dont les jetons kirino ACCESS sont acceptés directement — émetteur, secret partagé, audience optionnelle). Les deux sont validés au démarrage et documentés dans [Authentification & sécurité](./auth-security.md).

| Variable d'environnement | Clé TOML |
| --- | --- |
| `ARONA_HOST` | `server.host` |
| `ARONA_PORT` | `server.port` |
| `ARONA_DATA_DIR` | `server.data_dir` |
| `DATABASE_URL` | `database.url` |
| `ARONA_DB_MAX_CONNECTIONS` | `database.max_connections` |
| `JWT_SECRET` | `auth.jwt_secret` |
| `ARONA_ADMIN_TOKEN` | `auth.admin_token` |
| `ARONA_AGENT_TOKEN` | `auth.agent_token` |
| `ARONA_REGISTRATION_OPEN` | `auth.registration_open` |
| `ARONA_API_RATE_LIMIT_RPM` | `auth.rate_limit_rpm` |
| `ARONA_AUTH_MODE` | `auth.mode` |
| `ARONA_CONFIG_KEY` | `catalog.config_key` |
| `ARONA_MEMORY_URL` | `memory.url` |
| `ARONA_MEMORY_TOKEN` | `memory.token` |
| `ARONA_MEMORY_WRITEBACK` | `memory.writeback` |
| `ARONA_MEMORY_SCOPE` | `memory.scope` |
| `ARONA_MEDIA_URL_TTL_SECS` | `media.url_ttl_secs` |
| `ARONA_MEDIA_BUCKET` | `media.bucket` |
| `ARONA_MEDIA_BACKEND_LOCAL` | `media.backend_local` |
| `ARONA_RUSTFS_ENDPOINT` | `storage.endpoint` |
| `ARONA_RUSTFS_PUBLIC_ENDPOINT` | `storage.public_endpoint` |
| `ARONA_RUSTFS_REGION` | `storage.region` |
| `ARONA_RUSTFS_ACCESS_KEY` | `storage.access_key` |
| `ARONA_RUSTFS_SECRET_KEY` | `storage.secret_key` |
| `ARONA_EVERNIGHT_URL` | `bridge.evernight_url` |
| `ARONA_AGENT_NAME` | `agent.name` |
| `ARONA_PANEL_URL` | `agent.panel_url` |
| `ARONA_AGENT_BIND_ADDR` | `agent.bind_addr` |
| `ARONA_MISTRALRS` | `agent.mistralrs_force` |
| `ARONA_POINTS_PER_USD` | `billing.points_per_usd` |
| `MOCK_MODE` | `dev.mock` |
| `MOCK_SEED_PATH` | `dev.seed_path` |
| `HF_ENDPOINT` | `downloads.hf_endpoint` |
| `GITHUB_TOKEN` | `downloads.github_token` |

Le reste de cette page est la **référence des variables d'environnement** ; elle double comme référence sémantique des clés TOML mappées (mêmes noms, mêmes défauts).

## Tableau de référence

| Variable | Défaut | Rôle |
| --- | --- | --- |
| `DATABASE_URL` | *(requis)* | URL de connexion PostgreSQL. |
| `JWT_SECRET` | *(requis hors mode mock)* | Secret utilisé pour signer les JWT. |
| `ARONA_HOST` | `0.0.0.0` | Adresse d'écoute (repli sur `SHITTIM_CHEST_HOST`). |
| `ARONA_PORT` | `8420` | Port d'écoute (repli sur `SHITTIM_CHEST_PORT`). |
| `ARONA_DATA_DIR` | non défini | Répertoire de données local. |
| `ARONA_ADMIN_TOKEN` | non défini | Bearer token pour `/api/admin/*` et les méthodes RPC admin. |
| `ARONA_AGENT_TOKEN` | repli sur `ARONA_ADMIN_TOKEN` | Jeton Bearer côté agent pour `/ws/agent` ; doit correspondre au jeton opérateur du panneau. |
| `ARONA_REGISTRATION_OPEN` | `0` | Une valeur truthy (`1`/`true`/`yes`/`on`) ouvre l'inscription. |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | Limite de requêtes en mémoire par key et par minute ; `0` bloque tout. |
| `MOCK_MODE` | non défini | La présence (n'importe quelle valeur) active le mode mock de dev. |
| `MOCK_SEED_PATH` | non défini | Fichier SQL brut de seed exécuté en mode mock. |
| `ARONA_MEMORY_URL` | non défini | URL WebSocket du gateway mémoire Philia. |
| `ARONA_MEMORY_TOKEN` | non défini | Token pour le gateway mémoire. |
| `ARONA_MEMORY_WRITEBACK` | `true` | Écrit les tours de chat terminés en mémoire ; accepte `true`/`false` (toute autre valeur retombe sur la valeur par défaut). |
| `ARONA_AGENT_NAME` | `arona-agent` | Identité de l'agent de nœud GPU. |
| `ARONA_PANEL_URL` | `localhost:8080` | Où l'agent se connecte (`ws://<panel_url>/ws/agent`). |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | Agent evernight local pour les URL de backend `evernight://`. |
| `ARONA_MISTRALRS` | non défini | La présence force le moteur Mistral.rs pour les plans de modèles Gguf. |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | Interface sur laquelle un serveur de modèles llama.cpp lancé se lie. |
| `HF_ENDPOINT` | `https://huggingface.co` | URL de base Hugging Face pour les téléchargements de modèles. |
| `GITHUB_TOKEN` | non défini | Token d'accès pour le registre de modèles GitHub. |
| `RUST_LOG` | non défini | Filtre de tracing, p. ex. `info` ou `arona=debug,info`. |

## Variables requises

### `DATABASE_URL`

URL de connexion PostgreSQL. **Requise** : le serveur se termine avec
`FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.` quand elle
est vide, et la sous-commande CLI `migrate` refuse de s'exécuter. Le schéma est
créé/mis à jour automatiquement par `serve` au démarrage.

### `JWT_SECRET`

Secret utilisé pour signer les paires JWT access/refresh émises par
`auth.login` et `auth.register`. **Requis en production** : le code embarque un
repli de développement (`dev-secret-not-for-production-use-only-32chars`), mais
le serveur refuse de démarrer avec lui sauf si `MOCK_MODE=1` :

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

Utilisez une valeur longue et aléatoire (p. ex. `openssl rand -hex 32`).

## Serveur

### `ARONA_HOST` / `ARONA_PORT`

Adresse et port d'écoute du gateway. Pour compatibilité héritée, ils retombent
sur `SHITTIM_CHEST_HOST` / `SHITTIM_CHEST_PORT` ; les défauts ultimes sont (ordre : `server.host`/`server.port` du fichier de configuration priment sur les alias historiques — seuls `ARONA_HOST`/`ARONA_PORT` canoniques priment sur le fichier)
`0.0.0.0:8420`.

### `ARONA_DATA_DIR`

Répertoire de données local optionnel, porté sur l'état de l'application pour
les composants qui ont besoin d'un emplacement de travail. Non défini par
défaut.

## Sécurité et contrôle d'accès

### `ARONA_ADMIN_TOKEN`

Bearer token protégeant les routes HTTP `/api/admin/*` (`POST/GET/DELETE
/api/admin/backends`, `/api/admin/aliases`) et les méthodes RPC
`billing.plan.set` / `billing.video.pricing.set`. Quand il est **non défini**,
chacune de ces routes rejette avec `Admin access required` (401) — il n'y a pas
de valeur par défaut. Définissez-le avec une valeur aléatoire forte avant de
démarrer le serveur.

### `ARONA_REGISTRATION_OPEN`

Ouvre l'inscription en libre-service via `auth.register`. Les valeurs truthy
sont exactement `1`, `true`, `yes`, `on` (insensibles à la casse) ; tout le
reste — y compris `0`, `false`, `off`, ou une variable non définie/vide — reste
fermé. Le drapeau est lu une fois au démarrage. Le **premier utilisateur
enregistré est toujours autorisé** (même quand l'inscription est fermée) et
devient l'admin.

### `ARONA_API_RATE_LIMIT_RPM`

Limite de débit en mémoire par key à fenêtre glissante (requêtes par minute),
appliquée à chaque requête `/v1/*` authentifiée (chat, embeddings, vidéo,
models), indexée par le hash de l'API key (ou le libellé `u:<email>` pour le
`GET /v1/models` acceptant les JWT). Le plan RPC n'est pas limité par ce
limiteur — les extracteurs d'auth `/v1/*` sont les seuls appelants. Défaut
`60`. La valeur est parsée une fois dans un `OnceLock` à l'échelle du
processus. **Une valeur de `0` bloque chaque requête** — le contrôle est
`entry.len() >= rpm`, donc avec `0` aucune requête ne peut passer. C'est la
limite à l'échelle du gateway ; les tiers de facturation imposent leurs propres
limites par key par-dessus.

## Développement

### `MOCK_MODE`

Mode mock de dev, activé par **présence** — le contrôle est
`std::env::var("MOCK_MODE").is_ok()`, donc *n'importe quelle* valeur (y compris
`0` ou une valeur vide mais définie) l'active. Il :

- lève l'exigence de `JWT_SECRET` (le secret de dev intégré devient
  acceptable) ;
- seed quatre comptes de démo (`demiurge@celestia.world`, `momoi@celestia.world`,
  `midori@celestia.world`, `yuzu@celestia.world`, mot de passe `33550336`) ;
- attend la fin du seed avant de lier l'écouteur.

Ne l'utilisez jamais en production.

### `MOCK_SEED_PATH`

En mode mock uniquement, pointe vers un fichier SQL brut exécuté à la place du
seed de comptes intégré (instructions scindées sur `;`, commentaires `--`
ignorés). Si le fichier ne peut pas être lu, le seed intégré est utilisé comme
repli.

## Gateway mémoire

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

Configuration du gateway de mémoire à long terme (entelecheia Philia). La
mémoire est **entièrement désactivée** sauf si `ARONA_MEMORY_URL` **et**
`ARONA_MEMORY_TOKEN` sont définies et non vides. Quand elle est activée :

- les tours de chat terminés sont rappelés et injectés comme contexte, et
- `ARONA_MEMORY_WRITEBACK` (défaut `true`) contrôle si les tours terminés sont
  réécrits vers le service mémoire ; `0` ou `false` désactive le writeback.

Les échecs de mémoire ne bloquent jamais le chat ; l'état résultant est reflété
dans l'en-tête de réponse `X-Arona-Memory` (`enabled` / `disabled` / `offline`).

## Identité d'agent et cluster

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

Identité du binaire d'agent de nœud GPU (`_agent`) : `ARONA_AGENT_NAME`
(défaut `arona-agent`) est rapporté au panel comme nom/id de l'agent, et
`ARONA_PANEL_URL` (défaut `localhost:8080`) est l'endroit où l'agent se
connecte (`ws://<panel_url>/ws/agent`).

L'API HTTP propre de l'agent est **codée en dur** pour écouter sur
`0.0.0.0:5790` — il n'existe pas de variable d'environnement d'adresse d'écoute
pour elle.

### `ARONA_AGENT_BIND_ADDR`

Interface sur laquelle un **serveur de modèles llama.cpp lancé** écoute quand
l'agent déploie un modèle Gguf, pour que le moteur soit joignable depuis
d'autres machines (p. ex. `0.0.0.0`). Par défaut `127.0.0.1`. À noter : ce
n'est *pas* l'écoute de l'API HTTP de l'agent (qui est fixée à `0.0.0.0:5790`).

## Pont evernight

### `ARONA_EVERNIGHT_URL`

URL WebSocket de l'agent evernight local utilisé pour résoudre les URL de
backend `evernight://` en forwards TCP locaux. Défaut `ws://127.0.0.1:3001/ws`.

## Runtime de modèles et téléchargements

### `ARONA_MISTRALRS`

La présence (n'importe quelle valeur) force le moteur Mistral.rs pour les plans
de modèles Gguf qui utiliseraient sinon llama.cpp par défaut. Sémantique de
présence comme `MOCK_MODE`.

### `HF_ENDPOINT`

URL de base pour les téléchargements de modèles Hugging Face (sources `hf:`),
défaut `https://huggingface.co`. Définissez-la sur un miroir tel que
`https://hf-mirror.com` quand huggingface.co est injoignable. Lue par le
téléchargeur de modèles ; un slash final est supprimé.

### `GITHUB_TOKEN`

Token d'accès utilisé par le registre de modèles GitHub (sources `gh:`) pour
l'accès API. Non défini par défaut ; sans lui, les limites de débit de l'API
GitHub s'appliquent.

## Journalisation

### `RUST_LOG`

Filtre de tracing standard appliqué par `tracing_subscriber` au démarrage,
p. ex. `info` ou `arona=debug,info`. Suit la sémantique habituelle de
`RUST_LOG` (`error`/`warn`/`info`/`debug`/`trace`, remplacements par cible).

## Défauts en un coup d'œil

| Paramètre | Défaut |
| --- | --- |
| Adresse / port d'écoute | `0.0.0.0:8420` |
| Limite de débit API par key | 60 RPM |
| Nom d'agent | `arona-agent` |
| URL du panel | `localhost:8080` |
| Writeback mémoire | activé |
| Inscription | fermée |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/config/file.rs (discovery, interpolation, KEY_MAP), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
