---
title: "Arona"
description: "Plataforma de auto-implantação e gerenciamento remoto de modelos de IA — gateway, backends, billing, agents, memória."
---

# Arona

**Plataforma de auto-implantação e gerenciamento remoto de modelos de IA.**

Arona é uma plataforma **puramente backend** escrita em Rust (axum): é um
gateway de modelos compatível com OpenAI *e* um plano de gerenciamento para os
modelos que você executa no seu próprio hardware. Ela serve a API REST
compatível com OpenAI em `/v1/*`, o plano de gerenciamento JSON-RPC 2.0
(`/api/rpc`), o plano de controle de agents (`/ws/agent`) e uma UI Swagger em
`/docs`.

**Não há dashboard web embutido nem chat CLI embutido** — o chat + a UI de
admin vivem no [shittim-chest](https://github.com/celestia-island/shittim-chest),
que conversa com o Arona pela superfície RPC. O Arona foca no lado do servidor:
roteamento, billing, auth, implantação de modelos, agents e memória.

## Matriz de recursos

| Área | O que o Arona oferece |
| --- | --- |
| **Núcleo de conversa** | `chat.completions` compatível com OpenAI (stream + não-stream), `embeddings`, listagem de `models`; streaming com um chunk final `[DONE]` e uso real no último chunk. |
| **Backends** | Upstreams registrados por admin: `external` (qualquer API HTTP compatível com OpenAI), `ollama`, engine CEP (WebSocket), vídeo `minimax-cloud` e URLs de bridge `evernight://` para serviços industriais/de borda. |
| **Autenticação** | Pares JWT access/refresh (15 min / 7 dias), API keys `arona-{uuid}` armazenadas como hashes SHA-256, três tiers de admin, política de senha, limitação de taxa em duas trilhas. |
| **Billing e uso** | Tiers predefinidos (free / pro / enterprise), registros de uso por requisição em todos os canais, tabela de preços do plana, escopo de cota por projeto, 429 + `Retry-After`. |
| **Gerenciamento de modelos** | Download de modelos (fontes `hf:` / `ms:` / `gh:`), implantação em nós de GPU `_agent`, auto-registro de modelos implantados como backends roteáveis. |
| **Realtime e multimodal** | Sessões full-duplex `realtime.*`, canal de percepção/controle `engine.invoke`, jobs assíncronos de geração de vídeo (nuvem MiniMax). |
| **Cluster de agents** | Nós de GPU conectam via `/ws/agent`, alocação pelo menos carregado, afinidade de sessão, persistência de nós entre reinícios. |
| **Gateway de memória** | Memória de longo prazo via entelecheia Philia: injeção de recall, writeback de episódios, degradação explícita. |
| **Operações** | Health probes, tracing `RUST_LOG`, mapeamento de erros de upstream (502 vs 500), desligamento gracioso, auto-migração na inicialização. |

## Posicionamento

O Arona é um **gateway + plataforma**: ele roteia o tráfego de modelos para os
seus backends, implanta modelos nos seus agents de GPU e mede tudo.

- vs **pi** — o pi é um assistente de CLI que conversa com modelos; o arona não
  tem chat CLI. O Arona é a plataforma com a qual o pi (e outras ferramentas)
  conversa.
- vs **one-api / new-api** — são gateways de API key na frente de providers de
  modelos; o arona adiciona **implantação de modelos** (baixar pesos, executá-los
  nos seus agents), um plano RPC de gerenciamento completo, tiers de billing e
  memória.
- vs **LiteLLM** — um peer de gateway; o arona adicionalmente é dono do ciclo de
  vida de implantação dos modelos atrás do gateway.

## Comece por aqui

- [Quickstart](quickstart.md) — passo a passo completo com o mock upstream integrado.
- [Configuração](configuration.md) — todas as variáveis de ambiente.
- [Implantação](deployment.md) — bare metal, systemd, Docker, supervisão.
- [Backends](backends.md) — tipos de backend, semântica de URLs e probing.
- [API REST compatível com OpenAI](../api/openai-rest.md) — `/v1/*`.
- [API JSON-RPC](../api/jsonrpc.md) — o plano de gerenciamento completo.

## Estrutura do repositório

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

O dashboard web foi removido deste repositório e agora vive no
[shittim-chest](https://github.com/celestia-island/shittim-chest) (chest #291).

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
