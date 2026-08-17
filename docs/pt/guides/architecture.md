---
title: "Arquitetura"
description: "Como o Arona é montado — estrutura do workspace, o caminho da requisição pelo gateway, roteamento, health probing, memória, sessões e os tradeoffs de design deliberados."
---

# Arquitetura

Esta página percorre como o Arona é estruturado e como uma requisição flui por
ele: a estrutura do workspace, o caminho da requisição, o gateway e o router,
health checking, o cliente de memória, sessões e notificações e, finalmente, os
limites e tradeoffs deliberados que o design aceita. Veja
[quickstart](quickstart.md) para um exemplo em execução e
[operações](operations.md) para preocupações de runtime do dia a dia.

## Estrutura do workspace

O repositório é um Cargo workspace com três packages:

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC,
│            # backends, migration, entities, providers, models, registry,
│            # orchestration
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

- `packages/core` é o crate de biblioteca (`_core`). Ele contém tudo que o
  servidor precisa: o gateway axum (`gateway/`), o router de modelos (`routing/`),
  os adaptadores de backend (`backends/`), billing (`billing/`), auth (`auth.rs`),
  o cliente de memória (`memory/`), o plano JSON-RPC (`gateway/rpc.rs`), o schema
  (`migration/`, `entity/`), metadados de modelos (`models/`, `providers/`,
  `registry/`) e orquestração de modelos (`orchestration/`).
- `packages/agent` compila o binário `_agent` que roda em nós de GPU e conecta
  de volta via `/ws/agent` (veja [cluster de agents](agent-cluster.md)).
- `packages/cli` compila o binário `_cli` usado para operações de install, deploy,
  serve, migrate e download.

Não há mais dashboard web neste repositório: o dashboard Vue foi removido e
agora vive no [shittim-chest](https://github.com/celestia-island/shittim-chest)
(chest #291), que conversa com o Arona pela superfície JSON-RPC. O próprio Arona
é um backend puro (veja a [visão geral](./README.md)).

## Caminho da requisição

O ponto de entrada é o router axum montado em `GatewayServer::app`
(`packages/core/src/gateway/server.rs`). Sua tabela de rotas (server.rs:128-163)
cobre a superfície REST compatível com OpenAI (`/v1/chat/completions`,
`/v1/embeddings`, `/v1/models`, `/v1/health`), geração de vídeo, o endpoint
JSON-RPC `/api/rpc` (POST + upgrade WebSocket), o sidecar SSE
`/api/rpc/events`, o plano de controle de agents `/ws/agent`, a UI Swagger em
`/docs` e os endpoints de admin de gerenciamento de backend/alias.

O router é envolvido em uma pequena pilha de layers (server.rs:158-162):

1. Auth managers como `Extension`s, para que os extractors por handler possam
   alcançá-los.
2. Um layer de request-id que reutiliza um header `X-Request-ID` de entrada ou
   gera um, expondo-o a handlers e logs (`gateway/request_id.rs`).
3. Um limite de corpo de requisição de 1 MB (`RequestBodyLimitLayer`).
4. Um layer CORS permissivo (origin `*`, headers `*`).

Como o axum aplica layers de baixo para cima, o layer CORS é o mais externo.

Todo handler `/v1/*` então roda pelo mesmo esqueleto:

1. **Extração de auth** — `ApiKeyAuth` para os endpoints somente-chave
   (`/v1/chat/completions`, `/v1/embeddings`, vídeo) e `ApiKeyOrJwt` para
   `GET /v1/models`, que deve aceitar tanto API keys quanto JWTs de sessão
   (`gateway/middleware.rs`). O extractor resolve a chave/JWT para um email de
   usuário, prefixo de chave, uma chave de limite de taxa (o hash SHA-256 da API
   key, ou um rótulo `u:<email>` para JWTs, para que tokens rotacionados não
   resetem a janela) e um escopo de projeto opcional.
2. **Gates de billing** — `enforce_billing_gates` (server.rs:492-539) rejeita a
   requisição com HTTP 429 + `Retry-After` quando a cota mensal do tier do
   usuário ou o limite de taxa por minuto é excedido. Falhas de DB abrem: o
   billing é best-effort, nunca uma dependência dura de servir um chat.
3. **Recall de memória** (caminhos de chat) — se o cliente de memória está
   configurado e a requisição o pede, memórias de longo prazo relevantes são
   injetadas como uma seção de sistema (veja
   [Cliente de memória](#memory-client) abaixo). Falha nunca bloqueia o
   chat; o estado resultante é ecoado no header `X-Arona-Memory`.
4. **Persistência de conversa** — um `conversation_id` opcional é verificado
   quanto a ownership, e a rodada do usuário é persistida no momento do envio.
5. **Dispatch do gateway** — a requisição é entregue ao `Gateway`, que resolve
   um backend, trima o contexto e chama a trait de backend.
6. **Gravação de uso** — o uso retornado (ou terminal-de-stream) é gravado e
   persistido através do `UsageTracker` sob o prefixo da chave.

O próprio `Gateway` vive em `AppState` como um `Arc<Gateway>` — não há mutex
externo; mutabilidade interior impede que chamadas concorrentes de
chat/embeddings/stream segurem um lock através de uma round trip HTTP de
upstream (`gateway/mod.rs:29-53`).

## Gateway e router

O `Gateway` (`packages/core/src/gateway/mod.rs`) é dono de:

- **Estado do router** — a lista de backends e aliases, guardada por um
  `tokio::sync::RwLock`. A resolução do lado de leitura empresta através de
  awaits; mutações (register/remove/alias) tomam um lock de escrita curto e
  nunca o seguram através de uma chamada de upstream.
- **Um contador de requisições** (`AtomicU64`) e um `start_time` usados por
  `system.status` e pelos endpoints de health.
- **Um mapa de deployments** (`model_id → backend name`) para modelos
  implantados por agents. `register_agent_backend` constrói um
  `ExternalApiBackend` chamado `agent-{model_id}` e o insere no router;
  re-registrar o mesmo modelo substitui o backend anterior, e
  `unregister_agent_backend` o remove em um frame `stop_result` (veja
  [cluster de agents](agent-cluster.md)).

A resolução de backend acontece no `Router` (`packages/core/src/routing/mod.rs`):

1. **Resolução de alias** — um alias configurado é reescrito para seu alvo.
2. **Afinidade de sessão** — quando um `conversation_id` está presente, o router
   verifica um mapa de referência fraca que fixa a conversa no backend que a
   serviu primeiro. Referências fracas mantêm o mapa vivo apenas enquanto o
   backend está registrado ou em voo, então backends removidos desaparecem sem
   drift de índice. Um circuit breaker disparado ou um backend fixado unhealthy
   degrada para uma seleção nova, que re-fixa a conversa.
3. **Filtragem de candidatos** — uma dica `provider` opcional filtra por
   nome/kind de backend; candidatos devem estar healthy *e* ter um circuit
   breaker aberto, e devem listar o modelo solicitado. Ids de modelo casam
   exatamente ou pela convenção de sufixo `:latest` (uma requisição simples por
   `nomic-embed-text` casa com um `nomic-embed-text:latest` listado).
4. **Escolha por menos carregado** — candidatos sobreviventes são ordenados pelo
   seu contador de hits e o menos carregado é escolhido; o pin de conversa (se
   houver) é gravado ao mesmo tempo.

Antes de o backend ser chamado, `RequestPipeline::transform`
(`packages/core/src/pipeline.rs:422+`) trima a lista de mensagens para o
`max_context_length` do backend: mensagens de sistema são mantidas por inteiro,
mensagens não-sistema são mantidas das mais novas para as mais antigas enquanto
couberem, e uma única mensagem grande demais é truncada duramente por caracteres
(o contador de tokens heurístico não consegue truncar com precisão de token). A
chamada então passa pela trait `InferenceBackend`; sucessos e falhas são
gravados de volta no circuit breaker por backend do router (3 falhas, recuperação
de 30s, 1 chamada half-open — routing/mod.rs:57-64).

## Health checker e probing

`run_health_checks` (`packages/core/src/gateway/health_checker.rs`) roda como
uma task de background spawnada na inicialização (run.rs:135-150) e sonda todo
backend registrado uma vez por intervalo de 60 segundos. Dois detalhes
importam:

- A lista de backends é **buscada fresca a cada rodada** através de uma closure
  fetcher assíncrona, então backends registrados após a inicialização (ex. pela
  API de admin) são capturados sem reinício.
- A primeira rodada roda imediatamente, antes de o primeiro intervalo decorrer,
  então o estado de health é estabelecido assim que o processo inicia.

`probe_backend` é o caminho de código único do probe. Ele é reutilizado pelos
**probes de tempo de registro** one-off: depois que um admin registra um backend
(server.rs:688-693) ou um backend persistido é restaurado no boot (run.rs:122-127),
um probe fire-and-forget torna o backend healthy em ~1–2s em vez de permanecer
fail-closed até a próxima rodada de 60s. É isso que faz a lista de modelos de um
backend external recém-registrado aparecer em `GET /v1/models` quase
imediatamente.

O probe em si é uma chamada leve de backend (ex. o backend external atinge
`/v1/models` com um timeout de probe de 2s); o resultado é cacheado no backend e
o roteamento só seleciona backends cujo health cacheado é `Healthy` (mais um
circuit breaker aberto).

## Cliente de memória

O cliente de memória (`packages/core/src/memory/mod.rs`) é construído a partir
da configuração de ambiente na inicialização do servidor (server.rs:95-97):
quando `ARONA_MEMORY_URL` e `ARONA_MEMORY_TOKEN` estão definidas, requisições de
chat consultam o serviço de memória entelecheia Philia por um WebSocket JSON-RPC
e `recall_and_inject` pré-anexa as memórias relevantes como uma seção de sistema
(`## Relevant Long-Term Memories`) no contexto de saída. Rodadas concluídas são
gravadas de volta como episódios via `writeback_dialogue` — trabalho
fire-and-forget spawnado depois de a resposta do assistente ser persistida, para
que falhas de memória nunca bloqueiem ou atrasem o caminho de resposta de chat.
`ARONA_MEMORY_WRITEBACK` (padrão ativado) alterna o writeback. Veja
[gateway de memória](memory-gateway.md) para o panorama completo.

Toda resposta de chat carrega um header `X-Arona-Memory` com um de três
estados: `enabled` (o recall rodou e injetou), `disabled` (não configurado ou a
requisição passou `memory: false`) ou `offline` (configurado mas o serviço
estava inalcançável).

## Sessões e notificações

`AppState` guarda um `SessionManager` do `plana` (`state.sessions`). RPCs de
streaming como `chat.send` criam um session id (`gateway/rpc.rs:1701`) e
empurram notificações — tokens `chat.stream`, progresso de deploy
`models.progress`, `realtime.event` — para o canal da sessão. Clientes as
consomem do sidecar SSE `GET /api/rpc/events?session=<id>` (server.rs:191-200);
veja [eventos](../api/events.md) para o formato de notificação e a ressalva da
janela de pré-subscrição.

O canal de sessão também é usado para chamadas RPC request/response: quando um
cliente envia um header `x-session-id` em `POST /api/rpc`, o servidor empurra o
resultado para o canal da sessão também (server.rs:184-188, rpc.rs:128-144),
para que um cliente possa multiplexar uma resposta RPC em um stream SSE já
aberto.

## Limites e tradeoffs de design

O design aceita deliberadamente uma série de limites; conheça-os antes do uso em
produção:

- **Limite de corpo de requisição de 1 MB** — corpos maiores são rejeitados pelo
  layer; se você precisa de chamadas de contexto grande, esta é a primeira coisa
  a ajustar.
- **CORS `*`** — o gateway responde chamadas cross-origin de qualquer lugar.
  Tudo bem para uma API, mas se você a expor além de clientes confiáveis,
  coloque à frente um proxy que aplique sua própria política de CORS.
- **Billing fail-open** — a aplicação de cota/limite de taxa degrada para
  permitir a requisição quando o DB está indisponível. Billing é medição, não
  controle de acesso.
- **Sem timeout geral em streams SSE** — chamadas de streaming não carregam
  deadline total (gerações longas são legais); a detecção de hang depende de um
  timeout idle de 120s por leitura (`backends/external.rs:24-31`). Chamadas
  não-streaming recebem um deadline geral de 600s.
- **Uso estimado por tokenizer** — backends que nunca reportam uso (ollama,
  ws_engine) são cobrados de uma estimativa local de tokenizer com consciência
  de CJK, gravada como está (veja [billing e uso](billing-usage.md)).
- **Janelas de limite de taxa e revogação em memória** — a janela deslizante por
  chave e o conjunto de chaves revogadas vivem na memória do processo
  (`auth.rs`), então um reinício as reseta. O limitador de nível auth limita
  requisições por chave por janela; o limitador de tier de billing é apoiado em
  DB (veja [auth e segurança](auth-security.md) e
  [billing e uso](billing-usage.md)).
- **`/ws/agent` é não autenticado** — o plano de controle de agents aceita
  qualquer WebSocket que fale o protocolo register/heartbeat. Ele só é seguro em
  uma rede que você controla.
- **Sem TLS no gateway** — o servidor faz bind em HTTP puro; termine o TLS à
  frente (reverse proxy) em qualquer implantação que cruze uma fronteira de
  rede. Veja [implantação](deployment.md).

No lado gracioso, o servidor executa um desligamento gracioso: instala handlers
de Ctrl+C e SIGTERM, loga "draining connections" e deixa requisições em voo
terminarem antes de o processo sair (`gateway/run.rs:14-38` e o wiring
`with_graceful_shutdown` em run.rs:154-159).

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table); gateway/mod.rs:29-53; routing/mod.rs:102-200; health_checker.rs:14-37; run.rs:135-159; memory/mod.rs:207-241; rpc.rs:128-144,1701 -->
