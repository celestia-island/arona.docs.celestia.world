---
title: "Operações"
description: "Endpoints de health, tracing RUST_LOG, timeouts de upstream, mapeamento de erros e solução de problemas para um arona-server em execução."
---

# Operações

Esta página é para operadores executando `arona-server serve`. Ela cobre os
endpoints de health que você sonda, as linhas de log que valem um grep, o
modelo de timeout aplicado a backends upstream, como falhas de backend mapeiam
para erros HTTP e os percalços operacionais que pegam as pessoas. A implantação
em si é coberta no [guia de implantação](./deployment.md).

## Matriz de health

Os três endpoints de health são sem autenticação e retornam `200 OK` sempre que
o processo está servindo — não há distinção liveness/readiness:

| Endpoint | Resposta |
| --- | --- |
| `/healthz`, `/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | o mesmo corpo detalhado acima |
| `/api/health` | `HealthResponse` do plana: `status`, `version` (`CARGO_PKG_VERSION`), `kind` (`Dev`), `uptime` (segundos), `network` (transporte / região / asn), `build_hash` (`BUILD_HASH`), `engine_version` (`"0.1.0"`) |

`/healthz` e `/readyz` são aliases do mesmo handler, e `/v1/health` o
compartilha, então os probes estilo Kubernetes e a rota de health compatível com
OpenAI são intercambiáveis. `/api/health` adiciona uptime, rede e versão do
engine. Use `/readyz` para load balancers e supervisores; use `/api/health`
quando precisar do payload mais rico.

## Logs

O servidor loga via `tracing`, filtrado com a variável `RUST_LOG` padrão
(`RUST_LOG=info` é a configuração comum; `RUST_LOG=debug` revela tráfego de
probe). Eventos que valem conhecer, em ordem aproximada de frequência:

| Linha de log | Nível | O que ela diz |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | Uma por requisição de chat, com `key_prefix`, `model`, `stream` e `request_id` — a trilha de auditoria por requisição mais simples. |
| `request completed` | info | Logada pelo helper `logging_middleware` após toda resposta **não-streaming** de `/v1/chat/completions` e `/v1/embeddings`: `method`, `path`, `status`, `latency_ms`, `trace_id`. (Chat streaming loga `chat completions SSE request` no início em vez disso.) |
| `usage recorded` / `usage persisted` | info | Uma linha de uso foi gravada (em memória, com tokens/custo) e depois escrita na tabela `usage_records`. |
| `external probe: sending` / `external probe: returned` | debug | Um health probe do `/v1/models` de um backend external; `matched` diz se o probe concluiu dentro do timeout de probe de 2s. |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | Uma requisição `/v1/*` recusada pelo gate de billing — o cliente recebeu 429 mais `Retry-After`. |
| `rpc billing gate rejected: monthly quota exceeded` | warn | O gate de cota do lado RPC para métodos autenticados por JWT (janela do usuário inteiro; resposta de erro JSON-RPC). |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | Restauração na inicialização: backends registrados por admin e nós de agent carregados do banco e tornados roteáveis novamente. |
| `Shutdown signal received, draining connections…` | info | O desligamento gracioso começou (SIGINT/SIGTERM). |

## Modelo de timeouts

Timeouts são aplicados no cliente de upstream usado para backends external
(`packages/core/src/backends/external.rs`):

| Timeout | Valor | Aplica-se a |
| --- | --- | --- |
| Connect | 10s | Estabelecer a conexão TCP/TLS com o upstream. |
| Read idle | 120s por leitura | Toda chamada de upstream; cada byte recebido reseta o relógio, então um stream saudável-mas-lento nunca é cortado. |
| Geral não-streaming | 600s | Chamadas não-streaming de chat/embeddings — um upstream lento-mas-vivo não pode segurar uma requisição para sempre. |
| Streaming (SSE) | nenhum | Chamadas de streaming não carregam **deadline geral**; gerações longas são legais e a detecção de hang depende do timeout de read idle. |
| Health probe | 2s | O probe de `/v1/models`. |

## Mapeamento de erros

Falhas de backend mapeiam para status HTTP nos handlers de chat/embeddings
(`packages/core/src/gateway/server.rs`):

| Condição | HTTP | `type` / `code` | Mensagem |
| --- | --- | --- | --- |
| Status não-2xx do upstream (`UpstreamStatus`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| Falha de transporte do upstream (`RequestFailed`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | a string do erro de transporte |
| Qualquer outro erro de backend | **500** | `server_error` / `backend_error` | a string do erro |
| Nenhum backend para o modelo (`NoBackend`) | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| API key inválida (`Unauthorized`) | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| Limite de taxa (`RateLimited`) | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

A intenção de design: callers conseguem distinguir "seu provider rejeitou ou
falhou" (502) de "o próprio gateway está quebrado" (500). Todo corpo de erro tem
o mesmo shape estilo OpenAI — `{"error":{"message":...,"type":...,"code":...}}`
(`json_error_response`). Os 429s do gate de billing adicionalmente carregam um
header `Retry-After` e usam `quota_error`/`quota_exceeded` (cota) e
`rate_limit_error`/`rate_limit_exceeded` (limite de taxa do tier),
respectivamente.

## Solução de problemas

### Um backend recém-registrado permanece fail-closed até ser sondado

Backends external começam em um estado de health desconhecido e reportam
`"<url> not probed yet"`. Eles ficam healthy quando (a) a primeira rodada do
health checker roda — imediatamente na inicialização, depois a cada 60s — ou
(b) o probe fire-and-forget lançado no registro ou na restauração tem sucesso,
normalmente em ~1-2 segundos. Até lá, requisições roteadas para o backend falham
fechadas por design.

### Um 404 no `/models` do probe é normal para alguns backends

O probe external atinge `GET {base}/v1/models` (ou `{base}/models` para URLs
base com um prefixo de path). Alguns servidores compatíveis com OpenAI
implementam chat mas não expõem listagem de modelos — o endpoint de plano de
coding da Zhipu GLM é um. Um **404 é tolerado**: o backend é marcado healthy e a
lista de models configurada por admin permanece autoritativa para o roteamento.
Apenas probes genuinamente falhos (timeout, erro de rede, outro não-2xx) marcam
o backend unhealthy.

### Streams SSE que não produzem nada não são cobrados

Uma resposta de streaming é gravada no uso apenas quando o stream produziu texto
**ou** carregou uso terminal; um stream que terminou sem nenhum dos dois não é
gravado. Se você vir uma requisição sem uma linha `usage recorded`
correspondente, verifique se o stream realmente produziu conteúdo.

### Reporte de versão

`version` nos corpos de health é `CARGO_PKG_VERSION`; `build_hash` é o valor
`BUILD_HASH` de build-time emitido por `packages/core/build.rs`. Compare
`build_hash` entre nós para confirmar que todos rodam o mesmo artefato.

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
