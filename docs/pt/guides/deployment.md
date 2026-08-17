---
title: "Implantação"
description: "Instale o arona-server a partir de um asset de release, script ou código-fonte; execute-o em bare metal com systemd, em Docker Compose ou sob supervisão do malkuth; e faça upgrade com segurança."
---

# Implantação

O Arona é distribuído como um **binário Rust único autocontido** construído a
partir do crate `_cli`. O workflow de release o publica como
`arona-server-<tag>-linux-x86_64`
(`.github/workflows/release.yml`), e o mesmo binário desempenha ambos os papéis:

- **Servidor de API** — `serve` (o gateway; `migrate` aplica migrações de
  schema explicitamente);
- **Ferramentas de modelo** — `install` (detecção de hardware), `status`,
  `deploy <model>`, `download <repo>` e `connect <panel-url>`.

O servidor requer PostgreSQL. O schema é criado e migrado automaticamente na
inicialização, então implantar é basicamente "pegue o binário, aponte-o para um
banco de dados, execute-o". Comece pelo [quickstart](./quickstart.md) para um
passo a passo completo e volte aqui para o layout de produção.

## Requisitos

- **Linux x86_64** para o asset de release pré-compilado; qualquer plataforma
  suportada por Rust 1.91+ pode compilar a partir do código-fonte.
- **PostgreSQL** — o exemplo de Docker Compose usa `postgres:16-alpine`;
  qualquer versão recente funciona. O servidor se recusa a iniciar sem
  `DATABASE_URL`.
- **`curl`, `python3`, `ca-certificates`** se você usar o script de instalação
  (em Debian/RedHat o script os instala sozinho via apt/dnf).
- Um local gravável para o binário (ex. `/usr/local/bin`) e, se você fizer
  downloads de modelos, espaço em disco para o cache de modelos sob
  `ARONA_DATA_DIR`.

## Instalação

### A partir de um asset de release

Baixe o asset para o tag desejado e torne-o executável:

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### A partir do script de instalação

O repositório inclui `scripts/install.sh`: ele resolve o tag de release mais
recente pela API do GitHub (ou honra o override `ARONA_VERSION`), baixa o asset
correspondente, instala-o como `arona-server` em `~/.local/bin` por padrão
(substitua o diretório com `ARONA_BIN_DIR`) e imprime os próximos passos:

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### A partir do código-fonte

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` é exatamente o que o workflow de release e o
Dockerfile compilam.

## Configuração

Toda a configuração é por variáveis de ambiente; a [referência de
configuração](./configuration.md) documenta cada variável, e `.env.example` na
raiz do repositório é um ponto de partida funcional. O conjunto mínimo:

| Variável | Obrigatória? | Finalidade |
| --- | --- | --- |
| `DATABASE_URL` | sim | String de conexão PostgreSQL; o servidor sai se não estiver definida. |
| `JWT_SECRET` | sim | Segredo de assinatura de token; o servidor se recusa a rodar com o segredo de desenvolvimento integrado a menos que `MOCK_MODE=1`. |
| `ARONA_ADMIN_TOKEN` | fortemente recomendada | Bearer token compartilhado para as rotas `/api/admin/*`; sem ela, essas rotas sempre retornam 401. |

Opcionais: `ARONA_HOST` (padrão `0.0.0.0`), `ARONA_PORT` (padrão `8420`),
`ARONA_REGISTRATION_OPEN` (truthy `1`/`true`/`yes`/`on` — abre o cadastro;
o primeiro usuário registrado vira admin), `ARONA_DATA_DIR` (raiz do cache de
modelos), `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`
(gateway de memória), `ARONA_API_RATE_LIMIT_RPM` (limite de taxa por chave) e
`RUST_LOG` (filtro de tracing).

## Migrações de banco de dados

`serve` conecta-se ao banco e aplica todas as migrações de schema pendentes na
inicialização (`init_database` → `Migrator::up`), então não há etapa de deploy
separada. Para aplicar migrações explicitamente — por exemplo, para verificá-las
antes do primeiro start — execute:

```bash
arona-server migrate
```

O usuário do banco precisa de privilégio **`CREATE`** no schema alvo, porque a
migração de inicialização cria tabelas. Não há migração de dados além do schema:
o banco de dados *é* o estado, então faça backup antes de upgrades (veja
[Upgrade e backup](#upgrade-and-backup)).

## Bare metal com systemd

Arquivo de unit de exemplo (`/etc/systemd/system/arona.service`). **Todos os
valores secretos abaixo são placeholders — substitua `CHANGE_ME` antes do uso:**

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

Depois habilite e inicie:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

A raiz do repositório inclui um `docker-compose.yml` com dois serviços:

- **`arona`** — compila a imagem a partir do `Dockerfile` (tag `arona:local`),
  publica `${ARONA_PORT:-8420}:8420` e requer `DATABASE_URL`, `JWT_SECRET` e
  `ARONA_ADMIN_TOKEN` — o Compose falha rápido com um erro estilo `:?` se
  qualquer uma estiver faltando no `.env`. Seu healthcheck executa
  `curl -fsS http://127.0.0.1:8420/readyz`.
- **`postgres`** — `postgres:16-alpine` com credenciais apenas-placeholder
  (`POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`; a senha padrão é
  `change-me` — substitua-a) e um volume nomeado `arona-pgdata`.

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

O serviço postgres agrupado é alcançável no host `postgres` dentro da rede do
Compose. O `Dockerfile` compila `_cli` e `_agent` em um builder
`rust:1.91-slim-bookworm`, instala `ca-certificates`, `curl` e `python3` em um
runtime `debian:bookworm-slim`, copia ambos os binários, expõe as portas 8420
(servidor) e 5790 (a API local do `_agent` agrupado) e executa
`arona-server serve` como entrypoint.

## Implantação supervisionada com malkuth (convenção do workspace)

Neste workspace, o arona roda sob supervisão do **malkuth** como
`arona-malkuth.service`. O padrão se aplica a qualquer serviço implantado aqui:

- o malkuth supervisiona o processo `arona-server serve` — ele o spawna,
  sonda-o, o reinicia em caso de falha e drena conexões antes do shutdown.
- Cada serviço supervisionado é exposto apenas por uma **porta de info**
  por serviço e uma **porta de proxy** por serviço; o próprio serviço faz bind
  em `127.0.0.1` e nunca é alcançável diretamente da rede.
- O supervisor é iniciado com `--watch <deployment-path>`: quando o binário
  naquele caminho muda — ex. um upgrade copia um novo arquivo por cima dele — o
  malkuth executa um **restart rolante**, uma instância por vez, drenando
  conexões primeiro.

Consequências operacionais:

- Faça bind de `ARONA_HOST=127.0.0.1` ao rodar atrás do proxy do supervisor;
  o proxy é o único ponto de entrada voltado para a rede.
- Fazer upgrade é "copiar o novo binário sobre o caminho de deployment": o
  watcher dispara o restart rolante. Você também pode reiniciar a unit
  supervisionada explicitamente.
- Aponte o health check do supervisor para `/readyz` (ou `/api/health`); veja
  [Health probes](#health-probes).

## Health probes

O servidor expõe duas famílias de health sem autenticação (ambas também
cobertas no [guia de operações](./operations.md)):

- `GET /healthz`, `GET /readyz` (aliases) e `GET /v1/health` retornam `200`
  com `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`.
- `GET /api/health` retorna o shape `HealthResponse` do plana: `status`,
  `version`, `kind`, `uptime` (segundos), `network`, `build_hash` e
  `engine_version`.

Aponte load balancers, supervisores e healthchecks de contêiner para `/readyz`;
use `/api/health` quando precisar de uptime e detalhes de rede.

## Upgrade e backup

- **Mantenha o binário anterior.** Antes de sobrescrever `arona-server`,
  copie-o de lado — `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak` —
  para que você possa reverter o binário imediatamente se a nova versão falhar
  ao iniciar.
- **Faça backup do PostgreSQL.** O banco contém todo o estado — backends, nós
  de agent, usuários, chaves, conversas e uso — e a única mudança de schema
  automática é a migração de inicialização. Faça `pg_dump` do banco `arona`
  antes de todo upgrade.
- **O usuário do banco precisa de direitos `CREATE`**, porque as migrações
  rodam na inicialização; um papel somente-leitura não consegue dar boot no
  servidor.
- **Desligue de forma graciosa.** O servidor drena conexões em voo em
  SIGINT/SIGTERM, então prefira `systemctl restart arona` ou um restart do
  supervisor em vez de `kill -9`.

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
