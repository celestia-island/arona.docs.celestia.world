---
title: "Gateway de Memória"
description: "Memória de longo prazo para chat — injeção de recall, writeback de episódios, controle por requisição, estados de header e os RPCs memory.status / memory.delete."
---

# Gateway de Memória

O Gateway de Memória dá às rodadas de chat acesso à **memória de longo prazo**
armazenada no serviço de memória entelecheia scepter / Philia. Em cada rodada de
chat, o Arona consulta o serviço por memórias relevantes à conversa, injeta-as
no prompt como uma seção de sistema e — após uma resposta concluída — grava a
rodada de volta como um episódio, para que conversas futuras possam recordá-la.

É um cliente WebSocket JSON-RPC para o Philia (`Sync.ConnectHandshake`,
`Sync.MemoryQueryRequest`, `Sync.MemoryStoreRequest`,
`Sync.MemoryDeleteRequest`). Conexões são estabelecidas de forma lazy, derrubadas
em qualquer erro e restabelecidas na próxima chamada; toda falha degrada
silenciosamente e **nunca bloqueia o caminho de chat**.

## Configuração

O gateway é controlado por três variáveis de ambiente:

| Variável | Significado |
| --- | --- |
| `ARONA_MEMORY_URL` | URL WebSocket do serviço scepter / Philia, ex. `ws://192.0.2.10:8424/ws`. |
| `ARONA_MEMORY_TOKEN` | Token para o serviço de memória. |
| `ARONA_MEMORY_WRITEBACK` | Se rodadas concluídas são gravadas de volta. Padrão **ativado**; defina `false` para desabilitar (parseado como um booleano estrito — `0` não é aceito). |

Tanto `ARONA_MEMORY_URL` **quanto** `ARONA_MEMORY_TOKEN` devem estar definidas e
não vazias, caso contrário o gateway fica **desabilitado**: recall e writeback
são pulados inteiramente e toda requisição reporta `disabled`. O token é enviado
tanto como um parâmetro de query `?token=` no upgrade WebSocket quanto dentro da
requisição `Sync.ConnectHandshake`.

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

Veja [Configuração](configuration.md) para a referência completa de ambiente.

## Injeção de recall

Com o gateway habilitado, **toda rodada de chat** — REST não-streaming
`/v1/chat/completions`, REST streaming (SSE) e RPC `chat.send` — consulta o
serviço de memória antes de a requisição ser encaminhada:

- A consulta é a **última mensagem do usuário** do contexto montado.
- Até **5** memórias são solicitadas (`limit = 5`).
- Resultados são renderizados como uma seção de sistema markdown intitulada
  `## Relevant Long-Term Memories`, um bullet `- [score] text` por memória
  (scores com duas casas decimais, entradas em branco puladas), e pré-anexados à
  lista de mensagens como uma mensagem `system`. A injeção é idempotente: um
  contexto que já carrega a seção não é re-injetado.
- Se nenhuma memória relevante é retornada, nada é injetado e a rodada prossegue
  inalterada.

O recall roda antes da persistência de conversa e do encaminhamento ao upstream;
um serviço de memória lento ou com falha **não adiciona garantia de latência**
além do seu próprio timeout RPC de 10 segundos e não pode falhar a requisição.

## Writeback

Após uma resposta de assistente concluída, a rodada é gravada de volta no
serviço de memória como um nó de **episódio**. O texto do episódio é uma
transcrição heurística da rodada — `User: <user content>\nAssistant: <assistant content>`
(qualquer lado omitido quando vazio; ambos vazios pulam o writeback). O
writeback é **fire-and-forget**: roda em uma task spawnada, nunca bloqueia a
resposta de chat e suas falhas são apenas logadas dentro do cliente de memória.
(No caminho de REST streaming, o writeback adicionalmente exige uma conversa
anexada à requisição; os caminhos REST não-streaming e RPC gravam de volta
independentemente.)

## Controle por requisição

Tanto o corpo da requisição de chat REST quanto os params do RPC `chat.send`
aceitam um campo `memory` opcional para substituir a configuração do servidor
**por chamada**:

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` — força recall ligado / desligado para esta
  rodada.
- omitido (`null`) — segue a configuração do servidor (`req.memory.unwrap_or(true)`),
  ou seja, habilitado se e somente se o gateway está configurado.

O override afeta o recall; o writeback segue apenas
`ARONA_MEMORY_WRITEBACK` mais o gateway estar habilitado.

## Estados de header

Respostas REST carregam o estado de memória da rodada no **header de resposta
`X-Arona-Memory`**; a resposta do RPC `chat.send` ecoa o mesmo valor em um campo
`memory` do seu resultado. Estados possíveis:

| Valor | Significado |
| --- | --- |
| `enabled` | Memória foi solicitada, o gateway está configurado, o recall teve sucesso e pelo menos uma memória foi injetada. |
| `disabled` | Gateway não configurado, ou `memory: false` na requisição, ou nenhuma mensagem de usuário para consultar, ou o recall teve sucesso mas retornou **nenhuma** memória relevante (nada para injetar). |
| `offline` | Memória foi solicitada e o gateway está configurado, mas a chamada de recall falhou (conexão recusada, erro RPC ou timeout). |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## Semântica de falhas

Tudo degrada explicitamente, na mesma direção — o chat sempre roda:

- **Falha de recall** — logada no nível `warn`; a requisição prossegue sem
  memórias injetadas e reporta `offline` no header.
- **Falha de writeback** — logada dentro do cliente de memória; a resposta de
  chat não é afetada.
- **Serviço de memória não configurado** — recall e writeback são no-ops; toda
  requisição reporta `disabled`.

Não há modo em que uma indisponibilidade de memória falhe ou atrase uma
requisição de chat além dos timeouts limitados do próprio cliente.

## Superfície RPC

Dois métodos de gerenciamento são expostos na superfície JSON-RPC (ambos exigem
um JWT; veja [API JSON-RPC](../api/jsonrpc.md)):

**`memory.status`** — snapshot do gateway:

```json
{
  "enabled": true,
  "writeback": true,
  "events": [
    { "kind": "recall", "detail": "…query…", "at": "2026-01-01T00:00:00.000Z" },
    { "kind": "writeback", "detail": "…node id…", "at": "…" },
    { "kind": "error", "detail": "…", "at": "…" }
  ]
}
```

`events` é um ring buffer em memória da atividade recente — eventos de recall,
writeback, delete e erro, mais recentes primeiro, até a contagem solicitada (o
handler de status solicita os últimos 50; o buffer em si é limitado a 100). Ele
**não** é um log de auditoria durável — ele reseta no reinício.

**`memory.delete`** — poda um nó armazenado por id:

```json
{ "node_id": "…" }
```

Retorna `{ "deleted": true | false }`. Falha com um erro quando `node_id` está
faltando ou quando o serviço de memória não está configurado.

## Relacionados

- [Configuração](configuration.md) — variáveis `ARONA_MEMORY_*`.
- [Quickstart](quickstart.md) — setup de ponta a ponta do gateway.
- [Backends](backends.md) — como requisições de chat são roteadas antes de o recall rodar.
- [Billing e uso](billing-usage.md) — como as mesmas rodadas de chat são medidas.
- [Operações](operations.md) — logs e health para a conexão de memória.
- [API JSON-RPC](../api/jsonrpc.md) — `memory.status`, `memory.delete`, `chat.send`.
- [Visão geral](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
