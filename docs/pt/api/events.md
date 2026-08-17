---
title: "Eventos e Notificações"
description: "Sidecar de server-sent events (SSE) — notificações chat.stream, models.progress, realtime.event e de vídeo."
---

# Eventos e Notificações

Tokens de streaming, progresso de deploy e eventos realtime **não** são
entregues no socket WebSocket JSON-RPC. Cada RPC de streaming cria um
**session id** e empurra notificações para o endpoint SSE como server-sent
events:

```
GET /api/rpc/events?session=<session_id>
```

## Receita: assine antes de enviar

Notificações emitidas entre a chamada RPC retornar um session id e a subscrição
SSE ser estabelecida são **descartadas** (a janela de pré-subscrição). O padrão
confiável é:

1. Abra o stream SSE primeiro (ele bloqueia até um session id ser anexado).
2. Dispare o RPC que retorna o session id (ex. `chat.send`, `agents.deploy`,
   `realtime.start`, `video.create`).
3. Leia notificações do stream SSE conforme elas chegam.

Toda notificação é uma mensagem estilo JSON-RPC 2.0 com `"jsonrpc": "2.0"`, um
`method` e um objeto `params`.

## Catálogo de notificações

### `chat.stream`

Uma notificação por token, produzida por `chat.send` (e qualquer caminho de
chat streaming que use um canal de sessão):

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — um delta de conteúdo.
- `is_complete` — `false` até o chunk final (quando o upstream anexa um finish
  reason, o chunk final de conteúdo pode já carregar `is_complete: true` com um
  token não vazio); a notificação **terminal** sempre vem em seguida com um
  `token` vazio e `is_complete: true`.
- Um erro de stream é entregue como uma notificação terminal com
  `token: "Stream error: ..."` e `is_complete: true` (veja
  `packages/core/src/gateway/rpc.rs`).

### `models.progress`

Progresso de download de modelo para `agents.deploy`, encaminhado do agent. O
`stream_id` vem da resposta de `agents.deploy`.

### `realtime.event`

Eventos de servidor para uma sessão realtime full-duplex aberta, empurrados
para o canal da sessão (`packages/core/src/gateway/realtime.rs`). Eventos de
cliente enviados via RPC `realtime.event` são encaminhados ao upstream; eventos
de servidor chegam aqui.

### Notificações de job de vídeo

Jobs de `video.create` empurram progresso pelo canal da sessão
(`packages/core/src/gateway/video.rs`):

| Método | Payload (params) | Significado |
| --- | --- | --- |
| `video.progress` | `job_id`, `stream_id`, `status: "running"`, `progress` (0–90) | O job está rodando. |
| `video.done` | `job_id`, `stream_id`, `result`, `cost` | O job terminou; `result` carrega a URL do artefato. |
| `video.failed` | `job_id`, `stream_id`, `error` | O job falhou ou foi cancelado. |

## Notas sobre ordenação

- O stream SSE é ordenado por sessão; tokens chegam na ordem de geração.
- Um único session id só pode ser consumido por um assinante SSE; abra o stream
  antes (ou imediatamente após) o RPC que retorna o id.
- O header `x-session-id` em `POST /api/rpc` anexa a própria **resposta** RPC a
  um canal de sessão também — usado por clientes que querem a resposta ecoada
  pelo mesmo stream.

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
