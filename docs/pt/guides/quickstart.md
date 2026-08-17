---
title: "Quickstart"
description: "Passo a passo completo do Arona com o mock upstream integrado: migrate, serve, registrar um backend, criar uma API key e conversar."
---

# Quickstart

Este guia percorre uma configuração completa do Arona de ponta a ponta em uma
única máquina usando o **mock upstream integrado** — sem pesos de modelo reais,
GPU ou conta de API externa. Ao final, você terá:

- um gateway Arona em execução (API REST compatível com OpenAI em `/v1/*` mais o
  plano de gerenciamento JSON-RPC em `/api/rpc`),
- o mock upstream registrado como um backend `external`,
- uma conta de usuário e uma API key,
- uma rodada de chat não-streaming **e** streaming funcionando contra o mock,
- registros de uso visíveis via `usage.list`.

## Pré-requisitos

- **Toolchain Rust** (veja `rust-toolchain.toml` na raiz do repositório).
- **Python 3** com `aiohttp` — necessário apenas para o mock upstream
  (`pip install aiohttp`).
- Uma instância **PostgreSQL em execução** e a URL de conexão para ela.

## 1. Defina o ambiente

O Arona lê a configuração de variáveis de ambiente **na inicialização do
processo**. Duas são obrigatórias: `DATABASE_URL` e `JWT_SECRET` — o servidor
se recusa a iniciar sem elas (a menos que `MOCK_MODE=1`). `ARONA_ADMIN_TOKEN` é
fortemente recomendada: sem ela, toda rota `/api/admin/*` retorna 401.

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

Essas variáveis são lidas uma vez quando o processo inicia — se você alterá-las,
reinicie o servidor. Veja [Configuração](configuration.md) para a referência
completa de variáveis.

## 2. Migre e inicie o servidor

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

Somente `serve` já basta para um banco novo: ele auto-migra na inicialização.
O servidor faz bind em `0.0.0.0:8420` por padrão (substitua com `ARONA_HOST` /
`ARONA_PORT`).

## 3. Inicie o mock upstream

Em um segundo terminal:

```bash
python3 scripts/mock/server.py
```

O mock é um servidor aiohttp que escuta em `127.0.0.1:8429` por padrão
(`ARONA_MOCK_PORT` substitui a porta). Ele imprime sua API key na inicialização
e também serve `GET /api/test-key`, que retorna
`{"api_key": ..., "base_url": ...}`. Ele expõe alguns ids de modelos —
incluindo `gpt-5.5`, usado abaixo — e responde tanto a completions de chat
simples quanto streaming.

Capture a chave impressa:

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. Registre o mock como um backend external

Backends são registrados pela API HTTP de admin:

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

O backend é sondado imediatamente no registro e fica healthy em ~1-2 segundos;
até o probe concluir, ele permanece em um estado fail-closed "ainda não
sondado" (veja a caixa de solução de problemas abaixo). A configuração é
persistida, então o backend sobrevive a um reinício.

## 5. Registre uma conta e faça login

As contas vivem no plano JSON-RPC, `POST /api/rpc`. Como
`ARONA_REGISTRATION_OPEN=1` está definida, `auth.register` está aberto; o
primeiro usuário registrado vira o admin.

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

As senhas devem ter pelo menos 8 caracteres **e** conter pelo menos 3 das 4
categorias de caracteres (maiúsculas, minúsculas, dígitos, especiais). Depois,
faça login para obter o par JWT:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

Exporte o `access_token` da resposta:

```bash
export JWT="<access_token from the login response>"
```

## 6. Crie uma API key

`keys.create` é autenticado por JWT e retorna o segredo **completo**
`arona-{uuid}` exatamente uma vez — o banco de dados armazena apenas o hash
SHA-256, então copie-o agora:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. Chat (não-streaming)

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

Você recebe de volta um objeto de completion no estilo OpenAI com um
`choices[0].message` e um bloco `usage`.

## 8. Chat (streaming)

O mesmo endpoint com `"stream": true` responde com server-sent events:
um chunk `data:` por token, terminado por um chunk final `data: [DONE]`:

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. Verifique o uso

Cada rodada de chat grava uma linha de uso sob o prefixo da chave. Consulte-a
com o JWT:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

Você deve ver um ou mais registros das requisições `gpt-5.5` feitas acima.

## Solução de problemas

- **`No backend available for model: gpt-5.5` (HTTP 404, `code:
  model_not_found`)** — nenhum backend registrado serve esse id de modelo. Ou o
  backend nunca foi registrado (ou a lista `models` dele não inclui o id), ou a
  chamada de registro falhou. Verifique com
  `GET /api/admin/backends` (token de admin).
- **`all backends unhealthy` (HTTP 500, `backend_error`)** — um backend *está*
  registrado para o modelo, mas nenhum candidato está healthy. Um backend
  external recém-registrado começa em um estado fail-closed "ainda não sondado"
  e fica healthy quando o probe do momento do registro conclui, ~1-2 s depois;
  se você conversar dentro dessa janela, encontrará esse erro. Tente novamente
  após um momento, ou verifique se o mock está de fato rodando em
  `127.0.0.1:8429`.
- **HTTP 401 em `/v1/*`** — um header `Authorization` ausente gera
  `Missing Authorization header. Use: Bearer <api_key>`; uma chave desconhecida
  gera `Invalid API key`. Confira `$AR_KEY` (segredo completo, não o prefixo).
- **HTTP 401 `Admin access required` em `/api/admin/*`** — o bearer token não
  corresponde a `ARONA_ADMIN_TOKEN`, ou a variável não está definida (nesse
  caso a rota sempre rejeita). Reinicie o servidor depois de defini-la.
- **`auth.register` falha com "Registration is closed"** — o registro está
  desabilitado quando `ARONA_REGISTRATION_OPEN` não é truthy. Defina
  `ARONA_REGISTRATION_OPEN=1` **antes** de iniciar o servidor (ela é lida na
  inicialização), ou seja o primeiro usuário — o primeiro usuário registrado é
  sempre permitido e vira o admin.
- **Limites de taxa HTTP 429** — três limites independentes podem disparar:
  - o limite em memória por chave, 60 RPM por padrão
    (`ARONA_API_RATE_LIMIT_RPM`) → `Rate limit exceeded. Try again later.`;
  - o limite de 10 RPM por chave do tier free de billing → 429 com um header
    `Retry-After: 60`;
  - a cota mensal de $1 / 100 mil tokens do tier free → 429 com `Retry-After`
    apontando para o próximo período de billing.

## Próximos passos

- [Configuração](configuration.md) — todas as variáveis de ambiente.
- [Backends](backends.md) — tipos de backend, semântica de URLs e probing.
- [Implantação](deployment.md) — bare metal, systemd, Docker.
- [API REST compatível com OpenAI](../api/openai-rest.md) — a superfície `/v1/*` completa.
- [API JSON-RPC](../api/jsonrpc.md) — o plano de gerenciamento usado acima.

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
