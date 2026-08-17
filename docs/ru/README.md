---
title: "Arona · Русский"
description: "Платформа для самостоятельного развёртывания и удалённого управления моделями ИИ — gateway, backends, billing, agents, memory."
---

# Arona · Русский

**Платформа для самостоятельного развёртывания и удалённого управления моделями ИИ.**

Arona — это серверная backend-платформа (Rust/axum): OpenAI-совместимый шлюз моделей
и управляющая плоскость для моделей, которые вы запускаете на собственном оборудовании.
Она обслуживает OpenAI-совместимый REST API (`/v1/*`), управляющую плоскость JSON-RPC 2.0
(`/api/rpc`), плоскость управления агентами (`/ws/agent`) и Swagger UI на `/docs`.

## Начало работы

- [Краткое руководство](guides/quickstart.md) — сквозной сценарий со встроенным mock upstream.
- [Конфигурация](guides/configuration.md) — все переменные окружения.
- [Развёртывание](guides/deployment.md) — bare metal, systemd, Docker, supervision.
- [Backends](guides/backends.md) — типы backends, семантика URL и probing.
- [OpenAI-совместимый REST API](api/openai-rest.md) — `/v1/*`.
- [JSON-RPC API](api/jsonrpc.md) — полная управляющая плоскость.
