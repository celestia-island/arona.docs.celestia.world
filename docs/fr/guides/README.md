---
title: "Arona"
description: "Plateforme de déploiement autonome et de gestion à distance de modèles d'IA — gateway, backends, facturation, agents, mémoire."
---

# Arona

**Plateforme de déploiement autonome et de gestion à distance de modèles d'IA.**

Arona est une plateforme **purement backend** écrite en Rust (axum) : c'est à la
fois un gateway de modèles compatible OpenAI *et* un plan de gestion pour les
modèles que vous exécutez sur votre propre matériel. Elle sert l'API REST
compatible OpenAI `/v1/*`, le plan de gestion JSON-RPC 2.0 (`/api/rpc`), le
plan de contrôle des agents (`/ws/agent`) et une interface Swagger à `/docs`.

Il n'y a **ni tableau de bord web intégré, ni chat CLI intégré** — l'interface
de chat + administration vit dans [shittim-chest](https://github.com/celestia-island/shittim-chest),
qui communique avec Arona via la surface RPC. Arona se concentre sur le côté
serveur : routage, facturation, authentification, déploiement de modèles,
agents et mémoire.

## Matrice des fonctionnalités

| Domaine | Ce qu'Arona fournit |
| --- | --- |
| **Cœur conversationnel** | `chat.completions` compatible OpenAI (stream + non-stream), `embeddings`, liste `models` ; streaming avec un chunk terminal `[DONE]` et l'usage réel sur le chunk final. |
| **Backends** | Upstreams enregistrés par l'admin : `external` (toute API HTTP compatible OpenAI), `ollama`, moteur CEP `engine` (WebSocket), vidéo `minimax-cloud`, et URL de pont `evernight://` vers les services industriels/edge. |
| **Authentification** | Paires JWT access/refresh (15 min / 7 jours), API keys `arona-{uuid}` stockées sous forme de hash SHA-256, trois niveaux d'admin, politique de mot de passe, limitation de débit à double voie. |
| **Facturation et usage** | Tiers préinstallés (free / pro / enterprise), enregistrements d'usage par requête sur chaque canal, table de tarification plana, quota par projet, 429 + `Retry-After`. |
| **Gestion des modèles** | Téléchargement de modèles (sources `hf:` / `ms:` / `gh:`), déploiement sur nœuds GPU `_agent`, auto-enregistrement des modèles déployés comme backends routables. |
| **Realtime et multimodal** | Sessions `realtime.*` full-duplex, canal de perception/contrôle `engine.invoke`, jobs asynchrones de génération vidéo (cloud MiniMax). |
| **Cluster d'agents** | Les nœuds GPU se connectent via `/ws/agent`, placement le moins chargé, affinité de session, persistance des nœuds après redémarrage. |
| **Gateway mémoire** | Mémoire à long terme via entelecheia Philia : injection de rappel, épisodes de writeback, dégradation explicite. |
| **Opérations** | Sondes de santé, tracing `RUST_LOG`, mapping des erreurs upstream (502 vs 500), arrêt gracieux, auto-migration au démarrage. |

## Positionnement

Arona est un **gateway + plateforme** : il route le trafic des modèles vers vos
backends, déploie les modèles sur vos agents GPU, et mesure tout.

- vs **pi** — pi est un assistant CLI qui parle aux modèles ; arona n'a pas de
  chat CLI. Arona est la plateforme à laquelle pi (et d'autres outils) parle.
- vs **one-api / new-api** — ce sont des gateways d'API keys devant les
  providers de modèles ; arona ajoute le **déploiement de modèles**
  (télécharger les poids, les exécuter sur vos agents), un plan RPC de gestion
  complet, des tiers de facturation et la mémoire.
- vs **LiteLLM** — un pair gateway ; arona possède en plus le cycle de vie de
  déploiement des modèles derrière le gateway.

## Commencer ici

- [Guide de démarrage rapide](quickstart.md) — de bout en bout avec le mock upstream intégré.
- [Configuration](configuration.md) — toutes les variables d'environnement.
- [Déploiement](deployment.md) — bare metal, systemd, Docker, supervision.
- [Backends](backends.md) — types de backend, sémantique des URL et sondes.
- [API REST compatible OpenAI](../api/openai-rest.md) — `/v1/*`.
- [API JSON-RPC](../api/jsonrpc.md) — le plan de gestion complet.

## Structure du dépôt

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

Le tableau de bord web a été retiré de ce dépôt et vit désormais dans
[shittim-chest](https://github.com/celestia-island/shittim-chest) (chest #291).

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
