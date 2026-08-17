---
title: "记忆网关"
description: "聊天的长期记忆——召回注入、episode 写回、按请求控制、header 状态以及 memory.status / memory.delete RPC。"
---

# 记忆网关

记忆网关让聊天能够访问存储在 entelecheia scepter / Philia 记忆服务中的
**长期记忆**。在每轮聊天时，Arona 向服务查询与会话相关的记忆，以 system 段落
的形式注入提示词，并在——回复完成之后——把该轮写回为 episode，供未来会话召回。

它是 Philia 的 WebSocket JSON-RPC 客户端（`Sync.ConnectHandshake`、
`Sync.MemoryQueryRequest`、`Sync.MemoryStoreRequest`、
`Sync.MemoryDeleteRequest`）。连接惰性建立、出错即断、下次调用重新建立；
任何故障都静默降级，**从不阻断聊天路径**。

## 配置

网关由三个环境变量控制：

| 变量 | 含义 |
| --- | --- |
| `ARONA_MEMORY_URL` | scepter / Philia 服务的 WebSocket URL，例如 `ws://192.0.2.10:8424/ws`。 |
| `ARONA_MEMORY_TOKEN` | 记忆服务的 token。 |
| `ARONA_MEMORY_WRITEBACK` | 是否写回已完成的轮次。默认**开启**；设为 `false` 禁用（按严格布尔解析——不接受 `0`）。 |

`ARONA_MEMORY_URL` **和** `ARONA_MEMORY_TOKEN` 都必须设置且非空，否则网关
**禁用**：召回与写回被完全跳过，每个请求报告 `disabled`。token 既作为
WebSocket 升级时的 `?token=` 查询参数发送，也放在 `Sync.ConnectHandshake`
请求内部。

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

完整环境参考见 [配置](configuration.md)。

## 召回注入

网关启用后，**每轮聊天**——REST 非流式 `/v1/chat/completions`、REST 流式
（SSE）和 RPC `chat.send`——都会在请求转发之前查询记忆服务：

- 查询内容是组装上下文的**最后一条用户消息**。
- 最多请求 **5** 条记忆（`limit = 5`）。
- 结果渲染为标题为 `## Relevant Long-Term Memories` 的 markdown system
  段落，每条记忆一个 `- [score] text` 列表项（分数保留两位小数，空白条目跳过），
  并作为 `system` 消息前置到消息列表。注入是幂等的：已携带该段落的上下文不会
  被重复注入。
- 如果没有返回相关记忆，则不注入任何内容，该轮照常进行。

召回在会话持久化和上游转发之前运行；慢或故障的记忆服务**不带来任何延迟保障**，
除其自身 10 秒的 RPC 超时外，也不会让请求失败。

## 写回

在 assistant 回复完成后，该轮作为 **episode** 节点写回记忆服务。episode 文本
是该轮对话的启发式转录——`User: <user content>\nAssistant: <assistant content>`
（任一侧为空则省略；两侧都空则跳过写回）。写回是**即发即忘**：它在派生的任务中
运行，从不阻塞聊天响应，其失败只在记忆客户端内部记录日志。（在 REST 流式路径
上，写回还要求请求附带一个会话；非流式 REST 和 RPC 路径则无条件写回。）

## 按请求控制

REST 聊天请求体和 RPC `chat.send` 参数都接受可选的 `memory` 字段，以**按调用**
覆盖服务器配置：

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` —— 强制本轮召回开启 / 关闭。
- 省略（`null`）—— 遵循服务器配置（`req.memory.unwrap_or(true)`），即网关
  配置了才启用。

该覆盖只影响召回；写回只遵循 `ARONA_MEMORY_WRITEBACK` 加上网关是否启用。

## Header 状态

REST 响应在 **`X-Arona-Memory`** 响应头中携带本轮的记忆状态；RPC
`chat.send` 响应在其结果的 `memory` 字段中回显同一值。可能的状态：

| 值 | 含义 |
| --- | --- |
| `enabled` | 请求了记忆、网关已配置、召回成功且至少注入了一条记忆。 |
| `disabled` | 网关未配置，或请求上为 `memory: false`，或没有可查询的用户消息，或召回成功但未返回**任何**相关记忆（没有可注入的内容）。 |
| `offline` | 请求了记忆且网关已配置，但召回调用失败（连接被拒绝、RPC 错误或超时）。 |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## 故障语义

一切都会明确地向同一方向降级——聊天始终运行：

- **召回失败** —— 以 `warn` 级别记录日志；请求在没有注入记忆的情况下继续，
  并在 header 中报告 `offline`。
- **写回失败** —— 在记忆客户端内部记录日志；聊天响应不受影响。
- **记忆服务未配置** —— 召回与写回都是 no-op；每个请求报告 `disabled`。

不存在任何模式会让记忆中断使聊天请求失败或延迟超过客户端自身的有界超时。

## RPC 接口

JSON-RPC 接口上暴露两个管理方法（都需要 JWT；见 [JSON-RPC API](../api/jsonrpc.md)）：

**`memory.status`** —— 网关快照：

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

`events` 是近期活动的内存环形缓冲区——召回、写回、删除和错误事件，最新的在前，
最多返回请求的数量（status 处理器请求最近 50 条；缓冲区本身上限为 100）。它
**不是**持久化的审计日志——重启即重置。

**`memory.delete`** —— 按 id 清理存储的节点：

```json
{ "node_id": "…" }
```

返回 `{ "deleted": true | false }`。当 `node_id` 缺失或记忆服务未配置时以
错误失败。

## 相关

- [配置](configuration.md) — `ARONA_MEMORY_*` 变量。
- [快速上手](quickstart.md) — 网关的端到端设置。
- [Backend](backends.md) — 召回运行之前聊天请求如何路由。
- [计费与用量](billing-usage.md) — 同样的聊天轮次如何被计量。
- [运维](operations.md) — 记忆连接的日志与健康。
- [JSON-RPC API](../api/jsonrpc.md) — `memory.status`、`memory.delete`、`chat.send`。
- [概述](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
