---
title: "Déploiement"
description: "Installer arona-server depuis un asset de release, un script ou les sources ; l'exécuter sur bare metal avec systemd, en Docker Compose, ou sous supervision malkuth ; et le mettre à niveau en toute sécurité."
---

# Déploiement

Arona est fourni sous forme d'un **binaire Rust autonome unique** construit à
partir du crate `_cli`. Le workflow de release le publie sous le nom
`arona-server-<tag>-linux-x86_64`
(`.github/workflows/release.yml`), et ce même binaire joue les deux rôles :

- **Serveur API** — `serve` (le gateway ; `migrate` applique les migrations de
  schéma explicitement) ;
- **Outillage modèles** — `install` (détection matérielle), `status`, `deploy
  <model>`, `download <repo>` et `connect <panel-url>`.

Le serveur requiert PostgreSQL. Le schéma est créé et migré automatiquement au
démarrage, donc le déploiement consiste surtout à « obtenir le binaire, le
pointer vers une base de données, l'exécuter ». Commencez par le
[guide de démarrage rapide](./quickstart.md) pour un parcours de bout en bout,
puis revenez ici pour la disposition de production.

## Prérequis

- **Linux x86_64** pour l'asset de release précompilé ; toute plateforme prise
  en charge par Rust 1.91+ peut compiler depuis les sources.
- **PostgreSQL** — l'exemple Docker Compose utilise `postgres:16-alpine` ;
  toute version récente convient. Le serveur refuse de démarrer sans
  `DATABASE_URL`.
- **`curl`, `python3`, `ca-certificates`** si vous utilisez le script
  d'installation (sur Debian/RedHat, le script les installe lui-même via
  apt/dnf).
- Un emplacement inscriptible pour le binaire (p. ex. `/usr/local/bin`) et, si
  vous exécutez des téléchargements de modèles, de l'espace disque pour le
  cache de modèles sous `ARONA_DATA_DIR`.

## Installation

### Depuis un asset de release

Téléchargez l'asset du tag souhaité et rendez-le exécutable :

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### Depuis le script d'installation

Le dépôt fournit `scripts/install.sh` : il résout le dernier tag de release
depuis l'API GitHub (ou honore le remplacement `ARONA_VERSION`), télécharge
l'asset correspondant, l'installe comme `arona-server` dans `~/.local/bin` par
défaut (remplacez le répertoire avec `ARONA_BIN_DIR`), et affiche les
prochaines étapes :

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### Depuis les sources

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` est exactement ce que le workflow de release et
le Dockerfile construisent.

## Configuration

Toute la configuration passe par des variables d'environnement ; la
[référence de configuration](./configuration.md) documente chaque variable, et
`.env.example` à la racine du dépôt est un point de départ fonctionnel. Le jeu
minimal :

| Variable | Requise ? | Rôle |
| --- | --- | --- |
| `DATABASE_URL` | oui | Chaîne de connexion PostgreSQL ; le serveur se termine si non définie. |
| `JWT_SECRET` | oui | Secret de signature des tokens ; le serveur refuse de fonctionner avec le secret de développement intégré sauf si `MOCK_MODE=1`. |
| `ARONA_ADMIN_TOKEN` | fortement recommandée | Bearer token partagé pour les routes `/api/admin/*` ; sans lui, ces routes renvoient toujours 401. |

Optionnel : `ARONA_HOST` (défaut `0.0.0.0`), `ARONA_PORT` (défaut `8420`),
`ARONA_REGISTRATION_OPEN` (truthy `1`/`true`/`yes`/`on` — ouvre l'inscription ;
le premier utilisateur enregistré devient admin), `ARONA_DATA_DIR` (racine du
cache de modèles), `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` /
`ARONA_MEMORY_WRITEBACK` (gateway mémoire), `ARONA_API_RATE_LIMIT_RPM`
(limite de débit par key) et `RUST_LOG` (filtre de tracing).

## Migrations de base de données

`serve` se connecte à la base de données et applique toutes les migrations de
schéma en attente au démarrage (`init_database` → `Migrator::up`), donc il n'y
a pas d'étape de déploiement séparée. Pour appliquer les migrations
explicitement — par exemple pour les vérifier avant le premier démarrage —
exécutez :

```bash
arona-server migrate
```

L'utilisateur de la base de données a besoin du privilège **`CREATE`** sur le
schéma cible, car la migration de démarrage crée des tables. Il n'y a pas de
migration de données au-delà du schéma : la base de données *est* l'état, donc
sauvegardez-la avant les mises à niveau (voir [Upgrade and backup](#upgrade-and-backup)).

## Bare metal avec systemd

Exemple de fichier unit (`/etc/systemd/system/arona.service`). **Toutes les
valeurs secrètes ci-dessous sont des espaces réservés — remplacez `CHANGE_ME`
avant utilisation :**

```ini
[Unit]
Description=Arona API server
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=arona
Environment=DATABASE_URL=postgres://arona:CHANGE_ME@127.0.0.1:5432/arona
Environment=JWT_SECRET=CHANGE_ME
Environment=ARONA_ADMIN_TOKEN=CHANGE_ME
Environment=ARONA_HOST=0.0.0.0
Environment=ARONA_PORT=8420
Environment=RUST_LOG=info
# Optional:
# Environment=ARONA_REGISTRATION_OPEN=1
# Environment=ARONA_MEMORY_URL=ws://127.0.0.1:8424/ws
# Environment=ARONA_MEMORY_TOKEN=CHANGE_ME
# Environment=ARONA_MEMORY_WRITEBACK=1
# Environment=ARONA_DATA_DIR=/var/lib/arona
ExecStart=/usr/local/bin/arona-server serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Ensuite, activez et démarrez-le :

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

La racine du dépôt fournit un `docker-compose.yml` avec deux services :

- **`arona`** — construit l'image depuis le `Dockerfile` (tag `arona:local`),
  publie `${ARONA_PORT:-8420}:8420`, et requiert `DATABASE_URL`,
  `JWT_SECRET` et `ARONA_ADMIN_TOKEN` — Compose échoue rapidement avec une
  erreur de style `:?` si l'un d'eux manque dans `.env`. Son healthcheck
  exécute `curl -fsS http://127.0.0.1:8420/readyz`.
- **`postgres`** — `postgres:16-alpine` avec des identifiants
  d'espace réservé uniquement (`POSTGRES_USER` / `POSTGRES_PASSWORD` /
  `POSTGRES_DB` ; le mot de passe par défaut est `change-me` — remplacez-le) et
  un volume nommé `arona-pgdata`.

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

Le service postgres fourni est joignable à l'hôte `postgres` dans le réseau
Compose. Le `Dockerfile` construit `_cli` et `_agent` dans un builder
`rust:1.91-slim-bookworm`, installe `ca-certificates`, `curl` et `python3`
dans un runtime `debian:bookworm-slim`, copie les deux binaires, expose les
ports 8420 (serveur) et 5790 (l'API locale du `_agent` fourni), et exécute
`arona-server serve` comme entrypoint.

## Déploiement supervisé avec malkuth (convention de l'espace de travail)

Dans cet espace de travail, arona tourne sous supervision **malkuth** comme
`arona-malkuth.service`. Le schéma s'applique à tout service déployé ici :

- malkuth supervise le processus `arona-server serve` — il le lance, le sonde,
  le redémarre en cas d'échec et draine les connexions avant l'arrêt.
- Chaque service supervisé n'est exposé que via un **port d'info** par service
  et un **port proxy** par service ; le service lui-même écoute sur
  `127.0.0.1` et n'est jamais joignable directement depuis le réseau.
- Le superviseur est démarré avec `--watch <deployment-path>` : quand le
  binaire à ce chemin change — p. ex. une mise à niveau copie un nouveau
  fichier par-dessus — malkuth effectue un **redémarrage progressif (rolling
  restart)**, une instance à la fois, en drainant d'abord les connexions.

Conséquences opérationnelles :

- Liez `ARONA_HOST=127.0.0.1` quand vous tournez derrière le proxy du
  superviseur ; le proxy est le seul point d'entrée exposé au réseau.
- Mettre à niveau consiste à « copier le nouveau binaire sur le chemin de
  déploiement » : le watcher déclenche le redémarrage progressif. Vous pouvez
  aussi redémarrer l'unité supervisée explicitement.
- Pointez le health check du superviseur sur `/readyz` (ou `/api/health`) ;
  voir [Health probes](#health-probes).

## Sondes de santé

Le serveur expose deux familles de santé non authentifiées (toutes deux aussi
couvertes dans le [guide des opérations](./operations.md)) :

- `GET /healthz`, `GET /readyz` (alias) et `GET /v1/health` renvoient
  `200` avec `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`.
- `GET /api/health` renvoie la forme `HealthResponse` de plana : `status`,
  `version`, `kind`, `uptime` (secondes), `network`, `build_hash` et
  `engine_version`.

Pointez les load balancers, superviseurs et healthchecks de conteneurs vers
`/readyz` ; utilisez `/api/health` quand vous avez besoin du détail uptime et
réseau.

## Mise à niveau et sauvegarde

- **Conservez le binaire précédent.** Avant d'écraser `arona-server`, copiez-le
  de côté — `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak` —
  pour pouvoir restaurer immédiatement le binaire si la nouvelle version ne
  démarre pas.
- **Sauvegardez PostgreSQL.** La base de données détient tout l'état —
  backends, nœuds d'agents, utilisateurs, keys, conversations et usage — et le
  seul changement de schéma automatique est la migration de démarrage.
  Exécutez `pg_dump` sur la base `arona` avant chaque mise à niveau.
- **L'utilisateur de la base de données a besoin des droits `CREATE`**, car les
  migrations s'exécutent au démarrage ; un rôle en lecture seule ne peut pas
  démarrer le serveur.
- **Arrêtez gracieusement.** Le serveur draine les connexions en vol sur
  SIGINT/SIGTERM, donc préférez `systemctl restart arona` ou un redémarrage du
  superviseur à `kill -9`.

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
