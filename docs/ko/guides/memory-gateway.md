---
title: "Memory Gateway"
description: "chat용 장기 memory — 회상 주입, 에피소드 writeback, 요청별 제어, 헤더 상태 및 memory.status / memory.delete RPC."
---

# Memory Gateway

Memory Gateway는 chat 턴에 entelecheia scepter / Philia memory 서비스에 저장된 **장기 memory**에 대한 액세스를 제공합니다. 각 chat 턴에서 Arona는 대화와 관련된 memory를 서비스에 쿼리하고, 시스템 섹션으로 프롬프트에 주입하며, 응답 완료 후 턴을 에피소드로 다시 기록(writeback)하여 향후 대화가 회상할 수 있게 합니다.

Philia의 WebSocket JSON-RPC 클라이언트입니다(`Sync.ConnectHandshake`, `Sync.MemoryQueryRequest`, `Sync.MemoryStoreRequest`, `Sync.MemoryDeleteRequest`). 연결은 지연(lazily) 설정되고, 오류 시 끊어지며, 다음 호출에서 재설정됩니다. 모든 실패는 조용히 저하되며 **chat 경로를 절대 차단하지 않습니다**.

## 구성

Gateway는 세 개의 환경 변수로 제어됩니다:

| 변수 | 의미 |
| --- | --- |
| `ARONA_MEMORY_URL` | scepter / Philia 서비스의 WebSocket URL, 예: `ws://192.0.2.10:8424/ws`. |
| `ARONA_MEMORY_TOKEN` | memory 서비스용 token. |
| `ARONA_MEMORY_WRITEBACK` | 완료된 턴이 다시 기록되는지 여부. 기본값 **켜짐**. `false`로 설정하면 비활성화(엄격한 boolean으로 파싱 — `0`은 허용되지 않음). |

`ARONA_MEMORY_URL` **및** `ARONA_MEMORY_TOKEN`이 모두 설정되고 비어 있지 않아야 하며, 그렇지 않으면 gateway가 **비활성화**됩니다: 회상과 writeback이 완전히 건너뛰어지고 모든 요청이 `disabled`를 보고합니다. Token은 WebSocket 업그레이드의 `?token=` 쿼리 매개변수와 `Sync.ConnectHandshake` 요청 내부 **모두**로 전송됩니다.

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

전체 환경 참조는 [구성](configuration.md)을 참조하세요.

## 회상 주입

Gateway가 활성화되면 **모든 chat 턴** — REST 비스트리밍 `/v1/chat/completions`, REST 스트리밍(SSE), RPC `chat.send` — 요청이 전달되기 전에 memory 서비스를 쿼리합니다:

- 쿼리는 조립된 컨텍스트의 **마지막 사용자 메시지**입니다.
- 최대 **5개**의 memory가 요청됩니다(`limit = 5`).
- 결과는 `## Relevant Long-Term Memories` 제목의 markdown 시스템 섹션으로 렌더링되며, memory당 하나의 `- [score] text` 불릿(점수는 소수 둘째 자리, 빈 항목은 건너뜀)으로 메시지 목록 앞에 `system` 메시지로 추가됩니다. 주입은 멱등적입니다: 이미 섹션을 담은 컨텍스트는 다시 주입되지 않습니다.
- 관련 memory가 반환되지 않으면 아무것도 주입되지 않고 턴이 그대로 진행됩니다.

회상은 대화 영속화와 업스트림 전달보다 먼저 실행됩니다. 느리거나 실패하는 memory 서비스는 자체 10초 RPC 타임아웃 이상의 **지연 보장을 추가하지 않으며**, 요청을 실패시킬 수 없습니다.

## Writeback

완료된 assistant 응답 후 턴은 **에피소드** 노드로 memory 서비스에 다시 기록됩니다. 에피소드 텍스트는 턴의 휴리스틱 트랜스크립트입니다 — `User: <user content>\nAssistant: <assistant content>`(어느 한쪽이 비어 있으면 생략, 둘 다 비어 있으면 writeback 건너뜀). Writeback은 **fire-and-forget**입니다: 생성된 작업에서 실행되며 chat 응답을 절대 차단하지 않고, 실패는 memory 클라이언트 내부에서만 로그됩니다. (REST 스트리밍 경로에서 writeback은 추가로 요청에 대화가 연결되어 있어야 합니다. 비스트리밍 REST와 RPC 경로는 조건 없이 다시 기록합니다.)

## 요청별 제어

REST chat 요청 본문과 RPC `chat.send` params 모두 선택적 `memory` 필드를 허용하여 서버 구성을 **호출별로** 재정의합니다:

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` — 이 턴의 회상을 강제로 켜거나 끕니다.
- 생략(`null`) — 서버 구성을 따릅니다(`req.memory.unwrap_or(true)`), 즉 gateway가 구성된 경우에만 활성화.

재정의는 회상에 영향을 줍니다. Writeback은 `ARONA_MEMORY_WRITEBACK`과 gateway 활성화만 따릅니다.

## 헤더 상태

REST 응답은 **`X-Arona-Memory`** 응답 헤더에 턴의 memory 상태를 담습니다. RPC `chat.send` 응답은 결과의 `memory` 필드에 동일한 값을 반영합니다. 가능한 상태:

| 값 | 의미 |
| --- | --- |
| `enabled` | Memory가 요청되었고, gateway가 구성되었으며, 회상이 성공하고 최소 하나의 memory가 주입되었습니다. |
| `disabled` | Gateway가 구성되지 않았거나, 요청에 `memory: false`가 있거나, 쿼리할 사용자 메시지가 없거나, 회상은 성공했지만 관련 memory가 **없음**(주입할 것이 없음). |
| `offline` | Memory가 요청되었고 gateway가 구성되었지만 회상 호출이 실패했습니다(연결 거부, RPC 오류 또는 타임아웃). |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## 실패 의미론

모든 것이 명시적으로, 같은 방향으로 저하됩니다 — chat은 항상 실행됩니다:

- **회상 실패** — `warn` 수준으로 로그됩니다. 요청은 주입된 memory 없이 진행되고 헤더에 `offline`을 보고합니다.
- **Writeback 실패** — memory 클라이언트 내부에서 로그됩니다. Chat 응답은 영향받지 않습니다.
- **Memory 서비스 미구성** — 회상과 writeback은 no-op입니다. 모든 요청이 `disabled`를 보고합니다.

Memory 중단이 클라이언트의 자체 제한된 타임아웃 너머로 chat 요청을 실패시키거나 지연시키는 모드는 없습니다.

## RPC 표면

JSON-RPC 표면에 두 개의 관리 메서드가 노출됩니다(둘 다 JWT 필요. [JSON-RPC API](../api/jsonrpc.md) 참조):

**`memory.status`** — gateway의 스냅샷:

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

`events`는 최근 활동의 인메모리 링 버퍼입니다 — 회상, writeback, 삭제, 오류 이벤트를 최신순으로 요청된 개수만큼(상태 핸들러는 마지막 50개를 요청, 버퍼 자체는 100개로 제한) 담습니다. 지속적 감사 로그가 **아닙니다** — 재시작 시 초기화됩니다.

**`memory.delete`** — id로 저장된 노드를 정리합니다:

```json
{ "node_id": "…" }
```

`{ "deleted": true | false }`를 반환합니다. `node_id`가 없거나 memory 서비스가 구성되지 않았으면 오류로 실패합니다.

## 관련

- [구성](configuration.md) — `ARONA_MEMORY_*` 변수.
- [빠른 시작](quickstart.md) — gateway의 end-to-end 설정.
- [Backends](backends.md) — 회상이 실행되기 전에 chat 요청이 라우팅되는 방식.
- [Billing 및 사용량](billing-usage.md) — 동일한 chat 턴이 계량되는 방식.
- [Operations](operations.md) — memory 연결의 로그와 health.
- [JSON-RPC API](../api/jsonrpc.md) — `memory.status`, `memory.delete`, `chat.send`.
- [개요](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
