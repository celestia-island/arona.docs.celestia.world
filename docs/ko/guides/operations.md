---
title: "운영"
description: "실행 중인 arona-server를 위한 health 엔드포인트, RUST_LOG 추적, 업스트림 타임아웃, 오류 매핑 및 문제 해결."
---

# 운영

이 페이지는 `arona-server serve`를 실행하는 운영자를 위한 것입니다. Probe하는 health 엔드포인트, grep할 가치가 있는 로그 라인, 업스트림 backend에 적용되는 타임아웃 모델, backend 실패가 HTTP 오류에 매핑되는 방식, 사람들을 넘어뜨리는 운영상의 함정을 다룹니다. 배포 자체는 [배포 가이드](./deployment.md)에서 다룹니다.

## Health 매트릭스

세 health 엔드포인트 모두 비인증이며 프로세스가 서빙 중일 때마다 `200 OK`를 반환합니다 — liveness/readiness 구분은 없습니다:

| 엔드포인트 | 응답 |
| --- | --- |
| `/healthz`, `/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | 위와 동일한 상세 본문 |
| `/api/health` | plana `HealthResponse`: `status`, `version`(`CARGO_PKG_VERSION`), `kind`(`Dev`), `uptime`(초), `network`(transport / region / asn), `build_hash`(`BUILD_HASH`), `engine_version`(`"0.1.0"`) |

`/healthz`와 `/readyz`는 같은 핸들러의 별칭이고, `/v1/health`도 이를 공유하므로, Kubernetes 스타일 probe와 OpenAI 호환 health 라우트는 상호 교환 가능합니다. `/api/health`는 uptime, network, engine version을 추가합니다. 로드 밸런서와 슈퍼바이저에는 `/readyz`를, 더 풍부한 페이로드가 필요할 때는 `/api/health`를 사용하세요.

## 로깅

서버는 `tracing`을 통해 로그하며 표준 `RUST_LOG` 변수로 필터링됩니다(`RUST_LOG=info`가 일반적인 설정, `RUST_LOG=debug`는 probe 트래픽을 드러냄). 알 가치가 있는 이벤트, 대략 빈도순:

| 로그 라인 | 수준 | 알려주는 내용 |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | Chat 요청당 하나씩, `key_prefix`, `model`, `stream`, `request_id` 포함 — 가장 단순한 요청별 감사 추적. |
| `request completed` | info | `logging_middleware` 헬퍼가 모든 **비스트리밍** `/v1/chat/completions` 및 `/v1/embeddings` 응답 후 기록: `method`, `path`, `status`, `latency_ms`, `trace_id`. (스트리밍 chat은 시작 시 `chat completions SSE request`를 대신 기록합니다.) |
| `usage recorded` / `usage persisted` | info | Usage 행이 기록되고(인메모리, tokens/cost 포함) `usage_records` 테이블에 쓰였습니다. |
| `external probe: sending` / `external probe: returned` | debug | External backend의 `/v1/models` health probe. `matched`는 probe가 2초 probe 타임아웃 내에 완료되었는지 여부를 말합니다. |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | Billing 게이트가 거부한 `/v1/*` 요청 — 클라이언트가 429와 `Retry-After`를 받았습니다. |
| `rpc billing gate rejected: monthly quota exceeded` | warn | JWT 인증 메서드용 RPC 측 quota 게이트(전체 사용자 윈도우. JSON-RPC 오류 응답). |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | 시작 시 복원: admin이 등록한 backend와 agent 노드가 데이터베이스에서 로드되어 다시 라우팅 가능해졌습니다. |
| `Shutdown signal received, draining connections…` | info | 정상 종료가 시작되었습니다(SIGINT/SIGTERM). |

## 타임아웃 모델

타임아웃은 external backend용 업스트림 클라이언트에서 집행됩니다(`packages/core/src/backends/external.rs`):

| 타임아웃 | 값 | 적용 대상 |
| --- | --- | --- |
| 연결(Connect) | 10초 | 업스트림 TCP/TLS 연결 설정. |
| 읽기 유휴(Read idle) | 읽기당 120초 | 모든 업스트림 호출. 수신된 각 바이트가 클록을 재설정하므로, healthy하지만 느린 스트림은 절대 잘리지 않습니다. |
| 비스트리밍 전체 | 600초 | 비스트리밍 chat/embeddings 호출 — 느리지만 살아있는 업스트림이 요청을 영원히 붙잡을 수 없습니다. |
| 스트리밍(SSE) | 없음 | 스트리밍 호출에는 **전체 데드라인이 없습니다**. 긴 생성은 합법이며, 중단 감지는 읽기 유휴 타임아웃에 의존합니다. |
| Health probe | 2초 | `/v1/models` probe. |

## 오류 매핑

Backend 실패는 chat/embeddings 핸들러에서 HTTP 상태로 매핑됩니다(`packages/core/src/gateway/server.rs`):

| 조건 | HTTP | `type` / `code` | 메시지 |
| --- | --- | --- | --- |
| 업스트림 비-2xx 상태(`UpstreamStatus`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| 업스트림 전송 실패(`RequestFailed`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | 전송 오류 문자열 |
| 기타 모든 backend 오류 | **500** | `server_error` / `backend_error` | 오류 문자열 |
| 모델용 backend 없음(`NoBackend`) | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| 잘못된 API key(`Unauthorized`) | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| Rate limit(`RateLimited`) | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

설계 의도: 호출자는 "provider가 거부했거나 실패"(502)를 "gateway 자체가 고장"(500)과 구분할 수 있습니다. 모든 오류 본문은 동일한 OpenAI 스타일 형태를 가집니다 — `{"error":{"message":...,"type":...,"code":...}}`(`json_error_response`). Billing 게이트 429는 추가로 `Retry-After` 헤더를 가지며 각각 `quota_error`/`quota_exceeded`(quota)와 `rate_limit_error`/`rate_limit_exceeded`(tier rate limit)를 사용합니다.

## 문제 해결

### 새로 등록된 backend는 probe될 때까지 fail-closed 상태 유지

External backend는 알 수 없는 health 상태로 시작하며 `"<url> not probed yet"`을 보고합니다. (a) health checker의 첫 라운드가 실행되면 — 시작 시 즉시, 이후 60초마다 — 또는 (b) 등록 또는 복원 시 시작된 fire-and-forget probe가 성공하면, 보통 ~1~2초 내에 healthy로 전환됩니다. 그때까지 해당 backend로 라우팅된 요청은 설계상 fail-closed입니다.

### 일부 backend에서는 probe의 `/models` 404가 정상

External probe는 `GET {base}/v1/models`(경로 접두어가 있는 기본 URL은 `{base}/models`)를 칩니다. 일부 OpenAI 호환 서버는 chat을 구현하지만 모델 목록을 노출하지 않습니다 — Zhipu GLM 코딩 플랜 엔드포인트가 그 예입니다. **404는 허용됩니다**: backend는 healthy로 표시되고 admin이 구성한 models 목록이 라우팅의 권위로 유지됩니다. 진짜 실패한 probe(타임아웃, 네트워크 오류, 기타 비-2xx)만 backend를 unhealthy로 표시합니다.

### 아무것도 생성하지 않는 SSE 스트림은 청구되지 않습니다

스트리밍 응답은 스트림이 텍스트를 생성했거나 터미널 사용량을 실었을 때만 사용량으로 기록됩니다. 둘 다 없이 끝난 스트림은 전혀 기록되지 않습니다. 일치하는 `usage recorded` 라인이 없는 요청이 보이면 스트림이 실제로 콘텐츠를 생성했는지 확인하세요.

### 버전 보고

Health 본문의 `version`은 `CARGO_PKG_VERSION`이고, `build_hash`는 `packages/core/build.rs`가 내보내는 빌드 시점 `BUILD_HASH` 값입니다. 노드 간 `build_hash`를 비교하여 모두 동일한 아티팩트를 실행하는지 확인하세요.

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
