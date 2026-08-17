---
title: "Arona · 繁體中文"
description: "AI 模型的自我部署與遠端管理平台——gateway、backends、計費、agents、記憶。"
---

# Arona · 繁體中文

**AI 模型的自我部署與遠端管理平台。**

Arona 是 Celestia 生態系的 AI 模型 gateway 與管理平台，提供 OpenAI 相容的
REST API、JSON-RPC 2.0 管理平面、agent 控制平面與視訊生成等能力。以下指引將
帶你快速上手：

- [快速入門](guides/quickstart.md) — 使用內建 mock upstream 的完整端到端流程。
- [設定](guides/configuration.md) — 所有環境變數參考。
- [部署](guides/deployment.md) — bare metal、systemd、Docker 與監督管理。
- [Backends](guides/backends.md) — backend 類型、URL 語意與探測（probe）。
- [OpenAI 相容 REST API](api/openai-rest.md) — `/v1/*` 介面。
- [JSON-RPC API](api/jsonrpc.md) — 完整的管理平面。
