---
title: "記憶 Gateway"
description: "聊天的長期記憶——recall 注入、episode 寫回、每請求控制、header 狀態與 memory.status／memory.delete RPC。"
---

# 記憶 Gateway

記憶 Gateway 讓聊天回合能存取儲存在 entelecheia scepter／Philia 記憶服務
中的**長期記憶**。每個聊天回合，Arona 都會向服務查詢與對話相關的記憶、
以 system 區段注入 prompt，並在回覆完成後——將回合以 episode 寫回，
讓未來的對話可以 recall 它。

它是 Philia 的 WebSocket JSON-RPC 客戶端（`Sync.ConnectHandshake`、
`Sync.MemoryQueryRequest`、`Sync.MemoryStoreRequest`、
`Sync.MemoryDeleteRequest`）。連線惰性建立，任何錯誤時斷開並於下次呼叫
重建；每個失敗都靜默降級，**絕不阻擋聊天路徑**。

## 設定

gateway 由三個環境變數控制：

| 變數 | 意義 |
| --- | --- |
| `ARONA_MEMORY_URL` | scepter／Philia 服務的 WebSocket URL，例如 `ws://192.0.2.10:8424/ws`。 |
| `ARONA_MEMORY_TOKEN` | 記憶服務的 token。 |
| `ARONA_MEMORY_WRITEBACK` | 是否將完成的回合寫回。預設**開啟**；設為 `false` 停用（以嚴格的 boolean 解析——不接受 `0`）。 |

`ARONA_MEMORY_URL` **與** `ARONA_MEMORY_TOKEN` 都必須設定且非空，
否則 gateway **停用**：recall 與寫回完全略過，每個請求都回報 `disabled`。
token 同時以 `?token=` 查詢參數（在 WebSocket upgrade 時）與
`Sync.ConnectHandshake` 請求內送出。

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

完整的環境變數參考見[設定](configuration.md)。

## Recall 注入

gateway 啟用時，**每個聊天回合**——REST 非串流
`/v1/chat/completions`、REST 串流（SSE）與 RPC `chat.send`——都會在
請求轉發前查詢記憶服務：

- 查詢是組裝後 context 的**最後一則使用者訊息**。
- 最多請求 **5** 筆記憶（`limit = 5`）。
- 結果以標題為 `## Relevant Long-Term Memories` 的 markdown system
  區段呈現，每筆記憶一個 `- [score] text` 項目（score 取兩位小數，
  空白條目略過），並以 `system` 訊息前置到訊息清單。注入是冪等的：
  已攜帶該區段的 context 不會被重複注入。
- 若沒有回傳相關記憶，則不注入任何內容，回合照常進行。

Recall 在對話持久化與 upstream 轉發之前執行；緩慢或失敗的記憶服務
**不保證延遲**，除了其自身的 10 秒 RPC 逾時，且不能使請求失敗。

## 寫回

在 assistant 回覆完成後，回合會以 **episode** 節點寫回記憶服務。
Episode 文字是回合的啟發式轉錄——
`User: <user content>\nAssistant: <assistant content>`（任一端為空時略過；
兩者皆空時跳過寫回）。寫回是**fire-and-forget**：它在一個產生的 task 中
執行，絕不阻擋聊天回應，其失敗只會記錄在記憶客戶端內部。（在 REST 串流
路徑上，寫回額外要求請求附掛一個對話；非串流 REST 與 RPC 路徑則
無論如何都會寫回。）

## 每請求控制

REST 聊天請求內文與 RPC `chat.send` 參數都接受選用的 `memory` 欄位，
以**逐呼叫**覆寫伺服器設定：

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true`／`memory: false` — 為此回合強制開啟／關閉 recall。
- 省略（`null`）— 依伺服器設定（`req.memory.unwrap_or(true)`），
  亦即 gateway 有設定時才啟用。

覆寫影響 recall；寫回只遵循 `ARONA_MEMORY_WRITEBACK` 加上 gateway 是否
啟用。

## Header 狀態

REST 回應在 **`X-Arona-Memory`** 回應 header 攜帶回合的記憶狀態；
RPC `chat.send` 回應在其結果的 `memory` 欄位回傳相同的值。可能的狀態：

| 值 | 意義 |
| --- | --- |
| `enabled` | 要求了記憶、gateway 已設定、recall 成功且至少注入一筆記憶。 |
| `disabled` | Gateway 未設定，或請求帶 `memory: false`，或沒有可查詢的使用者訊息，或 recall 成功但沒有回傳**任何**相關記憶（沒有可注入的內容）。 |
| `offline` | 要求了記憶且 gateway 已設定，但 recall 呼叫失敗（連線被拒、RPC 錯誤或逾時）。 |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## 失敗語意

一切都有明確的降級，方向一致——聊天永遠運行：

- **Recall 失敗** — 以 `warn` 等級記錄；請求在沒有注入記憶的情況下繼續，
  並在 header 回報 `offline`。
- **寫回失敗** — 記錄在記憶客戶端內部；聊天回應不受影響。
- **記憶服務未設定** — recall 與寫回都是 no-op；每個請求回報 `disabled`。

不存在任何模式會讓記憶中斷使聊天請求失敗或延遲，超出客戶端自身有界
的逾時。

## RPC 介面

JSON-RPC 介面暴露兩個管理方法（都需要 JWT；見
[JSON-RPC API](../api/jsonrpc.md)）：

**`memory.status`** — gateway 的快照：

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

`events` 是最近活動的記憶體內 ring buffer——recall、寫回、刪除與錯誤事件，
最新在前，最多到要求的數量（status handler 要求最後 50 筆；buffer 本身
上限 100）。它**不是**持久的稽核日誌——重啟時會重置。

**`memory.delete`** — 依 id 修剪已儲存的節點：

```json
{ "node_id": "…" }
```

回傳 `{ "deleted": true | false }`。當 `node_id` 缺失或記憶服務未設定時
以錯誤失敗。

## 相關

- [設定](configuration.md) — `ARONA_MEMORY_*` 變數。
- [快速入門](quickstart.md) — gateway 的端到端設定。
- [Backends](backends.md) — recall 執行前聊天請求如何路由。
- [計費與用量](billing-usage.md) — 相同的聊天回合如何計量。
- [營運](operations.md) — 記憶連線的日誌與健康。
- [JSON-RPC API](../api/jsonrpc.md) — `memory.status`、`memory.delete`、`chat.send`。
- [總覽](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
