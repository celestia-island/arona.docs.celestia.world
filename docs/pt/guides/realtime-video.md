---
title: "Realtime e Vídeo"
description: "Sessões realtime full-duplex (realtime.start/event/stop), o canal de percepção/controle engine.invoke e jobs assíncronos de geração de vídeo."
---

# Realtime e Vídeo

O Arona expõe dois canais multimodais além do chat de texto puro: **sessões
realtime full-duplex** (fala/vídeo de entrada e saída por um canal
bidirecional) e **geração de vídeo assíncrona** (jobs estilo task que rodam em
background e reportam progresso). Ambos são roteados para o backend que serve o
modelo solicitado e ambos são medidos pela camada de billing.

## Sessões realtime

Uma sessão realtime é um canal bidirecional entre **um cliente** e **um
upstream**: uma API realtime de nuvem (vocabulário WebSocket OpenAI-Realtime /
Qwen-Omni-Realtime) ou um engine CEP local. Eventos de cliente chegam via
JSON-RPC e são encaminhados ao upstream; eventos de servidor são empurrados de
volta como notificações `realtime.event` pelo canal SSE da sessão. O áudio
viaja como base64 PCM16 (16 kHz cliente→modelo, 24 kHz modelo→cliente),
correspondendo ao formato de wire dos vendors de nuvem, para que o gateway
passe bytes adiante intactos (`packages/core/src/backends/realtime.rs:1-19`).

### `realtime.start`

Abre uma sessão contra o backend que serve `model` (JWT; params `model`,
`config?`, `conversation_id?` — `packages/core/src/gateway/rpc.rs:1890-1898,
1914-1984`). O backend **deve** declarar a capacidade `realtime` (modalidades
áudio/vídeo); caso contrário a chamada falha explicitamente com
`model {model} does not support realtime sessions (no audio/video modality)` —
não há fallback silencioso para chat de texto
(`packages/core/src/gateway/realtime.rs:138-142`).

Dois kinds de upstream são suportados (`packages/core/src/gateway/realtime.rs:143-167`):

- **Upstream de engine CEP** — roteia eventos pelo canal de streaming
  `Engine.InvokeStart` do Celestia Engine Protocol, então um engine omni
  implantado localmente junta-se à mesma superfície de cliente sem novo formato
  de wire.
- **Upstream de nuvem** — uma URL `wss://` fixa falando o vocabulário de
  eventos realtime de nuvem (`session.update`, `input_audio_buffer.*`,
  `response.audio.delta`, ...). A impl de nuvem re-emite `session.update` na
  reconexão.

A resposta é `{ "session_id": ..., "stream_session": ... }` — assine
`/api/rpc/events?session=<stream_session>` antes (ou imediatamente após) a
chamada para receber eventos de servidor. O `conversation_id` opcional persiste
a transcrição de fala como mensagens de assistente e grava uso de tokens para
billing (`packages/core/src/gateway/realtime.rs:32-85`).

### `realtime.event`

Envia um evento de cliente para dentro da sessão (JWT; params `session_id`,
`event` — `packages/core/src/gateway/rpc.rs:1989-2013`). Eventos suportados
incluem `session.update`, `input_audio_buffer.append` / `.commit` / `.clear`,
`input_image_buffer.append`, `response.create`, `response.cancel` e
`session.stop`. `send_event` é **não-bloqueante**: o evento é enfileirado em um
canal mpsc e a task forwarder o escreve no upstream
(`packages/core/src/gateway/realtime.rs:254-280`). Apenas o dono da sessão pode
enviar eventos.

### `realtime.stop`

Fecha e remove a sessão (JWT; params `session_id` —
`packages/core/src/gateway/rpc.rs:2016-2034`). Cada sessão é dona de exatamente
uma **task forwarder** que segura o upstream e multiplexa ambas as direções:
eventos de cliente são consumidos da fila e eventos de upstream são pollados no
mesmo loop. A forwarder sai quando o upstream fecha ou a sessão é parada,
removendo a entrada do registry
(`packages/core/src/gateway/realtime.rs:195-250`).

Eventos de servidor são empurrados como notificações `realtime.event` com
params `{ session_id, event }` pelo canal da sessão — veja
[Eventos e notificações](../api/events.md).

## `engine.invoke`

`engine.invoke` é o canal genérico **síncrono** de métodos de engine
(ADMIN: JWT + `is_admin`; params `model`, `method`, `params?` —
`packages/core/src/gateway/rpc.rs:261-264,2049-2079`). Ele invoca um método
arbitrário no backend que serve `model` e retorna o resultado diretamente,
tornando-o o canal de alta frequência de percepção/controle: chamadas estilo
`sensor.ingest`, `control.setpoint` em loops de 20-30 Hz. Backends sem um canal
genérico de invocação (todos os backends HTTP compatíveis com OpenAI) rejeitam
explicitamente com `backend does not support generic invocation`
(`packages/core/src/backends/mod.rs:573-586`).

## Geração de vídeo (REST)

Jobs de vídeo são tasks assíncronas estilo OpenAI sobre a superfície REST (auth
por API key — `packages/core/src/gateway/server.rs:876-993`; veja
[API REST compatível com OpenAI](../api/openai-rest.md)):

**`POST /v1/video/generations`**

| Campo | Tipo | Notas |
| --- | --- | --- |
| `model` | string | obrigatório — seleciona um backend com capacidade de vídeo. |
| `prompt` | string | obrigatório. |
| `negative_prompt` | string? | |
| `images` | array? | Data URLs base64 (`data:image/png;base64,...`), carregadas como `{ data_base64, mime_type }`. |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | Dica de seleção de backend (casada contra o nome do backend). |
| `extra` | object? | Overrides específicos do backend (seed, steps, cfg, ...). |

Resposta:

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`** faz poll do job e retorna `id`, `object`,
`model`, `status`, `progress`, `result`, `error`, `cost`, `created_at`. Jobs
têm escopo do caller: um job de outro usuário retorna 404. A superfície REST
aplica os mesmos gates de billing (cota mensal, limite de taxa por minuto) que o
caminho de chat.

## Geração de vídeo (RPC)

A mesma capacidade está disponível via JSON-RPC (JWT —
`packages/core/src/gateway/rpc.rs:371-386,1296-1431`):

| Método | Params | Retorna |
| --- | --- | --- |
| `video.create` | mesmos campos da chamada REST | `{ job_id, stream_id }`. |
| `video.get` | `job_id` | A visão do job (status, progress, result, cost, ...). |
| `video.list` | `limit?` (padrão 20, limitado a 1-100) | `{ jobs: [...] }`, mais recentes primeiro. |
| `video.cancel` | `job_id` | `{ "ok": true }` — apenas o dono pode cancelar. |

`video.create` retorna um `stream_id`; assine
`/api/rpc/events?session=<stream_id>` para receber notificações de job
(`video.progress` / `video.done` / `video.failed` — veja
[Eventos e notificações](../api/events.md)).

## Backend

A geração de vídeo é **somente-nuvem**: a API MiniMax H3 Open Platform, kind de
backend `minimax-cloud` (`BackendKind::CloudVideo` —
`packages/core/src/backends/mod.rs:502-504,720-727,759-761`). O fluxo é estilo
task:

1. `POST /v1/video_generation_v2` → `task_id`
2. faça poll de `GET /v1/query/video_generation_v2?task_id=...` até `Success` /
   `Fail` / ainda `Pending`
3. no sucesso, resolva o artefato via
   `GET /v1/files/{file_id}/content` → `{ file: { download_url } }`

(`packages/core/src/backends/minimax_cloud.rs:66-210`). O backend MiniMax não
serve chat/embeddings; suas capacidades declaram `supports_video_generation` e
`realtime: false` (veja [Backends](./backends.md) para o modelo de
capacidades). O roteamento resolve requisições de vídeo apenas contra backends
com `supports_video_generation`, honrando a dica `provider` opcional
(`packages/core/src/routing/mod.rs:205-263`).

O **backend ComfyUI foi removido** durante a convergência da plataforma de
modelos: configurar o kind de backend `"comfyui"` falha com
`comfyui backend removed` (`packages/core/src/backends/mod.rs:756-757`). Não há
caminho de vídeo self-hosted; o vídeo sempre passa por um backend
`minimax-cloud`.

## Ciclo de vida dos jobs e precificação

Um job de vídeo move-se por `queued → running → done | failed | cancelled`
(`packages/core/src/gateway/video.rs`):

- **create** — a linha do job é persistida (`queued`, progresso 0) e uma task
  poller é spawnada (`video.rs:109-176`).
- **running** — o poller submete a task (progresso 5), depois faz poll a cada
  1,5 s, aumentando o progresso em 5 a cada poucas iterações até **90**
  (`video.rs:178-275`). Erros de poll são logados e tentados novamente.
- **done** — progresso 100, a URL do resultado e o custo calculado são
  persistidos, o uso é gravado e uma notificação `video.done` é disseminada
  (`video.rs:332-368`).
- **failed** — falha de submit ou poll → `video.failed`; após 900 iterações de
  poll (~22,5 minutos) o job falha com `generation timed out`.
- **cancelled** — `video.cancel` define uma flag que o poller observa na sua
  próxima passada; o job é marcado `cancelled` e `video.failed` dispara com erro
  `cancelled` (`video.rs:389-400`).

O uso é gravado com o custo específico de vídeo: `record_video` escreve um
registro de uso por requisição com zero tokens e um custo explícito em dólares
(`packages/core/src/billing/mod.rs:496-531`).

**Precificação** é específica por modelo, na tabela `video_pricing`
(`packages/core/src/billing/video.rs`):

| Modo | Fórmula |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (padrão) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` mapeia a chave de pixel do lado curto (ex. `"768"`) para um
multiplicador, com `"*"` como fallback. Modelos sem uma linha configurada caem
para: modo `per_second_resolution`, `base_price` 0,0, `price_per_second` 0,005,
`price_per_frame` 0,0, `resolution_coeff {"*": 1.0}`, moeda USD
(`billing/video.rs:20-32`). Consulte linhas com `billing.video.pricing.get`
(JWT) e faça upsert com `billing.video.pricing.set` (token de admin) — veja
[API JSON-RPC](../api/jsonrpc.md). Veja [Billing e uso](./billing-usage.md)
para como os registros de uso agregam no billing mensal.

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
