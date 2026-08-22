---
title: "Backends"
description: "Tipos de backend (external, ollama, engine, minimax-cloud, bridges evernight), semântica de URLs, health probing, descoberta de modelos, aliases e roteamento."
---

# Backends

Um **backend** é um upstream que serve tráfego de modelos. O Arona roteia
requisições compatíveis com OpenAI (`/v1/chat/completions`, `/v1/embeddings`,
listagem de modelos, jobs de vídeo) para um dos backends registrados, mede toda
requisição e mantém o health e o inventário de modelos de cada backend
atualizados.

Backends são registrados por um admin via `POST /api/admin/backends` (veja a
[API HTTP de admin](../api/admin-http.md)), persistidos na tabela
`backend_configs` e restaurados automaticamente na inicialização. Cada registro
carrega um `name`, um `type`, uma `url`, um `api_key` opcional e uma lista
estática `models` opcional. Backends persistidos sobrevivem a reinícios;
backends restaurados começam fail-closed e são sondados imediatamente.

## Tipos de backend

| `type` | Transporte | Protocolo | Finalidade |
| --- | --- | --- | --- |
| `external` | HTTP(S) | REST compatível com OpenAI | Qualquer API de chat/embeddings (nuvem ou self-hosted) |
| `ollama` | HTTP(S) | API nativa do Ollama (`/api/chat`, `/api/embed`, `/api/tags`) | Um servidor Ollama local ou remoto; construído apenas a partir da URL |
| `engine` | `ws://` / `wss://` | CEP (Celestia Engine Protocol), WebSocket + JSON-RPC | Qualquer engine que fale o padrão de intercâmbio CEP (`plana::engine`) |
| `minimax-cloud` | HTTPS | API estilo task do MiniMax H3 (submit + poll) | Geração de vídeo em nuvem |
| `evernight://<node>/<service>` | URL de bridge | Resolvida pelo agent evernight local em um forward TCP local | Serviços industriais/de borda alcançáveis apenas pela malha evernight |
| `agent-{model}` | HTTP | Compatível com OpenAI (external) | Auto-registrado quando um agent de GPU implanta um modelo |

### `external` — qualquer API HTTP compatível com OpenAI

O backend de propósito geral: completions de chat (streaming e não-streaming)
e embeddings contra qualquer servidor que fale o shape REST do OpenAI.
Configure-o com uma `url` base, um `api_key` (opcional) e uma lista estática
`models` opcional:

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

A lista estática `models` é autoritativa: ela é mesclada antes de quaisquer
modelos descobertos no momento do probe (veja
[Descoberta de modelos](#model-discovery)).

### `ollama` — construído apenas a partir da URL

Um backend Ollama é construído apenas a partir da URL — sem API key, sem lista
de modelos. Ele fala os protocolos nativos do Ollama: `/api/chat` para chat,
`/api/embed` para embeddings e `/api/tags` para health probing e descoberta de
modelos.

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — CEP sobre WebSocket

Um backend `engine` conecta-se a um engine que expõe `ws://` (ou `wss://`) e
fala o **Celestia Engine Protocol** (CEP): um padrão de intercâmbio WebSocket +
JSON-RPC 2.0 definido em `plana::engine`. Qualquer engine escrito em qualquer
linguagem que implemente o fluxo handshake → methods → streaming-notifications
registra-se como um backend de primeira classe com zero código de adaptador no
Arona. Methods no wire: `Engine.Handshake` (primeira mensagem; identidade +
capacidades), `Engine.Chat`, `Engine.ChatStart` (streaming; chunks chegam como
notificações `Engine.ChatChunk`), `Engine.Embeddings` e `Engine.Models`.
Conexões são estabelecidas de forma lazy no primeiro uso e derrubadas em
qualquer erro; a próxima chamada reconecta e refaz o handshake.

### `minimax-cloud` — geração de vídeo estilo task

O backend de vídeo em nuvem dirige a API MiniMax H3 Open Platform: submeta uma
task de geração, faça poll até a conclusão e leia a URL do artefato do
resultado. Foi isso que substituiu o backend ComfyUI removido (veja abaixo);
jobs de vídeo são submetidos via `/v1/video/generations` ou os métodos RPC
`video.*` e progridem através de notificações `video.progress` / `video.done` /
`video.failed` (veja [Realtime e vídeo](realtime-video.md)).

### URLs de bridge `evernight://`

Uma URL de backend da forma `evernight://<node>/<service>` **não** é contatada
diretamente. O agent evernight do host local a resolve (uma chamada JSON-RPC
`Bridge.Connect` sobre o endpoint WebSocket do agent) em um forward TCP local,
e o backend roda contra `http://127.0.0.1:<local_port>` em vez de um endereço
remoto hardcoded. Esta é a arquitetura de panel único: o panel Arona alcança
serviços em outros nós (engines CEP, scepter, ...) pela malha sem nunca embutir
um endereço remoto em uma configuração.

Uma task de keepalive sonda o túnel a cada 15 segundos; quando o lado remoto
reinicia e o túnel é restabelecido em uma nova porta local, o backend afetado é
**reconstruído de forma transparente** com a nova URL — a configuração
persistida mantém a URL `evernight://`, então reinícios a re-resolvem. Para
backends `engine`, o forward `http://127.0.0.1:<port>` resolvido é adaptado
para `ws://` no transporte WebSocket.

### Modelos implantados por agents se auto-registram

Quando um agent de GPU termina de implantar um modelo, o gateway registra um
backend chamado `agent-{model_id}` (um `ExternalApiBackend` sobre
`http://{agent host}:{port}`) para que o modelo se torne roteável
imediatamente; parar a implantação o desregistra novamente. Veja
[Cluster de agents](agent-cluster.md) para o ciclo de vida completo da
implantação.

### `comfyui` é rejeitado

O tipo de backend `comfyui` é explicitamente recusado com o erro
`comfyui backend removed`: o backend ComfyUI foi removido durante a
convergência da plataforma de modelos, e a geração de vídeo agora roda via
`minimax-cloud`. Registrar um backend `comfyui` retorna um HTTP 400.

## Semântica de URLs

Como uma URL base configurada mapeia para endpoints reais é decidido por se a
URL tem um componente de path:

- **Base estilo raiz** — uma URL cujo path é vazio ou `/` é tratada como raiz
  de host e mantém a convenção `/v1` do OpenAI: `{base}/v1/chat/completions`,
  `{base}/v1/models`. Exemplos: `http://192.0.2.20:8429`,
  `https://api.deepseek.com`.
- **Base estilo path** — uma URL com um path não vazio é tratada como o prefixo
  completo da API que o servidor realmente serve, e o endpoint é anexado
  diretamente: `{base}/chat/completions`, `{base}/models`. É o que servidores
  compatíveis com OpenAI fora da convenção `/v1` precisam. O plano de coding GLM
  da Zhipu é o exemplo canônico: sua API vive em
  `https://open.bigmodel.cn/api/coding/paas/v4` com chat diretamente em
  `{base}/chat/completions` e **nenhum endpoint `/models`** — a raiz padrão
  `/api/paas/v4` retorna erros de balance para chaves de plano de coding.
- Uma **barra final** na URL base configurada é normalizada para fora, para que
  o join nunca produza uma barra dupla.

## Probing e health

Um health checker em background sonda todo backend registrado a cada **60
segundos**; a lista de backends é buscada fresca a cada rodada, então backends
registrados após a inicialização são capturados sem reinício. Cada registro de
admin também dispara um probe imediato, para que o backend fique healthy em
~1–2 segundos em vez de esperar a próxima rodada do checker.

- **Backends external** sondam `GET {base}/models` (ou `{base}/v1/models` para
  bases estilo raiz) com um **timeout de 2 segundos**. Um **404 é tolerado**:
  alguns servidores implementam chat mas não expõem listagem de modelos (o
  plano de coding GLM não tem endpoint `/models`), então um 404 marca o backend
  como healthy e a lista `models` configurada por admin vira a fonte de
  roteamento. Timeouts, falhas de rede e outras respostas não-2xx marcam o
  backend como unhealthy.
- **Backends ollama** sondam `/api/tags` com o mesmo timeout de 2 segundos.
- Backends começam **fail-closed** — reportados como `not probed yet` — até o
  primeiro probe bem-sucedido, então um backend recém-registrado (ou restaurado)
  nunca recebe tráfego antes de ser verificado.

O estado de health é cacheado por backend e consultado pelo router a cada
requisição; backends unhealthy são excluídos da seleção de candidatos (veja
[Roteamento](#routing)).

## Descoberta de modelos

Um backend anuncia os ids de modelos que serve, e o router casa requisições
contra esse anúncio:

- Backends **external** anunciam os modelos parseados da resposta do probe
  (tanto um array `data` quanto um `models` são aceitos), mesclados com a lista
  estática configurada por admin — ids estáticos mantêm ordem e precedência,
  ids dinâmicos são deduplicados e anexados. Quando um servidor não tem endpoint
  de models, apenas a lista estática é a fonte de roteamento.
- Backends **ollama** anunciam os tags retornados por `/api/tags`.
- Modelos **implantados por agents** anunciam exatamente o `model_id`
  implantado.

A superfície pública é `GET /v1/models` (autenticado), que lista os modelos
roteáveis de todo backend healthy (veja a
[API REST compatível com OpenAI](../api/openai-rest.md)).

## Aliases e normalização de nomes

Aliases mapeiam um id de modelo requisitado para um id alvo. O alias é resolvido
primeiro no roteamento, então uma requisição pelo alias é servida por qualquer
backend que anuncie o alvo:

```json
{ "alias": "fast-chat", "target": "deepseek-chat" }
```

Aliases são gerenciados pelos endpoints de admin `/api/admin/aliases` e têm
efeito imediato.

O casamento de nomes também normaliza tags estilo Ollama: um backend que lista
`nomic-embed-text:latest` casa com uma requisição simples por
`nomic-embed-text`, então requisições de embedding/chat resolvem sem a
contabilidade do sufixo `:latest`. Um tag explícito (`qwen3:0.6b`) casa apenas
com aquele tag exato.

## Roteamento

Toda requisição resolve pelo router, que seleciona um backend:

1. **Resolução de alias** — o id de modelo requisitado é mapeado pela tabela de
   aliases (se houver).
2. **Dica de provider** — um campo `provider` opcional filtra candidatos por
   nome de backend (ou nome de kind, ex. `cloud` para backends de vídeo).
3. **Apenas candidatos healthy** — um backend deve reportar `Healthy` *e*
   passar no seu circuit breaker (3 falhas consecutivas abrem o breaker por 30
   segundos, com uma chamada de teste half-open) para ser selecionável.
4. **Escolha por menos contagem** — candidatos que servem o modelo são
   ordenados pelo contador de requisições por backend e o menos carregado é
   escolhido. Isso distribui a carga entre backends healthy que servem o mesmo
   modelo.
5. **Afinidade de sessão** — quando uma requisição carrega um
   `conversation_id`, a conversa é fixada no backend que ela usou primeiro. O
   pin vive em um mapa de referência `Weak`, então um backend removido
   desaparece do mapa sem drift de índice. A afinidade é best-effort: reutilizar
   o mesmo backend nas rodadas de uma conversa permite que o upstream reutilize
   estado de runtime por conversa (contextos quentes, caches KV). Se o backend
   fixado ficou unhealthy (ou o pin morreu com um backend removido), o router
   cai para uma nova seleção por menos contagem e **re-fixa** a conversa.

Se nenhum backend healthy serve o modelo, o roteamento falha: um modelo
desconhecido gera `model not found` (HTTP 404), um modelo conhecido-mas-
inalcançável gera `all backends unhealthy`, que é exposto como um erro interno
500. HTTP 502 é reservado para falhas reportadas por um upstream *alcançável*
(respostas não-2xx do upstream e falhas de transporte após a seleção). Veja
[Operações](operations.md) para o mapeamento de erros completo.

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
