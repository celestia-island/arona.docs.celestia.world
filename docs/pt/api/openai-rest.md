---
title: "API REST compatível com OpenAI"
description: "Referência /v1/* estilo OpenAI — completions de chat, embeddings, listagem de modelos, gerações de vídeo assíncronas, shapes de erro e limites de taxa."
---

# API REST compatível com OpenAI

O Arona expõe uma superfície REST compatível com OpenAI sob `/v1/*` para chat
LLM, embeddings, listagem de modelos, health probing e geração assíncrona de
vídeo. Qualquer SDK OpenAI apontado para a URL base funciona para chat e
embeddings; os endpoints de vídeo seguem a convenção submit/poll estilo task do
OpenAI.

Todos os corpos de requisição e resposta são JSON. Erros usam um shape uniforme
(veja [Erros](#errors)); falhas de autenticação na camada de middleware são a
única exceção e são retornadas como texto puro (veja
[Autenticação](#authentication)).

## Endpoints em resumo

| Método | Path | Descrição |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | Rodada de chat, streaming ou não-streaming. |
| `POST` | `/v1/embeddings` | Vetores de embedding para uma ou muitas entradas. |
| `GET` | `/v1/models` | Modelos do router mesclados com modelos quick-start. |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`. |
| `POST` | `/v1/video/generations` | Submete uma task assíncrona de geração de vídeo. |
| `GET` | `/v1/video/generations/{id}` | Faz poll do status / resultado de uma task de vídeo. |

`/api/health`, `/healthz` e `/readyz` são probes adicionais de readiness
(aliases estilo Kubernetes de `/v1/health`).

## Autenticação

Endpoints de chat, embeddings e vídeo autenticam com uma **API key** no header
`Authorization: Bearer`. API keys são criadas pelo plano de gerenciamento
(`keys.create`, veja a [API JSON-RPC](./jsonrpc.md#keys)) e parecem com
`arona-<uuid>`. Elas são armazenadas no lado do servidor como hashes SHA-256.

```
Authorization: Bearer arona-CHANGE_ME
```

- **Header ausente** → `401` texto puro: `Missing Authorization header. Use: Bearer <api_key>`.
- **Chave inválida ou revogada** → `401` texto puro: `Invalid API key`.
- `GET /v1/models` adicionalmente aceita um **JWT** access token (emitido por
  `auth.login` / `auth.register`) para que o dashboard web possa listar modelos
  com o mesmo token que usa no plano RPC. Para esse endpoint, as mensagens são
  `Missing Authorization header. Use: Bearer <api_key_or_jwt>` e
  `Invalid API key or JWT`.

Rejeições em nível de middleware são corpos de texto puro, não o shape de erro
JSON descrito em [Erros](#errors) — o shape JSON é produzido apenas quando uma
requisição chega a um handler.

Toda requisição `/v1` autenticada também passa por um **limitador de taxa em
memória por chave** (padrão 60 RPM, janela de 60 segundos, configurável via
`ARONA_API_RATE_LIMIT_RPM`). Excedê-lo retorna `429` texto puro:
`Rate limit exceeded. Try again later.` Cotas e limites de taxa em nível de tier
são aplicados separadamente e retornam 429s JSON com um header `Retry-After`
(veja [429 e Retry-After](#429-and-retry-after)).

> Gerenciar API keys, projetos e seu escopo é coberto em
> [Autenticação e Segurança](../guides/auth-security.md).

## POST /v1/chat/completions

O endpoint de chat compatível com OpenAI central, com suporte a streaming e
extensões específicas do arona (`conversation_id`, `memory`, `extra`,
`provider`).

### Corpo da requisição

| Campo | Tipo | Obrigatório | Notas |
| --- | --- | --- | --- |
| `model` | string | sim | Id do modelo conforme listado por `GET /v1/models`. |
| `messages` | array | sim | Rodadas de chat, veja abaixo. |
| `stream` | boolean | não | Padrão `false`. Quando `true`, a resposta é um stream SSE (veja [Streaming](#streaming)). |
| `temperature` | number | não | Temperatura de amostragem, encaminhada ao upstream. |
| `max_tokens` | integer | não | Teto de tokens de completion, encaminhado ao upstream. |
| `conversation_id` | string | não | Afinidade de sessão + persistência. A conversa deve existir e pertencer ao usuário da API key (`403` `conversation_forbidden` caso contrário, `404` `conversation_not_found` se ela não existir). A rodada do usuário é persistida no momento do envio e a resposta do assistente quando a rodada conclui; o roteamento fixa a conversa no backend que a serviu primeiro. |
| `memory` | boolean | não | Override do gateway de memória. Padrão `true` (o recall de memória é injetado quando o gateway de memória está habilitado); `false` desabilita a injeção de recall para esta requisição. |
| `extra` | object | não | Pass-through de forma livre mesclado no nível superior do payload do upstream (veja abaixo). |
| `tools` | array | não | Definições de function-call estilo OpenAI, passadas verbatim ao upstream. |
| `provider` | string | não | Dica explícita de seleção de backend casando com um **nome** de backend (ou kind) de forma insensível a maiúsculas/minúsculas. Quando definida, apenas backends que casam com a dica são candidatos. |

**Entradas de `messages`** são `{ "role": "user" | "assistant" | "system", "content": "..." }`.
Duas extensões são encaminhadas ao upstream para workloads multimodais / de
agent:

- `images` — imagens anexadas para requisições de visão (um array de objetos
  `{ "media_type", "data", "position" }`; o backend external as renderiza como
  partes de conteúdo OpenAI `image_url`).
- `tool_calls` — payloads de function-call produzidos pelo modelo do upstream,
  para serem ecoados de volta em rodadas de acompanhamento. O backend external
  DEVE encaminhá-los ou pipelines de agent (ex. cadeias de skill do scepter)
  perdem toda invocação de tool.

**Regras de merge do `extra`**: toda chave de `extra` é mesclada no payload de
requisição do upstream no nível superior, com duas garantias duras — as chaves
reservadas `model`, `messages`, `stream`, `temperature`, `max_tokens` e
`options` **nunca** são sobrescritas, e tampouco qualquer chave que o próprio
gateway já definiu. Valores de `extra` não-objeto são ignorados.

**Entradas de `tools`** são `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }` e são encaminhadas verbatim.

### Resposta não-streaming

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1720000000,
  "model": "Qwen/Qwen3-1.7B",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 12, "completion_tokens": 2, "total_tokens": 14 }
}
```

- `choices[].message` pode carregar `tool_calls` para rodadas de function-calling.
- O estado de memória da requisição é ecoado no **header de resposta
  `X-Arona-Memory`**: `enabled` | `disabled` | `offline`.

### Streaming

Defina `"stream": true`. A resposta é um stream SSE `text/event-stream` — uma
linha `data:` por chunk, cada uma carregando um único `ChatChunk` JSON:

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta` carrega `content` (e deltas de `tool_calls` com
  `index`/`id`/`type`/`function` para streams de function-calling).
- Em upstreams compatíveis com OpenAI, o **chunk terminal** carrega um campo
  `usage` com as contagens reais de tokens; o gateway o grava (caindo para uma
  estimativa local de tokenizer para upstreams que nunca reportam uso, ex.
  ollama / ws_engine).
- O stream termina com `data: [DONE]`.
- Um erro de stream é entregue como um evento `data:` carregando
  `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}`;
  o evento `[DONE]` ainda vem em seguida, e a gravação de uso mais a persistência
  do assistente são puladas para o stream com falha.
- O header `X-Arona-Memory` também está presente na resposta SSE.

### Exemplo

```bash
curl http://192.0.2.10:8080/v1/chat/completions \
  -H "Authorization: Bearer arona-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-1.7B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

## POST /v1/embeddings

| Campo | Tipo | Obrigatório | Notas |
| --- | --- | --- | --- |
| `model` | string | sim | Id do modelo de embedding (ex. `nomic-embed-text` — um nome simples também casa com um tag `:latest`). |
| `input` | string ou string[] | sim | Uma entrada, ou muitas. |

Resposta: `{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`.

## GET /v1/models

Lista os modelos roteáveis hoje: a listagem de modelos de todo backend healthy
registrado, mesclada com os **modelos quick-start** integrados (sempre
anunciados, mesmo antes de um backend ser registrado): `Qwen/Qwen3-0.6B`,
`Qwen/Qwen3-1.7B`, `HuggingFaceTB/SmolLM2-1.7B-Instruct`,
`google/gemma-3-1b-it`, `microsoft/Phi-4-mini-instruct`,
`deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`.

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

Modelos quick-start aparecem com `owned_by` definido para seu provider; modelos
do router carregam o nome do backend dono.

## Geração de vídeo

Endpoints de vídeo estilo task para backends com capacidade de vídeo (ex.
`minimax-cloud`, veja [Backends](../guides/backends.md)). Jobs progridem
assincronamente; faça poll do endpoint de status até `done`.

### POST /v1/video/generations

| Campo | Tipo | Obrigatório | Notas |
| --- | --- | --- | --- |
| `model` | string | sim | Id de modelo de vídeo registrado em um backend com capacidade de vídeo. |
| `prompt` | string | sim | Prompt de geração. |
| `negative_prompt` | string | não | Prompt negativo. |
| `images` | array | não | Imagens de condicionamento/referência como um array de objetos `{ "data_base64": "...", "mime_type": "image/png" }`. |
| `duration_seconds` | integer | não | Duração solicitada. |
| `width` / `height` | integer | não | Resolução de saída. |
| `provider` | string | não | Dica explícita de seleção de backend (nome do backend). |
| `extra` | object | não | Overrides de workflow específicos do backend (seed, steps, cfg, ...). |

Sucesso → `200`:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

Erros: `400` `missing_fields` quando `model` ou `prompt` está ausente; `503`
`video_backend_error` / `no_backend` quando nenhum backend healthy com
capacidade de vídeo serve o modelo; `429` `quota_error` / `quota_exceeded`
quando a cota mensal está esgotada.

### GET /v1/video/generations/{id}

Retorna o status da task:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "running",
  "progress": 40,
  "result": null,
  "error": null,
  "cost": 0.0,
  "created_at": 1720000000
}
```

- `status`: `queued` | `running` | `done` | `failed` | `cancelled`; `progress`
  avança de 0–90 enquanto roda e chega a 100 em `done`.
- `result` (em `done`): `{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }` — `url` aponta para o artefato gerado servido pelo backend.
- `error` (em `failed` / `cancelled`) e `cost` são preenchidos quando aplicável.
- Erros: `400` `bad_id` para um id não-UUID; `404` `no_job` quando o job não
  existe ou pertence a outra API key.

Jobs de vídeo também disseminam progresso pelo sidecar SSE do RPC
(`video.progress` / `video.done` / `video.failed`, veja
[Eventos e notificações](./events.md#video-job-notifications)).

## Erros

Erros em nível de gateway usam um shape único (`json_error_response`):

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| Status | `type` / `code` | Quando |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`, `missing_index`, `bad_id`, ... | Campos de requisição malformados ou ausentes. |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id` pertence a outro usuário. |
| `404` | `invalid_request_error` / `model_not_found` | Nenhum backend serve o modelo solicitado. Mensagem: `No backend available for model: <model>`. |
| `404` | `invalid_request` / `conversation_not_found` | Conversa não encontrada. |
| `404` | `not_found` / `no_job` | Job de vídeo não encontrado. |
| `502` | `server_error` / `bad_gateway` | Não-2xx do upstream: mensagem `upstream <status>: <detail>` (detail do corpo de erro do upstream, limitado a 4 KB). Falhas de transporte (connect/read/timeout) também mapeiam para 502 com a string do erro. |
| `500` | `server_error` / `backend_error` | Outras falhas de backend (ex. o backend não suporta a operação). |
| `500` | `server_error` / `internal_error` | Qualquer erro interno restante do gateway. |
| `429` | veja abaixo | Rejeições de cota / limite de taxa com `Retry-After`. |

## 429 e Retry-After

Respostas 429 incluem um header `Retry-After` (segundos) para que clientes
compatíveis com OpenAI recuem:

| Gatilho | Corpo do status | `Retry-After` |
| --- | --- | --- |
| Cota mensal excedida | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | Segundos até o próximo mês. |
| Limite de taxa por minuto do tier | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`. |
| Limitador em memória por chave (60 RPM padrão) | texto puro `Rate limit exceeded. Try again later.` | nenhum (rejeição de middleware). |

Tiers, escopo de cota e contabilidade de uso são descritos em
[Billing e uso](../guides/billing-usage.md).

## Gravação de uso

Toda requisição `/v1` grava uma linha de uso sob o prefixo da API key (`arona-XX`)
quando conclui (chat não-streaming, chat streaming no chunk terminal,
embeddings e jobs de vídeo na conclusão com seu custo calculado). Veja
[Billing e uso](../guides/billing-usage.md) para o modelo de gravação e como a
cota é aplicada.

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
