---
title: "Billing 및 사용량"
description: "사용 기록(usage records), 모델별 비용, billing tiers, quota 및 rate-limit 집행, 프로젝트 범위 키, 비디오 가격 책정 및 usage.list RPC."
---

# Billing 및 사용량

Arona는 모든 모델 요청을 계량하고 gateway에서 계층화된 quota와 rate limits을 집행합니다. 모델별 가격은 공유 plana 가격표에서 가져오며(arona에서 절대 재구현하지 않음), 사용량은 `usage_records` 행으로 영속화되며, 월간 전체 그림은 `usage.list` RPC를 통해 노출됩니다.

## 사용 기록

모든 계량된 요청은 `usage_records` 테이블의 한 행이 됩니다(`m20250101_000006_create_usage_records`):

| 열 | 유형 | 의미 |
| --- | --- | --- |
| `id` | `UUID` | 기본 키, 생성됨. |
| `api_key_id` | `VARCHAR(64)` | **키 접두어** — API key의 처음 8자(키는 `arona-{uuid}` 형태) 또는 JWT 귀속 RPC 채널용 합성 `jwt-<user-uuid>` id. |
| `model` | `VARCHAR(128)` | 요청이 라우팅된 모델 id. |
| `backend` | `VARCHAR(64)` | Backend 종류: `gateway`, `rpc`, `realtime` 또는 backend 기능 이름. |
| `prompt_tokens` | `INTEGER` | 입력 token, 업스트림 보고 또는 추정. |
| `completion_tokens` | `INTEGER` | 출력 token, 업스트림 보고 또는 추정. |
| `total_tokens` | `INTEGER` | 두 값의 합. |
| `cost` | `DOUBLE PRECISION` | 계산된 USD 비용. 모델에 가격 행이 없으면 `NULL`. |
| `created_at` | `TIMESTAMPTZ` | 요청이 완료된 시점. |

`api_key_id`, `model`, `created_at`(월간 집계와 rate-limit 윈도우가 스캔하는 열)에 인덱스가 있습니다.

## 기록 채널

사용량은 모든 계량 채널에서 기록됩니다:

1. **REST 비스트리밍** — `POST /v1/chat/completions`와 `POST /v1/embeddings`는 응답이 생성된 후 정확한 업스트림 보고 사용량을 기록합니다.
2. **REST 스트리밍(SSE)** — 스트림이 사용량을 실은 경우(OpenAI 호환 터미널 청크 `usage` 필드) 업스트림 보고 사용량이 우선합니다. 그렇지 않으면 로컬 CJK 인식 토크나이저 추정(`estimate_usage`)이 그대로 기록됩니다. 텍스트도 사용량도 생성하지 않은 스트림은 **전혀** 기록되지 않습니다.
3. **RPC `chat.send`** — 동일한 추정-대-업스트림 로직이 적용됩니다. 행은 합성 `jwt-<user-uuid>` id로 귀속되어 사용자에게 다시 조인됩니다.
4. **Realtime 세션** — 각 완료된 `response_done` 트랜스크립트는(0이 아닐 때) `jwt-<user-uuid>` 아래에 backend `realtime`으로 token 사용량을 기록합니다.
5. **Video 작업** — 완료된 작업은 명시적 달러 비용을 기록합니다([비디오 가격 책정](#video-pricing) 참조). Token 수는 0입니다.

기록은 최선 노력입니다. 실패한 insert는 로그로 남고 요청을 실패시키지 않습니다.

## 비용 계산

비용은 표준 1M token당 가격표(`plana_llm_provider::metering::lookup_pricing`, 모든 celestia-island 서비스에서 공유)에서 계산됩니다:

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

테이블의 모델 매칭은 소문자 모델 id에 대한 부분 문자열 기반입니다(더 구체적인 패밀리가 우선). 모델에 가격 행이 없으면 `cost`는 `NULL`입니다. **arona에서 가격 책정을 재구현하지 마세요 — plana의 테이블을 업데이트하세요.**

## Tiers

Tiers는 `billing_tiers` 테이블에 있으며, 첫 마이그레이션 시 시드됩니다(`m20250101_000007_create_billing_tiers`). `NULL` quota 열은 해당 차원이 "무제한"임을 의미합니다. `tier_id`가 없는 사용자는 시드된 `free` tier로 대체됩니다.

| Tier | 월간 USD quota | 월간 token quota | 키별 RPM |
| --- | --- | --- | --- |
| `free` | $1.00 | 100,000 | 10 |
| `pro` | $20.00 | 5,000,000 | 120 |
| `enterprise` | 무제한(`NULL`) | 무제한(`NULL`) | 1000 |

Tier 할당은 admin 작업입니다(`billing.plan.set` RPC). 현재 tier와 사용량은 `billing.plan`을 통해 표시됩니다.

## Quota 및 rate-limit 집행

### REST(`/v1/*`)

모든 **계량된** REST 엔드포인트 — `/v1/chat/completions`, `/v1/embeddings`, `/v1/video/generations` — 앞에서 gateway는 키 인증 요청에 대해 두 개의 게이트를 집행합니다:

- **월간 quota** — tier의 `monthly_quota_usd` 및 `quota_tokens` 제한을 현재 달력 월의 첫 순간부터 누적된 사용량과 비교합니다. 어느 차원이든 한도에 도달하면 게이트가 작동합니다.
- **키별 rate limit** — tier의 `rate_limit_rpm`을 지난 60초 윈도우에서 이 키 접두어에 기록된 요청 수와 비교합니다. (`/v1/models`와 health 엔드포인트는 계량되지 않고 게이트도 적용되지 않습니다.)

거부는 `Retry-After` 헤더와 OpenAI 스타일 오류 본문이 있는 HTTP **429 Too Many Requests**입니다:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| 거부 | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| 월간 quota 소진 | `quota_error` / `quota_exceeded` | **다음 달력 월** 시작까지의 초 |
| Tier rate limit 초과 | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

JWT 인증 `chat.send`는 동일한 월간 quota 게이트를 거치지만, **전체 사용자** 윈도우에 대해 적용됩니다(호출에 API key가 없음). 거부는 구현 정의 코드 `-32006`(`QUOTA_ERROR`)와 REST quota 거부와 동일한 메시지를 가진 JSON-RPC 오류입니다. RPC 경로에는 키별 rate limit이 없습니다 — rate limiting은 키 범위이며 RPC 호출에는 키가 없습니다. Realtime 및 video **RPC** 메서드는 quota 게이트가 적용되지 않습니다.

## Fail-open 트레이드오프

Billing은 **설계상 최선 노력**입니다. Quota 또는 rate-limit 검사 뒤의 데이터베이스 쿼리가 실패하면 검사는 `Unknown`을 반환하고 요청은 chat을 차단하는 대신 **허용**(로그만)됩니다. 운영자는 429에 의존해 용량을 보호할 수 있지만, 데이터베이스가 unhealthy할 때 이를 확실한 보장으로 취급해서는 안 됩니다 — 문서화된 트레이드오프는 엄격한 계량보다 chat 경로의 가용성입니다.

## 프로젝트 범위 키

API keys는 `project` 라벨과 함께 생성될 수 있습니다(`api_keys.project`, 설정되지 않으면 `default`). Quota 집행이 이를 존중합니다:

- `default`가 아닌 project로 태그된 키는 **해당 project 자체 버킷**에 귀속된 사용량에 대해 quota를 확인합니다(`PROJECT_MONTHLY_USAGE_SQL`).
- `default` / 태그 없는 키는 **전체 사용자** 윈도우를 유지하며, 프로젝트 이전 동작과 일치합니다.

JWT 귀속 RPC 행(`jwt-<user-uuid>`)은 project 라벨이 없으며 **의도적으로** project 윈도우에서 제외됩니다 — 여전히 전체 사용자 윈도우에 계산되므로, RPC 채널로 트래픽을 보내 project를 "숨길" 수 없습니다.

## 비디오 가격 책정

비디오 생성은 모델별 작업 스타일 가격 책정을 사용합니다(token별 가격 책정은 비디오에 의미가 없음). 가격 행은 `video_pricing` 테이블에 있습니다. `compute_cost`는 행이 구성되지 않았을 때 플레이스홀더 기본값으로 대체됩니다.

| 모드 | 비용 |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution`(기본값) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff`는 짧은 변 픽셀 값(예: `"768"`)을 키로 하는 JSON 객체입니다. `"*"` 키가 대체입니다. 기본 가격 행은 모드 `per_second_resolution`, `base_price` 0.0, `price_per_second` 0.005, `resolution_coeff {"*": 1.0}`입니다. 행은 `billing.video.pricing.get`(모든 JWT)과 `billing.video.pricing.set`(Bearer `ARONA_ADMIN_TOKEN`)으로 관리됩니다. 계산된 비용은 작업이 완료될 때 작업의 키에 대해 기록됩니다.

## usage.list

`usage.list`(JWT)는 호출자의 페이지네이션된 사용 기록을 반환하며, **API-key 행**(키 접두어로 조인)과 JWT 귀속 행(합성 `jwt-<user-uuid>` id로 조인) **모두**를 최신순으로 포함합니다:

| 매개변수 | 기본값 | 참고 |
| --- | --- | --- |
| `limit` | `50` | `1..=200`으로 클램프됩니다. |
| `offset` | `0` | 페이지 오프셋. |
| `project` | 설정 안 됨 | 설정되면 해당 project 라벨이 있는 키에 귀속된 레코드만(JWT 행 제외). |

응답은 `{ "records": [...], "total", "limit", "offset", "project" }`이며, 각 레코드는 `id`, `model`, `backend`, `prompt_tokens`, `completion_tokens`, `total_tokens`, `cost`, `created_at`을 가집니다. 월간 quota 집계는 동일한 조인 형태를 사용하므로 quota 계산과 사용량 보기가 항상 범위에 대해 일치합니다.

## 관련

- [빠른 시작](quickstart.md) — 키를 얻고 첫 계량 요청을 만듭니다.
- [구성](configuration.md) — gateway용 환경 변수.
- [인증 및 보안](auth-security.md) — API key 생성과 `project` 라벨.
- [Realtime & Video](realtime-video.md) — 비디오 가격 책정 뒤의 video 작업 수명 주기.
- [Operations](operations.md) — health probes 및 관찰 가능성.
- [OpenAI 호환 REST API](../api/openai-rest.md) — `/v1/*` 표면.
- [JSON-RPC API](../api/jsonrpc.md) — `usage.list`, `billing.plan`, `billing.video.pricing.*`.
- [개요](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
