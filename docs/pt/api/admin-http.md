---
title: "API HTTP de Admin"
description: "Superfície de admin com bearer token — registre/liste/remova backends e gerencie aliases de modelos por /api/admin/*."
---

# API HTTP de Admin

A superfície `/api/admin/*` gerencia os **backends** do gateway (providers de
modelo upstream) e os **aliases** (redirecionamento nome-de-modelo → id-de-modelo).
É a contraparte HTTP do plano de gerenciamento RPC (veja a
[API JSON-RPC](./jsonrpc.md)) e é usada principalmente por operadores e pela UI
de admin.

## Autenticação

Toda rota `/api/admin/*` requer:

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN` é lida do ambiente no início do processo
(`GatewayServer::new`). Se a variável está **não definida**, ou o token
apresentado não corresponde, a requisição é rejeitada com `401`:

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

O prefixo bearer é casado de forma insensível a maiúsculas/minúsculas (`Bearer`
ou `bearer`).

> Diferente da superfície `/v1/*`, a auth de admin nunca cai para API keys ou
> JWTs, e é aplicada com uma comparação exata de token — rotacione o token
> reiniciando o processo com um valor novo.

## Backends

Backends são os upstreams roteáveis atrás do gateway. O registro torna um
backend roteável imediatamente, persiste sua configuração para restauração no
reinício, sonda-o (fica healthy em ~1–2 s) e, para URLs de bridge, mantém o
túnel vivo. Tipos de backend e semântica de URLs são detalhados em
[Backends](../guides/backends.md).

### POST /api/admin/backends — registra um backend

Corpo da requisição (todos os campos opcionais, exceto onde indicado):

| Campo | Tipo | Notas |
| --- | --- | --- |
| `type` | string | Kind de backend. Um de `external` (qualquer API HTTP compatível com OpenAI), `ollama` (servidor ollama local ou remoto), `engine` (engine CEP sobre `ws://`/`wss://`), `minimax-cloud` (API de vídeo em nuvem). Nomes de engine MDD (`llama_cpp`, `vllm`, `ollama`, `cloud`, `external_api`, `candle`, `native`, ...) resolvem pelo planner. `comfyui` é **rejeitado** (`comfyui backend removed`); qualquer outra coisa → `400` `unknown_type`. Padrão `ollama` quando ausente. |
| `url` | string | URL base do backend. URLs de bridge `evernight://<node>/<service>` são resolvidas pelo agent evernight local em um forward TCP local (falha de resolução → `502` `evernight_unreachable`). Padrão `http://localhost:11434`. |
| `api_key` | string | API key de upstream opcional, enviada como `Authorization: Bearer` nas chamadas de upstream. |
| `name` | string | Nome do backend. Padrão do valor de `type` quando ausente. Usado como a dica de `provider` do roteamento e para a identidade da linha de configuração. |
| `models` | string[] | Lista estática de modelos. A fonte de roteamento quando o probing não descobre nenhuma. Para backends `external`, modelos descobertos são mesclados após a lista estática (ids estáticos mantêm precedência); backends `engine` retornam seu cache de modelos descobertos primeiro e anexam ids estáticos depois; `minimax-cloud` não faz descoberta de modelos (seu probe apenas faz health-ping de `/v1/query/available_models`) e serve apenas a lista estática. Ignorada por `ollama`, que descobre modelos de `/api/tags`. |
| `workflow` | object | Opcional. Legado — historicamente consumido pelo backend ComfyUI removido; nenhum backend atual o lê (mantido para compatibilidade da coluna `backend_configs`). |

Exemplo:

```bash
curl -X POST http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "name": "mock-upstream",
    "url": "http://192.0.2.20:11434",
    "api_key": "sk-xxx",
    "models": ["Qwen/Qwen3-1.7B"]
  }'
```

Sucesso → `200`:

```json
{ "status": "ok", "message": "backend registered" }
```

Efeitos colaterais do registro:

- O backend é **registrado e roteável imediatamente** (sem necessidade de
  reinício).
- A configuração é **persistida** na tabela `backend_configs` e restaurada na
  inicialização (uma falha de DB é logada, mas nunca bloqueia a resposta).
- Um **probe** fire-and-forget roda na hora, para que o backend fique healthy
  em ~1–2 s em vez de permanecer fail-closed até a próxima rodada de 60 s do
  health checker.
- Para URLs `evernight://`, uma **task de keepalive** vigia o túnel: na
  reconexão com uma nova porta local, ela reconstrói e re-registra
  transparentemente o backend sob o mesmo nome.

### GET /api/admin/backends — lista backends

```json
{
  "backends": {
    "count": 2,
    "health": [
      ["backend_0:ExternalApi", "Healthy"],
      ["backend_1:Ollama", { "Unhealthy": "connection refused" }]
    ]
  },
  "models": [
    { "id": "Qwen/Qwen3-1.7B", "object": "model", "owned_by": "mock-upstream" }
  ]
}
```

- `backends.count` — número de backends **healthy**.
- `backends.health` — rótulo `backend_<index>:<kind>` por backend e estado de
  health (`Healthy` / `Degraded` / `Unhealthy`). O `<index>` é o índice de
  registro no router usado por `DELETE /api/admin/backends`.
- `models` — todo id de modelo roteável hoje (mesma listagem de
  `GET /v1/models`, sem o merge quick-start; veja
  [REST compatível com OpenAI](./openai-rest.md#get-v1models)).

### DELETE /api/admin/backends — remove um backend

Identificado pelo seu **índice** no router no corpo JSON — não pelo nome:

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| Campo | Tipo | Obrigatório | Notas |
| --- | --- | --- | --- |
| `index` | integer | sim | Índice de registro no router, casando com o rótulo `backend_<index>` no relatório de health de `GET /api/admin/backends`. |

- `index` ausente → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`.
- Índice fora do intervalo → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`.
- Sucesso → `200` `{ "status": "ok", "message": "backend removed" }`.
- A linha `backend_configs` persistida é deletada best-effort: o nome do backend
  é recuperado do `owned_by` da sua listagem de modelos; uma incompatibilidade
  deixa a linha no store (falhas de DB são logadas, nunca fatais).

## Aliases

Aliases mapeiam um nome de modelo para outro (`alias` → `target`), para que
requisições por um id de modelo roteiem para um modelo de backend diferente.
Aliases são resolvidos antes do roteamento, então se aplicam uniformemente a
lookups de chat, embeddings e vídeo.

> Aliases são **apenas estado de router em memória** — não são persistidos e são
> perdidos no reinício. Registre-os após a inicialização ou recrie-os a partir
> do seu próprio estado de provisioning.

### POST /api/admin/aliases — adiciona um alias

| Campo | Tipo | Obrigatório | Notas |
| --- | --- | --- | --- |
| `alias` | string | sim | O nome de modelo que clientes solicitarão. |
| `target` | string | sim | O id de modelo para o qual requisições são roteadas. |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- `alias` ausente → `400` `missing_alias`; `target` ausente → `400`
  `missing_target`.
- Sucesso → `200` `{ "status": "ok", "message": "alias added" }`.
- Adicionar um alias existente substitui seu target.

### GET /api/admin/aliases — lista aliases

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

Pares são retornados ordenados por alias.

### DELETE /api/admin/aliases — remove um alias

| Campo | Tipo | Obrigatório | Notas |
| --- | --- | --- | --- |
| `alias` | string | sim | O alias a remover. |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- `alias` ausente → `400` `missing_alias`.
- Remover um alias desconhecido é um sucesso no-op → `200`
  `{ "status": "ok", "message": "alias removed" }`.

## Resumo de persistência

| Recurso | Persistido? | Restauração no reinício |
| --- | --- | --- |
| Backends | Sim — tabela `backend_configs` (chave `name`, upsert no registro, delete na remoção). | Sim: restaurados na inicialização; backends external começam fail-closed e ficam healthy após a primeira rodada de probe. URLs `evernight://` são re-resolvidas pela bridge na inicialização. |
| Aliases | Não — apenas `Router.aliases` em memória. | Não. |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
