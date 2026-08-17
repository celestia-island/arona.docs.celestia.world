---
title: "Referência da API JSON-RPC"
description: "API JSON-RPC 2.0 do plano de gerenciamento do Arona em /api/rpc — métodos de chat, realtime, engine, auth, keys, providers, agents, memory, conversations, usage, billing, video e system por HTTP e WebSocket."
---

# Referência da API JSON-RPC

O Arona expõe uma superfície JSON-RPC 2.0 em `/api/rpc` para o plano de
gerenciamento: auth, keys, providers, agents, memory, conversations, usage,
billing, video, realtime e chat streaming. Ela complementa a superfície REST
compatível com OpenAI (`/v1/*`, veja [API REST compatível com OpenAI](./openai-rest.md));
use REST para workloads de inferência autenticados por chave e JSON-RPC para
gerenciamento de sessão/conta e controle de streaming. O
[Quickstart](../guides/quickstart.md) percorre a primeira rodada de ponta a
ponta.

A superfície despacha **39 métodos de requisição** mais um método de liveness
anônimo somente-WebSocket, `system.probe` (40 métodos no total). Toda
requisição é um objeto JSON-RPC 2.0 com `jsonrpc: "2.0"`, uma string `method`,
um objeto `params` opcional e um `id` opcional.

## Transporte

- **HTTP POST `/api/rpc`** — requisição/resposta. Envie `Content-Type:
  application/json`. O JWT viaja no header `Authorization: Bearer <jwt>`.
  O corpo da requisição é limitado a 1 MiB.
- **WebSocket `GET /api/rpc`** — conexão de longa duração. Browsers não podem
  definir custom headers no upgrade WebSocket, então o JWT viaja como um
  parâmetro de query `?token=<jwt>`; o servidor o dobra em um header
  `Authorization: Bearer` internamente (veja `packages/core/src/gateway/server.rs`).
  Sockets autenticados podem permanecer conectados indefinidamente.
- **Requisições batch** — um corpo POST que é um array JSON é executado
  elemento por elemento e respondido com um array JSON de respostas na mesma
  ordem.
- **Acesso anônimo** — por WebSocket sem um JWT, os métodos públicos
  (`auth.register`/`auth.login`/`auth.refresh`, `providers.list`,
  `system.status`) permanecem chamáveis, e `system.probe` é respondido com um
  único ack antes de o socket fechar. Todo outro método requer um JWT válido;
  os métodos com gate de admin adicionalmente requerem uma conta de admin (veja
  a legenda abaixo). Sockets anônimos também estão sujeitos a um timeout de
  idle de 10 segundos.
- **Anexo de sessão** — um header `x-session-id` em `POST /api/rpc`
  adicionalmente empurra a própria resposta RPC para o canal da sessão, junto
  com as notificações de streaming.

## Ids

Valores de `id` de requisição são ecoados com fidelidade de tipo: `null` →
`null`, strings → strings, inteiros → números, e qualquer outra coisa (floats,
objetos, inteiros fora do intervalo i64) → a renderização como string JSON. Um
`id` omitido é respondido com `null`.

## Notificações servidor → cliente (sidecar SSE)

Tokens, progresso de deploy e eventos realtime **não** são entregues no socket
WebSocket. Cada RPC de streaming cria um session id e empurra notificações para
`GET /api/rpc/events?session=<session_id>` como server-sent events. Assine o
endpoint SSE **antes ou imediatamente após** a chamada RPC retornar um session
id — notificações emitidas entre a chamada retornar e a subscrição SSE ser
estabelecida são descartadas (a janela de pré-subscrição). O padrão recomendado
é abrir o stream SSE primeiro e depois disparar o RPC.

Métodos de notificação: `chat.stream` (um token por evento de `chat.send`),
`models.progress` (progresso de download de modelo de agent de `agents.deploy`),
`realtime.event` (eventos de servidor para uma sessão realtime aberta) e
`video.progress` / `video.done` / `video.failed` (jobs assíncronos de vídeo).
Veja o catálogo completo em [Eventos e notificações](./events.md).

## Códigos de erro

| Código | Nome | Significado |
| --- | --- | --- |
| `-32700` | Parse error | O corpo da requisição não é JSON válido. |
| `-32600` | Invalid request | O objeto de requisição está malformado, ex. um `method` ausente. |
| `-32601` | Method not found | String `method` desconhecida; a mensagem a ecoa. |
| `-32602` | Invalid params | `params` falhou na desserialização para o método. |
| `-32603` | Internal error | Falha inesperada do servidor. |
| `-32000` | `APP_ERROR` | Erro de aplicação genérico — ex. conversa/provider/agent não encontrado, nenhum agent online disponível para deploy. |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` — JWT ausente ou inválido. Também usado por métodos de admin-token quando o bearer token não corresponde a `ARONA_ADMIN_TOKEN` (`"Admin access required"`). |
| `-32006` | `QUOTA_ERROR` | Cota mensal de billing excedida para um método RPC com gate de JWT (`chat.send`). |
| `-32007` | `ADMIN_REQUIRED` | **Não-admin** autenticado chamando um método com gate de admin (`agents.*`, `engine.invoke`); a mensagem inclui uma dica específica do método. |

> Os métodos `agents.*` e `engine.invoke` são somente-admin: eles exigem um
> JWT cuja conta tenha `users.is_admin = true`. Um não-admin autenticado é
> rejeitado com `-32007` (`ADMIN_REQUIRED`); um caller não autenticado
> recebe o padrão `AUTH_ERROR`, para que o servidor não revele que o
> método é privilegiado.

## Legenda de auth

| Legenda | Credenciais |
| --- | --- |
| **public** | Nenhuma credencial exigida. |
| **JWT** | `Authorization: Bearer <jwt>` no HTTP, ou `?token=<jwt>` no WebSocket. |
| **admin (JWT + is_admin)** | Bearer JWT de uma conta com `users.is_admin = true`. |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN` (configurada por env; quando não definida, o método é sempre negado, default-deny). |

Todos os exemplos de credenciais e endereços neste documento são placeholders
(IPs de documentação RFC 5737, chaves `sk-xxx`). Veja
[Autenticação e Segurança](../guides/auth-security.md) para o modelo de auth
completo por trás desta legenda.

## Chat

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model` (string), `messages` (array de `{ role, content, images?, tool_calls? }`), `temperature?` (number), `max_tokens?` (integer), `conversation_id?` (string), `memory?` (bool), `extra?` (object), `tools?` (array de definições de função estilo OpenAI), `provider?` (string) | Envia uma rodada de chat streaming. Retorna `{ "stream_id", "memory" }` — `memory` é o estado de recall (`enabled` / `disabled` / `offline`); tokens chegam como notificações `chat.stream` no sidecar SSE. Com um `conversation_id`, o histórico persistido concluído é montado no lado do servidor e a rodada é persistida. Com gate de billing (cota mensal → `-32006`); o uso é gravado sob `jwt-<user-uuid>`. |

## Realtime (sessões full-duplex de áudio/vídeo)

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model` (string), `config?` (objeto de configuração de sessão), `conversation_id?` (string) | Abre uma sessão full-duplex contra o backend que serve `model`. Retorna `{ "session_id", "stream_session" }`: use `session_id` para `realtime.event` / `realtime.stop`, e assine `stream_session` no sidecar SSE para receber notificações `realtime.event`. |
| `realtime.event` | JWT | `session_id` (string), `event` (evento de cliente — append/commit/clear de áudio, frame de imagem, response create/cancel, session stop) | Envia um evento de cliente para dentro de uma sessão aberta; ele é encaminhado ao backend upstream. Retorna `{ "ok": true }`. |
| `realtime.stop` | JWT | `session_id` (string) | Fecha e remove uma sessão. Retorna `{ "removed": bool }`. |

## Engine (canal genérico de percepção/controle)

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `engine.invoke` | admin (JWT + is_admin) | `model` (string), `method` (string), `params?` (object) | Invocação síncrona request/response de um método de engine arbitrário no backend que serve `model` — o canal de alta frequência para chamadas estilo `sensor.ingest` / `control.setpoint` (loops de 20–30 Hz). O resultado é a resposta bruta do backend. |

## Auth

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `auth.register` | public | `email`, `password`, `name?` | Registra uma conta. Permitido apenas enquanto o registro está aberto (`ARONA_REGISTRATION_OPEN`); o primeiro usuário registrado vira o admin. Retorna a mesma resposta de token que `auth.login` (`access_token`, `refresh_token`, `token_type`, `expires_in`, `user`). |
| `auth.login` | public | `email`, `password` | Faz login. Retorna `access_token`, `refresh_token`, `token_type`, `expires_in`, `user` (`{ id, email, name, is_admin }`). Com limite de taxa por IP e conta. |
| `auth.refresh` | public | `refresh_token` | Troca um refresh token por um access token novo (e um refresh token novo). Refresh tokens reutilizados ou expirados são rejeitados com `AUTH_ERROR`. |
| `auth.me` | JWT | — | Perfil do usuário atual: `{ "id", "email", "name" }`. |

## Chaves

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | Lista as API keys do caller (id, name, `key_prefix`, project, timestamps, flag ativa). |
| `keys.create` | JWT | `name`, `project?` | Cria uma API key. Retorna `{ id, name, key, key_prefix, project, created_at }` — o segredo completo `arona-<uuid>` em `key` é mostrado **uma vez**; armazene-o imediatamente. |
| `keys.revoke` | JWT | `key_id` | Revoga uma API key. Retorna `{ "ok": true }`. |

## Providers

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | Lista providers conhecidos: entradas oficiais integradas mais as customizadas, como metadados de exibição (`id`, `name`, `description`, `website_domain`, `is_official`, `is_operator`). Público por design — a lista não carrega credenciais; apenas as mutações abaixo têm gate de JWT. |
| `providers.add` | JWT | `id`, `name`, `description?`, `website_domain?` | Adiciona uma entrada de provider customizada. Retorna `{ "ok": true }`. |
| `providers.update` | JWT | `provider_id`, `name?`, `description?`, `website_domain?` | Atualiza os campos de um provider customizado (apenas os fornecidos). Retorna `{ "ok": true }`. |
| `providers.remove` | JWT | `provider_id` | Remove um provider customizado. Retorna `{ "ok": true }`. |
| `providers.test` | JWT | — | Testa uma conexão de provider. Stub: retorna `{ "ok": true, "message": "Provider connection test not yet implemented" }`. |

## Agents

Todos os métodos `agents.*` são somente-admin (JWT + `is_admin`). Nós de agent
conectam outbound por `GET /ws/agent`; este grupo RPC controla o registry (veja
[Cluster de Agents](../guides/agent-cluster.md)).

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `agents.list` | admin (JWT + is_admin) | — | Lista nós de agent registrados: id, name, host, status `online`/`offline` (baseado em heartbeat), resumo de GPU, modelos implantados, version, timestamps. |
| `agents.register` | admin (JWT + is_admin) | `machine_name`, `version` | Registra um nó de agent no tunnel manager. Retorna `{ "agent_id", "token" }` (o token é a credencial do plano de controle do agent). |
| `agents.deregister` | admin (JWT + is_admin) | `agent_id` | Desregistra (desconecta) um agent. Retorna `{ "ok": true }`. |
| `agents.status` | admin (JWT + is_admin) | `agent_id` | Status por agent: flag online, host, resumo de GPU, modelos carregados, utilização de GPU, timestamps de heartbeat/conexão. |
| `agents.deploy` | admin (JWT + is_admin) | `model_id`, `agent_id?` (vazio/ausente = nó menos carregado; erro se nenhum estiver online) | Implanta um modelo em um agent. Retorna `{ "ok": true, "stream_id" }` — assine `stream_id` no sidecar SSE para notificações de download `models.progress`. |
| `agents.stop` | admin (JWT + is_admin) | `agent_id`, `model_id` | Para um modelo implantado. Retorna `{ "ok": true, "stream_id": null }` (sem stream de progresso). |

## Memória

A memória de longo prazo é servida pelo serviço entelecheia Philia por um
WebSocket; falhas nunca bloqueiam o chat (veja
[Gateway de Memória](../guides/memory-gateway.md)).

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | Estado do gateway de memória: `{ "enabled", "writeback", "events" }` — flags mais até 50 eventos de atividade recentes (mais recentes primeiro). |
| `memory.delete` | JWT | `node_id` | Deleta um nó de memória armazenado. Retorna `{ "deleted": bool }`. |

## Conversas

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | Lista as conversas do caller com timestamps de idade relativa. |
| `conversations.create` | JWT | `title?` (padrão `New Conversation`) | Cria uma conversa. Retorna o objeto de conversa novo. |
| `conversations.get` | JWT | `conversation_id` (alias legado: `id`) | Busca uma conversa com suas mensagens. Verificado quanto a ownership; acesso cross-user é rejeitado. |
| `conversations.delete` | JWT | `conversation_id` (alias legado: `id`) | Deleta uma conversa (apenas o dono). Retorna `{ "ok": true }`. |

> `conversations.get` / `conversations.delete` também aceitam a chave `id`
> legada de clientes de dashboard antigos; `conversation_id` vence quando ambos
> estão presentes.

## Uso

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?` (integer, padrão 50, limitado a 1–200), `offset?` (integer, padrão 0), `project?` (string) | Registros de uso paginados do caller, mais recentes primeiro, cobrindo tanto linhas de API key (prefixo `arona-XX`) quanto linhas atribuídas por JWT (`jwt-<user-uuid>`). Retorna `{ "records", "total", "limit", "offset", "project" }`; o filtro `project` restringe a linhas marcadas por chave apenas. |

## Billing

Tiers, cotas e contabilidade de uso são descritos em
[Billing e uso](../guides/billing-usage.md).

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | Estado de billing atual: `{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` — uso mensal (`cost_usd`, tokens, contagem de requisições) e cota restante. |
| `billing.plan.set` | admin token | `user_email`, `tier` | Define o tier de billing de um usuário. Retorna `{ "ok": true }`. Negado com `AUTH_ERROR` quando o bearer não corresponde a `ARONA_ADMIN_TOKEN`. |
| `billing.video.pricing.get` | JWT | — | Tabela de precificação de vídeo. Retorna `{ "pricing": [...] }`. |
| `billing.video.pricing.set` | admin token | `model`, `mode?` (padrão `per_second_resolution`), `base_price?` (number, padrão 0), `price_per_second?` (number, padrão 0), `price_per_frame?` (number, padrão 0), `resolution_coeff?` (object), `currency?` (padrão `USD`), `enabled?` (bool, padrão `true`) | Faz upsert da precificação de vídeo para um modelo. Retorna `{ "ok": true }`. Negado com `AUTH_ERROR` quando o bearer não corresponde a `ARONA_ADMIN_TOKEN`. |

## Vídeo

Jobs assíncronos de geração de vídeo (veja
[Realtime e Vídeo](../guides/realtime-video.md)). O progresso do job é
empurrado como notificações `video.progress` / `video.done` / `video.failed` no
canal da sessão.

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`, `prompt`, `negative_prompt?`, `images?` (array de `{ data_base64, mime_type }`), `duration_seconds?` (integer), `width?` (integer), `height?` (integer), `provider?` (string), `extra?` (object) | Submete um job assíncrono de geração de vídeo. Retorna `{ "job_id", "stream_id" }` — assine `stream_id` para notificações de progresso. |
| `video.get` | JWT | `job_id` (UUID) | Faz poll do status/resultado de um job (status, progress, result, error, cost). |
| `video.list` | JWT | `limit?` (integer, padrão 20) | Lista os jobs do caller. Retorna `{ "jobs": [...] }`. |
| `video.cancel` | JWT | `job_id` (UUID) | Cancela um job em execução. Retorna `{ "ok": true }`. |

## Sistema

| Método | Auth | Params | Descrição |
| --- | --- | --- | --- |
| `system.status` | public | — | Status agregado do gateway: `{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`. |
| `system.probe` | anonymous (somente WS) | — | Probe de liveness de disparo único pelo transporte WebSocket. O servidor responde com ack `{ "ok": true, "status": "ok" }` e fecha o socket — visitantes anônimos nunca seguram uma conexão aberta. Qualquer outro método em um socket não autenticado é rejeitado com `AUTH_ERROR`. |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
