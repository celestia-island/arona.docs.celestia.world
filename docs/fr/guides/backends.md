---
title: "Backends"
description: "Types de backend (external, ollama, engine, minimax-cloud, ponts evernight), sémantique des URL, sondes de santé, découverte de modèles, alias et routage."
---

# Backends

Un **backend** est un upstream qui sert le trafic des modèles. Arona route les
requêtes compatibles OpenAI (`/v1/chat/completions`, `/v1/embeddings`, liste
des modèles, jobs vidéo) vers l'un des backends enregistrés, mesure chaque
requête, et maintient à jour la santé et l'inventaire de modèles de chaque
backend.

Les backends sont enregistrés par un admin via
`POST /api/admin/backends` (voir l'[API HTTP d'administration](../api/admin-http.md)),
persistés dans la table `backend_configs`, et restaurés automatiquement au
démarrage. Chaque enregistrement porte un `name`, un `type`, une `url`, une
`api_key` optionnelle et une liste statique `models` optionnelle. Les backends
persistés survivent aux redémarrages ; les backends restaurés démarrent en
fail-closed et sont sondés immédiatement.

## Types de backend

| `type` | Transport | Protocole | Rôle |
| --- | --- | --- | --- |
| `external` | HTTP(S) | REST compatible OpenAI | Toute API de chat/embeddings (cloud ou auto-hébergée) |
| `ollama` | HTTP(S) | API native Ollama (`/api/chat`, `/api/embed`, `/api/tags`) | Un serveur Ollama local ou distant ; construit à partir de la seule URL |
| `engine` | `ws://` / `wss://` | CEP (Celestia Engine Protocol), WebSocket + JSON-RPC | Tout moteur parlant le standard d'échange CEP (`plana::engine`) |
| `minimax-cloud` | HTTPS | API de style tâche MiniMax H3 (soumission + sondage) | Génération vidéo cloud |
| `evernight://<node>/<service>` | URL de pont | Résolue via l'agent evernight local en un forward TCP local | Services industriels/edge joignables uniquement via le mesh evernight |
| `agent-{model}` | HTTP | Compatible OpenAI (external) | Auto-enregistré quand un agent GPU déploie un modèle |

### `external` — toute API HTTP compatible OpenAI

Le backend polyvalent : complétions de chat (streaming et non-streaming) et
embeddings contre n'importe quel serveur parlant la forme REST OpenAI.
Configurez-le avec une `url` de base, une `api_key` (optionnelle) et une liste
statique `models` optionnelle :

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

La liste statique `models` fait autorité : elle est fusionnée avant tout modèle
découvert au moment de la sonde (voir [Model discovery](#model-discovery)).

### `ollama` — construit à partir de la seule URL

Un backend Ollama est construit à partir de la seule URL — pas d'API key, pas
de liste de modèles. Il parle les protocoles natifs d'Ollama : `/api/chat`
pour le chat, `/api/embed` pour les embeddings, et `/api/tags` pour les sondes
de santé et la découverte de modèles.

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — CEP sur WebSocket

Un backend `engine` se connecte à un moteur exposant `ws://` (ou `wss://`) et
parle le **Celestia Engine Protocol** (CEP) : un standard d'échange
WebSocket + JSON-RPC 2.0 défini dans `plana::engine`. Tout moteur écrit dans
n'importe quel langage qui implémente le flux handshake → méthodes →
notifications de streaming s'enregistre comme backend de première classe avec
zéro code d'adaptateur dans Arona. Méthodes filaires : `Engine.Handshake`
(premier message ; identité + capacités), `Engine.Chat`, `Engine.ChatStart`
(streaming ; les chunks arrivent comme notifications `Engine.ChatChunk`),
`Engine.Embeddings` et `Engine.Models`. Les connexions sont établies
paresseusement à la première utilisation et coupées à la moindre erreur ;
l'appel suivant se reconnecte et refait le handshake.

### `minimax-cloud` — génération vidéo de style tâche

Le backend vidéo cloud pilote l'API MiniMax H3 Open Platform : soumettez une
tâche de génération, sondez son achèvement, puis lisez l'URL de l'artefact
dans le résultat. C'est ce qui a remplacé le backend ComfyUI supprimé (voir
ci-dessous) ; les jobs vidéo sont soumis via `/v1/video/generations` ou les
méthodes RPC `video.*` et progressent via les notifications `video.progress` /
`video.done` / `video.failed` (voir [Realtime & Vidéo](realtime-video.md)).

### URL de pont `evernight://`

Une URL de backend de la forme `evernight://<node>/<service>` **n'est pas**
contactée directement. L'agent evernight de l'hôte local la résout (un appel
JSON-RPC `Bridge.Connect` sur l'endpoint WebSocket de l'agent) en un forward
TCP local, et le backend fonctionne contre `http://127.0.0.1:<local_port>`
plutôt qu'une adresse distante codée en dur. C'est l'architecture à panel
unique : le panel Arona atteint les services des autres nœuds (moteurs CEP,
scepter, ...) à travers le mesh sans jamais embarquer d'adresse distante dans
une configuration.

Une tâche keepalive sonde le tunnel toutes les 15 secondes ; quand le côté
distant redémarre et que le tunnel est rétabli sur un nouveau port local, le
backend concerné est **transparentment reconstruit** avec la nouvelle URL — la
configuration persistée garde l'URL `evernight://` pour que les redémarrages
la re-résolvent. Pour les backends `engine`, le forward résolu
`http://127.0.0.1:<port>` est adapté en `ws://` pour le transport WebSocket.

### Les modèles déployés par les agents s'auto-enregistrent

Quand un agent GPU termine le déploiement d'un modèle, le gateway enregistre un
backend nommé `agent-{model_id}` (un `ExternalApiBackend` sur
`http://{agent host}:{port}`) pour que le modèle devienne immédiatement
routable ; l'arrêt du déploiement le désenregistre à nouveau. Voir
[Cluster d'agents](agent-cluster.md) pour le cycle de vie de déploiement
complet.

### `comfyui` est rejeté

Le type de backend `comfyui` est explicitement refusé avec l'erreur
`comfyui backend removed` : le backend ComfyUI a été supprimé lors de la
convergence de la plateforme de modèles, et la génération vidéo passe désormais
par `minimax-cloud`. L'enregistrement d'un backend `comfyui` renvoie un HTTP
400.

## Sémantique des URL

La manière dont une URL de base configurée mappe vers les endpoints réels est
décidée par la présence ou non d'un composant de chemin dans l'URL :

- **Base de style racine** — une URL dont le chemin est vide ou `/` est traitée
  comme une racine d'hôte et conserve la convention OpenAI `/v1` :
  `{base}/v1/chat/completions`, `{base}/v1/models`. Exemples :
  `http://192.0.2.20:8429`, `https://api.deepseek.com`.
- **Base de style chemin** — une URL avec un chemin non vide est traitée comme
  le préfixe API complet que le serveur sert réellement, et l'endpoint est
  ajouté directement : `{base}/chat/completions`, `{base}/models`. C'est ce
  dont ont besoin les serveurs compatibles OpenAI hors convention `/v1`. Le
  plan de codage Zhipu GLM est l'exemple canonique : son API vit à
  `https://open.bigmodel.cn/api/coding/paas/v4` avec le chat directement à
  `{base}/chat/completions` et **aucun endpoint `/models` du tout** — la racine
  standard `/api/paas/v4` renvoie des erreurs de solde pour les keys du plan de
  codage.
- Un **slash final** sur l'URL de base configurée est normalisé, pour que la
  jonction ne produise jamais de double slash.

## Sondes et santé

Un vérificateur de santé en arrière-plan sonde chaque backend enregistré
toutes les **60 secondes** ; la liste des backends est récupérée à nouveau à
chaque tour, donc les backends enregistrés après le démarrage sont pris en
compte sans redémarrage. Chaque enregistrement admin déclenche aussi une sonde
immédiate pour que le backend passe à healthy en ~1–2 secondes au lieu
d'attendre le tour suivant du vérificateur.

- **Les backends external** sondent `GET {base}/models` (ou
  `{base}/v1/models` pour les bases de style racine) avec un **timeout de
  2 secondes**. Un **404 est toléré** : certains serveurs implémentent le chat
  mais n'exposent aucune liste de modèles (le plan de codage GLM n'a pas
  d'endpoint `/models`), donc un 404 marque le backend healthy et la liste
  `models` configurée par l'admin devient la source du routage. Les timeouts,
  échecs réseau et autres réponses non-2xx marquent le backend unhealthy.
- **Les backends Ollama** sondent `/api/tags` avec le même timeout de
  2 secondes.
- Les backends démarrent en **fail-closed** — signalés comme `not probed yet` —
  jusqu'à la première sonde réussie, donc un backend fraîchement enregistré (ou
  restauré) ne reçoit jamais de trafic avant d'avoir été vérifié.

L'état de santé est mis en cache par backend et consulté par le routeur à
chaque requête ; les backends unhealthy sont exclus de la sélection des
candidats (voir [Routing](#routing)).

## Découverte de modèles

Un backend annonce les ids de modèles qu'il sert, et le routeur fait
correspondre les requêtes à cette annonce :

- **External** — les backends annoncent les modèles parsés depuis la réponse
  de la sonde (un tableau `data` comme un tableau `models` sont acceptés),
  fusionnés avec la liste statique configurée par l'admin — les ids statiques
  gardent l'ordre et la précédence, les ids dynamiques sont dédupliqués et
  ajoutés. Quand un serveur n'a pas d'endpoint de modèles, la liste statique
  seule est la source du routage.
- **Ollama** — les backends annoncent les tags renvoyés par `/api/tags`.
- **Modèles déployés par agents** — annoncent exactement le `model_id` déployé.

La surface publique est `GET /v1/models` (authentifié), qui liste les modèles
routables de chaque backend healthy (voir l'[API REST compatible
OpenAI](../api/openai-rest.md)).

## Alias et normalisation des noms

Les alias font correspondre un id de modèle demandé à un id cible. L'alias est
résolu en premier dans le routage, donc une requête pour l'alias est servie par
le backend qui annonce la cible :

```json
{ "alias": "fast-chat", "target": "deepseek-v4-flash" }
```

Les alias sont gérés via les endpoints admin `/api/admin/aliases` et prennent
effet immédiatement.

La correspondance des noms normalise aussi les tags de style Ollama : un
backend listant `nomic-embed-text:latest` correspond à une requête nue pour
`nomic-embed-text`, donc les requêtes d'embedding/chat se résolvent sans
comptabilité de suffixe `:latest`. Un tag explicite (`qwen3:0.6b`) ne
correspond qu'à ce tag exact.

## Routage

Chaque requête passe par le routeur, qui sélectionne un backend :

1. **Résolution d'alias** — l'id de modèle demandé est mappé via la table
   d'alias (le cas échéant).
2. **Indice de provider** — un champ optionnel `provider` filtre les candidats
   par nom de backend (ou nom de type, p. ex. `cloud` pour les backends
   vidéo).
3. **Candidats healthy uniquement** — un backend doit signaler `Healthy` *et*
   passer son circuit breaker (3 échecs consécutifs ouvrent le breaker pendant
   30 secondes, avec un appel de test half-open) pour être sélectionnable.
4. **Choix le moins chargé** — les candidats servant le modèle sont triés par
   leur compteur de requêtes par backend et le moins chargé est choisi. Cela
   répartit la charge entre les backends healthy servant le même modèle.
5. **Affinité de session** — quand une requête porte un `conversation_id`, la
   conversation est épinglée au backend qu'elle a utilisé en premier.
   L'épingle vit dans une map de références `Weak`, donc un backend supprimé
   disparaît de la map sans dérive d'index. L'affinité est best-effort :
   réutiliser le même backend sur les tours d'une conversation laisse
   l'upstream réutiliser l'état d'exécution par conversation (contextes
   chauds, caches KV). Si le backend épinglé est devenu unhealthy (ou si
   l'épingle est morte avec un backend supprimé), le routeur retombe sur une
   nouvelle sélection la moins chargée et **ré-épingle** la conversation.

Si aucun backend healthy ne sert le modèle, le routage échoue : un modèle
inconnu donne `model not found` (HTTP 404), un modèle connu mais injoignable
donne `all backends unhealthy`, remonté comme erreur interne 500. HTTP 502 est
réservé aux échecs rapportés par un upstream *joignable* (réponses upstream
non-2xx et échecs de transport après sélection). Voir
[Opérations](operations.md) pour le mapping d'erreurs complet.

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
