---
title: "Testes"
description: "A pirâmide de testes do Arona — testes unitários, integração hermética, integração com gate de PostgreSQL, smoke com servidor ativo, o mock server e a disciplina de smoke com credenciais reais."
---

# Testes

Os testes do Arona são organizados em camadas para que a execução padrão de
`cargo test` seja rápida, hermética e não precise de banco de dados nem de
rede, enquanto as suítes mais pesadas são opt-ins explícitos que exercitam a
superfície de wire real e um PostgreSQL real. Esta página mapeia as camadas, os
comandos que as executam e a disciplina do workspace em torno de execuções de
smoke com credenciais reais.

## Testes unitários

O grosso da cobertura são testes unitários simples dentro de `packages/core/src`:
217 funções `#[test]` / `#[tokio::test]`, mais ~23 em `packages/agent` e
`packages/cli`. Eles rodam com:

```bash
cargo test --workspace
```

Sem rede, sem banco de dados. Suítes-chave:

- **auth.rs** — a política de senha (≥8 caracteres E ≥3 de 4 categorias de
  caracteres), casts `::uuid` no SQL bruto de INSERT/REVOKE, defaults de
  requisição e leituras de flag de admin que caem para `false`.
- **billing/mod.rs** — matemática de cota nas dimensões de custo *ou* tokens, a
  janela mensal (`month_start`, `seconds_until_month_end`), o teto de limite de
  taxa (dispara apenas *no* RPM, `None` = ilimitado), guards de shape SQL para
  as queries de uso mensal / tier / janela de chave e `estimate_usage`
  preferindo números reportados pelo upstream.
- **routing/mod.rs** — resolução de alias, casamento de sufixo `:latest`, dicas
  de provider, seleção por menos carregado e fixação de conversa.
- **gateway/mod.rs** — registro de backend de agent: registrar
  `agent-{model_id}`, re-registro substituindo (não duplicando) e
  desregistro restaurando o router.

## Integração hermética (sempre executada, sem banco)

`packages/core/tests/gateway_integration.rs` contém três testes sempre
executados que exercitam lógica real de serialização/contrato sem tocar em um
banco de dados:

- **A1** — o contrato de serialização de eco de id JSON-RPC: ids de requisição
  numéricos, string e null fazem round-trip pelo enum `Id` do plana com
  fidelidade de tipo.
- **A2** — o contrato de código de erro do gate de admin: `AUTH_ERROR` (-32005,
  anônimo) e `ADMIN_REQUIRED` (-32007, não-admin autenticado) permanecem
  distintos, vivem no intervalo definido pela implementação e nunca colidem com
  os códigos do plana nem com o `QUOTA_ERROR` (-32006) de billing.
- **A3** — `estimate_usage`: uso reportado pelo upstream vence verbatim; sem
  ele, a estimativa local de tokenizer produz contagens não-zero de
  prompt/completion cuja soma total é a soma delas.

`packages/core/tests/smoke.rs` adiciona mais três testes sempre executados:
detecção de hardware, o caminho raiz do registry de modelos e defaults de
configuração sob `MOCK_MODE=1`.

## Integração com gate de PostgreSQL

A suíte completa do gateway em processo — `packages/core/tests/gateway_integration.rs`
— gira o router axum inteiro em uma porta loopback aleatória, registra mocks
upstream descartáveis compatíveis com OpenAI pela API de admin real e dirige a
superfície de wire com reqwest. Como o `AuthManager` conversa com PostgreSQL em
todo caminho (mesmo `MOCK_MODE=1` apenas semeia contas *no banco de dados*),
esta suíte tem gate atrás de `ARONA_TEST_PG=1` e é pulada por padrão. Os 10
testes:

- **T1** register + login + `keys.create`/`keys.list` (chave bruta mascarada nas
  listagens, prefixo `arona-`).
- **T2** chat REST com persistência de registro de uso no PostgreSQL.
- **T3** eco de id JSON-RPC pelo wire (caminhos de sucesso e erro).
- **T4** gate de admin em `agents.list`: anônimo → `AUTH_ERROR`, não-admin →
  `ADMIN_REQUIRED`.
- **T5** 401 do upstream → HTTP 502 `bad_gateway` com o detalhe do upstream.
- **T6** o probe de tempo de registro publica modelos (o modelo aparece em
  `GET /v1/models` dentro de 10s sem uma lista estática de modelos).
- **T7** persistência de conversa via `chat.send` (ambas as rodadas caem em
  `conversations.get`).
- **T8** gate de billing do tier free: 10 RPM por chave, a 11ª requisição na
  janela é 429 `rate_limit_exceeded`.
- **T9** stream SSE com uso terminal gravado do upstream.
- **T10** JSON malformado → 400; modelo desconhecido → 404 `model_not_found`.

Execute-a com o one-liner de Postgres descartável dos docs do módulo
(gateway_integration.rs:18-26):

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

Estas são credenciais de exemplo apenas para o contêiner de teste descartável —
nunca aponte isso para um banco real.

## Smoke com servidor ativo

`packages/core/tests/auth_flow.rs` percorre a cadeia completa
`register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` contra um servidor Arona **ativo**, espelhando o loop de auth
implantado. Ele é `#[ignore]` por padrão — a execução simples de `cargo test`
nunca toca a rede. Execute-o explicitamente:

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

Knobs:

- `ARONA_TEST_URL` — URL base (padrão `http://127.0.0.1:8420`).
- `ARONA_TEST_EXPECT_CHAT=1` — asserção dura de que `POST /v1/chat/completions`
  retorna 200. Sem ela, o teste apenas assere que a auth passou (não 401/403),
  porque o ambiente alvo pode não ter um provider de inferência configurado.

A suíte também inclui testes negativos: um chat completion não autenticado e um
`GET /v1/models` não autenticado devem ambos ser rejeitados com 401.

## Mock server

`scripts/mock/server.py` é um fake compatível com OpenAI baseado em aiohttp
usado pelo quickstart e por execuções de smoke. Ele serve
`POST /v1/chat/completions` (não-stream e SSE), `GET /v1/models`,
`GET /api/health`, a superfície WebSocket/HTTP JSON-RPC em `/api/rpc`, um sidecar
SSE em `/api/rpc/events` e `GET /api/test-key`, que retorna a API key do mock
para que outros serviços possam descobri-la. Ele escuta na porta 8429 por
padrão (substitua com `ARONA_MOCK_PORT`, host com `ARONA_MOCK_HOST`). O
[quickstart](quickstart.md) o usa para montar um ambiente de ponta a ponta sem
providers de modelo reais.

## Disciplina de smoke com credenciais reais

Execuções de smoke contra providers reais (DeepSeek / GLM) são deliberadamente
**não** testes de repositório — elas exigem credenciais reais e dinheiro real,
então não podem viver no CI nem na árvore git. A convenção do workspace,
documentada nos docs do módulo gateway_integration (gateway_integration.rs:54-55),
é:

- Arquivos de evidência vivem em `/mnt/work/arona-pr*-smoke.md` — locais ao
  workspace, nunca commitados no git.
- Credenciais vêm apenas do ambiente; orçamentos são mantidos pequenos.
- Todo PR que toca o caminho de inferência recebe um registro de evidência
  escrito.

O mock server é o substituto dessas execuções no CI e no desenvolvimento local;
o smoke com credenciais reais é um passo humano no momento do release.

## CI

`.github/workflows/ci.yml` roda `cargo fmt`, `cargo clippy`, `cargo test
--workspace` e `cargo-deny` nos runners self-hosted da org
(`[self-hosted, linux, x64, local]`); `ci-hosted.yml` espelha as mesmas
verificações em runners hospedados no GitHub. `.github/workflows/docs.yml`
compila este site de docs com o lagrange e o implanta no GitHub Pages em pushes
que tocam `docs/**`.

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
