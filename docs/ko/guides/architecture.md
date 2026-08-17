---
title: "아키텍처"
description: "Arona가 어떻게 구성되는지 — 워크스페이스 레이아웃, gateway를 통한 요청 경로, 라우팅, health probing, memory, 세션 및 의도적인 설계 트레이드오프."
---

# 아키텍처

이 페이지는 Arona가 어떻게 구조화되고 요청이 어떻게 흐르는지 안내합니다: 워크스페이스 레이아웃, 요청 경로, gateway와 라우터, health checking, memory 클라이언트, 세션과 알림, 마지막으로 설계가 수용하는 의도적인 한계와 트레이드오프. 실행 예제는 [quickstart](quickstart.md), 일상적인 런타임 관심사는 [operations](operations.md)을 참조하세요.

## 워크스페이스 레이아웃

저장소는 세 개의 패키지를 가진 Cargo 워크스페이스입니다:

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC,
│            # backends, migration, entities, providers, models, registry,
│            # orchestration
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

- `packages/core`는 라이브러리 크레이트(`_core`)입니다. 서버에 필요한 모든 것이 들어 있습니다: axum gateway(`gateway/`), 모델 라우터(`routing/`), backend 어댑터(`backends/`), billing(`billing/`), auth(`auth.rs`), memory 클라이언트(`memory/`), JSON-RPC 평면(`gateway/rpc.rs`), 스키마(`migration/`, `entity/`), 모델 메타데이터(`models/`, `providers/`, `registry/`), 모델 오케스트레이션(`orchestration/`).
- `packages/agent`는 GPU 노드에서 실행되고 `/ws/agent`로 다시 연결되는 `_agent` 바이너리를 빌드합니다([agent-cluster](agent-cluster.md) 참조).
- `packages/cli`는 install, deploy, serve, migrate, download 작업에 사용되는 `_cli` 바이너리를 빌드합니다.

이 저장소에는 더 이상 웹 대시보드가 없습니다: Vue 대시보드는 제거되었으며 이제 [shittim-chest](https://github.com/celestia-island/shittim-chest)(chest #291)에 있고, JSON-RPC 표면을 통해 Arona와 통신합니다. Arona 자체는 순수 backend입니다([개요](./README.md) 참조).

## 요청 경로

진입점은 `GatewayServer::app`(`packages/core/src/gateway/server.rs`)에서 조립되는 axum 라우터입니다. 라우트 테이블(server.rs:128-163)은 OpenAI 호환 REST 표면(`/v1/chat/completions`, `/v1/embeddings`, `/v1/models`, `/v1/health`), 비디오 생성, `/api/rpc` JSON-RPC 엔드포인트(POST + WebSocket 업그레이드), SSE 사이드카 `/api/rpc/events`, agent 제어 평면 `/ws/agent`, `/docs`의 Swagger UI, admin backend/alias 관리 엔드포인트를 다룹니다.

라우터는 작은 계층 스택으로 감싸져 있습니다(server.rs:158-162):

1. 핸들러별 추출기가 접근할 수 있도록 `Extension`으로서의 auth 관리자.
2. 인바운드 `X-Request-ID` 헤더를 재사용하거나 생성하여 핸들러와 로그에 노출하는 request-id 계층(`gateway/request_id.rs`).
3. 1MB 요청 본문 제한(`RequestBodyLimitLayer`).
4. 관대한 CORS 계층(`*` origin, `*` headers).

axum은 계층을 bottom-up으로 적용하므로 CORS 계층이 가장 바깥쪽입니다.

모든 `/v1/*` 핸들러는 동일한 골격을 거칩니다:

1. **Auth 추출** — 키 전용 엔드포인트(`/v1/chat/completions`, `/v1/embeddings`, video)는 `ApiKeyAuth`, API keys와 세션 JWTs 모두를 허용해야 하는 `GET /v1/models`는 `ApiKeyOrJwt`(`gateway/middleware.rs`). 추출기는 키/JWT를 사용자 이메일, 키 접두어, rate-limit 키(API key의 SHA-256 해시, 또는 회전하는 token이 윈도우를 재설정하지 않도록 JWT의 `u:<email>` 라벨), 선택적 project 범위로 해석합니다.
2. **Billing 게이트** — `enforce_billing_gates`(server.rs:492-539)는 사용자 tier의 월간 quota 또는 분당 rate limit이 초과되면 HTTP 429 + `Retry-After`로 요청을 거부합니다. DB 실패는 fail open입니다: billing은 최선 노력이며, chat 제공의 하드 의존성이 아닙니다.
3. **Memory 회상**(chat 경로) — memory 클라이언트가 구성되고 요청이 요청하면 관련 장기 memory가 시스템 섹션으로 주입됩니다(아래 [Memory client](#memory-client) 참조). 실패는 chat을 절대 차단하지 않습니다. 결과 상태는 `X-Arona-Memory` 헤더에 반영됩니다.
4. **대화 영속화** — 선택적 `conversation_id`는 소유권을 검사받고 사용자 턴은 전송 시점에 영속화됩니다.
5. **Gateway 디스패치** — 요청이 `Gateway`에 전달되며, backend를 해석하고, 컨텍스트를 정리하고, backend 트레이트를 호출합니다.
6. **사용량 기록** — 반환된(또는 스트림 터미널) 사용량이 키 접두어 아래 `UsageTracker`를 통해 기록되고 영속화됩니다.

`Gateway` 자체는 `AppState`에 `Arc<Gateway>`로 존재합니다 — 외부 뮤텍스가 없습니다. 내부 가변성(interior mutability) 덕분에 동시 chat/embeddings/stream 호출이 업스트림 HTTP 왕복에 걸쳐 락을 쥐지 않습니다(`gateway/mod.rs:29-53`).

## Gateway 및 라우터

`Gateway`(`packages/core/src/gateway/mod.rs`)는 다음을 소유합니다:

- **라우터 상태** — `tokio::sync::RwLock`으로 보호되는 backend 목록과 aliases. 읽기 측 해석은 await에 걸쳐 빌림합니다. 변경(등록/제거/alias)은 짧은 쓰기 락을 잡고 업스트림 호출에 걸쳐 보유하지 않습니다.
- **요청 카운터**(`AtomicU64`)와 `system.status` 및 health 엔드포인트가 사용하는 `start_time`.
- **배포 맵**(`model_id → backend name`) — agent 배포 모델용. `register_agent_backend`는 `agent-{model_id}`라는 이름의 `ExternalApiBackend`를 빌드하여 라우터에 삽입합니다. 동일한 모델 재등록은 이전 backend를 대체하고, `unregister_agent_backend`는 `stop_result` 프레임에서 제거합니다([agent-cluster](agent-cluster.md) 참조).

Backend 해석은 `Router`(`packages/core/src/routing/mod.rs`)에서 일어납니다:

1. **Alias 해석** — 구성된 alias가 대상으로 재작성됩니다.
2. **세션 고정** — `conversation_id`가 있으면 라우터가 대화를 처음 제공한 backend에 고정하는 약한 참조 맵을 확인합니다. 약한 참조는 backend가 등록되거나 진행 중인 동안만 맵을 살아 있게 하므로, 제거된 backend는 인덱스 드리프트 없이 사라집니다. 작동한 회로 차단기 또는 unhealthy 고정 backend는 새 선택으로 저하되며, 대화를 재바인딩합니다.
3. **후보 필터링** — 선택적 `provider` 힌트가 backend 이름/종류로 필터링합니다. 후보는 healthy *이고* 회로 차단기가 열려 있어야 하며 요청된 모델을 나열해야 합니다. 모델 id는 정확히 또는 `:latest` 접미사 관례를 통해 매칭됩니다(단순 `nomic-embed-text` 요청은 나열된 `nomic-embed-text:latest`와 매칭).
4. **최소 부하 선택** — 살아남은 후보를 히트 카운터로 정렬하고 가장 부하가 적은 것을 선택합니다. 대화 고정(있는 경우)은 동시에 기록됩니다.

backend가 호출되기 전에 `RequestPipeline::transform`(`packages/core/src/pipeline.rs:422+`)이 메시지 목록을 backend의 `max_context_length`로 정리합니다: 시스템 메시지는 전체가 유지되고, 비시스템 메시지는 맞는 동안 최신순으로 유지되며, 단일 초대형 메시지는 문자 단위로 하드 트렁케이트됩니다(휴리스틱 token 카운터는 token 정밀하게 트렁케이트할 수 없음). 호출은 `InferenceBackend` 트레이트를 통해 진행됩니다. 성공과 실패는 라우터의 backend별 회로 차단기(3회 실패, 30초 복구, half-open 호출 1회 — routing/mod.rs:57-64)에 다시 기록됩니다.

## Health checker 및 probing

`run_health_checks`(`packages/core/src/gateway/health_checker.rs`)는 시작 시 생성된(run.rs:135-150) 백그라운드 작업으로 실행되며 60초 간격마다 등록된 모든 backend를 probe합니다. 두 가지 세부 사항이 중요합니다:

- backend 목록은 비동기 fetcher 클로저를 통해 **매 라운드 새로 가져오므로**, 시작 후(예: admin API를 통해) 등록된 backend도 재시작 없이 선택됩니다.
- 첫 라운드는 첫 간격이 경과하기 전에 즉시 실행되므로, 프로세스가 시작되자마자 health 상태가 확립됩니다.

`probe_backend`는 단일 probe 코드 경로입니다. 일회성 **등록 시점 probe**에도 재사용됩니다: admin이 backend를 등록한 후(server.rs:688-693) 또는 영속화된 backend가 부팅 시 복원된 후(run.rs:122-127), fire-and-forget probe가 다음 60초 라운드까지 fail-closed로 남는 대신 약 1~2초 내에 backend를 healthy로 전환합니다. 이것이 갓 등록된 external backend의 모델 목록이 `GET /v1/models`에 거의 즉시 나타나는 이유입니다.

probe 자체는 가벼운 backend 호출입니다(예: external backend는 2초 probe 타임아웃으로 `/v1/models`를 침). 결과는 backend에 캐시되며, 라우팅은 캐시된 health가 `Healthy`인(그리고 회로 차단기가 열린) backend만 선택합니다.

## Memory 클라이언트

memory 클라이언트(`packages/core/src/memory/mod.rs`)는 서버 시작 시 환경 구성에서 생성됩니다(server.rs:95-97): `ARONA_MEMORY_URL`과 `ARONA_MEMORY_TOKEN`이 설정되면 chat 요청이 JSON-RPC WebSocket을 통해 entelecheia Philia memory 서비스를 쿼리하고, `recall_and_inject`가 관련 memory를 시스템 섹션(`## Relevant Long-Term Memories`)으로 나가는 컨텍스트 앞에 추가합니다. 완료된 턴은 `writeback_dialogue`를 통해 에피소드로 다시 기록됩니다 — assistant 응답이 영속화된 후 생성되는 fire-and-forget 작업이므로 memory 실패가 chat 응답 경로를 절대 차단하거나 느리게 하지 않습니다. `ARONA_MEMORY_WRITEBACK`(기본 켜짐)이 writeback을 토글합니다. 전체 그림은 [memory-gateway](memory-gateway.md)를 참조하세요.

모든 chat 응답은 세 가지 상태 중 하나의 `X-Arona-Memory` 헤더를 가집니다: `enabled`(회상이 실행되어 주입됨), `disabled`(구성되지 않았거나 요청이 `memory: false`를 전달), `offline`(구성되었지만 서비스에 도달할 수 없음).

## 세션 및 알림

`AppState`는 plana `SessionManager`(`state.sessions`)를 보유합니다. `chat.send` 같은 스트리밍 RPC는 세션 id를 생성하고(`gateway/rpc.rs:1701`) 알림 — `chat.stream` tokens, `models.progress` 배포 진행, `realtime.event` — 을 해당 세션의 채널에 푸시합니다. 클라이언트는 SSE 사이드카 `GET /api/rpc/events?session=<id>`(server.rs:191-200)에서 이를 소비합니다. 알림 형식과 사전 구독 윈도우 주의 사항은 [events](../api/events.md)를 참조하세요.

세션 채널은 요청/응답 RPC 호출에도 사용됩니다: 클라이언트가 `POST /api/rpc`에 `x-session-id` 헤더를 보내면 서버가 결과를 해당 세션 채널에도 푸시합니다(server.rs:184-188, rpc.rs:128-144). 따라서 클라이언트는 RPC 응답을 이미 열린 SSE 스트림에 다중화할 수 있습니다.

## 한계 및 설계 트레이드오프

설계는 의도적으로 많은 한계를 수용합니다. 프로덕션 사용 전에 알아 두세요:

- **1MB 요청 본문 제한** — 더 큰 본문은 계층에서 거부됩니다. 큰 컨텍스트 호출이 필요하면 이것이 가장 먼저 조정할 것입니다.
- **CORS `*`** — gateway는 어디서든 교차 origin 호출에 응답합니다. API에는 괜찮지만, 신뢰하는 클라이언트 너머로 노출한다면 자체 CORS 정책을 강제하는 proxy를 앞에 두세요.
- **Fail-open billing** — DB를 사용할 수 없을 때 quota/rate-limit 집행이 요청 허용으로 저하됩니다. Billing은 계량이지 접근 제어가 아닙니다.
- **SSE 스트림에 전체 타임아웃 없음** — 스트리밍 호출에는 총 데드라인이 없습니다(긴 생성은 합법). 중단 감지는 읽기당 120초 유휴 타임아웃(`backends/external.rs:24-31`)에 의존합니다. 비스트리밍 호출은 600초 전체 데드라인을 받습니다.
- **토크나이저 추정 사용량** — 사용량을 절대 보고하지 않는 backend(ollama, ws_engine)는 로컬 CJK 인식 토크나이저 추정으로 청구되며, 그대로 기록됩니다([billing-usage](billing-usage.md) 참조).
- **인메모리 rate-limit 윈도우 및 폐기** — 키별 슬라이딩 윈도우와 폐기된 키 집합이 프로세스 메모리에 있습니다(`auth.rs`). 따라서 재시작 시 재설정됩니다. auth 수준 리미터는 윈도우당 키당 요청을 제한하고, billing-tier 리미터는 DB 기반입니다([auth-security](auth-security.md) 및 [billing-usage](billing-usage.md) 참조).
- **`/ws/agent`는 인증되지 않음** — agent 제어 평면은 register/heartbeat 프로토콜을 말하는 모든 WebSocket을 받아들입니다. 통제하는 네트워크에서만 안전합니다.
- **gateway에 TLS 없음** — 서버는 일반 HTTP에 바인딩합니다. 네트워크 경계를 넘는 모든 배포에서는 앞에서 TLS를 종료(리버스 프록시)하세요. [deployment](deployment.md) 참조.

정상 측면에서 서버는 graceful shutdown을 수행합니다: Ctrl+C 및 SIGTERM 핸들러를 설치하고, "draining connections"을 로그하며, 프로세스가 종료되기 전에 진행 중인 요청이 끝나게 합니다(`gateway/run.rs:14-38`, run.rs:154-159의 `with_graceful_shutdown` 배선).

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table); gateway/mod.rs:29-53; routing/mod.rs:102-200; health_checker.rs:14-37; run.rs:135-159; memory/mod.rs:207-241; rpc.rs:128-144,1701 -->
