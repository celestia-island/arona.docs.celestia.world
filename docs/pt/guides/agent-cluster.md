---
title: "Cluster de Agents"
description: "Clusters de GPU multi-nó — download de pesos de modelos com a CLI, execução do binário _agent em nós de GPU e condução de implantações pela superfície RPC agents.*."
---

# Cluster de Agents

A história de implantação do Arona se divide em duas metades. O **panel** (o
binário do servidor `arona`) é dono do roteamento, billing, auth e do plano de
gerenciamento. Cada nó de GPU executa um processo **`_agent`** que é dono dos
pesos de modelos e dos processos locais de servir. Agents abrem um WebSocket de
longa duração de volta para o plano de controle de agents do panel
(`/ws/agent`); o panel empurra comandos `deploy` / `stop` por esse socket e o
agent faz stream de progresso de download, heartbeats e resultados de comandos
de volta para cima. Uma vez que um modelo está rodando em um agent, o panel o
registra como um backend roteável, para que o tráfego `/v1/*` e RPC o alcance —
o plano de controle é WebSocket, o plano de dados é HTTP puro para a porta
local do engine do agent.

## Download de pesos de modelos (CLI)

O binário `_cli` baixa pesos de modelos do HuggingFace, ModelScope ou releases
do GitHub:

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **Formas de repo** — `hf:owner/repo` (padrão; um `owner/repo` simples também
  resolve para o HuggingFace), `ms:owner/repo` (ModelScope), `gh:owner/repo[:tag]`
  (release do GitHub, tag opcional). Os prefixos longos `huggingface:`,
  `modelscope:` e `github:` também são aceitos; um id simples sem barra resolve
  para o registry do Ollama
  (`packages/core/src/models/download.rs:21-28,55-86`).
- **`--filter <glob|prefix>`** — repetível; apenas arquivos de manifest que
  casam com o glob (ou prefixo) são baixados. Sem filtro, o **repo inteiro** é
  selecionado.
- **Confirmação** — um download sem filtro sempre pergunta `Continue? [y/N]`
  antes de começar, a menos que `--yes` seja passado. Um download filtrado nunca
  pergunta; quando o total selecionado está em ou acima de 2 GiB, ele imprime um
  banner informativo em vez disso (`NO_CONFIRM_THRESHOLD`,
  `packages/cli/src/main.rs:12-15, 439-464`).
- **`--out <dir>`** — substitui o destino padrão
  `~/.arona/models/<repo-id>`.
- **`--revision <rev>`** — substitui qualquer sufixo `:rev` inline
  (`hf:owner/repo:rev`).
- **Resume** — downloads interrompidos retomam automaticamente: um arquivo
  `.part` é mantido e o download continua do comprimento atual via uma requisição
  Range; arquivos completos são pulados por tamanho e, quando o manifest carrega
  um digest, verificados por SHA-256 (`packages/cli/src/main.rs`
  `verify_sha256`/`summarize`).
- **Retries** — erros de rede tentam novamente até 3 vezes com um atraso de 5
  segundos; erros não-rede falham imediatamente (`packages/cli/src/main.rs:277-302`).
- **`HF_ENDPOINT`** — troca a URL base do HuggingFace, ex. um mirror:

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

Os outros comandos CLI (`packages/cli/src/main.rs:28-53`):

| Comando | Finalidade |
| --- | --- |
| `install` | Setup de ambiente em um clique: detecta o perfil de hardware e imprime recomendações de backend / quantização. |
| `status` | Imprime o perfil de hardware. |
| `deploy <model>` | Resolve um modelo localmente e reporta se ele já está em cache. |
| `download` | Baixa pesos de modelos (acima). |
| `serve` | Inicia o servidor de API (panel). |
| `connect <url>` | Conecta-se a um panel de gerenciamento. |
| `migrate` | Executa migrações de banco de dados. |

## O binário `_agent`

`_agent` roda em cada nó de GPU e é configurado puramente por variáveis de
ambiente (`packages/core/src/config.rs:37-40`):

| Variável | Padrão | Significado |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | Id único do nó; o panel o usa como `agent_id`. |
| `ARONA_PANEL_URL` | `localhost:8080` | `host:port` do panel; o agent se conecta a `ws://{ARONA_PANEL_URL}/ws/agent`. |

Veja [Configuração](./configuration.md) para a referência completa de variáveis
de ambiente (variáveis do lado do panel, banco de dados, segredos).

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

Comportamento:

- **Conexão de controle** — o agent conecta de volta a
  `ws://{ARONA_PANEL_URL}/ws/agent` (`packages/agent/src/panel.rs:23`). Ao
  conectar, ele envia um frame `register` carregando `agent_name`, `gpu_info` e
  a lista de modelos já implantados; o panel registra o endereço TCP peer do
  agent como seu `host`.
- **Backoff de reconexão** — começa em 1 segundo e dobra até um teto de 60
  segundos (`packages/agent/src/panel.rs:27,33-34,63`).
- **Heartbeats** — a cada 30 segundos o agent reporta utilização de GPU,
  contagem de modelos carregados e uptime. O panel considera um agent offline
  quando seu último heartbeat tem mais de 30 segundos.
- **API HTTP local** — faz bind no endereço **fixo** `0.0.0.0:5790`; não há
  variável de ambiente de endereço de bind (`packages/agent/src/main.rs:109`). O
  panel combina esta porta com o host registrado do agent para construir a URL
  do plano de dados para modelos implantados.
- **Comandos** — o panel enfileira comandos `deploy` / `stop` pelo socket. Um
  comando `deploy` carrega `model_id` e um `stream_id`; o progresso de download
  é transmitido de volta como frames `deploy_progress` no mesmo socket (o panel
  os encaminha como notificações SSE `models.progress`, veja
  [Eventos e notificações](../api/events.md)), e o frame final `deploy_result`
  reporta a `port` e o `backend` locais do engine. `stop` é respondido com
  `stop_result`.

Rode `_agent` sob um supervisor de serviço (systemd, malkuth, ...) para que ele
reconecte automaticamente; o panel tolera reinícios em qualquer lado (veja
[persistência de nós](#node-persistence) abaixo).

## RPC do plano de controle de agents

Toda a superfície de agents tem gate de admin: todo método exige um JWT válido
**e** uma conta de admin (`validate_admin_jwt` verifica `is_admin_email`;
`packages/core/src/gateway/rpc.rs:106-118,301-337`).

| Método | Params | Retorna |
| --- | --- | --- |
| `agents.list` | — | Topologia do cluster: `id`, `name`, `host`, `status` (`online`/`offline`), resumo de GPU, `models`, `last_heartbeat`, `version`, `connected_at`. |
| `agents.register` | `machine_name`, `version` | `agent_id`, `token`. |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — remove o nó. |
| `agents.status` | `agent_id` | `online`, resumo de GPU, `gpu_utilization`, `models`, `host`, `connected_at`, `last_heartbeat`. |
| `agents.deploy` | `model_id`, `agent_id?` | `{ "ok": true, "stream_id" }` — `agent_id` vazio auto-seleciona o nó menos carregado. |
| `agents.stop` | `agent_id`, `model_id` | `{ "ok": true }` — interrompe a implantação. |

`agents.deploy` retorna um `stream_id`; assine
`/api/rpc/events?session=<stream_id>` **antes** ou imediatamente após a chamada
para receber notificações de download `models.progress` (veja
[Eventos e notificações](../api/events.md)). Veja
[API JSON-RPC](../api/jsonrpc.md) para os detalhes de transporte e auth.

## Auto-registro de modelos implantados

Quando um frame `deploy_result` reporta sucesso, o panel registra um
`ExternalApiBackend` chamado **`agent-{model_id}`** no router do gateway, com
URL base `http://{agent-host}:{port}` — o host registrado do agent mais a porta
do engine que ele reportou (`packages/core/src/gateway/server.rs:310-366`,
`packages/core/src/gateway/mod.rs:253-270`). O modelo implantado vira um backend
roteável normal: `/v1/chat/completions`, embeddings e chat RPC todos o
alcançam, aliases se aplicam e o health checker o sonda (veja
[Backends](./backends.md) para tipos de backend e semântica de probing).

- Re-implantar o mesmo modelo (ex. em um agent diferente) substitui o backend
  anterior.
- Um `stop_result` bem-sucedido o desregistra novamente
  (`packages/core/src/gateway/mod.rs:274-287`); o id do modelo para de resolver.

## Alocação

Implantações sem um `agent_id` explícito passam pela alocação por menos
carregado (`packages/core/src/gateway/tunnel.rs:214-229`): candidatos são agents
cujo último heartbeat está abaixo de 30 segundos, e o de **menor utilização
média de GPU** é escolhido. Agents sem telemetria ficam por último, mas
permanecem selecionáveis. Se nenhum agent está online, o RPC falha com
`No online agents available for deployment`.

No lado do roteamento, conversas são **fixadas em um backend** (afinidade de
sessão): o primeiro backend que serve uma conversa é registrado e reutilizado
nas rodadas seguintes, para que estado por conversa como um cache KV de runtime
permaneça quente (`packages/core/src/routing/mod.rs:31-34,110-138`). Se o
backend fixado ficar unhealthy, o roteamento degrada para uma seleção nova e
re-fixa.

## Persistência de nós

Nós de agent persistem na tabela `agent_nodes` (`agent_id`, `machine_name`,
`version`, `host`, `gpu_info`, `models`, `connected_at`, `last_heartbeat`;
`packages/core/src/gateway/tunnel.rs:8-12`). Na inicialização do panel, as
linhas persistidas são restauradas, para que nós registrados anteriormente
permaneçam visíveis entre reinícios; entradas restauradas ficam **sem sender**
até que cada agent reconecte via WebSocket
(`packages/core/src/gateway/run.rs:74-85`). Implantar em um nó restaurado mas
desconectado, portanto, falha até que seu `_agent` reconecte.

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
