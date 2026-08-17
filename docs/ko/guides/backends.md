---
title: "Backends"
description: "Backend 유형(external, ollama, engine, minimax-cloud, evernight 브리지), URL 의미, health probing, 모델 발견, aliases 및 라우팅."
---

# Backends

**backend**는 모델 트래픽을 제공하는 업스트림입니다. Arona는 OpenAI 호환 요청(`/v1/chat/completions`, `/v1/embeddings`, 모델 목록, video 작업)을 등록된 backend 중 하나로 라우팅하고, 모든 요청을 계량하며, 각 backend의 health와 모델 인벤토리를 최신 상태로 유지합니다.

Backend는 관리자가 `POST /api/admin/backends`([Admin HTTP API](../api/admin-http.md) 참조)를 통해 등록하고, `backend_configs` 테이블에 영속화되며, 시작 시 자동으로 복원됩니다. 각 등록은 `name`, `type`, `url`, 선택적 `api_key`, 선택적 정적 `models` 목록을 포함합니다. 영속화된 backend는 재시작 후에도 유지됩니다. 복원된 backend는 fail-closed로 시작하며 즉시 probe됩니다.

## Backend 유형

| `type` | 전송 | 프로토콜 | 용도 |
| --- | --- | --- | --- |
| `external` | HTTP(S) | OpenAI 호환 REST | 모든 chat/embeddings API(클라우드 또는 자체 호스팅) |
| `ollama` | HTTP(S) | Ollama 네이티브 API(`/api/chat`, `/api/embed`, `/api/tags`) | 로컬 또는 원격 Ollama 서버. URL만으로 구성 |
| `engine` | `ws://` / `wss://` | CEP(Celestia Engine Protocol), WebSocket + JSON-RPC | CEP 교환 표준(`plana::engine`)을 말하는 모든 엔진 |
| `minimax-cloud` | HTTPS | MiniMax H3 작업 스타일 API(submit + poll) | 클라우드 비디오 생성 |
| `evernight://<node>/<service>` | 브리지 URL | 로컬 evernight agent를 통해 로컬 TCP 포워드로 해석 | Evernight mesh를 통해서만 도달 가능한 산업/엣지 서비스 |
| `agent-{model}` | HTTP | OpenAI 호환(external) | GPU agent가 모델을 배포할 때 자동 등록 |

### `external` — 모든 OpenAI 호환 HTTP API

범용 backend: OpenAI REST 형태를 말하는 모든 서버에 대한 chat completions(스트리밍 및 비스트리밍)와 embeddings. 기본 `url`, `api_key`(선택), 선택적 정적 `models` 목록으로 구성합니다:

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

정적 `models` 목록은 권위가 있습니다. Probe 시 발견된 모델보다 앞서 병합됩니다([모델 발견](#model-discovery) 참조).

### `ollama` — URL만으로 구성

Ollama backend는 URL만으로 구성됩니다 — API key도 모델 목록도 필요 없습니다. Ollama 네이티브 프로토콜을 사용합니다: chat은 `/api/chat`, embeddings는 `/api/embed`, health probing과 모델 발견은 `/api/tags`.

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — WebSocket을 통한 CEP

`engine` backend는 `ws://`(또는 `wss://`)를 노출하는 엔진에 연결하며 **Celestia Engine Protocol**(CEP)을 사용합니다: `plana::engine`에 정의된 WebSocket + JSON-RPC 2.0 교환 표준. 핸드셰이크 → 메서드 → 스트리밍 알림 흐름을 구현하는 모든 언어의 엔진은 Arona에서 어댑터 코드 없이 일급 backend로 등록됩니다. 와이어 메서드: `Engine.Handshake`(첫 메시지; 식별자 + 기능), `Engine.Chat`, `Engine.ChatStart`(스트리밍; 청크는 `Engine.ChatChunk` 알림으로 도착), `Engine.Embeddings`, `Engine.Models`. 연결은 첫 사용 시 지연(lazily) 설정되며 오류 시 끊어집니다. 다음 호출에서 재연결하고 다시 핸드셰이크합니다.

### `minimax-cloud` — 작업 스타일 비디오 생성

클라우드 비디오 backend는 MiniMax H3 Open Platform API를 구동합니다: 생성 작업 제출, 완료 poll, 결과에서 아티팩트 URL 읽기. 이것은 제거된 ComfyUI backend를 대체한 것입니다(아래 참조). Video 작업은 `/v1/video/generations` 또는 `video.*` RPC 메서드로 제출되며, `video.progress` / `video.done` / `video.failed` 알림으로 진행 상황이 전달됩니다([Realtime & Video](realtime-video.md) 참조).

### `evernight://` 브리지 URL

`evernight://<node>/<service>` 형태의 backend URL은 **직접** 접촉되지 않습니다. 로컬 호스트 evernight agent가(agent의 WebSocket 엔드포인트를 통한 `Bridge.Connect` JSON-RPC 호출) 로컬 TCP 포워드로 해석하며, backend는 하드코딩된 원격 주소 대신 `http://127.0.0.1:<local_port>`에 대해 실행됩니다. 이것은 단일 패널 아키텍처입니다: Arona 패널은 구성에 원격 주소를 박아 넣지 않고도 mesh를 통해 다른 노드의 서비스(CEP engines, scepter, ...)에 도달합니다.

Keepalive 작업이 15초마다 터널을 probe합니다. 원격 측이 재시작되고 터널이 새 로컬 포트에 재설정되면, 영향을 받은 backend는 새 URL로 **투명하게 재구축**됩니다 — 영속화된 구성은 `evernight://` URL을 유지하므로 재시작 시 다시 해석됩니다. `engine` backend의 경우 해석된 `http://127.0.0.1:<port>` 포워드가 WebSocket 전송을 위해 `ws://`로 변환됩니다.

### Agent 배포 모델 자동 등록

GPU agent가 모델 배포를 완료하면 gateway가 `agent-{model_id}`라는 이름의 backend(`http://{agent host}:{port}` 위의 `ExternalApiBackend`)를 등록하여 모델이 즉시 라우팅 가능해집니다. 배포를 중지하면 다시 등록이 해제됩니다. 전체 배포 수명 주기는 [Agent Cluster](agent-cluster.md)를 참조하세요.

### `comfyui`는 거부됩니다

`comfyui` backend 유형은 `comfyui backend removed` 오류로 명시적으로 거부됩니다: ComfyUI backend는 모델 플랫폼 통합 중에 제거되었으며, 비디오 생성은 이제 `minimax-cloud`를 통해 실행됩니다. `comfyui` backend를 등록하면 HTTP 400이 반환됩니다.

## URL 의미

구성된 기본 URL이 실제 엔드포인트에 매핑되는 방식은 URL에 경로 구성 요소가 있는지 여부에 따라 결정됩니다:

- **루트 스타일 기본 URL** — 경로가 비어 있거나 `/`인 URL은 호스트 루트로 처리되며 OpenAI `/v1` 관례를 유지합니다: `{base}/v1/chat/completions`, `{base}/v1/models`. 예: `http://192.0.2.20:8429`, `https://api.deepseek.com`.
- **경로 스타일 기본 URL** — 비어 있지 않은 경로가 있는 URL은 서버가 실제로 제공하는 전체 API 접두어로 처리되며 엔드포인트가 직접 추가됩니다: `{base}/chat/completions`, `{base}/models`. 이것은 `/v1` 관례 밖에 있는 OpenAI 호환 서버에 필요합니다. Zhipu GLM 코딩 플랜이 대표적인 예입니다: API는 `https://open.bigmodel.cn/api/coding/paas/v4`에 있으며, chat은 `{base}/chat/completions`에 직접 있고 `/models` 엔드포인트는 **전혀 없습니다** — 표준 `/api/paas/v4` 루트는 코딩 플랜 키에 대해 잔액 오류를 반환합니다.
- 구성된 기본 URL의 **끝 슬래시**는 정규화되어 제거되므로 조인이 이중 슬래시를 만들지 않습니다.

## Probing 및 health

백그라운드 health checker가 등록된 모든 backend를 **60초**마다 probe합니다. Backend 목록은 각 라운드에서 새로 가져오므로, 시작 후 등록된 backend도 재시작 없이 선택됩니다. 또한 각 admin 등록은 즉시 probe를 실행하여 backend가 다음 checker 라운드를 기다리는 대신 약 1~2초 내에 healthy로 전환됩니다.

- **External backend**는 **2초 타임아웃**으로 `GET {base}/models`(루트 스타일 기본 URL의 경우 `{base}/v1/models`)를 probe합니다. **404는 허용됩니다**: 일부 서버는 chat을 구현하지만 모델 목록을 노출하지 않으므로(GLM 코딩 플랜에는 `/models` 엔드포인트가 없음), 404는 backend를 healthy로 표시하고 admin이 구성한 `models` 목록이 라우팅 소스가 됩니다. 타임아웃, 네트워크 실패 및 기타 비-2xx 응답은 backend를 unhealthy로 표시합니다.
- **Ollama backend**는 동일한 2초 타임아웃으로 `/api/tags`를 probe합니다.
- Backend는 첫 번째 성공적인 probe까지 **fail-closed** — "not probed yet"으로 보고됨 — 상태로 시작하므로, 갓 등록된(또는 복원된) backend는 검증되기 전에 트래픽을 받지 않습니다.

Health 상태는 backend별로 캐시되며 라우터가 모든 요청에서 참조합니다. Unhealthy backend는 후보 선택에서 제외됩니다([라우팅](#routing) 참조).

## 모델 발견

Backend는 제공하는 모델 id를 광고하며, 라우터는 요청을 그 광고와 매칭합니다:

- **External** backend는 probe 응답에서 파싱한 모델(`data`와 `models` 배열 모두 허용)을 admin이 구성한 정적 목록과 병합하여 광고합니다 — 정적 id는 순서와 우선순위를 유지하고, 동적 id는 중복 제거 후 추가됩니다. 서버에 models 엔드포인트가 없으면 정적 목록만이 라우팅 소스입니다.
- **Ollama** backend는 `/api/tags`가 반환한 태그를 광고합니다.
- **Agent 배포** 모델은 정확히 배포된 `model_id`를 광고합니다.

공개 표면은 `GET /v1/models`(인증 필요)이며, 모든 healthy backend의 라우팅 가능한 모델을 나열합니다([OpenAI 호환 REST API](../api/openai-rest.md) 참조).

## Aliases 및 이름 정규화

Aliases는 요청된 모델 id를 대상 id에 매핑합니다. 라우팅에서 alias가 먼저 해석되므로, alias에 대한 요청은 대상을 광고하는 backend가 제공합니다:

```json
{ "alias": "fast-chat", "target": "deepseek-chat" }
```

Aliases는 `/api/admin/aliases` admin 엔드포인트를 통해 관리되며 즉시 적용됩니다.

이름 매칭은 Ollama 스타일 태그도 정규화합니다: `nomic-embed-text:latest`를 나열하는 backend는 `nomic-embed-text`에 대한 단순 요청과 매칭되므로, embedding/chat 요청은 `:latest` 접미사 관리를 하지 않고 해석됩니다. 명시적 태그(`qwen3:0.6b`)는 정확히 해당 태그와만 매칭됩니다.

## 라우팅

모든 요청은 하나의 backend를 선택하는 라우터를 통해 해석됩니다:

1. **Alias 해석** — 요청된 모델 id가 alias 테이블을 통해 매핑됩니다(있는 경우).
2. **Provider 힌트** — 선택적 `provider` 필드가 backend 이름(또는 종류 이름, 예: video backend의 `cloud`)으로 후보를 필터링합니다.
3. **Healthy 후보만** — backend가 `Healthy`로 보고 *되고* 회로 차단기(circuit breaker)를 통과해야(연속 3회 실패 시 30초 동안 차단기 개방, half-open 테스트 호출 1회) 선택될 수 있습니다.
4. **최소 카운트 선택** — 모델을 제공하는 후보를 backend별 요청 카운터로 정렬하고 가장 부하가 적은 것을 선택합니다. 이렇게 하면 동일한 모델을 제공하는 healthy backend 간에 부하가 분산됩니다.
5. **세션 고정** — 요청에 `conversation_id`가 있으면 대화가 처음 사용한 backend에 고정됩니다. 고정은 `Weak` 참조 맵에 있으므로, 제거된 backend는 인덱스 드리프트 없이 맵에서 사라집니다. 고정은 최선 노력(best-effort)입니다: 대화의 턴에서 동일한 backend를 재사용하면 업스트림이 대화별 런타임 상태(웜 컨텍스트, KV 캐시)를 재사용할 수 있습니다. 고정된 backend가 unhealthy가 되면(또는 고정이 제거된 backend와 함께 소멸하면), 라우터는 새 최소 카운트 선택으로 대체하고 대화를 **재바인딩**합니다.

모델을 제공하는 healthy backend가 없으면 라우팅이 실패합니다: 알 수 없는 모델은 `model not found`(HTTP 404), 알려졌지만 도달할 수 없는 모델은 `all
backends unhealthy`를 생성하며, 이는 500 내부 서버 오류로 표시됩니다. HTTP 502는 *도달 가능한* 업스트림이 보고한 실패(선택 후 비-2xx 업스트림 응답 및 전송 실패)를 위해 예약됩니다. 전체 오류 매핑은 [Operations](operations.md)를 참조하세요.

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
