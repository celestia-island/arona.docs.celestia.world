---
title: "Events 및 알림"
description: "Server-sent event(SSE) 사이드카 — chat.stream, models.progress, realtime.event 및 video 알림."
---

# Events 및 알림

스트리밍 token, 배포 진행, realtime 이벤트는 JSON-RPC WebSocket 소켓에서 **전달되지 않습니다**. 각 스트리밍 RPC는 **세션 id**를 생성하고 server-sent events로 SSE 엔드포인트에 알림을 푸시합니다:

```
GET /api/rpc/events?session=<session_id>
```

## 구독-후-전송 레시피

RPC 호출이 세션 id를 반환한 시점과 SSE 구독이 설정되는 사이에 발생한 알림은 **버려집니다**(사전 구독 윈도우). 신뢰할 수 있는 패턴은:

1. 먼저 SSE 스트림을 엽니다(세션 id가 첨부될 때까지 블록).
2. 세션 id를 반환하는 RPC를 발사합니다(예: `chat.send`, `agents.deploy`, `realtime.start`, `video.create`).
3. SSE 스트림에서 알림을 도착하는 대로 읽습니다.

모든 알림은 `"jsonrpc": "2.0"`, `method`, `params` 객체를 가진 JSON-RPC 2.0 스타일 메시지입니다.

## 알림 카탈로그

### `chat.stream`

Token당 하나의 알림으로, `chat.send`(및 세션 채널을 사용하는 모든 스트리밍 chat 경로)가 생성합니다:

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — 하나의 콘텐츠 델타.
- `is_complete` — 최종 청크까지 `false`(업스트림이 finish reason을 붙이면 최종 콘텐츠 청크가 비어 있지 않은 token과 함께 `is_complete:
  true`를 이미 담을 수 있음). **터미널** 알림은 항상 빈 `token`과 `is_complete: true`로 뒤따릅니다.
- 스트림 오류는 `token: "Stream error: ..."`과 `is_complete: true`를 가진 터미널 알림으로 전달됩니다(`packages/core/src/gateway/rpc.rs` 참조).

### `models.progress`

`agents.deploy`의 모델 다운로드 진행 상황, agent에서 전달됩니다. `stream_id`는 `agents.deploy` 응답에서 옵니다.

### `realtime.event`

열린 전이중 realtime 세션의 서버 이벤트, 세션 채널에 푸시됩니다(`packages/core/src/gateway/realtime.rs`). `realtime.event` RPC로 보낸 클라이언트 이벤트는 업스트림으로 전달됩니다. 서버 이벤트는 여기에 도착합니다.

### Video 작업 알림

`video.create` 작업은 세션 채널로 진행 상황을 푸시합니다(`packages/core/src/gateway/video.rs`):

| 메서드 | Payload(params) | 의미 |
| --- | --- | --- |
| `video.progress` | `job_id`, `stream_id`, `status: "running"`, `progress`(0–90) | 작업이 실행 중입니다. |
| `video.done` | `job_id`, `stream_id`, `result`, `cost` | 작업이 완료되었습니다. `result`가 아티팩트 URL을 담습니다. |
| `video.failed` | `job_id`, `stream_id`, `error` | 작업이 실패했거나 취소되었습니다. |

## 순서 참고 사항

- SSE 스트림은 세션별로 정렬됩니다. Token은 생성 순서로 도착합니다.
- 단일 세션 id는 하나의 SSE 구독자만 소비할 수 있습니다. Id를 반환하는 RPC 전에(또는 직후에) 스트림을 여세요.
- `POST /api/rpc`의 `x-session-id` 헤더는 RPC **응답** 자체도 세션 채널에 첨부합니다 — 응답이 같은 스트림으로 에코되기를 원하는 클라이언트가 사용합니다.

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
