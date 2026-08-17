---
title: "JSON-RPC API 참조"
description: "/api/rpc의 Arona 관리 평면 JSON-RPC 2.0 API — HTTP와 WebSocket을 통한 chat, realtime, engine, auth, keys, providers, agents, memory, conversations, usage, billing, video 및 system 메서드."
---

# JSON-RPC API 참조

Arona는 관리 평면을 위해 `/api/rpc`에 JSON-RPC 2.0 표면을 노출합니다: auth, keys, providers, agents, memory, conversations, usage, billing, video, realtime 및 스트리밍 chat. 이는 OpenAI 호환 REST 표면(`/v1/*`, [OpenAI 호환 REST API](./openai-rest.md) 참조)을 보완합니다. 키 인증 inference 워크로드에는 REST를, 세션/계정 관리와 스트리밍 제어에는 JSON-RPC를 사용하세요. [Quickstart](../guides/quickstart.md)가 첫 end-to-end 턴을 안내합니다.

표면은 **39개의 요청 메서드**와 하나의 익명 WebSocket 전용 liveness 메서드 `system.probe`(총 40개 메서드)를 디스패치합니다. 모든 요청은 `jsonrpc: "2.0"`, `method` 문자열, 선택적 `params` 객체, 선택적 `id`를 가진 JSON-RPC 2.0 객체입니다.

## 전송

- **HTTP POST `/api/rpc`** — 요청/응답. `Content-Type:
  application/json`을 보내세요. JWT는 `Authorization: Bearer <jwt>` 헤더로 이동합니다. 요청 본문은 1MiB로 제한됩니다.
- **WebSocket `GET /api/rpc`** — 장기 연결. 브라우저는 WebSocket 업그레이드에 사용자 지정 헤더를 설정할 수 없으므로, JWT는 `?token=<jwt>` 쿼리 매개변수로 이동합니다. 서버는 내부적으로 `Authorization: Bearer` 헤더로 접습니다(`packages/core/src/gateway/server.rs` 참조). 인증된 소켓은 무기한 연결 상태를 유지할 수 있습니다.
- **배치 요청** — JSON 배열인 POST 본문은 요소별로 실행되고 같은 순서의 응답 JSON 배열로 응답됩니다.
- **익명 액세스** — JWT 없이 WebSocket에서 공개 메서드(`auth.register`/`auth.login`/`auth.refresh`, `providers.list`, `system.status`)는 계속 호출 가능하며, `system.probe`는 소켓이 닫히기 전에 단일 ack로 응답됩니다. 다른 모든 메서드는 유효한 JWT가 필요하며, admin 게이트 메서드는 추가로 admin 계정이 필요합니다(아래 범례 참조). 익명 소켓은 10초 유휴 타임아웃에도 묶입니다.
- **세션 첨부** — `POST /api/rpc`의 `x-session-id` 헤더는 스트리밍 알림과 함께 RPC 응답 자체도 해당 세션 채널에 푸시합니다.

## Ids

요청 `id` 값은 유형 충실도로 에코됩니다: `null` → `null`, 문자열 → 문자열, 정수 → 숫자, 그 외의 것(부동소수점, 객체, i64 범위 밖의 정수) → JSON 문자열 렌더링. 생략된 `id`는 `null`로 응답됩니다.

## 서버 → 클라이언트 알림(SSE 사이드카)

Token, 배포 진행, realtime 이벤트는 WebSocket 소켓에서 **전달되지 않습니다**. 각 스트리밍 RPC는 세션 id를 생성하고 server-sent events로 `GET /api/rpc/events?session=<session_id>`에 알림을 푸시합니다. RPC 호출이 세션 id를 반환한 **전에 또는 직후에** SSE 엔드포인트를 구독하세요 — 호출이 반환된 시점과 SSE 구독이 설정되는 사이에 발생한 알림은 버려집니다(사전 구독 윈도우). 권장 패턴은 먼저 SSE 스트림을 연 다음 RPC를 발사하는 것입니다.

알림 메서드: `chat.stream`(`chat.send`의 이벤트당 하나의 token), `models.progress`(`agents.deploy`의 agent 모델 다운로드 진행), `realtime.event`(열린 realtime 세션의 서버 이벤트), `video.progress` / `video.done` / `video.failed`(비동기 video 작업). 전체 카탈로그는 [Events & Notifications](./events.md)를 참조하세요.

## 오류 코드

| 코드 | 이름 | 의미 |
| --- | --- | --- |
| `-32700` | Parse error | 요청 본문이 유효한 JSON이 아님. |
| `-32600` | Invalid request | 요청 객체가 잘못됨, 예: `method` 누락. |
| `-32601` | Method not found | 알 수 없는 `method` 문자열. 메시지가 이를 에코합니다. |
| `-32602` | Invalid params | `params`가 메서드에 대한 역직렬화에 실패. |
| `-32603` | Internal error | 예상치 못한 서버 실패. |
| `-32000` | `APP_ERROR` | 일반 애플리케이션 오류 — 예: 대화/provider/agent를 찾을 수 없음, 배포할 온라인 agent 없음. |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` — 누락되거나 잘못된 JWT. Bearer token이 `ARONA_ADMIN_TOKEN`과 일치하지 않을 때 admin-token 메서드에서도 사용됩니다(`"Admin access required"`). |
| `-32006` | `QUOTA_ERROR` | JWT 게이트 RPC 메서드(`chat.send`)의 월간 billing quota 초과. |
| `-32007` | `ADMIN_REQUIRED` | Admin 게이트 메서드(`agents.*`, `engine.invoke`)를 호출하는 인증된 **비관리자**. 메시지에 메서드별 힌트가 포함됩니다. |

> `agents.*` 및 `engine.invoke` 메서드는 admin 전용입니다: `users.is_admin = true`인 계정의 JWT가 필요합니다. 인증된 비관리자는 `-32007`(`ADMIN_REQUIRED`)로 거부됩니다. 인증되지 않은 호출자는 표준 `AUTH_ERROR`를 받으므로 서버가 메서드가 특권적임을 드러내지 않습니다.

## 인증 범례

| 범례 | 자격 증명 |
| --- | --- |
| **public** | 자격 증명 불필요. |
| **JWT** | HTTP에서 `Authorization: Bearer <jwt>`, WebSocket에서 `?token=<jwt>`. |
| **admin (JWT + is_admin)** | `users.is_admin = true`인 계정의 Bearer JWT. |
| **admin token** | Bearer `ARONA_ADMIN_TOKEN`(env로 구성. 설정되지 않으면 메서드가 항상 거부됨, 기본 거부). |

이 문서의 모든 예제 자격 증명과 주소는 플레이스홀더입니다(RFC 5737 문서화 IP, `sk-xxx` 키). 이 범례 뒤의 전체 인증 모델은 [인증 및 보안](../guides/auth-security.md)을 참조하세요.

## Chat

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model`(string), `messages`(`{ role, content, images?, tool_calls? }` 배열), `temperature?`(number), `max_tokens?`(integer), `conversation_id?`(string), `memory?`(bool), `extra?`(object), `tools?`(OpenAI 스타일 함수 정의 배열), `provider?`(string) | 스트리밍 chat 턴을 보냅니다. `{ "stream_id", "memory" }`를 반환합니다 — `memory`는 회상 상태(`enabled` / `disabled` / `offline`). Token은 SSE 사이드카의 `chat.stream` 알림으로 도착합니다. `conversation_id`가 있으면 완료된 영속화 기록이 서버 측에서 조립되고 턴이 영속화됩니다. Billing 게이트 적용(월간 quota → `-32006`). 사용량은 `jwt-<user-uuid>` 아래에 기록됩니다. |

## Realtime(전이중 오디오/비디오 세션)

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model`(string), `config?`(세션 구성 객체), `conversation_id?`(string) | `model`을 제공하는 backend에 대해 전이중 세션을 엽니다. `{ "session_id", "stream_session" }`을 반환합니다: `realtime.event` / `realtime.stop`에는 `session_id`를 사용하고, SSE 사이드카의 `stream_session`을 구독하여 `realtime.event` 알림을 받습니다. |
| `realtime.event` | JWT | `session_id`(string), `event`(클라이언트 이벤트 — 오디오 append/commit/clear, 이미지 프레임, response create/cancel, session stop) | 열린 세션에 클라이언트 이벤트 하나를 보냅니다. 업스트림 backend로 전달됩니다. `{ "ok": true }`를 반환합니다. |
| `realtime.stop` | JWT | `session_id`(string) | 세션을 닫고 제거합니다. `{ "removed": bool }`를 반환합니다. |

## Engine(일반 인지/제어 채널)

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `engine.invoke` | admin (JWT + is_admin) | `model`(string), `method`(string), `params?`(object) | `model`을 제공하는 backend에서 임의의 엔진 메서드를 동기 요청/응답 호출 — `sensor.ingest` / `control.setpoint` 스타일 호출(20~30Hz 루프)의 고주파 채널. 결과는 backend의 원시 응답입니다. |

## Auth

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `auth.register` | public | `email`, `password`, `name?` | 계정을 등록합니다. 가입이 열린 동안에만(`ARONA_REGISTRATION_OPEN`) 허용됩니다. 첫 번째 등록 사용자가 admin이 됩니다. `auth.login`과 동일한 token 응답(`access_token`, `refresh_token`, `token_type`, `expires_in`, `user`)을 반환합니다. |
| `auth.login` | public | `email`, `password` | 로그인. `access_token`, `refresh_token`, `token_type`, `expires_in`, `user`(`{ id, email, name, is_admin }`)를 반환합니다. IP 및 계정별 rate-limited. |
| `auth.refresh` | public | `refresh_token` | Refresh token을 새 access token(및 새 refresh token)으로 교환합니다. 재사용되거나 만료된 refresh token은 `AUTH_ERROR`로 거부됩니다. |
| `auth.me` | JWT | — | 현재 사용자 프로필: `{ "id", "email", "name" }`. |

## Keys

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | 호출자의 API keys를 나열합니다(id, name, `key_prefix`, project, 타임스탬프, active 플래그). |
| `keys.create` | JWT | `name`, `project?` | API key를 생성합니다. `{ id, name, key, key_prefix, project, created_at }`를 반환합니다 — `key`의 전체 `arona-<uuid>` 시크릿은 **한 번만** 표시됩니다. 즉시 저장하세요. |
| `keys.revoke` | JWT | `key_id` | API key를 폐기합니다. `{ "ok": true }`를 반환합니다. |

## Providers

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | 알려진 providers를 나열합니다: 내장 공식 항목과 사용자 지정 항목, 표시 메타데이터(`id`, `name`, `description`, `website_domain`, `is_official`, `is_operator`)로. 설계상 공개 — 목록은 자격 증명을 담지 않습니다. 아래 변경 메서드만 JWT 게이트입니다. |
| `providers.add` | JWT | `id`, `name`, `description?`, `website_domain?` | 사용자 지정 provider 항목을 추가합니다. `{ "ok": true }`를 반환합니다. |
| `providers.update` | JWT | `provider_id`, `name?`, `description?`, `website_domain?` | 사용자 지정 provider의 필드를 업데이트합니다(제공된 것만). `{ "ok": true }`를 반환합니다. |
| `providers.remove` | JWT | `provider_id` | 사용자 지정 provider를 제거합니다. `{ "ok": true }`를 반환합니다. |
| `providers.test` | JWT | — | Provider 연결을 테스트합니다. 스텁: `{ "ok": true, "message": "Provider connection test not yet implemented" }`를 반환합니다. |

## Agents

모든 `agents.*` 메서드는 admin 전용입니다(JWT + `is_admin`). Agent 노드는 `GET /ws/agent`로 아웃바운드 연결합니다. 이 RPC 그룹은 레지스트리를 제어합니다([Agent Cluster](../guides/agent-cluster.md) 참조).

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `agents.list` | admin (JWT + is_admin) | — | 등록된 agent 노드를 나열합니다: id, name, host, `online`/`offline` 상태(하트비트 기반), GPU 요약, 배포된 모델, version, 타임스탬프. |
| `agents.register` | admin (JWT + is_admin) | `machine_name`, `version` | 터널 관리자에 agent 노드를 등록합니다. `{ "agent_id", "token" }`을 반환합니다(token은 agent의 제어 평면 자격 증명). |
| `agents.deregister` | admin (JWT + is_admin) | `agent_id` | Agent의 등록을 해제(연결 끊기)합니다. `{ "ok": true }`를 반환합니다. |
| `agents.status` | admin (JWT + is_admin) | `agent_id` | Agent별 상태: online 플래그, host, GPU 요약, 로드된 모델, GPU 사용률, 하트비트/연결 타임스탬프. |
| `agents.deploy` | admin (JWT + is_admin) | `model_id`, `agent_id?`(비어 있거나 없음 = 최소 부하 노드. 온라인 없음이면 오류) | Agent에 모델을 배포합니다. `{ "ok": true, "stream_id" }`를 반환합니다 — `models.progress` 다운로드 알림은 SSE 사이드카의 `stream_id`를 구독하세요. |
| `agents.stop` | admin (JWT + is_admin) | `agent_id`, `model_id` | 배포된 모델을 중지합니다. `{ "ok": true, "stream_id": null }`을 반환합니다(진행 스트림 없음). |

## Memory

장기 memory는 WebSocket을 통해 entelecheia Philia 서비스가 제공합니다. 실패는 chat을 절대 차단하지 않습니다([Memory Gateway](../guides/memory-gateway.md) 참조).

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | Memory gateway 상태: `{ "enabled", "writeback", "events" }` — 플래그와 최대 50개의 최근 활동 이벤트(최신순). |
| `memory.delete` | JWT | `node_id` | 저장된 memory 노드를 삭제합니다. `{ "deleted": bool }`를 반환합니다. |

## Conversations

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | 상대 나이 타임스탬프와 함께 호출자의 대화를 나열합니다. |
| `conversations.create` | JWT | `title?`(기본값 `New Conversation`) | 대화를 생성합니다. 새 대화 객체를 반환합니다. |
| `conversations.get` | JWT | `conversation_id`(레거시 별칭: `id`) | 메시지와 함께 대화를 가져옵니다. 소유권 검사. 교차 사용자 액세스는 거부됩니다. |
| `conversations.delete` | JWT | `conversation_id`(레거시 별칭: `id`) | 대화를 삭제합니다(소유자만). `{ "ok": true }`를 반환합니다. |

> `conversations.get` / `conversations.delete`는 이전 대시보드 클라이언트의 레거시 `id` 키도 허용합니다. 둘 다 있으면 `conversation_id`가 우선합니다.

## Usage

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?`(integer, 기본 50, 1–200으로 클램프), `offset?`(integer, 기본 0), `project?`(string) | 호출자의 페이지네이션된 사용 기록, 최신순, API-key 행(`arona-XX` 접두어)과 JWT 귀속 행(`jwt-<user-uuid>`) 모두 포함. `{ "records", "total", "limit", "offset", "project" }`를 반환합니다. `project` 필터는 키 태그 행만 좁힙니다. |

## Billing

Tiers, quota 및 사용량 회계는 [Billing 및 사용량](../guides/billing-usage.md)에 설명되어 있습니다.

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | 현재 billing 상태: `{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` — 월간 사용량(`cost_usd`, tokens, 요청 수)과 남은 quota. |
| `billing.plan.set` | admin token | `user_email`, `tier` | 사용자의 billing tier를 설정합니다. `{ "ok": true }`를 반환합니다. Bearer가 `ARONA_ADMIN_TOKEN`과 일치하지 않으면 `AUTH_ERROR`로 거부됩니다. |
| `billing.video.pricing.get` | JWT | — | Video pricing 테이블. `{ "pricing": [...] }`를 반환합니다. |
| `billing.video.pricing.set` | admin token | `model`, `mode?`(기본 `per_second_resolution`), `base_price?`(number, 기본 0), `price_per_second?`(number, 기본 0), `price_per_frame?`(number, 기본 0), `resolution_coeff?`(object), `currency?`(기본 `USD`), `enabled?`(bool, 기본 `true`) | 모델에 대한 video pricing을 upsert합니다. `{ "ok": true }`를 반환합니다. Bearer가 `ARONA_ADMIN_TOKEN`과 일치하지 않으면 `AUTH_ERROR`로 거부됩니다. |

## Video

비동기 비디오 생성 작업([Realtime & Video](../guides/realtime-video.md) 참조). 작업 진행은 세션 채널에 `video.progress` / `video.done` / `video.failed` 알림으로 푸시됩니다.

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`, `prompt`, `negative_prompt?`, `images?`(`{ data_base64, mime_type }` 배열), `duration_seconds?`(integer), `width?`(integer), `height?`(integer), `provider?`(string), `extra?`(object) | 비동기 비디오 생성 작업을 제출합니다. `{ "job_id", "stream_id" }`를 반환합니다 — 진행 알림은 `stream_id`를 구독하세요. |
| `video.get` | JWT | `job_id`(UUID) | 작업의 상태/결과를 폴링합니다(status, progress, result, error, cost). |
| `video.list` | JWT | `limit?`(integer, 기본 20) | 호출자의 작업을 나열합니다. `{ "jobs": [...] }`를 반환합니다. |
| `video.cancel` | JWT | `job_id`(UUID) | 실행 중인 작업을 취소합니다. `{ "ok": true }`를 반환합니다. |

## System

| 메서드 | Auth | Params | 설명 |
| --- | --- | --- | --- |
| `system.status` | public | — | Gateway 상태 집계: `{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`. |
| `system.probe` | anonymous (WS only) | — | WebSocket 전송을 통한 일회성 liveness probe. 서버는 `{ "ok": true, "status": "ok" }`를 ack하고 소켓을 닫습니다 — 익명 방문자는 열린 연결을 유지하지 않습니다. 인증되지 않은 소켓의 다른 모든 메서드는 `AUTH_ERROR`로 거부됩니다. |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
