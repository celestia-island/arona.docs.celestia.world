---
title: "Realtime & Video"
description: "전이중 realtime 세션(realtime.start/event/stop), engine.invoke 인지/제어 채널 및 비동기 비디오 생성 작업."
---

# Realtime & Video

Arona는 일반 텍스트 chat 너머의 두 가지 멀티모달 채널을 노출합니다: **전이중 realtime 세션**(하나의 양방향 채널을 통한 음성/비디오 입출력)과 **비동기 비디오 생성**(백그라운드에서 실행되고 진행 상황을 보고하는 작업 스타일 작업). 둘 다 요청된 모델을 제공하는 backend로 라우팅되며 둘 다 billing 계층을 통해 계량됩니다.

## Realtime 세션

Realtime 세션은 **한 클라이언트**와 **한 업스트림** 사이의 양방향 채널입니다: 클라우드 realtime API(OpenAI-Realtime / Qwen-Omni-Realtime WebSocket 어휘) 또는 로컬 CEP 엔진. 클라이언트 이벤트는 JSON-RPC를 통해 도착하여 업스트림으로 전달됩니다. 서버 이벤트는 세션 SSE 채널을 통해 `realtime.event` 알림으로 다시 푸시됩니다. 오디오는 base64 PCM16(클라이언트→모델 16kHz, 모델→클라이언트 24kHz)로 전송되며, 클라우드 벤더의 와이어 형식과 일치하므로 gateway는 바이트를 그대로 통과시킵니다(`packages/core/src/backends/realtime.rs:1-19`).

### `realtime.start`

`model`을 제공하는 backend에 대해 세션을 엽니다(JWT. params `model`, `config?`, `conversation_id?` — `packages/core/src/gateway/rpc.rs:1890-1898,
1914-1984`). Backend **반드시** `realtime` 기능(오디오/비디오 양식)을 선언해야 합니다. 그렇지 않으면 호출이 `model {model} does not support realtime sessions (no audio/video modality)`로 명시적으로 실패합니다 — 텍스트 chat으로의 조용한 대체는 없습니다(`packages/core/src/gateway/realtime.rs:138-142`).

두 가지 업스트림 종류가 지원됩니다(`packages/core/src/gateway/realtime.rs:143-167`):

- **CEP 엔진 업스트림** — Celestia Engine Protocol `Engine.InvokeStart` 스트리밍 채널로 이벤트를 라우팅하므로, 로컬에 배포된 omni 엔진이 새 와이어 형식 없이 동일한 클라이언트 표면에 합류합니다.
- **클라우드 업스트림** — 클라우드 realtime 이벤트 어휘(`session.update`, `input_audio_buffer.*`, `response.audio.delta`, ...)를 말하는 고정 `wss://` URL. 클라우드 구현은 재연결 시 `session.update`를 다시 발행합니다.

응답은 `{ "session_id": ..., "stream_session": ... }`입니다 — 호출 전에(또는 직후에) `/api/rpc/events?session=<stream_session>`을 구독하여 서버 이벤트를 받으세요. 선택적 `conversation_id`는 음성 트랜스크립트를 assistant 메시지로 유지하고 billing용 token 사용량을 기록합니다(`packages/core/src/gateway/realtime.rs:32-85`).

### `realtime.event`

세션에 클라이언트 이벤트 하나를 보냅니다(JWT. params `session_id`, `event` — `packages/core/src/gateway/rpc.rs:1989-2013`). 지원되는 이벤트에는 `session.update`, `input_audio_buffer.append` / `.commit` / `.clear`, `input_image_buffer.append`, `response.create`, `response.cancel`, `session.stop`이 포함됩니다. `send_event`는 **비차단**입니다: 이벤트는 mpsc 채널에 큐에 들어가고 포워더 작업이 업스트림에 씁니다(`packages/core/src/gateway/realtime.rs:254-280`). 세션 소유자만 이벤트를 보낼 수 있습니다.

### `realtime.stop`

세션을 닫고 제거합니다(JWT. params `session_id` — `packages/core/src/gateway/rpc.rs:2016-2034`). 각 세션은 업스트림을 보유하고 양방향을 다중화하는 정확히 하나의 **포워더 작업**을 소유합니다: 클라이언트 이벤트는 큐에서 소비되고 업스트림 이벤트는 같은 루프에서 폴링됩니다. 포워더는 업스트림이 닫히거나 세션이 중지되면 종료되어 레지스트리 항목을 제거합니다(`packages/core/src/gateway/realtime.rs:195-250`).

서버 이벤트는 params `{ session_id, event }`와 함께 `realtime.event` 알림으로 세션 채널에 푸시됩니다 — [Events & Notifications](../api/events.md) 참조.

## `engine.invoke`

`engine.invoke`는 일반 **동기** 엔진 메서드 채널입니다(ADMIN: JWT + `is_admin`. params `model`, `method`, `params?` — `packages/core/src/gateway/rpc.rs:261-264,2049-2079`). `model`을 제공하는 backend에서 임의의 메서드를 호출하고 결과를 직접 반환하므로, 고주파 인지/제어 채널이 됩니다: `sensor.ingest`, `control.setpoint` 스타일 호출을 20~30Hz 루프로. 일반 호출 채널이 없는 backend(모든 OpenAI 호환 HTTP backend)는 `backend does not support generic invocation`으로 명시적으로 거부합니다(`packages/core/src/backends/mod.rs:573-586`).

## 비디오 생성(REST)

Video 작업은 REST 표면 위의 OpenAI 스타일 비동기 작업입니다(API key 인증 — `packages/core/src/gateway/server.rs:876-993`. [OpenAI 호환 REST API](../api/openai-rest.md) 참조):

**`POST /v1/video/generations`**

| 필드 | 유형 | 참고 |
| --- | --- | --- |
| `model` | string | 필수 — 비디오 지원 backend를 선택합니다. |
| `prompt` | string | 필수. |
| `negative_prompt` | string? | |
| `images` | array? | Base64 데이터 URL(`data:image/png;base64,...`), `{ data_base64, mime_type }`로 전달. |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | Backend 선택 힌트(backend 이름과 매칭). |
| `extra` | object? | Backend별 재정의(seed, steps, cfg, ...). |

응답:

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

**`GET /v1/video/generations/{id}`**는 작업을 폴링하고 `id`, `object`, `model`, `status`, `progress`, `result`, `error`, `cost`, `created_at`을 반환합니다. 작업은 호출자 범위입니다: 다른 사용자가 소유한 작업은 404를 반환합니다. REST 표면은 chat 경로와 동일한 billing 게이트(월간 quota, 분당 rate limit)를 집행합니다.

## 비디오 생성(RPC)

동일한 기능이 JSON-RPC로 제공됩니다(JWT — `packages/core/src/gateway/rpc.rs:371-386,1296-1431`):

| 메서드 | Params | 반환값 |
| --- | --- | --- |
| `video.create` | REST 호출과 동일한 필드 | `{ job_id, stream_id }`. |
| `video.get` | `job_id` | 작업 보기(status, progress, result, cost, ...). |
| `video.list` | `limit?`(기본 20, 1~100으로 클램프) | `{ jobs: [...] }`, 최신순. |
| `video.cancel` | `job_id` | `{ "ok": true }` — 소유자만 취소할 수 있습니다. |

`video.create`는 `stream_id`를 반환합니다. 작업 알림(`video.progress` / `video.done` / `video.failed` — [Events & Notifications](../api/events.md) 참조)을 받으려면 `/api/rpc/events?session=<stream_id>`를 구독하세요.

## Backend

비디오 생성은 **클라우드 전용**입니다: MiniMax H3 Open Platform API, backend 종류 `minimax-cloud`(`BackendKind::CloudVideo` — `packages/core/src/backends/mod.rs:502-504,720-727,759-761`). 흐름은 작업 스타일입니다:

1. `POST /v1/video_generation_v2` → `task_id`
2. `Success` / `Fail` / 여전히 `Pending`일 때까지 `GET /v1/query/video_generation_v2?task_id=...` 폴링
3. 성공 시 `GET /v1/files/{file_id}/content` → `{ file: { download_url } }`로 아티팩트 해석

(`packages/core/src/backends/minimax_cloud.rs:66-210`). MiniMax backend는 chat/embeddings를 제공하지 않습니다. 기능은 `supports_video_generation`을 선언하고 `realtime: false`입니다(기능 모델은 [Backends](./backends.md) 참조). 라우팅은 선택적 `provider` 힌트를 존중하며 `supports_video_generation`이 있는 backend에 대해서만 video 요청을 해석합니다(`packages/core/src/routing/mod.rs:205-263`).

**ComfyUI backend는 제거되었습니다** — 모델 플랫폼 통합 중에: backend 종류 `"comfyui"` 구성은 `comfyui backend removed`로 실패합니다(`packages/core/src/backends/mod.rs:756-757`). 자체 호스팅 video 경로는 없습니다. Video는 항상 `minimax-cloud` backend를 통해 진행됩니다.

## 작업 수명 주기 및 가격 책정

Video 작업은 `queued → running → done | failed | cancelled`를 거칩니다(`packages/core/src/gateway/video.rs`):

- **create** — 작업 행이 영속화되고(`queued`, progress 0) 폴러 작업이 생성됩니다(`video.rs:109-176`).
- **running** — 폴러가 작업을 제출하고(progress 5), 1.5초마다 폴링하며, 몇 번의 반복마다 progress를 5씩 올려 최대 **90**까지(`video.rs:178-275`). 폴 오류는 로그되고 재시도됩니다.
- **done** — progress 100, 결과 URL과 계산된 비용이 영속화되고, 사용량이 기록되며, `video.done` 알림이 팬아웃됩니다(`video.rs:332-368`).
- **failed** — 제출 또는 폴 실패 → `video.failed`. 900회 폴 반복(약 22.5분) 후 작업은 `generation timed out`으로 실패합니다.
- **cancelled** — `video.cancel`이 다음 패스에서 폴러가 관찰하는 플래그를 설정합니다. 작업은 `cancelled`로 표시되고 `video.failed`가 `cancelled` 오류와 함께 발생합니다(`video.rs:389-400`).

사용량은 video 특정 비용으로 기록됩니다: `record_video`는 token 0개와 명시적 달러 비용으로 요청별 사용 기록을 씁니다(`packages/core/src/billing/mod.rs:496-531`).

**가격 책정**은 모델별이며 `video_pricing` 테이블에 있습니다(`packages/core/src/billing/video.rs`):

| 모드 | 공식 |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution`(기본값) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff`는 짧은 변 픽셀 키(예: `"768"`)를 승수에 매핑하며 `"*"`가 대체입니다. 구성된 행이 없는 모델은 모드 `per_second_resolution`, `base_price` 0.0, `price_per_second` 0.005, `price_per_frame` 0.0, `resolution_coeff {"*": 1.0}`, 통화 USD로 대체됩니다(`billing/video.rs:20-32`). 행은 `billing.video.pricing.get`(JWT)으로 조회하고 `billing.video.pricing.set`(admin token)으로 upsert하세요 — [JSON-RPC API](../api/jsonrpc.md) 참조. 사용 기록이 월간 billing으로 집계되는 방식은 [Billing 및 사용량](./billing-usage.md)을 참조하세요.

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
