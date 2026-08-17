---
title: "Arona"
description: "Plataforma de autodespliegue y gestión remota de modelos de IA — gateway, backends, billing, agentes y memoria."
---

# Arona

**Plataforma de autodespliegue y gestión remota de modelos de IA.**

Arona es una plataforma **puramente backend** escrita en Rust (axum): es a la
vez un gateway de modelos compatible con OpenAI *y* un plano de gestión para
los modelos que ejecuta en su propio hardware. Sirve la API REST compatible
con OpenAI `/v1/*`, el plano de gestión JSON-RPC 2.0 (`/api/rpc`), el plano de
control de agentes (`/ws/agent`) y una interfaz Swagger en `/docs`.

**No incluye ningún panel web ni chat CLI integrados** — el chat y la interfaz
de administración viven en [shittim-chest](https://github.com/celestia-island/shittim-chest),
que se comunica con Arona a través de la superficie RPC. Arona se centra en el
lado del servidor: routing, billing, auth, despliegue de modelos, agentes y
memoria.

## Matriz de funciones

| Área | Lo que ofrece Arona |
| --- | --- |
| **Núcleo de conversación** | `chat.completions` compatible con OpenAI (stream y no stream), `embeddings`, listado de `models`; streaming con un chunk terminal `[DONE]` y uso real en el chunk final. |
| **Backends** | Upstreams registrados por el admin: `external` (cualquier API HTTP compatible con OpenAI), `ollama`, motor CEP `engine` (WebSocket), vídeo `minimax-cloud` y URLs de bridge `evernight://` hacia servicios industriales/edge. |
| **Autenticación** | Pares JWT access/refresh (15 min / 7 días), API keys `arona-{uuid}` almacenadas como hashes SHA-256, tres niveles de admin, política de contraseñas y limitación de tasa de doble vía. |
| **Billing y uso** | Tiers predefinidos (free / pro / enterprise), registros de uso por solicitud en cada canal, tabla de precios de plana, delimitación de cuotas por proyecto, 429 + `Retry-After`. |
| **Gestión de modelos** | Descarga de modelos (fuentes `hf:` / `ms:` / `gh:`), despliegue en nodos GPU `_agent`, auto-registro de los modelos desplegados como backends enrutables. |
| **Realtime y multimodal** | Sesiones `realtime.*` full-duplex, canal de percepción/control `engine.invoke`, trabajos asíncronos de generación de vídeo (cloud de MiniMax). |
| **Clúster de agentes** | Los nodos GPU se conectan por `/ws/agent`, colocación por menor carga, afinidad de sesión y persistencia de nodos entre reinicios. |
| **Gateway de memoria** | Memoria a largo plazo mediante entelecheia Philia: inyección de recuperación, writeback de episodios y degradación explícita. |
| **Operaciones** | Probes de salud, tracing con `RUST_LOG`, mapeo de errores upstream (502 frente a 500), apagado correcto y auto-migración al iniciar. |

## Posicionamiento

Arona es un **gateway + plataforma**: enruta el tráfico de modelos hacia sus
backends, despliega modelos en sus agentes GPU y contabiliza todo.

- vs **pi** — pi es un asistente CLI que habla con los modelos; arona no tiene
  chat CLI. Arona es la plataforma con la que pi (y otras herramientas) habla.
- vs **one-api / new-api** — son gateways de API keys delante de los providers
  de modelos; arona añade el **despliegue de modelos** (descargar los pesos y
  ejecutarlos en sus agentes), un plano RPC de gestión completo, tiers de
  billing y memoria.
- vs **LiteLLM** — un gateway homólogo; arona además posee el ciclo de vida de
  despliegue de los modelos que hay detrás del gateway.

## Empiece aquí

- [Inicio rápido](quickstart.md) — de extremo a extremo con el upstream mock integrado.
- [Configuración](configuration.md) — todas las variables de entorno.
- [Despliegue](deployment.md) — bare metal, systemd, Docker y supervisión.
- [Backends](backends.md) — tipos de backend, semántica de URLs y probing.
- [API REST compatible con OpenAI](../api/openai-rest.md) — `/v1/*`.
- [API JSON-RPC](../api/jsonrpc.md) — el plano de gestión completo.

## Estructura del repositorio

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

El panel web se eliminó de este repositorio y ahora vive en
[shittim-chest](https://github.com/celestia-island/shittim-chest) (chest #291).

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
