---
title: "Autenticação e Segurança"
description: "Sessões JWT, API keys, os três gates de admin, política de senha, limitação de taxa em duas trilhas e o modelo de segurança."
---

# Autenticação e Segurança

O Arona autentica callers em duas trilhas: **tokens de sessão JWT** para
clientes interativos (o chat + a UI de admin, chamadas RPC) e **API keys**
(`arona-…`) para tráfego programático compatível com OpenAI. Um token de admin
separado protege as superfícies administrativas. Esta página documenta a
mecânica, o modelo de segurança e os leftovers conhecidos de baixo risco de uma
auditoria de segurança.

## Sessões JWT

Sessões usam pares de tokens JWT access/refresh emitidos pelo token manager
`kirino_session`:

- **TTL do access token: 900 segundos (15 minutos).**
- **TTL do refresh token: 604.800 segundos (7 dias).**

Access tokens autenticam o plano JSON-RPC (`/api/rpc`) e `GET /v1/models`; o
sidecar SSE (`/api/rpc/events`) é chaveado pelo seu session id, uma capacidade
cunhada durante chamadas RPC autenticadas em vez de uma credencial bearer. Os
endpoints `/v1/chat/completions`, `/v1/embeddings` e `/v1/video/*` exigem uma
**API key** (um JWT não é aceito lá). Access tokens são de vida curta, então um
token roubado é utilizável apenas por pouco tempo. Refresh tokens são trocados
por pares novos via `auth.refresh`.

O refresh usa **rotação de família de tokens**: consumir um refresh token o
invalida e emite um novo, e reutilizar um refresh token consumido revoga a
família inteira — `auth.refresh` responde com `AUTH_ERROR` e a mensagem
`Refresh token reused` (o erro subjacente é `TokenReused`, "refresh token has
been reused — token family revoked"), e a conta precisa fazer login novamente.
A revogação de família é **em memória** (um conjunto `revoked_families`): um
reinício do servidor o limpa, então a proteção é best-effort entre reinícios (o
estado de sessão por usuário não sobrevive a um reinício).

O segredo de assinatura vem da variável de ambiente `JWT_SECRET`. Fora de
`MOCK_MODE=1`, o servidor **se recusa a iniciar** se `JWT_SECRET` não estiver
definida ou ainda for igual ao segredo de desenvolvimento integrado, então uma
instância de produção nunca pode acidentalmente servir tokens assinados com uma
constante pública. Use um segredo forte e aleatório e nunca o commite.

## Chaves de API

API keys são a credencial de máquina para a superfície compatível com OpenAI:

- **Formato:** `arona-{uuid}`.
- **Armazenamento:** apenas o **hash SHA-256** da chave é armazenado na tabela
  `api_keys` — o texto simples é retornado exatamente uma vez, na resposta de
  `keys.create`, e nunca pode ser recuperado depois.
- **Prefixo da chave:** os primeiros 8 caracteres (`key_prefix`) são
  armazenados em claro para exibição e atribuição de uso; a UI mostra uma forma
  mascarada como `arona-XXXX…abcd`.
- **Revogação:** o lookup de chave junta `api_keys.is_active = TRUE`, então uma
  chave revogada para de validar imediatamente — não há TTL de cache para
  esperar.

## Tiers de admin

Há três gates de admin distintos, cada um com sua própria credencial:

1. **Rotas `/api/admin/*`** — gerenciamento de backends e aliases
   (`POST/GET/DELETE /api/admin/backends`, `POST/GET/DELETE /api/admin/aliases`)
   exigem o header `Authorization: Bearer ARONA_ADMIN_TOKEN`. Quando
   `ARONA_ADMIN_TOKEN` não está definida, `check_admin` sempre falha e toda rota
   de admin retorna **401 "Admin access required"** — a superfície de
   gerenciamento inteira fica desabilitada em vez de aberta.

2. **Métodos RPC `agents.*` e `engine.invoke`** — o cluster de agents e o plano
   de controle de engine exigem um JWT cuja conta tenha `users.is_admin = true`.
   Um não-admin autenticado é rejeitado com o código definido pela
   implementação **-32007 (`ADMIN_REQUIRED`)** mais uma dica específica do
   método (ex. `agents.deploy starts model deployments on GPU nodes`); um caller
   **não autenticado** recebe o padrão **-32005 (`AUTH_ERROR`)**, para que o
   servidor não revele que o método é privilegiado.

3. **Métodos RPC `billing.plan.set` e `billing.video.pricing.set`** — mutações
   de billing exigem o mesmo Bearer `ARONA_ADMIN_TOKEN` que as rotas HTTP de
   admin; sem ele, retornam `AUTH_ERROR` "Admin access required".

O **primeiro usuário registrado vira o admin** (`users.is_admin = true`). Todo
registro posterior é um usuário comum, e o registro só está aberto enquanto
`ARONA_REGISTRATION_OPEN` estiver definida com um valor truthy.

## Política de senha

Senhas devem satisfazer **ambas** as regras (aplicadas no registro e em qualquer
caminho de troca de senha):

- pelo menos **8 caracteres**, e
- pelo menos **3 das 4 categorias de caracteres**: maiúsculas, minúsculas,
  dígitos, especiais.

## Limitação de taxa

A limitação de taxa roda em duas trilhas independentes; qualquer uma pode
rejeitar uma requisição com **429**:

### 1. Janela deslizante em memória (por identidade)

Toda requisição `/v1` autenticada passa por um limitador de janela deslizante
em memória chaveado pela identidade do caller:

- **Chamadas com API key** são chaveadas pelo **hash SHA-256** da chave;
- **Chamadas JWT** são chaveadas por `u:<email>` — JWTs rotacionam a cada 15
  minutos, então chavear a janela pela instância do token a reiniciaria
  silenciosamente a cada refresh.

O orçamento padrão é **60 requisições por minuto**, substituível com
`ARONA_API_RATE_LIMIT_RPM` (defina mais alto para pipelines de agent que fazem
fan-out de muitas chamadas LLM paralelas). Definir como **0 bloqueia toda
requisição**.

### 2. Limite de taxa do tier (por chave, do banco de dados)

Tiers de billing carregam um `rate_limit_rpm` por chave. A verificação conta
linhas `usage_records` para o prefixo da chave na **janela dos últimos 60
segundos** (o uso é persistido após cada resposta, então a janela fica atrasada
em no máximo uma requisição em voo; falhas de DB falham abertas). O **tier
free semeados é 10 RPM**; tiers pro/enterprise elevam o teto. A aplicação da
cota mensal compartilha o mesmo caminho de rejeição.

### Limitação de taxa de login

A adivinhação de credenciais é throttled no endpoint de login: **5 tentativas
falhas por janela de 5 minutos por email** e **20 por janela de 5 minutos por
IP**, cada uma seguida de um lockout de 15 minutos.

### `Retry-After`

Toda resposta 429 carrega um header `Retry-After` para que clientes compatíveis
com OpenAI recuem em vez de martelar o endpoint: rejeições por cota o definem
como **segundos até o fim do mês**; rejeições por limite de taxa o definem como
**60**. Veja [Billing e uso](billing-usage.md) para o modelo de cota.

## Notas do modelo de segurança

- **CORS permite qualquer origin** (`allow_origin(Any)`) — o Arona é um backend
  consumido por muitos clientes first-party e third-party; se sua implantação
  precisar restringir origins, coloque à frente um reverse proxy que aplique
  CORS.
- **Corpos de requisição são limitados a 1 MB** (`RequestBodyLimitLayer`),
  limitando o uso de memória no gateway.
- **O gateway não termina TLS** — ele escuta em HTTP puro. Coloque-o atrás de
  um reverse proxy (veja [Implantação](deployment.md)) que termine HTTPS.
- **Segredos vêm apenas do ambiente**: `ARONA_ADMIN_TOKEN` e `JWT_SECRET` são
  lidas de env vars e devem ser valores aleatórios fortes nunca commitados no
  repositório.
- O endereço de bind padrão do servidor é `0.0.0.0`; restrinja a exposição na
  camada de rede.

## Leftovers conhecidos de baixo risco (da auditoria)

Os itens a seguir são documentados como estão; são intencionais ou aceitos por
enquanto, mas vale saber ao expor uma instância além de uma rede confiável:

- **`providers.list` é público**, enquanto `providers.add` / `providers.update` /
  `providers.remove` / `providers.test` exigem um JWT. O caminho de leitura
  público revela o catálogo de providers, mas nada secreto.
- **`/ws/agent` é um plano de controle não autenticado**: agents de GPU se
  conectam sem credencial e se auto-registram (frames `register` / `heartbeat` /
  command-result). Qualquer um que alcance a porta WebSocket pode registrar um
  agent falso. Veja [Cluster de agents](agent-cluster.md) para os tradeoffs
  operacionais.
- **`memory.delete` é apenas-JWT sem verificação de ownership**: qualquer
  usuário autenticado pode deletar um nó de memória por `node_id`. Deletar
  memória exige estar logado, mas não ser dono do nó.

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
