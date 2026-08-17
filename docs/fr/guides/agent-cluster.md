---
title: "Cluster d'agents"
description: "Clusters GPU multi-nœuds — télécharger les poids de modèles avec la CLI, exécuter le binaire _agent sur les nœuds GPU, et piloter les déploiements via la surface RPC agents.*."
---

# Cluster d'agents

L'histoire de déploiement d'Arona se divise en deux moitiés. Le **panel** (le
binaire serveur `arona`) possède le routage, la facturation, l'auth et le plan
de gestion. Chaque nœud GPU exécute un **processus `_agent`** qui possède les
poids de modèles et les processus de service locaux. Les agents ouvrent un
WebSocket longue durée vers le plan de contrôle d'agents du panel
(`/ws/agent`) ; le panel pousse les commandes `deploy` / `stop` sur ce socket
et l'agent remonte les progrès de téléchargement, les heartbeats et les
résultats de commandes. Une fois un modèle en cours d'exécution sur un agent,
le panel l'enregistre comme backend routable pour que le trafic `/v1/*` et RPC
l'atteigne — le plan de contrôle est en WebSocket, le plan de données est en
HTTP simple vers le port de moteur local de l'agent.

## Téléchargement des poids de modèles (CLI)

Le binaire `_cli` télécharge les poids de modèles depuis HuggingFace, ModelScope
ou les releases GitHub :

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **Formes de repo** — `hf:owner/repo` (défaut ; un `owner/repo` nu se résout
  aussi vers HuggingFace), `ms:owner/repo` (ModelScope), `gh:owner/repo[:tag]`
  (release GitHub, tag optionnel). Les longs préfixes `huggingface:`,
  `modelscope:` et `github:` sont aussi acceptés ; un id nu sans slash se
  résout vers le registre Ollama
  (`packages/core/src/models/download.rs:21-28,55-86`).
- **`--filter <glob|prefix>`** — répétable ; seuls les fichiers du manifest
  correspondant au glob (ou au préfixe) sont téléchargés. Sans filtre, le
  **repo entier** est sélectionné.
- **Confirmation** — un téléchargement non filtré demande toujours
  `Continue? [y/N]` avant de démarrer sauf si `--yes` est passé. Un
  téléchargement filtré ne demande jamais ; quand le total sélectionné atteint
  ou dépasse 2 Gio, il affiche à la place une bannière informative
  (`NO_CONFIRM_THRESHOLD`, `packages/cli/src/main.rs:12-15, 439-464`).
- **`--out <dir>`** — remplace la destination par défaut
  `~/.arona/models/<repo-id>`.
- **`--revision <rev>`** — remplace tout suffixe `:rev` inline
  (`hf:owner/repo:rev`).
- **Reprise** — les téléchargements interrompus reprennent automatiquement : un
  fichier `.part` est conservé et le téléchargement continue depuis sa longueur
  courante via une requête Range ; les fichiers complets sont sautés par taille
  et, quand le manifest porte un digest, vérifiés en SHA-256
  (`packages/cli/src/main.rs` `verify_sha256`/`summarize`).
- **Nouvelles tentatives** — les erreurs réseau réessaient jusqu'à 3 fois avec
  un délai de 5 secondes ; les erreurs non réseau échouent immédiatement
  (`packages/cli/src/main.rs:277-302`).
- **`HF_ENDPOINT`** — change l'URL de base HuggingFace, p. ex. un miroir :

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

Les autres commandes CLI (`packages/cli/src/main.rs:28-53`) :

| Commande | Rôle |
| --- | --- |
| `install` | Configuration d'environnement en un clic : détecte le profil matériel et affiche les recommandations de backend / quantification. |
| `status` | Affiche le profil matériel. |
| `deploy <model>` | Résout un modèle localement et indique s'il est déjà en cache. |
| `download` | Télécharge les poids de modèles (ci-dessus). |
| `serve` | Démarre le serveur API (panel). |
| `connect <url>` | Se connecte à un panel de gestion. |
| `migrate` | Exécute les migrations de base de données. |

## Le binaire `_agent`

`_agent` tourne sur chaque nœud GPU et est configuré uniquement par des
variables d'environnement (`packages/core/src/config.rs:37-40`) :

| Variable | Défaut | Signification |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | Id de nœud unique ; le panel l'utilise comme `agent_id`. |
| `ARONA_PANEL_URL` | `localhost:8080` | `host:port` du panel ; l'agent se connecte à `ws://{ARONA_PANEL_URL}/ws/agent`. |

Voir [Configuration](./configuration.md) pour la référence complète des
variables d'environnement (variables côté panel, base de données, secrets).

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

Comportement :

- **Connexion de contrôle** — l'agent se reconnecte à
  `ws://{ARONA_PANEL_URL}/ws/agent` (`packages/agent/src/panel.rs:23`). À la
  connexion, il envoie une frame `register` portant `agent_name`, `gpu_info` et
  la liste des modèles déjà déployés ; le panel enregistre l'adresse de peer
  TCP de l'agent comme son `host`.
- **Backoff de reconnexion** — commence à 1 seconde et double jusqu'à un
  plafond de 60 secondes (`packages/agent/src/panel.rs:27,33-34,63`).
- **Heartbeats** — toutes les 30 secondes, l'agent rapporte l'utilisation GPU,
  le nombre de modèles chargés et l'uptime. Le panel considère un agent hors
  ligne quand son dernier heartbeat date de plus de 30 secondes.
- **API HTTP locale** — écoute sur l'adresse **fixe** `0.0.0.0:5790` ; il
  n'existe pas de variable d'environnement d'adresse d'écoute
  (`packages/agent/src/main.rs:109`). Le panel combine ce port avec le host
  enregistré de l'agent pour construire l'URL du plan de données des modèles
  déployés.
- **Commandes** — le panel met en file les commandes `deploy` / `stop` sur le
  socket. Une commande `deploy` porte `model_id` et un `stream_id` ; la
  progression du téléchargement est remontée en streaming comme frames
  `deploy_progress` sur le même socket (le panel les transmet comme
  notifications SSE `models.progress`, voir
  [Événements et notifications](../api/events.md)), et la frame finale
  `deploy_result` rapporte le `port` et le `backend` du moteur local. `stop`
  reçoit une réponse `stop_result`.

Exécutez `_agent` sous un superviseur de services (systemd, malkuth, ...) pour
qu'il se reconnecte automatiquement ; le panel tolère les redémarrages des deux
côtés (voir [node persistence](#node-persistence) ci-dessous).

## RPC du plan de contrôle d'agents

Toute la surface d'agents est gardée par admin : chaque méthode requiert un JWT
valide **et** un compte admin (`validate_admin_jwt` vérifie
`is_admin_email` ; `packages/core/src/gateway/rpc.rs:106-118,301-337`).

| Méthode | Params | Retours |
| --- | --- | --- |
| `agents.list` | — | Topologie du cluster : `id`, `name`, `host`, `status` (`online`/`offline`), résumé GPU, `models`, `last_heartbeat`, `version`, `connected_at`. |
| `agents.register` | `machine_name`, `version` | `agent_id`, `token`. |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — supprime le nœud. |
| `agents.status` | `agent_id` | `online`, résumé GPU, `gpu_utilization`, `models`, `host`, `connected_at`, `last_heartbeat`. |
| `agents.deploy` | `model_id`, `agent_id?` | `{ "ok": true, "stream_id" }` — un `agent_id` vide cible automatiquement le nœud le moins chargé. |
| `agents.stop` | `agent_id`, `model_id` | `{ "ok": true }` — arrête le déploiement. |

`agents.deploy` renvoie un `stream_id` ; abonnez-vous à
`/api/rpc/events?session=<stream_id>` **avant** ou immédiatement après l'appel
pour recevoir les notifications de téléchargement `models.progress` (voir
[Événements et notifications](../api/events.md)). Voir l'[API
JSON-RPC](../api/jsonrpc.md) pour les détails de transport et d'auth.

## Auto-enregistrement des modèles déployés

Quand une frame `deploy_result` rapporte un succès, le panel enregistre un
`ExternalApiBackend` nommé **`agent-{model_id}`** dans le routeur du gateway,
avec l'URL de base `http://{agent-host}:{port}` — le host enregistré de
l'agent plus le port du moteur qu'il a rapporté
(`packages/core/src/gateway/server.rs:310-366`,
`packages/core/src/gateway/mod.rs:253-270`). Le modèle déployé devient un
backend routable normal : `/v1/chat/completions`, les embeddings et le chat RPC
l'atteignent tous, les alias s'appliquent, et le vérificateur de santé le sonde
(voir [Backends](./backends.md) pour les types de backend et la sémantique des
sondes).

- Redéployer le même modèle (p. ex. sur un agent différent) remplace le backend
  précédent.
- Un `stop_result` réussi le désenregistre à nouveau
  (`packages/core/src/gateway/mod.rs:274-287`) ; l'id de modèle cesse de se
  résoudre.

## Placement

Les déploiements sans `agent_id` explicite passent par un placement le moins
chargé (`packages/core/src/gateway/tunnel.rs:214-229`) : les candidats sont les
agents dont le dernier heartbeat date de moins de 30 secondes, et celui avec
l'**utilisation GPU moyenne la plus basse** est choisi. Les agents sans
télémétrie sont triés en dernier mais restent sélectionnables. Si aucun agent
n'est en ligne, le RPC échoue avec `No online agents available for deployment`.

Côté routage, les conversations sont **épinglées à un backend** (affinité de
session) : le premier backend qui sert une conversation est enregistré et
réutilisé pour les tours suivants, pour que l'état par conversation tel qu'un
cache KV d'exécution reste chaud (`packages/core/src/routing/mod.rs:31-34,110-138`).
Si le backend épinglé devient unhealthy, le routage dégrade vers une nouvelle
sélection et ré-épingle.

## Persistance des nœuds

Les nœuds d'agents persistent dans la table `agent_nodes` (`agent_id`,
`machine_name`, `version`, `host`, `gpu_info`, `models`, `connected_at`,
`last_heartbeat` ; `packages/core/src/gateway/tunnel.rs:8-12`). Au démarrage du
panel, les lignes persistées sont restaurées pour que les nœuds précédemment
enregistrés restent visibles à travers les redémarrages ; les entrées
restaurées sont **sans émetteur** jusqu'à ce que chaque agent se reconnecte via
WebSocket (`packages/core/src/gateway/run.rs:74-85`). Déployer vers un nœud
restauré mais déconnecté échoue donc jusqu'à ce que son `_agent` se
reconnecte.

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
