---
title: "Billing e Uso"
description: "Registros de uso, custo por modelo, tiers de billing, aplicação de cota e limite de taxa, chaves com escopo de projeto, precificação de vídeo e o RPC usage.list."
---

# Billing e Uso

O Arona mede toda requisição de modelo e aplica cotas e limites de taxa por tier
no gateway. Preços por modelo vêm da tabela de preços compartilhada do plana
(nunca reimplementada no arona), o uso é persistido como linhas `usage_records`
e o panorama mensal inteiro é exposto pelo RPC `usage.list`.

## Registros de uso

Toda requisição medida vira uma linha na tabela `usage_records`
(`m20250101_000006_create_usage_records`):

| Coluna | Tipo | Significado |
| --- | --- | --- |
| `id` | `UUID` | Chave primária, gerada. |
| `api_key_id` | `VARCHAR(64)` | O **prefixo da chave** — os primeiros 8 caracteres da API key (chaves parecem com `arona-{uuid}`) — ou um id sintético `jwt-<user-uuid>` para canais RPC atribuídos por JWT. |
| `model` | `VARCHAR(128)` | Id do modelo para o qual a requisição foi roteada. |
| `backend` | `VARCHAR(64)` | Kind de backend: `gateway`, `rpc`, `realtime` ou o nome de capacidade do backend. |
| `prompt_tokens` | `INTEGER` | Tokens de entrada, reportados pelo upstream ou estimados. |
| `completion_tokens` | `INTEGER` | Tokens de saída, reportados pelo upstream ou estimados. |
| `total_tokens` | `INTEGER` | Soma dos dois. |
| `cost` | `DOUBLE PRECISION` | Custo em USD calculado; `NULL` quando o modelo não tem linha de preço. |
| `created_at` | `TIMESTAMPTZ` | Quando a requisição foi concluída. |

Existem índices em `api_key_id`, `model` e `created_at` (as colunas que a
agregação mensal e as janelas de limite de taxa varrem).

## Canais de gravação

O uso é gravado em todo canal medido:

1. **REST não-streaming** — `POST /v1/chat/completions` e
   `POST /v1/embeddings` gravam o uso exato reportado pelo upstream assim que a
   resposta é produzida.
2. **REST streaming (SSE)** — o uso reportado pelo upstream vence quando o
   stream o carregou (campo `usage` do chunk terminal compatível com OpenAI);
   caso contrário, uma estimativa local de tokenizer com consciência de CJK
   (`estimate_usage`) é gravada como está. Streams que não produziram nem texto
   nem uso **não** são gravados.
3. **RPC `chat.send`** — a mesma lógica estimativa-vs-upstream se aplica; linhas
   são atribuídas com o id sintético `jwt-<user-uuid>` para que façam join de
   volta ao usuário.
4. **Sessões realtime** — cada transcrição `response_done` concluída grava seu
   uso de tokens (quando não zero) sob `jwt-<user-uuid>` com backend `realtime`.
5. **Jobs de vídeo** — um job concluído grava um custo explícito em dólares
   (veja [Precificação de vídeo](#video-pricing)); contagens de tokens
   são zero.

A gravação é best-effort: um insert que falha é logado e nunca falha a
requisição.

## Cálculo de custo

O custo é calculado a partir da tabela de preços canônica por 1M de tokens
(`plana_llm_provider::metering::lookup_pricing`, compartilhada entre todos os
serviços celestia-island):

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

O casamento de modelos na tabela é por substring no id de modelo em minúsculas
(famílias mais específicas vencem). Quando um modelo não tem linha de preço,
`cost` é `NULL`. **Não reimplemente preços no arona — atualize a tabela do
plana.**

## Tiers

Tiers vivem na tabela `billing_tiers`, predefinidos na primeira migração
(`m20250101_000007_create_billing_tiers`). Uma coluna de cota `NULL` significa
"ilimitado" para aquela dimensão. Usuários sem `tier_id` caem para o tier
`free` predefinido.

| Tier | Cota mensal USD | Cota mensal de tokens | RPM por chave |
| --- | --- | --- | --- |
| `free` | $1,00 | 100.000 | 10 |
| `pro` | $20,00 | 5.000.000 | 120 |
| `enterprise` | ilimitada (`NULL`) | ilimitada (`NULL`) | 1000 |

A atribuição de tier é uma operação de admin (RPC `billing.plan.set`); o tier
atual e o uso são expostos via `billing.plan`.

## Aplicação de cota e limite de taxa

### REST (`/v1/*`)

Antes de todo endpoint REST **medido** — `/v1/chat/completions`,
`/v1/embeddings` e `/v1/video/generations` — o gateway aplica dois gates para
requisições autenticadas por chave:

- **Cota mensal** — os limites `monthly_quota_usd` e `quota_tokens` do tier são
  verificados contra o uso acumulado desde o primeiro instante do mês
  calendário atual. Qualquer dimensão atingindo seu limite dispara o gate.
- **Limite de taxa por chave** — o `rate_limit_rpm` do tier é verificado contra
  o número de requisições gravadas para este prefixo de chave na janela dos
  últimos 60 segundos. (`/v1/models` e os endpoints de health não são medidos e
  não passam por gates.)

Uma rejeição é um HTTP **429 Too Many Requests** com um header `Retry-After` e
um corpo de erro estilo OpenAI:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| Rejeição | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| Cota mensal esgotada | `quota_error` / `quota_exceeded` | Segundos até o início do **próximo mês calendário** |
| Limite de taxa do tier excedido | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

O `chat.send` autenticado por JWT passa pelo mesmo gate de cota mensal, mas
contra a janela do **usuário inteiro** (a chamada não carrega API key). Uma
rejeição é um erro JSON-RPC com o código definido pela implementação `-32006`
(`QUOTA_ERROR`) e a mesma mensagem da rejeição de cota REST. Não há limite de
taxa por chave no caminho RPC — a limitação de taxa tem escopo de chave e
chamadas RPC não têm chave. Métodos RPC de realtime e vídeo não passam por
gates de cota.

## Tradeoff fail-open

O billing é **best-effort por design**. Se a query de banco por trás de uma
verificação de cota ou limite de taxa falhar, a verificação retorna `Unknown` e
a requisição é **permitida** (apenas logada) em vez de bloquear o chat. Um
operador pode confiar em 429s para proteger capacidade, mas não deve tratá-los
como uma garantia dura quando o banco está unhealthy — o tradeoff documentado é
a disponibilidade do caminho de chat em vez de medição estrita.

## Chaves com escopo de projeto

API keys podem ser criadas com um rótulo `project` (`api_keys.project`,
`default` quando não definido). A aplicação de cota o honra:

- Uma chave marcada com um projeto diferente de `default` verifica sua cota
  contra o uso atribuído ao **bucket do próprio projeto**
  (`PROJECT_MONTHLY_USAGE_SQL`).
- Chaves `default` / sem marcação mantêm a janela do **usuário inteiro**,
  correspondendo ao comportamento pré-projeto.

Linhas RPC atribuídas por JWT (`jwt-<user-uuid>`) não carregam rótulo de
projeto e são **intencionalmente excluídas** das janelas de projeto — elas ainda
contam para a janela do usuário inteiro, então um projeto não pode ser
"escondido" enviando tráfego pelo canal RPC.

## Precificação de vídeo

A geração de vídeo usa precificação específica por modelo, estilo task
(precificação por token não faz sentido para um vídeo). Linhas de preço vivem na
tabela `video_pricing`; `compute_cost` cai para um padrão placeholder quando
nenhuma linha está configurada.

| Modo | Custo |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (padrão) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` é um objeto JSON chaveado pelo valor de pixel do lado curto
(ex. `"768"`); a chave `"*"` é o fallback. A linha de preço padrão é o modo
`per_second_resolution`, `base_price` 0,0, `price_per_second` 0,005,
`resolution_coeff {"*": 1.0}`. Linhas são gerenciadas via
`billing.video.pricing.get` (qualquer JWT) e `billing.video.pricing.set`
(Bearer `ARONA_ADMIN_TOKEN`); o custo calculado é gravado contra a chave do job
quando o job conclui.

## usage.list

`usage.list` (JWT) retorna os registros de uso paginados do caller cobrindo
**tanto** linhas de API key (join via prefixo de chave) quanto linhas atribuídas
por JWT (join via id sintético `jwt-<user-uuid>`), mais recentes primeiro:

| Parâmetro | Padrão | Notas |
| --- | --- | --- |
| `limit` | `50` | Limitado a `1..=200`. |
| `offset` | `0` | Offset de página. |
| `project` | não definido | Quando definido, apenas registros atribuídos a chaves com aquele rótulo de projeto (linhas JWT são excluídas). |

A resposta é `{ "records": [...], "total", "limit", "offset", "project" }`,
onde cada registro carrega `id`, `model`, `backend`, `prompt_tokens`,
`completion_tokens`, `total_tokens`, `cost` e `created_at`. A agregação de cota
mensal usa o mesmo shape de join, então a matemática de cota e a visão de uso
sempre concordam sobre o escopo.

## Relacionados

- [Quickstart](quickstart.md) — obtenha uma chave e faça sua primeira requisição medida.
- [Configuração](configuration.md) — variáveis de ambiente para o gateway.
- [Autenticação e Segurança](auth-security.md) — criação de API keys e o rótulo `project`.
- [Realtime e vídeo](realtime-video.md) — o ciclo de vida dos jobs de vídeo por trás da precificação de vídeo.
- [Operações](operations.md) — health probes e observabilidade.
- [API REST compatível com OpenAI](../api/openai-rest.md) — a superfície `/v1/*`.
- [API JSON-RPC](../api/jsonrpc.md) — `usage.list`, `billing.plan`, `billing.video.pricing.*`.
- [Visão geral](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
