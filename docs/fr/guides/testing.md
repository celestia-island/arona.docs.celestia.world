---
title: "Tests"
description: "La pyramide de tests d'Arona — tests unitaires, intégration hermétique, intégration gardée par PostgreSQL, smoke tests sur serveur live, le serveur mock, et la discipline des smoke runs à identifiants réels."
---

# Tests

Les tests d'Arona sont organisés en couches pour que l'exécution par défaut de
`cargo test` soit rapide, hermétique et n'ait besoin ni de base de données ni
du réseau, tandis que les suites plus lourdes sont des opt-ins explicites qui
exercent la surface filaire réelle et un PostgreSQL réel. Cette page fait le
plan des couches, des commandes qui les exécutent, et de la discipline de
l'espace de travail autour des smoke runs à identifiants réels.

## Tests unitaires

L'essentiel de la couverture consiste en tests unitaires simples dans
`packages/core/src` : 217 fonctions `#[test]` / `#[tokio::test]`, plus ~23
autres dans `packages/agent` et `packages/cli`. Elles s'exécutent avec :

```bash
cargo test --workspace
```

Pas de réseau, pas de base de données. Suites clés :

- **auth.rs** — la politique de mot de passe (≥8 caractères ET ≥3 des 4
  catégories de caractères), les casts `::uuid` dans le SQL brut
  INSERT/REVOKE, les défauts de requête, et les lectures de drapeau admin qui
  retombent sur `false`.
- **billing/mod.rs** — le calcul de quota sur la dimension coût *ou* tokens, la
  fenêtre mensuelle (`month_start`, `seconds_until_month_end`), le plafond de
  limite de débit (déclenche uniquement *à* la valeur RPM, `None` = illimité),
  les gardes de forme SQL pour les requêtes d'usage mensuel / tier / fenêtre de
  key, et `estimate_usage` préférant les nombres rapportés par l'upstream.
- **routing/mod.rs** — la résolution d'alias, la correspondance de suffixe
  `:latest`, les indices de provider, la sélection la moins chargée et
  l'épinglage de conversation.
- **gateway/mod.rs** — l'enregistrement des backends d'agents : enregistrer
  `agent-{model_id}`, le ré-enregistrement remplaçant (pas dupliquant), et le
  désenregistrement restaurant le routeur.

## Intégration hermétique (toujours exécutée, sans DB)

`packages/core/tests/gateway_integration.rs` contient trois tests toujours
exécutés qui exercent la logique réelle de sérialisation/contrat sans toucher
à une base de données :

- **A1** — le contrat de sérialisation d'écho d'id JSON-RPC : les ids de
  requête numériques, chaînes et null font l'aller-retour à travers l'enum
  `Id` de plana avec fidélité de type.
- **A2** — le contrat de codes d'erreur de la porte admin : `AUTH_ERROR`
  (-32005, anonyme) et `ADMIN_REQUIRED` (-32007, non-admin authentifié)
  restent distincts, vivent dans la plage définie par l'implémentation, et ne
  rentrent jamais en collision avec les codes de plana ou le `QUOTA_ERROR`
  (-32006) de facturation.
- **A3** — `estimate_usage` : l'usage rapporté par l'upstream gagne verbatim ;
  sans lui, l'estimation du tokenizer local produit des comptes
  prompt/completion non nuls dont le total est leur somme.

`packages/core/tests/smoke.rs` ajoute trois autres tests toujours exécutés :
la détection matérielle, le chemin racine du registre de modèles, et les
défauts de configuration sous `MOCK_MODE=1`.

## Intégration gardée par PG

La suite complète du gateway in-process — `packages/core/tests/gateway_integration.rs`
— fait tourner le routeur axum entier sur un port loopback aléatoire,
enregistre des mock upstreams compatibles OpenAI jetables via la vraie API
admin, et pilote la surface filaire avec reqwest. Parce que `AuthManager`
parle à PostgreSQL sur chaque chemin (même `MOCK_MODE=1` ne fait que seeder
les comptes *dans la base de données*), cette suite est gardée derrière
`ARONA_TEST_PG=1` et ignorée par défaut. Les 10 tests :

- **T1** register + login + `keys.create`/`keys.list` (key brute masquée dans
  les listings, préfixe `arona-`).
- **T2** chat REST avec persistance d'enregistrements d'usage dans PostgreSQL.
- **T3** écho d'id JSON-RPC sur le fil (chemins succès et erreur).
- **T4** porte admin sur `agents.list` : anonyme → `AUTH_ERROR`, non-admin →
  `ADMIN_REQUIRED`.
- **T5** upstream 401 → HTTP 502 `bad_gateway` avec le détail upstream.
- **T6** la sonde au moment de l'enregistrement publie les modèles (le modèle
  apparaît dans `GET /v1/models` en 10 s sans liste statique de modèles).
- **T7** persistance de conversation via `chat.send` (les deux tours atterrissent
  dans `conversations.get`).
- **T8** porte de facturation du tier free : 10 RPM par key, la 11e requête
  dans la fenêtre est 429 `rate_limit_exceeded`.
- **T9** stream SSE avec usage terminal enregistré depuis l'upstream.
- **T10** JSON malformé → 400 ; modèle inconnu → 404 `model_not_found`.

Exécutez-la avec le one-liner Postgres jetable des docs de module
(gateway_integration.rs:18-26) :

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

Ce sont des identifiants d'exemple pour le conteneur de test jetable uniquement
— ne pointez jamais cela vers une vraie base de données.

## Smoke test sur serveur live

`packages/core/tests/auth_flow.rs` parcourt la chaîne complète
`register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` contre un serveur Arona **live**, reflétant la boucle d'auth
déployée. Il est `#[ignore]`d par défaut — l'exécution simple de `cargo test`
ne touche jamais le réseau. Exécutez-le explicitement :

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

Réglages :

- `ARONA_TEST_URL` — URL de base (défaut `http://127.0.0.1:8420`).
- `ARONA_TEST_EXPECT_CHAT=1` — asserte durement que `POST /v1/chat/completions`
  renvoie 200. Sans lui, le test n'asserte que le
  passage de l'auth (pas 401/403), car l'environnement cible peut n'avoir aucun
  provider d'inférence configuré.

La suite inclut aussi des tests négatifs : une complétion de chat non
authentifiée et un `GET /v1/models` non authentifié doivent tous deux être
rejetés avec 401.

## Serveur mock

`scripts/mock/server.py` est un faux compatible OpenAI basé sur aiohttp utilisé
par le guide de démarrage rapide et les smoke runs. Il sert `POST /v1/chat/completions`
(non-stream et SSE), `GET /v1/models`, `GET /api/health`, la surface WebSocket/HTTP JSON-RPC à `/api/rpc`, un sidecar SSE à
`/api/rpc/events`, et `GET /api/test-key`, qui renvoie l'API key du mock pour
que d'autres services puissent la découvrir. Il écoute sur le port 8429 par
défaut (remplaçable avec `ARONA_MOCK_PORT`, hôte avec `ARONA_MOCK_HOST`). Le
[guide de démarrage rapide](quickstart.md) l'utilise pour monter un
environnement de bout en bout sans vrais providers de modèles.

## Discipline des smoke runs à identifiants réels

Les smoke runs contre de vrais providers (DeepSeek / GLM) ne sont délibérément
**pas** des tests de dépôt — ils requièrent de vrais identifiants et de
l'argent réel, donc ils ne peuvent pas vivre dans CI ni dans l'arbre git. La
convention de l'espace de travail, documentée dans les docs de module
gateway_integration (gateway_integration.rs:54-55), est :

- Les fichiers de preuve vivent sous `/mnt/work/arona-pr*-smoke.md` — locaux à
  l'espace de travail, jamais commités dans git.
- Les identifiants ne viennent que de l'environnement ; les budgets sont
  gardés petits.
- Chaque PR qui touche le chemin d'inférence reçoit un dossier de preuve écrit.

Le serveur mock est le remplaçant de ces runs en CI et en développement local ;
le smoke run à identifiants réels est une étape humaine au moment de la
release.

## CI

`.github/workflows/ci.yml` exécute `cargo fmt`, `cargo clippy`, `cargo test
--workspace` et `cargo-deny` sur les runners auto-hébergés de l'org
(`[self-hosted, linux, x64, local]`) ; `ci-hosted.yml` reflète les mêmes
vérifications sur les runners hébergés par GitHub. `.github/workflows/docs.yml`
construit ce site de documentation avec lagrange et le déploie sur GitHub
Pages lors des push touchant `docs/**`.

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
