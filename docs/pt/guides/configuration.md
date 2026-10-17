---
title: "Configuração"
description: "Referência do arquivo de configuração TOML fixo e de todas as variáveis de ambiente lidas pelo servidor Arona, com padrões e semântica."
---

# Configuração
O Arona lê sua configuração de um **arquivo TOML fixo**, com variáveis de ambiente atuando como substituições por chave. Cada chave resolve como **variável de ambiente explícita → valor TOML → padrão embutido**; implantações existentes baseadas em env (unidades systemd, docker-compose, `.env.example`) continuam funcionando sem mudanças, e novas implantações podem fixar tudo em um único arquivo.

## O arquivo de configuração

O arquivo é descoberto nesta ordem — o primeiro encontrado vence:
1. o caminho em `ARONA_CONFIG` (deve existir; um arquivo ausente ou ilegível é um erro fatal de inicialização);
2. `./arona.toml` no diretório de trabalho;
3. `/etc/arona/arona.toml`;
4. nenhum → modo legado somente variáveis de ambiente.

Não há **fallback silencioso**: um arquivo descoberto que falhe ao analisar, contenha uma chave desconhecida ou mantenha uma interpolação `${VAR}` não resolvida aborta a inicialização com um erro fatal. A interpolação `${VAR}` / `${VAR:-default}` permite que os segredos fiquem no ambiente ou num `LoadCredential` do systemd enquanto o arquivo contém apenas referências. Um modelo comentado é distribuído como `arona.toml.example` na raiz do repositório.

Chaves booleanas (`registration_open`, `writeback`, `backend_local`, `mistralrs_force`, `mock`) devem ser booleanos TOML no arquivo; suas formas env mantêm a semântica de string truthy ou de presença documentada abaixo. As variantes `ARONA_RUSTFS_ACCESS_KEY_FILE` / `ARONA_RUSTFS_SECRET_KEY_FILE` continuam somente via ambiente.

A seção auth também carrega dois blocos **de valor estruturado** sem forma env: `[auth.oauth]` (o provedor de retransmissão OAuth2 — URLs dos endpoints, credenciais do cliente, escopos, política de cadastro) e `[[auth.trusted_issuers]]` (painéis cujos tokens kirino ACCESS são aceitos diretamente — emissor, segredo compartilhado, audience opcional). Ambos são validados na inicialização e documentados em [Autenticação e segurança](./auth-security.md).

| Variável de ambiente | Chave TOML |
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

O restante desta página é a **referência de variáveis de ambiente**; também serve como referência semântica das chaves TOML mapeadas (mesmos nomes, mesmos padrões).

## Tabela de referência

| Variável | Padrão | Finalidade |
| --- | --- | --- |
| `DATABASE_URL` | *(obrigatória)* | URL de conexão PostgreSQL. |
| `JWT_SECRET` | *(obrigatória fora do modo mock)* | Segredo usado para assinar JWTs. |
| `ARONA_HOST` | `0.0.0.0` | Endereço de bind (cai para `SHITTIM_CHEST_HOST`). |
| `ARONA_PORT` | `8420` | Porta de bind (cai para `SHITTIM_CHEST_PORT`). |
| `ARONA_DATA_DIR` | não definida | Diretório local de dados. |
| `ARONA_ADMIN_TOKEN` | não definida | Bearer token para `/api/admin/*` e os métodos RPC de admin. |
| `ARONA_REGISTRATION_OPEN` | `0` | Truthy (`1`/`true`/`yes`/`on`) abre o cadastro. |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | Limite em memória por chave de requisições por minuto; `0` bloqueia tudo. |
| `MOCK_MODE` | não definida | Presença (qualquer valor) habilita o modo mock de dev. |
| `MOCK_SEED_PATH` | não definida | Arquivo de seed SQL bruto executado no modo mock. |
| `ARONA_MEMORY_URL` | não definida | URL WebSocket do gateway de memória Philia. |
| `ARONA_MEMORY_TOKEN` | não definida | Token para o gateway de memória. |
| `ARONA_MEMORY_WRITEBACK` | `true` | Grava rodadas de chat concluídas de volta na memória; aceita `true`/`false` (qualquer outro valor cai para o padrão). |
| `ARONA_AGENT_NAME` | `arona-agent` | Identidade do agent de nó de GPU. |
| `ARONA_PANEL_URL` | `localhost:8080` | Onde o agent se conecta (`ws://<panel_url>/ws/agent`). |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | Agent evernight local para URLs de backend `evernight://`. |
| `ARONA_MISTRALRS` | não definida | Presença força o engine Mistral.rs para planos de modelo Gguf. |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | Interface à qual um servidor de modelo llama.cpp spawnado faz bind. |
| `HF_ENDPOINT` | `https://huggingface.co` | URL base do Hugging Face para downloads de modelos. |
| `GITHUB_TOKEN` | não definida | Access token para o registry de modelos do GitHub. |
| `RUST_LOG` | não definida | Filtro de tracing, ex. `info` ou `arona=debug,info`. |

## Variáveis obrigatórias

### `DATABASE_URL`

URL de conexão PostgreSQL. **Obrigatória**: o servidor sai com
`FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.` quando ela
está vazia, e o subcomando CLI `migrate` se recusa a rodar. O schema é
criado/atualizado automaticamente pelo `serve` na inicialização.

### `JWT_SECRET`

Segredo usado para assinar os pares JWT access/refresh emitidos por `auth.login`
e `auth.register`. **Obrigatória em produção**: o código embute um fallback de
desenvolvimento (`dev-secret-not-for-production-use-only-32chars`), mas o
servidor se recusa a iniciar com ele a menos que `MOCK_MODE=1`:

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

Use um valor longo e aleatório (ex. `openssl rand -hex 32`).

## Servidor

### `ARONA_HOST` / `ARONA_PORT`

Endereço e porta de bind do gateway. Para compatibilidade legada, eles caem
para `SHITTIM_CHEST_HOST` / `SHITTIM_CHEST_PORT`; os padrões finais são (ordem: `server.host`/`server.port` do arquivo de configuração têm precedência sobre os apelidos legados; apenas `ARONA_HOST`/`ARONA_PORT` canônicos têm precedência sobre o arquivo)
`0.0.0.0:8420`.

### `ARONA_DATA_DIR`

Diretório local de dados opcional, carregado no estado do app para componentes
que precisam de um local de scratch. Não definido por padrão.

## Segurança e controle de acesso

### `ARONA_ADMIN_TOKEN`

Bearer token que protege as rotas HTTP `/api/admin/*` (`POST/GET/DELETE
/api/admin/backends`, `/api/admin/aliases`) e os métodos RPC
`billing.plan.set` / `billing.video.pricing.set`. Quando ela está **não
definida**, todas essas rotas rejeitam com `Admin access required` (401) — não
há padrão. Defina um valor aleatório forte antes de iniciar o servidor.

### `ARONA_REGISTRATION_OPEN`

Abre o cadastro self-service via `auth.register`. Valores truthy são
exatamente `1`, `true`, `yes`, `on` (insensível a maiúsculas/minúsculas);
qualquer outra coisa — incluindo `0`, `false`, `off` ou uma variável
não definida/vazia — permanece fechado. A flag é lida uma vez na inicialização.
O **primeiro usuário registrado é sempre permitido** (mesmo quando o registro
está fechado) e vira o admin.

### `ARONA_API_RATE_LIMIT_RPM`

Limite em memória de janela deslizante por chave (requisições por minuto),
aplicado a toda requisição `/v1/*` autenticada (chat, embeddings, vídeo,
models), chaveado pelo hash da API key (ou o rótulo `u:<email>` para o
`GET /v1/models` que aceita JWT). O plano RPC não é limitado por este limitador
— os extractors de auth de `/v1/*` são os únicos callers. Padrão `60`. O valor
é parseado uma vez em um `OnceLock` de processo. **Um valor de `0` bloqueia
toda requisição** — a verificação é `entry.len() >= rpm`, então com `0` nenhuma
requisição passa. Este é o limite geral do gateway; os tiers de billing impõem
seus próprios limites por chave por cima.

## Desenvolvimento

### `MOCK_MODE`

Modo mock de dev, habilitado por **presença** — a verificação é
`std::env::var("MOCK_MODE").is_ok()`, então *qualquer* valor (incluindo `0` ou
um valor vazio-mas-definido) o habilita. Ele:

- remove o requisito de `JWT_SECRET` (o segredo de dev integrado passa a ser
  aceitável);
- semeia quatro contas demo (`demiurge@celestia.world`, `momoi@celestia.world`,
  `midori@celestia.world`, `yuzu@celestia.world`, senha `33550336`);
- espera o seed concluir antes de fazer bind do listener.

Nunca o use em produção.

### `MOCK_SEED_PATH`

Apenas no modo mock, aponta para um arquivo SQL bruto executado no lugar do
seed de contas integrado (declarações divididas em `;`, comentários `--`
ignorados). Se o arquivo não puder ser lido, o seed integrado é usado como
fallback.

## Gateway de memória

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

Configuração do gateway de memória de longo prazo (entelecheia Philia). A
memória fica **totalmente desabilitada** a menos que `ARONA_MEMORY_URL` e
`ARONA_MEMORY_TOKEN` estejam ambas definidas e não vazias. Quando habilitada:

- rodadas de chat concluídas são recuperadas e injetadas como contexto, e
- `ARONA_MEMORY_WRITEBACK` (padrão `true`) controla se rodadas concluídas são
  gravadas de volta no serviço de memória; `0` ou `false` desabilita o
  writeback.

Falhas de memória nunca bloqueiam o chat; o estado resultante é ecoado no
header de resposta `X-Arona-Memory` (`enabled` / `disabled` / `offline`).

## Identidade do agent e cluster

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

Identidade do binário de agent de nó de GPU (`_agent`): `ARONA_AGENT_NAME`
(padrão `arona-agent`) é reportado ao panel como o nome/id do agent, e
`ARONA_PANEL_URL` (padrão `localhost:8080`) é onde o agent se conecta
(`ws://<panel_url>/ws/agent`).

A própria API HTTP do agent está **hardcoded** para fazer bind em `0.0.0.0:5790`
— não há variável de ambiente de endereço de bind para ela.

### `ARONA_AGENT_BIND_ADDR`

Interface à qual um **servidor de modelo llama.cpp spawnado** faz bind quando o
agent implanta um modelo Gguf, para que o engine possa ser alcançado de outras
máquinas (ex. `0.0.0.0`). Padrão `127.0.0.1`. Note que este *não* é o bind da
API HTTP do agent (que é fixo em `0.0.0.0:5790`).

## Bridge do evernight

### `ARONA_EVERNIGHT_URL`

URL WebSocket do agent evernight local usado para resolver URLs de backend
`evernight://` em forwards TCP locais. Padrão `ws://127.0.0.1:3001/ws`.

## Runtime de modelos e downloads

### `ARONA_MISTRALRS`

Presença (qualquer valor) força o engine Mistral.rs para planos de modelo Gguf
que de outra forma usariam llama.cpp. Semântica de presença como `MOCK_MODE`.

### `HF_ENDPOINT`

URL base para downloads de modelos do Hugging Face (fontes `hf:`), padrão
`https://huggingface.co`. Defina um mirror como `https://hf-mirror.com` quando
huggingface.co estiver inacessível. Lida pelo downloader de modelos; uma barra
final é removida.

### `GITHUB_TOKEN`

Access token usado pelo registry de modelos do GitHub (fontes `gh:`) para
acesso à API. Não definido por padrão; sem ele, os rate limits da API do GitHub
se aplicam.

## Logs

### `RUST_LOG`

Filtro de tracing padrão aplicado pelo `tracing_subscriber` na inicialização,
ex. `info` ou `arona=debug,info`. Segue a semântica usual de `RUST_LOG`
(`error`/`warn`/`info`/`debug`/`trace`, overrides por target).

## Padrões em resumo

| Configuração | Padrão |
| --- | --- |
| Endereço / porta de bind | `0.0.0.0:8420` |
| Rate limit de API por chave | 60 RPM |
| Nome do agent | `arona-agent` |
| URL do panel | `localhost:8080` |
| Writeback de memória | ativado |
| Registro | fechado |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/config/file.rs (discovery, interpolation, KEY_MAP), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
