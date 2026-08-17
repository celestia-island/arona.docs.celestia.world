---
title: "Facturation et usage"
description: "Enregistrements d'usage, coût par modèle, tiers de facturation, application des quotas et limites de débit, keys scopées par projet, tarification vidéo et le RPC usage.list."
---

# Facturation et usage

Arona mesure chaque requête de modèle et applique des quotas et limites de
débit par tier au niveau du gateway. Les prix par modèle viennent de la table
de tarification partagée plana (jamais réimplémentée dans arona), l'usage est
persisté comme lignes `usage_records`, et l'ensemble du tableau mensuel est
exposé via le RPC `usage.list`.

## Enregistrements d'usage

Chaque requête mesurée aboutit à une ligne dans la table `usage_records`
(`m20250101_000006_create_usage_records`) :

| Colonne | Type | Signification |
| --- | --- | --- |
| `id` | `UUID` | Clé primaire, générée. |
| `api_key_id` | `VARCHAR(64)` | Le **préfixe de key** — les 8 premiers caractères de l'API key (les keys ressemblent à `arona-{uuid}`) — ou un id synthétique `jwt-<user-uuid>` pour les canaux RPC attribués par JWT. |
| `model` | `VARCHAR(128)` | Id de modèle vers lequel la requête a été routée. |
| `backend` | `VARCHAR(64)` | Type de backend : `gateway`, `rpc`, `realtime`, ou le nom de capacité du backend. |
| `prompt_tokens` | `INTEGER` | Tokens d'entrée, rapportés par l'upstream ou estimés. |
| `completion_tokens` | `INTEGER` | Tokens de sortie, rapportés par l'upstream ou estimés. |
| `total_tokens` | `INTEGER` | Somme des deux. |
| `cost` | `DOUBLE PRECISION` | Coût USD calculé ; `NULL` quand le modèle n'a pas de ligne de tarification. |
| `created_at` | `TIMESTAMPTZ` | Quand la requête s'est terminée. |

Des index existent sur `api_key_id`, `model` et `created_at` (les colonnes que
l'agrégation mensuelle et les fenêtres de limite de débit scannent).

## Canaux d'enregistrement

L'usage est enregistré sur chaque canal mesuré :

1. **REST non-streaming** — `POST /v1/chat/completions` et
   `POST /v1/embeddings` enregistrent l'usage exact rapporté par l'upstream une
   fois la réponse produite.
2. **REST streaming (SSE)** — l'usage rapporté par l'upstream gagne quand le
   stream le portait (champ `usage` du chunk terminal compatible OpenAI) ;
   sinon, une estimation locale par tokenizer conscient du CJK
   (`estimate_usage`) est enregistrée telle quelle. Les streams qui n'ont
   produit ni texte ni usage **ne sont pas** enregistrés du tout.
3. **RPC `chat.send`** — la même logique estimation-vs-upstream s'applique ;
   les lignes sont attribuées avec l'id synthétique `jwt-<user-uuid>` pour être
   rattachées à l'utilisateur.
4. **Sessions realtime** — chaque transcription `response_done` terminée
   enregistre son usage de tokens (quand il est non nul) sous
   `jwt-<user-uuid>` avec le backend `realtime`.
5. **Jobs vidéo** — un job terminé enregistre un coût en dollars explicite
   (voir [Video pricing](#video-pricing)) ; les compteurs de tokens
   sont à zéro.

L'enregistrement est best-effort : un insert échoué est journalisé et ne fait
jamais échouer la requête.

## Calcul du coût

Le coût est calculé depuis la table de tarification canonique par million de
tokens (`plana_llm_provider::metering::lookup_pricing`, partagée par tous les
services celestia-island) :

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

La correspondance de modèles dans la table est par sous-chaîne sur l'id de
modèle en minuscules (les familles plus spécifiques gagnent). Quand un modèle
n'a pas de ligne de tarification, `cost` est `NULL`. **Ne réimplémentez pas la
tarification dans arona — mettez à jour la table de plana.**

## Tiers

Les tiers vivent dans la table `billing_tiers`, seedée à la première migration
(`m20250101_000007_create_billing_tiers`). Une colonne de quota `NULL`
signifie « illimité » pour cette dimension. Les utilisateurs sans `tier_id`
retombent sur le tier `free` seedé.

| Tier | Quota mensuel USD | Quota mensuel de tokens | RPM par key |
| --- | --- | --- | --- |
| `free` | 1,00 $ | 100 000 | 10 |
| `pro` | 20,00 $ | 5 000 000 | 120 |
| `enterprise` | illimité (`NULL`) | illimité (`NULL`) | 1000 |

L'attribution de tier est une opération admin (RPC `billing.plan.set`) ; le
tier courant et l'usage sont remontés via `billing.plan`.

## Application des quotas et limites de débit

### REST (`/v1/*`)

Avant chaque endpoint REST **mesuré** — `/v1/chat/completions`,
`/v1/embeddings` et `/v1/video/generations` — le gateway applique deux portes
pour les requêtes authentifiées par key :

- **Quota mensuel** — les limites `monthly_quota_usd` et `quota_tokens` du
  tier sont vérifiées contre l'usage accumulé depuis la première seconde du
  mois calendaire courant. Chaque dimension atteignant sa limite déclenche la
  porte.
- **Limite de débit par key** — le `rate_limit_rpm` du tier est vérifié contre
  le nombre de requêtes enregistrées pour ce préfixe de key dans la fenêtre des
  60 dernières secondes. (`/v1/models` et les endpoints de santé ne sont pas
  mesurés et ne sont pas soumis aux portes.)

Un rejet est un HTTP **429 Too Many Requests** avec un en-tête `Retry-After` et
un corps d'erreur de style OpenAI :

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| Rejet | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| Quota mensuel épuisé | `quota_error` / `quota_exceeded` | Secondes jusqu'au début du **prochain mois calendaire** |
| Limite de débit de tier dépassée | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

Le `chat.send` authentifié par JWT passe par la même porte de quota mensuel,
mais contre la fenêtre **de tout l'utilisateur** (l'appel ne porte pas d'API
key). Un rejet est une erreur JSON-RPC avec le code défini par
l'implémentation `-32006` (`QUOTA_ERROR`) et le même message que le rejet de
quota REST. Il n'y a pas de limite de débit par key sur le chemin RPC — la
limitation de débit est scopée par key et les appels RPC n'ont pas de key. Les
méthodes **RPC** realtime et vidéo ne sont pas soumises au quota.

## Compromis fail-open

La facturation est **best-effort par conception**. Si la requête de base de
données derrière un contrôle de quota ou de limite de débit échoue, le contrôle
renvoie `Unknown` et la requête est **autorisée** (seulement journalisée) au
lieu de bloquer le chat. Un opérateur peut compter sur les 429 pour protéger la
capacité, mais ne doit pas les traiter comme une garantie dure quand la base de
données est en mauvaise santé — le compromis documenté est la disponibilité du
chemin de chat au détriment d'une mesure stricte.

## Keys scopées par projet

Les API keys peuvent être créées avec un libellé `project`
(`api_keys.project`, `default` quand non défini). L'application du quota le
respecte :

- Une key marquée d'un projet autre que `default` vérifie son quota contre
  l'usage attribué au **bucket propre à ce projet** (`PROJECT_MONTHLY_USAGE_SQL`).
- Les keys `default` / non marquées gardent la fenêtre **de tout l'utilisateur**,
  conformément au comportement d'avant-projet.

Les lignes RPC attribuées par JWT (`jwt-<user-uuid>`) ne portent pas de
libellé de projet et sont **intentionnellement exclues** des fenêtres de
projet — elles comptent toujours dans la fenêtre de tout l'utilisateur, donc un
projet ne peut pas être « caché » en envoyant du trafic via le canal RPC.

## Tarification vidéo

La génération vidéo utilise une tarification par modèle, de style tâche (la
tarification par token n'a pas de sens pour une vidéo). Les lignes de
tarification vivent dans la table `video_pricing` ; `compute_cost` retombe sur
un défaut d'espace réservé quand aucune ligne n'est configurée.

| Mode | Coût |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (défaut) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` est un objet JSON indexé par la valeur de pixel du petit
côté (p. ex. `"768"`) ; la clé `"*"` est le repli. La ligne de tarification par
défaut est le mode `per_second_resolution`, `base_price` 0.0,
`price_per_second` 0.005, `resolution_coeff {"*": 1.0}`. Les lignes sont gérées
via `billing.video.pricing.get` (tout JWT) et `billing.video.pricing.set`
(Bearer `ARONA_ADMIN_TOKEN`) ; le coût calculé est enregistré contre la key du
job quand le job se termine.

## usage.list

`usage.list` (JWT) renvoie les enregistrements d'usage paginés de l'appelant
couvrant **à la fois** les lignes par API key (jointes via le préfixe de key)
et les lignes attribuées par JWT (jointes via l'id synthétique
`jwt-<user-uuid>`), les plus récentes d'abord :

| Param | Défaut | Notes |
| --- | --- | --- |
| `limit` | `50` | Borné à `1..=200`. |
| `offset` | `0` | Décalage de page. |
| `project` | non défini | Quand défini, seuls les enregistrements attribués aux keys portant ce libellé de projet (les lignes JWT sont exclues). |

La réponse est `{ "records": [...], "total", "limit", "offset", "project" }`
où chaque enregistrement porte `id`, `model`, `backend`, `prompt_tokens`,
`completion_tokens`, `total_tokens`, `cost` et `created_at`. L'agrégation du
quota mensuel utilise la même forme de jointure, donc le calcul du quota et la
vue d'usage s'accordent toujours sur le périmètre.

## Rubriques connexes

- [Guide de démarrage rapide](quickstart.md) — obtenir une key et faire votre
  première requête mesurée.
- [Configuration](configuration.md) — les variables d'environnement du gateway.
- [Authentification et sécurité](auth-security.md) — la création d'API keys et
  le libellé `project`.
- [Realtime et vidéo](realtime-video.md) — le cycle de vie des jobs vidéo
  derrière la tarification vidéo.
- [Opérations](operations.md) — les sondes de santé et l'observabilité.
- [API REST compatible OpenAI](../api/openai-rest.md) — la surface `/v1/*`.
- [API JSON-RPC](../api/jsonrpc.md) — `usage.list`, `billing.plan`,
  `billing.video.pricing.*`.
- [Vue d'ensemble](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
