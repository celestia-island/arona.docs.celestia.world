---
title: "인증 및 보안"
description: "JWT 세션, API keys, 세 가지 admin 게이트, 비밀번호 정책, 이중 트랙 rate limiting 및 보안 모델."
---

# 인증 및 보안

Arona는 두 트랙으로 호출자를 인증합니다: 대화형 클라이언트(chat + 관리 UI, RPC 호출)를 위한 **JWT 세션 token**과 프로그래매틱 OpenAI 호환 트래픽을 위한 **API keys**(`arona-…`). 별도의 admin token이 관리 표면을 보호합니다. 이 페이지는 메커니즘, 보안 모델, 보안 감사에서 알려진 위험이 낮은 잔여 항목을 문서화합니다.

## JWT 세션

세션은 `kirino_session` token 관리자가 발급하는 JWT access/refresh token 쌍을 사용합니다:

- **Access token TTL: 900초(15분).**
- **Refresh token TTL: 604,800초(7일).**

Access token은 JSON-RPC 평면(`/api/rpc`)과 `GET /v1/models`를 인증합니다. SSE 사이드카(`/api/rpc/events`)는 세션 id로 키가 지정되며, 이는 bearer 자격 증명이 아니라 인증된 RPC 호출 중에 발급된 기능입니다. `/v1/chat/completions`, `/v1/embeddings`, `/v1/video/*` 엔드포인트는 **API key**가 필요합니다(여기서는 JWT가 허용되지 않음). Access token은 수명이 짧으므로 도난된 token은 잠시 동안만 사용할 수 있습니다. Refresh token은 `auth.refresh`를 통해 새 쌍으로 교환됩니다.

Refresh는 **token 패밀리 로테이션**을 사용합니다: refresh token을 사용하면 무효화되고 새 것이 발급되며, 사용된 refresh token을 재사용하면 패밀리 전체가 폐기됩니다 — `auth.refresh`는 `AUTH_ERROR`와 `Refresh token reused` 메시지로 응답합니다(기본 오류는 `TokenReused`, "refresh token has been reused — token family revoked"), 계정은 다시 로그인해야 합니다. 패밀리 폐기는 **인메모리**(`revoked_families` 집합)입니다. 서버 재시작 시 지워지므로, 보호는 재시작에 걸쳐 최선 노력입니다(사용자별 세션 상태는 재시작 후에도 유지되지 않습니다).

서명 시크릿은 `JWT_SECRET` 환경 변수에서 옵니다. `MOCK_MODE=1` 밖에서 `JWT_SECRET`이 설정되지 않았거나 내장 개발 시크릿과 여전히 같으면 서버는 **시작을 거부**하므로, 프로덕션 인스턴스가 공개 상수로 서명된 token을 실수로 제공할 수 없습니다. 강력하고 무작위한 시크릿을 사용하고 절대 커밋하지 마세요.

## API keys

API keys는 OpenAI 호환 표면의 머신 자격 증명입니다:

- **형식:** `arona-{uuid}`.
- **저장:** 키의 **SHA-256 해시**만 `api_keys` 테이블에 저장됩니다 — 평문은 `keys.create` 응답에서 정확히 한 번 반환되며, 이후에는 복구할 수 없습니다.
- **키 접두어:** 첫 8자(`key_prefix`)는 표시 및 사용량 귀속을 위해 평문으로 저장됩니다. UI는 `arona-XXXX…abcd` 같은 마스킹된 형태를 보여줍니다.
- **폐기:** 키 조회는 `api_keys.is_active = TRUE`를 조인하므로, 폐기된 키는 즉시 검증을 중지합니다 — 기다릴 캐시 TTL이 없습니다.

## Admin 등급

세 가지 별개의 admin 게이트가 있으며 각각 자체 자격 증명이 있습니다:

1. **`/api/admin/*` 라우트** — backend 및 alias 관리(`POST/GET/DELETE /api/admin/backends`, `POST/GET/DELETE /api/admin/aliases`)는 `Authorization: Bearer ARONA_ADMIN_TOKEN` 헤더가 필요합니다. `ARONA_ADMIN_TOKEN`이 설정되지 않으면 `check_admin`은 항상 실패하고 모든 admin 라우트는 **401 "Admin access required"**를 반환합니다 — 관리 표면 전체가 열리는 대신 비활성화됩니다.

2. **`agents.*` 및 `engine.invoke` RPC 메서드** — agent 클러스터와 엔진 제어 평면은 `users.is_admin =
   true`인 계정의 JWT가 필요합니다. 인증된 비관리자는 구현 정의 코드 **-32007(`ADMIN_REQUIRED`)**과 메서드별 힌트(예: `agents.deploy starts model deployments on GPU nodes`)로 거부됩니다. **인증되지 않은** 호출자는 표준 **-32005(`AUTH_ERROR`)**를 받으므로 서버가 메서드가 특권적임을 드러내지 않습니다.

3. **`billing.plan.set` 및 `billing.video.pricing.set` RPC 메서드** — billing 변경은 admin HTTP 라우트와 동일한 Bearer `ARONA_ADMIN_TOKEN`이 필요합니다. 없으면 `AUTH_ERROR` "Admin access required"를 반환합니다.

**첫 번째 등록 사용자가 admin이 됩니다**(`users.is_admin = true`). 이후의 모든 등록은 일반 사용자이며, `ARONA_REGISTRATION_OPEN`이 truthy 값으로 설정된 동안에만 가입이 열립니다.

## 비밀번호 정책

비밀번호는 **두** 규칙을 모두 충족해야 합니다(등록 시 및 모든 비밀번호 변경 경로에서 적용):

- 최소 **8자**, 그리고
- **4가지 문자 범주 중 최소 3가지**: 대문자, 소문자, 숫자, 특수 문자.

## Rate limiting

Rate limiting은 두 개의 독립적인 트랙에서 실행됩니다. 둘 중 하나가 요청을 **429**로 거부할 수 있습니다:

### 1. 인메모리 슬라이딩 윈도우(신원별)

인증된 모든 `/v1` 요청은 호출자의 신원을 키로 하는 인메모리 슬라이딩 윈도우 리미터를 통과합니다:

- **API-key 호출**은 키의 **SHA-256 해시**로 키가 지정됩니다.
- **JWT 호출**은 `u:<email>`로 키가 지정됩니다 — JWT는 15분마다 회전하므로, token 인스턴스로 윈도우를 키로 하면 매 refresh마다 조용히 재설정됩니다.

기본 예산은 **분당 60회 요청**이며, `ARONA_API_RATE_LIMIT_RPM`으로 재정의할 수 있습니다(병렬 LLM 호출을 많이 펼치는 agent 파이프라인의 경우 더 높게 설정). **0으로 설정하면 모든 요청을 차단합니다**.

### 2. Tier rate limit(키별, 데이터베이스에서)

Billing tiers는 키별 `rate_limit_rpm`을 가집니다. 검사는 **지난 60초** 동안 키 접두어의 `usage_records` 행 수를 셉니다(사용량은 각 응답 후 영속화되므로 윈도우는 최대 하나의 진행 중 요청만큼 지연됩니다. DB 실패는 fail open). 시드된 **free tier는 10 RPM**입니다. Pro/enterprise tier는 상한을 높입니다. 월간 quota 집행은 동일한 거부 경로를 공유합니다.

### 로그인 rate limiting

로그인 엔드포인트에서 자격 증명 추측이 제한됩니다: **이메일당 5분 윈도우에 5회 실패 시도**, **IP당 5분 윈도우에 20회**, 각각 15분 잠금이 이어집니다.

### `Retry-After`

모든 429 응답은 `Retry-After` 헤더를 가지므로 OpenAI 호환 클라이언트가 엔드포인트를 두드리는 대신 백오프합니다: quota 거부는 **월말까지의 초**로 설정하고, rate-limit 거부는 **60**으로 설정합니다. Quota 모델은 [Billing & Usage](billing-usage.md)를 참조하세요.

## 보안 모델 참고 사항

- **CORS는 모든 origin을 허용합니다**(`allow_origin(Any)`) — Arona는 많은 일방 및 타사 클라이언트가 소비하는 backend입니다. 배포에서 origin을 제한해야 한다면 CORS를 강제하는 리버스 프록시를 앞에 두세요.
- **요청 본문은 1MB로 제한됩니다**(`RequestBodyLimitLayer`) — gateway의 메모리 사용을 제한합니다.
- **Gateway는 TLS를 종료하지 않습니다** — 일반 HTTP로 수신합니다. HTTPS를 종료하는 리버스 프록시 뒤에 두세요([배포](deployment.md) 참조).
- **시크릿은 환경에서만 옵니다**: `ARONA_ADMIN_TOKEN`과 `JWT_SECRET`은 env vars에서 읽히며, 저장소에 커밋되지 않는 강력한 무작위 값이어야 합니다.
- 기본 서버 바인딩 주소는 `0.0.0.0`입니다. 네트워크 계층에서 노출을 제한하세요.

## 알려진 위험 낮은 잔여 항목(감사에서)

다음은 있는 그대로 문서화됩니다. 의도적이거나 현재 수용된 것이지만, 신뢰할 수 있는 네트워크 너머로 인스턴스를 노출할 때 알아 둘 가치가 있습니다:

- **`providers.list`는 공개**인 반면, `providers.add` / `providers.update` / `providers.remove` / `providers.test`는 JWT가 필요합니다. 공개 읽기 경로는 provider 카탈로그를 드러내지만 비밀은 아닙니다.
- **`/ws/agent`는 인증되지 않은 제어 평면입니다**: GPU agents가 자격 증명 없이 연결하여 자체 등록합니다(`register` / `heartbeat` / command-result 프레임). WebSocket 포트에 도달할 수 있는 사람은 누구나 가짜 agent를 등록할 수 있습니다. 운영상의 트레이드오프는 [Agent Cluster](agent-cluster.md)를 참조하세요.
- **`memory.delete`는 소유권 검사가 없는 JWT 전용입니다**: 인증된 사용자라면 누구나 `node_id`로 memory 노드를 삭제할 수 있습니다. Memory를 삭제하려면 로그인이 필요하지만 노드를 소유할 필요는 없습니다.

<!-- src: packages/core/src/auth.rs:22-23,504-534,553-557,630-649,694-705 (tokens, keys, rate limit) / packages/core/src/gateway/server.rs:577-600 (admin gate) / packages/core/src/gateway/rpc.rs:39,90-118 (admin RPC gate) -->
<!-- note: over the RPC surface, auth.refresh reports a reused refresh token as `AUTH_ERROR` "Refresh token reused" (rpc.rs:463-481); the longer Display text "refresh token has been reused — token family revoked" is the AuthError variant itself (auth.rs:726). -->
<!-- note: /v1/chat/completions, /v1/embeddings and /v1/video/* accept API keys only (ApiKeyAuth); only GET /v1/models accepts a JWT too (ApiKeyOrJwt, middleware.rs:88-100). -->
