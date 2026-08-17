---
title: "Arona"
description: "AI 模型的自我部署與遠端管理平台——gateway、backends、計費、agents、記憶。"
---

# Arona

**AI 模型的自我部署與遠端管理平台。**

Arona 是以 Rust（axum）撰寫的**純後端（pure backend）**平台：既是
OpenAI 相容的模型 gateway，也是你自有硬體上運行模型的管理平面。它提供
`/v1/*` OpenAI 相容 REST API、JSON-RPC 2.0 管理平面（`/api/rpc`）、
agent 控制平面（`/ws/agent`），以及在 `/docs` 的 Swagger UI。

**沒有內建的網頁儀表板，也沒有內建的 CLI 聊天**——聊天與管理介面位於
[shittim-chest](https://github.com/celestia-island/shittim-chest)，
它透過 RPC 介面與 Arona 溝通。Arona 專注於伺服器端：路由、計費、認證、
模型部署、agents 與記憶。

## 功能矩陣

| 領域 | Arona 提供的功能 |
| --- | --- |
| **對話核心** | OpenAI 相容的 `chat.completions`（串流與非串流）、`embeddings`、`models` 列表；串流以終端 `[DONE]` chunk 收尾，並在最後一個 chunk 帶上真實 usage。 |
| **Backends** | 由管理端註冊的 upstream：`external`（任何 OpenAI 相容 HTTP API）、`ollama`、CEP `engine`（WebSocket）、`minimax-cloud` 視訊，以及通往工業／邊緣服務的 `evernight://` bridge URL。 |
| **認證** | JWT access/refresh 配對（15 分鐘／7 天）、以 SHA-256 雜湊儲存的 API keys `arona-{uuid}`、三級管理權限、密碼政策、雙軌速率限制。 |
| **計費與用量** | 預設 tier（free／pro／enterprise）、每個通道的每請求 usage records、plana 定價表、依專案（project）的 quota 範圍、429 + `Retry-After`。 |
| **模型管理** | 模型下載（`hf:`／`ms:`／`gh:` 來源）、`_agent` GPU 節點部署、已部署模型自動註冊為可路由的 backends。 |
| **Realtime 與多模態** | 全雙工 `realtime.*` session、`engine.invoke` 感知／控制通道、非同步視訊生成任務（MiniMax 雲端）。 |
| **Agent 叢集** | GPU 節點透過 `/ws/agent` 連線、最少負載放置、session affinity、跨重啟的節點持續性。 |
| **記憶 gateway** | 透過 entelecheia Philia 的長期記憶：recall 注入、episode 寫回、明確的降級行為。 |
| **營運** | 健康探測（health probe）、`RUST_LOG` 追蹤、upstream 錯誤對映（502 vs 500）、優雅關閉、啟動時自動 migration。 |

## 定位

Arona 是 **gateway + 平台**：它將模型流量路由到你的 backends、將模型部署到你的
GPU agents，並為所有流量計量。

- vs **pi** — pi 是與模型對話的 CLI 助手；arona 沒有 CLI 聊天。Arona 是 pi（及其他
  工具）所要溝通的平台。
- vs **one-api / new-api** — 那些是模型 provider 前面的 API-key gateway；arona
  額外提供**模型部署**（下載權重、在你的 agents 上運行）、完整的管理 RPC 平面、
  計費 tier 與記憶。
- vs **LiteLLM** — 同為 gateway 同級產品；arona 另外掌管 gateway 後方模型的
  部署生命週期。

## 從這裡開始

- [快速入門](quickstart.md) — 使用內建 mock upstream 的完整端到端流程。
- [設定](configuration.md) — 所有環境變數。
- [部署](deployment.md) — bare metal、systemd、Docker、監督管理。
- [Backends](backends.md) — backend 類型、URL 語意與探測。
- [OpenAI 相容 REST API](../api/openai-rest.md) — `/v1/*`。
- [JSON-RPC API](../api/jsonrpc.md) — 完整的管理平面。

## 儲存庫結構

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

網頁儀表板已從此儲存庫移除，現位於
[shittim-chest](https://github.com/celestia-island/shittim-chest)（chest #291）。

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
