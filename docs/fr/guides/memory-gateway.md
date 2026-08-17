---
title: "Gateway mémoire"
description: "Mémoire à long terme pour le chat — injection de rappel, writeback d'épisodes, contrôle par requête, états d'en-tête et les RPC memory.status / memory.delete."
---

# Gateway mémoire

Le Gateway Mémoire donne aux tours de chat accès à la **mémoire à long terme**
stockée dans le service mémoire entelecheia scepter / Philia. À chaque tour de
chat, Arona interroge le service pour les mémoires pertinentes pour la
conversation, les injecte dans le prompt comme section système, et — après une
réponse terminée — écrit le tour comme épisode pour que les futures
conversations puissent le rappeler.

C'est un client WebSocket JSON-RPC vers Philia (`Sync.ConnectHandshake`,
`Sync.MemoryQueryRequest`, `Sync.MemoryStoreRequest`,
`Sync.MemoryDeleteRequest`). Les connexions sont établies paresseusement,
coupées à la moindre erreur et rétablies à l'appel suivant ; chaque échec
dégrade silencieusement et **ne bloque jamais le chemin de chat**.

## Configuration

Le gateway est contrôlé par trois variables d'environnement :

| Variable | Signification |
| --- | --- |
| `ARONA_MEMORY_URL` | URL WebSocket du service scepter / Philia, p. ex. `ws://192.0.2.10:8424/ws`. |
| `ARONA_MEMORY_TOKEN` | Token pour le service mémoire. |
| `ARONA_MEMORY_WRITEBACK` | Si les tours terminés sont réécrits. Défaut **activé** ; définissez `false` pour désactiver (parsé comme booléen strict — `0` n'est pas accepté). |

`ARONA_MEMORY_URL` **et** `ARONA_MEMORY_TOKEN` doivent toutes deux être
définies et non vides, sinon le gateway est **désactivé** : le rappel et le
writeback sont entièrement ignorés et chaque requête rapporte `disabled`. Le
token est envoyé à la fois comme paramètre de requête `?token=` sur la mise à
niveau WebSocket et dans la requête `Sync.ConnectHandshake`.

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

Voir [Configuration](configuration.md) pour la référence complète de
l'environnement.

## Injection de rappel

Avec le gateway activé, **chaque tour de chat** — REST non-streaming
`/v1/chat/completions`, REST streaming (SSE) et RPC `chat.send` — interroge le
service mémoire avant que la requête ne soit transmise :

- La requête est le **dernier message utilisateur** du contexte assemblé.
- Jusqu'à **5** mémoires sont demandées (`limit = 5`).
- Les résultats sont rendus comme section système markdown intitulée
  `## Relevant Long-Term Memories`, une puce `- [score] text` par mémoire
  (scores à deux décimales, entrées vides ignorées), et préfixés à la liste de
  messages comme message `system`. L'injection est idempotente : un contexte
  qui porte déjà la section n'est pas ré-injecté.
- Si aucune mémoire pertinente n'est renvoyée, rien n'est injecté et le tour
  se déroule inchangé.

Le rappel s'exécute avant la persistance de la conversation et la transmission
à l'upstream ; un service mémoire lent ou en échec n'ajoute **aucune garantie
de latence** au-delà de son propre timeout RPC de 10 secondes et ne peut pas
faire échouer la requête.

## Writeback

Après une réponse d'assistant terminée, le tour est réécrit vers le service
mémoire comme nœud **épisode**. Le texte de l'épisode est une transcription
heuristique du tour — `User: <user content>\nAssistant: <assistant content>`
(chaque côté omis quand vide ; les deux vides ignorent le writeback). Le
writeback est **fire-and-forget** : il s'exécute dans une tâche lancée, ne
bloque jamais la réponse de chat, et ses échecs ne sont journalisés qu'à
l'intérieur du client mémoire. (Sur le chemin REST streaming, le writeback
requiert en outre qu'une conversation soit attachée à la requête ; les chemins
REST non-streaming et RPC écrivent quel que soit.)

## Contrôle par requête

Le corps de requête de chat REST et les params du RPC `chat.send` acceptent
tous deux un champ `memory` optionnel pour remplacer la configuration serveur
**par appel** :

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` — force le rappel activé / désactivé pour
  ce tour.
- omis (`null`) — suit la configuration serveur (`req.memory.unwrap_or(true)`),
  c'est-à-dire activé si et seulement si le gateway est configuré.

Le remplacement affecte le rappel ; le writeback ne suit que
`ARONA_MEMORY_WRITEBACK` plus le fait que le gateway soit activé.

## États d'en-tête

Les réponses REST portent l'état mémoire du tour dans l'en-tête de réponse
**`X-Arona-Memory`** ; la réponse RPC `chat.send` reflète la même valeur dans
un champ `memory` de son résultat. États possibles :

| Valeur | Signification |
| --- | --- |
| `enabled` | La mémoire a été demandée, le gateway est configuré, le rappel a réussi et au moins une mémoire a été injectée. |
| `disabled` | Gateway non configuré, ou `memory: false` sur la requête, ou aucun message utilisateur à interroger, ou rappel réussi mais n'ayant renvoyé **aucune** mémoire pertinente (rien à injecter). |
| `offline` | La mémoire a été demandée et le gateway est configuré, mais l'appel de rappel a échoué (connexion refusée, erreur RPC ou timeout). |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## Sémantique des échecs

Tout dégrade explicitement, dans la même direction — le chat tourne toujours :

- **Échec de rappel** — journalisé au niveau `warn` ; la requête se poursuit
  sans mémoires injectées et rapporte `offline` dans l'en-tête.
- **Échec de writeback** — journalisé à l'intérieur du client mémoire ; la
  réponse de chat n'est pas affectée.
- **Service mémoire non configuré** — le rappel et le writeback sont des no-ops ;
  chaque requête rapporte `disabled`.

Il n'existe aucun mode dans lequel une panne de mémoire fait échouer ou retarde
une requête de chat au-delà des timeouts bornés du client lui-même.

## Surface RPC

Deux méthodes de gestion sont exposées sur la surface JSON-RPC (toutes deux
requièrent un JWT ; voir l'[API JSON-RPC](../api/jsonrpc.md)) :

**`memory.status`** — instantané du gateway :

```json
{
  "enabled": true,
  "writeback": true,
  "events": [
    { "kind": "recall", "detail": "…query…", "at": "2026-01-01T00:00:00.000Z" },
    { "kind": "writeback", "detail": "…node id…", "at": "…" },
    { "kind": "error", "detail": "…", "at": "…" }
  ]
}
```

`events` est un buffer circulaire en mémoire de l'activité récente — événements
de rappel, writeback, suppression et erreur, les plus récents d'abord, jusqu'au
compte demandé (le handler de status demande les 50 derniers ; le buffer
lui-même est plafonné à 100). Ce **n'est pas** un journal d'audit durable — il
se réinitialise au redémarrage.

**`memory.delete`** — élague un nœud stocké par id :

```json
{ "node_id": "…" }
```

Renvoie `{ "deleted": true | false }`. Échoue avec une erreur quand `node_id`
est manquant ou quand le service mémoire n'est pas configuré.

## Rubriques connexes

- [Configuration](configuration.md) — les variables `ARONA_MEMORY_*`.
- [Guide de démarrage rapide](quickstart.md) — configuration de bout en bout du
  gateway.
- [Backends](backends.md) — comment les requêtes de chat sont routées avant que
  le rappel ne s'exécute.
- [Facturation et usage](billing-usage.md) — comment les mêmes tours de chat
  sont mesurés.
- [Opérations](operations.md) — logs et santé pour la connexion mémoire.
- [API JSON-RPC](../api/jsonrpc.md) — `memory.status`, `memory.delete`, `chat.send`.
- [Vue d'ensemble](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
