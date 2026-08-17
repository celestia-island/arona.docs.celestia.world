---
title: "OpenAI 호환 REST API"
description: "OpenAI 스타일 /v1/* 참조 — chat completions, embeddings, 모델 목록, 비동기 비디오 생성, 오류 형태 및 rate limits."
---

# OpenAI 호환 REST API

Arona는 LLM chat, embeddings, 모델 목록, health probing, 비동기 비디오 생성을 위해 `/v1/*` 아래에 OpenAI 호환 REST 표면을 노출합니다. 기본 URL을 가리키는 모든 OpenAI SDK는 chat과 embeddings에서 작동합니다. Video 엔드포인트는 OpenAI의 작업 스타일 submit/poll 관례를 따릅니다.

모든 요청과 응답 본문은 JSON입니다. 오류는 균일한 형태를 사용합니다([오류](#errors) 참조). 미들웨어 계층의 인증 실패가 유일한 예외이며 일반 텍스트로 반환됩니다([인증](#authentication) 참조).

## 엔드포인트 한눈에 보기

| 메서드 | 경로 | 설명 |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | Chat 턴, 스트리밍 또는 비스트리밍. |
| `POST` | `/v1/embeddings` | 하나 또는 여러 입력에 대한 embedding 벡터. |
| `GET` | `/v1/models` | Quick-start 모델과 병합된 라우터 모델. |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`. |
| `POST` | `/v1/video/generations` | 비동기 비디오 생성 작업 제출. |
| `GET` | `/v1/video/generations/{id}` | Video 작업의 상태 / 결과 폴링. |

`/api/health`, `/healthz`, `/readyz`는 추가 readiness probe입니다(`/v1/health`의 Kubernetes 스타일 별칭).

## 인증

Chat, embeddings, video 엔드포인트는 `Authorization: Bearer` 헤더의 **API key**로 인증합니다. API keys는 관리 평면(`keys.create`, [JSON-RPC API](./jsonrpc.md#keys) 참조)을 통해 생성되며 `arona-<uuid>` 형태입니다. 서버 측에 SHA-256 해시로 저장됩니다.

```
Authorization: Bearer arona-CHANGE_ME
```

- **헤더 누락** → `401` 일반 텍스트: `Missing Authorization header. Use: Bearer <api_key>`.
- **잘못되거나 폐기된 키** → `401` 일반 텍스트: `Invalid API key`.
- `GET /v1/models`는 추가로 **JWT** 액세스 token(`auth.login` / `auth.register`가 발급)을 허용하므로 웹 대시보드가 RPC 평면에 사용하는 것과 동일한 token으로 모델을 나열할 수 있습니다. 이 엔드포인트의 메시지는 `Missing Authorization header. Use: Bearer <api_key_or_jwt>`와 `Invalid API key or JWT`입니다.

미들웨어 수준 거부는 [오류](#errors)에 설명된 JSON 오류 형태가 아닌 일반 텍스트 본문입니다 — JSON 형태는 요청이 핸들러에 도달한 후에만 생성됩니다.

인증된 모든 `/v1` 요청은 또한 **인메모리 키별 rate limiter**(기본 60 RPM, 60초 윈도우, `ARONA_API_RATE_LIMIT_RPM`으로 구성 가능)를 통과합니다. 초과하면 `429` 일반 텍스트를 반환합니다: `Rate limit exceeded. Try again later.` Tier 수준 quota와 rate limits은 별도로 집행되며 `Retry-After` 헤더가 있는 JSON 429를 반환합니다([429 및 Retry-After](#429-and-retry-after) 참조).

> API keys, projects 및 그 범위 관리에 대해서는 [인증 및 보안](../guides/auth-security.md)에서 다룹니다.

## POST /v1/chat/completions

스트리밍 지원과 arona 특정 확장(`conversation_id`, `memory`, `extra`, `provider`)이 있는 핵심 OpenAI 호환 chat 엔드포인트.

### 요청 본문

| 필드 | 유형 | 필수 | 참고 |
| --- | --- | --- | --- |
| `model` | string | 예 | `GET /v1/models`에 나열된 모델 id. |
| `messages` | array | 예 | Chat 턴, 아래 참조. |
| `stream` | boolean | 아니요 | 기본값 `false`. `true`이면 응답이 SSE 스트림입니다([스트리밍](#streaming) 참조). |
| `temperature` | number | 아니요 | 샘플링 온도, 업스트림으로 전달. |
| `max_tokens` | integer | 아니요 | Completion token 상한, 업스트림으로 전달. |
| `conversation_id` | string | 아니요 | 세션 고정 + 영속화. 대화가 존재하고 API-key 사용자 소유여야 합니다(그렇지 않으면 `403` `conversation_forbidden`, 존재하지 않으면 `404` `conversation_not_found`). 사용자 턴은 전송 시점에, assistant 응답은 턴 완료 시 영속화됩니다. 라우팅은 대화를 처음 제공한 backend에 고정합니다. |
| `memory` | boolean | 아니요 | Memory gateway 재정의. 기본값 `true`(memory gateway가 활성화되면 memory 회상이 주입됨). `false`는 이 요청의 회상 주입을 비활성화합니다. |
| `extra` | object | 아니요 | 업스트림 페이로드 최상위에 병합되는 자유 형식 패스스루(아래 참조). |
| `tools` | array | 아니요 | OpenAI 스타일 함수 호출 정의, 업스트림에 그대로 전달. |
| `provider` | string | 아니요 | Backend **이름**(또는 종류)과 대소문자 구분 없이 매칭되는 명시적 backend 선택 힌트. 설정되면 힌트와 일치하는 backend만 후보입니다. |

**`messages` 항목**은 `{ "role": "user" | "assistant" | "system", "content": "..." }`입니다. 멀티모달 / agent 워크로드용 확장 두 가지가 업스트림으로 전달됩니다:

- `images` — vision 요청용 첨부 이미지(`{ "media_type", "data", "position" }` 객체 배열. external backend가 이를 OpenAI `image_url` 콘텐츠 파트로 렌더링).
- `tool_calls` — 업스트림 모델이 생성한 함수 호출 페이로드로, 후속 턴에서 에코백됩니다. External backend가 이를 전달해야 합니다. 그렇지 않으면 agent 파이프라인(예: scepter 스킬 체인)이 모든 도구 호출을 잃습니다.

**`extra` 병합 규칙**: 모든 `extra` 키는 업스트림 요청 페이로드 최상위에 병합되며, 두 가지 하드 보장이 있습니다 — 예약 키 `model`, `messages`, `stream`, `temperature`, `max_tokens`, `options`는 **절대** 재정의되지 않으며, gateway가 이미 설정한 키도 재정의되지 않습니다. 객체가 아닌 `extra` 값은 무시됩니다.

**`tools` 항목**은 `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }`이며 그대로 전달됩니다.

### 비스트리밍 응답

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1720000000,
  "model": "Qwen/Qwen3-1.7B",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 12, "completion_tokens": 2, "total_tokens": 14 }
}
```

- `choices[].message`는 함수 호출 턴에 `tool_calls`를 담을 수 있습니다.
- 요청의 memory 상태는 **`X-Arona-Memory`** 응답 헤더에 반영됩니다: `enabled` | `disabled` | `offline`.

### 스트리밍

`"stream": true`로 설정합니다. 응답은 `text/event-stream` SSE 스트림입니다 — 청크당 하나의 `data:` 라인, 각각 단일 JSON `ChatChunk`:

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- `choices[].delta`는 `content`를 담습니다(함수 호출 스트림의 경우 `index`/`id`/`type`/`function`이 있는 `tool_calls` 델타도).
- OpenAI 호환 업스트림에서 **터미널 청크**는 실제 token 수를 담은 `usage` 필드를 가집니다. Gateway가 이를 기록합니다(사용량을 절대 보고하지 않는 업스트림 — 예: ollama / ws_engine — 은 로컬 토크나이저 추정으로 대체).
- 스트림은 `data: [DONE]`으로 종료됩니다.
- 스트림 오류는 `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}`를 담은 `data:` 이벤트로 전달됩니다. `[DONE]` 이벤트는 여전히 뒤따르며, 실패한 스트림에 대해서는 사용량 기록과 assistant 영속화가 건너뜁니다.
- `X-Arona-Memory` 헤더는 SSE 응답에도 있습니다.

### 예제

```bash
curl http://192.0.2.10:8080/v1/chat/completions \
  -H "Authorization: Bearer arona-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-1.7B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

## POST /v1/embeddings

| 필드 | 유형 | 필수 | 참고 |
| --- | --- | --- | --- |
| `model` | string | 예 | Embedding 모델 id(예: `nomic-embed-text` — 단순 이름은 `:latest` 태그와도 매칭). |
| `input` | string 또는 string[] | 예 | 하나 또는 여러 입력. |

응답: `{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`.

## GET /v1/models

오늘 라우팅 가능한 모델을 나열합니다: 모든 healthy 등록 backend의 모델 목록에 내장 **quick-start 모델**(backend 등록 전에도 항상 광고됨)을 병합한 것: `Qwen/Qwen3-0.6B`, `Qwen/Qwen3-1.7B`, `HuggingFaceTB/SmolLM2-1.7B-Instruct`, `google/gemma-3-1b-it`, `microsoft/Phi-4-mini-instruct`, `deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`.

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

Quick-start 모델은 `owned_by`가 해당 provider로 나타납니다. 라우터 모델은 소유 backend의 이름을 담습니다.

## 비디오 생성

비디오 지원 backend(예: `minimax-cloud`, [Backends](../guides/backends.md) 참조)용 작업 스타일 video 엔드포인트. 작업은 비동기로 진행됩니다. `done`까지 상태 엔드포인트를 폴링하세요.

### POST /v1/video/generations

| 필드 | 유형 | 필수 | 참고 |
| --- | --- | --- | --- |
| `model` | string | 예 | 비디오 지원 backend에 등록된 video 모델 id. |
| `prompt` | string | 예 | 생성 프롬프트. |
| `negative_prompt` | string | 아니요 | 부정 프롬프트. |
| `images` | array | 아니요 | `{ "data_base64": "...", "mime_type": "image/png" }` 객체 배열로 된 컨디셔닝/참조 이미지. |
| `duration_seconds` | integer | 아니요 | 요청된 지속 시간. |
| `width` / `height` | integer | 아니요 | 출력 해상도. |
| `provider` | string | 아니요 | 명시적 backend 선택 힌트(backend 이름). |
| `extra` | object | 아니요 | Backend별 워크플로우 재정의(seed, steps, cfg, ...). |

성공 → `200`:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

오류: `model` 또는 `prompt`가 없으면 `400` `missing_fields`. 모델을 제공하는 healthy 비디오 지원 backend가 없으면 `503` `video_backend_error` / `no_backend`. 월간 quota가 소진되면 `429` `quota_error` / `quota_exceeded`.

### GET /v1/video/generations/{id}

작업 상태를 반환합니다:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "running",
  "progress": 40,
  "result": null,
  "error": null,
  "cost": 0.0,
  "created_at": 1720000000
}
```

- `status`: `queued` | `running` | `done` | `failed` | `cancelled`. `progress`는 running 중 0–90으로 진행되고 `done`에서 100에 도달합니다.
- `result`(`done` 시): `{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }` — `url`은 backend가 제공하는 생성 아티팩트를 가리킵니다.
- `error`(`failed` / `cancelled` 시)와 `cost`는 해당될 때 채워집니다.
- 오류: 비-UUID id는 `400` `bad_id`. 작업이 존재하지 않거나 다른 API key 소유이면 `404` `no_job`.

Video 작업은 RPC SSE 사이드카로도 진행 상황을 팬아웃합니다(`video.progress` / `video.done` / `video.failed`, [Events & Notifications](./events.md#video-job-notifications) 참조).

## 오류

Gateway 수준 오류는 하나의 형태를 사용합니다(`json_error_response`):

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| 상태 | `type` / `code` | 시점 |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`, `missing_index`, `bad_id`, ... | 잘못되거나 누락된 요청 필드. |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id`가 다른 사용자 소유. |
| `404` | `invalid_request_error` / `model_not_found` | 요청된 모델을 제공하는 backend 없음. 메시지: `No backend available for model: <model>`. |
| `404` | `invalid_request` / `conversation_not_found` | 대화를 찾을 수 없음. |
| `404` | `not_found` / `no_job` | Video 작업을 찾을 수 없음. |
| `502` | `server_error` / `bad_gateway` | 업스트림 비-2xx: 메시지 `upstream <status>: <detail>`(업스트림 오류 본문의 상세, 4KB로 제한). 전송 실패(connect/read/timeout)도 오류 문자열과 함께 502로 매핑됩니다. |
| `500` | `server_error` / `backend_error` | 기타 backend 실패(예: backend가 작업을 지원하지 않음). |
| `500` | `server_error` / `internal_error` | 기타 남은 gateway 내부 오류. |
| `429` | 아래 참조 | `Retry-After`가 있는 quota / rate-limit 거부. |

## 429 및 Retry-After

429 응답에는 `Retry-After` 헤더(초)가 포함되어 OpenAI 호환 클라이언트가 백오프합니다:

| 트리거 | 상태 본문 | `Retry-After` |
| --- | --- | --- |
| 월간 quota 초과 | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | 다음 달까지의 초. |
| Tier 분당 rate limit | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`. |
| 인메모리 키별 리미터(기본 60 RPM) | 일반 텍스트 `Rate limit exceeded. Try again later.` | 없음(미들웨어 거부). |

Tiers, quota 범위 및 사용량 회계는 [Billing 및 사용량](../guides/billing-usage.md)에 설명되어 있습니다.

## 사용량 기록

모든 `/v1` 요청은 완료 시 API-key 접두어(`arona-XX`) 아래에 사용량 행을 기록합니다(비스트리밍 chat, 터미널 청크의 스트리밍 chat, embeddings, 완료 시 계산된 비용이 있는 video 작업). 기록 모델과 quota 집행 방식은 [Billing 및 사용량](../guides/billing-usage.md)을 참조하세요.

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
