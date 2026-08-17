---
title: "Admin HTTP API"
description: "Bearer-token admin 표면 — /api/admin/*에서 backend 등록/목록/제거 및 모델 aliases 관리."
---

# Admin HTTP API

`/api/admin/*` 표면은 gateway의 **backends**(업스트림 모델 provider)와 **aliases**(모델 이름 → 모델 id 리다이렉션)를 관리합니다. RPC 관리 평면의 HTTP 대응물이며([JSON-RPC API](./jsonrpc.md) 참조), 주로 운영자와 admin UI가 사용합니다.

## 인증

모든 `/api/admin/*` 라우트는 다음을 요구합니다:

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

`ARONA_ADMIN_TOKEN`은 프로세스 시작 시 환경에서 읽힙니다(`GatewayServer::new`). 변수가 **설정되지 않았거나** 제시된 token이 일치하지 않으면 요청은 `401`로 거부됩니다:

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

Bearer 접두어는 대소문자 구분 없이 매칭됩니다(`Bearer` 또는 `bearer`).

> `/v1/*` 표면과 달리 admin 인증은 API keys나 JWTs로 절대 대체되지 않으며, 정확한 token 비교로 집행됩니다 — 새 값으로 프로세스를 재시작하여 token을 회전하세요.

## Backends

Backend는 gateway 뒤의 라우팅 가능한 업스트림입니다. 등록은 backend를 즉시 라우팅 가능하게 만들고, 재시작 복원을 위해 구성을 영속화하며, probe하고(~1~2초 내에 healthy로 전환), 브리지 URL의 경우 터널을 살아 있게 유지합니다. Backend 유형과 URL 의미는 [Backends](../guides/backends.md)에 자세히 있습니다.

### POST /api/admin/backends — backend 등록

요청 본문(명시된 경우 외 모든 필드는 선택):

| 필드 | 유형 | 참고 |
| --- | --- | --- |
| `type` | string | Backend 종류. `external`(모든 OpenAI 호환 HTTP API), `ollama`(로컬 또는 원격 ollama 서버), `engine`(`ws://`/`wss://` 위의 CEP 엔진), `minimax-cloud`(클라우드 video API) 중 하나. MDD 엔진 이름(`llama_cpp`, `vllm`, `ollama`, `cloud`, `external_api`, `candle`, `native`, ...)은 planner를 통해 해석됩니다. `comfyui`는 **거부**됩니다(`comfyui backend removed`). 그 외의 것은 `400` `unknown_type`. 누락 시 기본값 `ollama`. |
| `url` | string | Backend 기본 URL. `evernight://<node>/<service>` 브리지 URL은 로컬 evernight agent를 통해 로컬 TCP 포워드로 해석됩니다(해석 실패 → `502` `evernight_unreachable`). 기본값 `http://localhost:11434`. |
| `api_key` | string | 선택적 업스트림 API key, 업스트림 호출 시 `Authorization: Bearer`로 전송. |
| `name` | string | Backend 이름. 누락 시 `type` 값이 기본값. 라우팅 `provider` 힌트 및 구성 행 식별자로 사용. |
| `models` | string[] | 정적 모델 목록. Probing이 아무것도 발견하지 못할 때의 라우팅 소스. `external` backend의 경우 발견된 모델이 정적 목록 뒤에 병합됩니다(정적 id가 우선권 유지). `engine` backend는 발견된 모델 캐시를 먼저 반환하고 정적 id를 뒤에 추가합니다. `minimax-cloud`는 모델 발견을 수행하지 않으며(probe는 `/v1/query/available_models`를 health ping만) 정적 목록만 제공합니다. `/api/tags`에서 모델을 발견하는 `ollama`는 무시합니다. |
| `workflow` | object | 선택. 레거시 — 역사적으로 제거된 ComfyUI backend가 소비했으며 현재 backend는 읽지 않습니다(`backend_configs` 열 호환성용으로 유지). |

예:

```bash
curl -X POST http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "name": "mock-upstream",
    "url": "http://192.0.2.20:11434",
    "api_key": "sk-xxx",
    "models": ["Qwen/Qwen3-1.7B"]
  }'
```

성공 → `200`:

```json
{ "status": "ok", "message": "backend registered" }
```

등록 부작용:

- Backend가 **즉시 등록되고 라우팅 가능**해집니다(재시작 불필요).
- 구성이 `backend_configs` 테이블에 **영속화**되고 시작 시 복원됩니다(DB 실패는 로그되지만 응답을 절대 차단하지 않음).
- 다음 60초 health-checker 라운드까지 fail-closed로 남는 대신, fire-and-forget **probe**가 즉시 실행되어 backend가 ~1~2초 내에 healthy로 전환됩니다.
- `evernight://` URL의 경우 **keepalive 작업**이 터널을 감시합니다: 새 로컬 포트로 재연결되면 동일한 이름으로 backend를 투명하게 재구축하고 재등록합니다.

### GET /api/admin/backends — backend 목록

```json
{
  "backends": {
    "count": 2,
    "health": [
      ["backend_0:ExternalApi", "Healthy"],
      ["backend_1:Ollama", { "Unhealthy": "connection refused" }]
    ]
  },
  "models": [
    { "id": "Qwen/Qwen3-1.7B", "object": "model", "owned_by": "mock-upstream" }
  ]
}
```

- `backends.count` — **healthy** backend 수.
- `backends.health` — backend별 `backend_<index>:<kind>` 라벨과 health 상태(`Healthy` / `Degraded` / `Unhealthy`). `<index>`는 `DELETE /api/admin/backends`가 사용하는 라우터 등록 인덱스입니다.
- `models` — 오늘 라우팅 가능한 모든 모델 id(quick-start 병합이 없는 `GET /v1/models`와 동일한 목록. [OpenAI 호환 REST](./openai-rest.md#get-v1models) 참조).

### DELETE /api/admin/backends — backend 제거

이름이 아닌 JSON 본문의 라우터 **인덱스**로 식별됩니다:

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| 필드 | 유형 | 필수 | 참고 |
| --- | --- | --- | --- |
| `index` | integer | 예 | `GET /api/admin/backends` health 보고서의 `backend_<index>` 라벨과 일치하는 라우터 등록 인덱스. |

- `index` 누락 → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`.
- 인덱스 범위 밖 → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`.
- 성공 → `200` `{ "status": "ok", "message": "backend removed" }`.
- 영속화된 `backend_configs` 행은 최선 노력으로 삭제됩니다: backend 이름은 모델 목록의 `owned_by`에서 복구됩니다. 불일치는 행을 저장소에 남깁니다(DB 실패는 로그되며 치명적이지 않음).

## Aliases

Aliases는 한 모델 이름을 다른 모델 이름(`alias` → `target`)에 매핑하여, 한 모델 id에 대한 요청이 다른 backend 모델로 라우팅되게 합니다. Aliases는 라우팅 전에 해석되므로 chat, embeddings, video 조회에 균일하게 적용됩니다.

> Aliases는 **인메모리 라우터 상태일 뿐**입니다 — 영속화되지 않으며 재시작 시 손실됩니다. 시작 후 등록하거나 자체 프로비저닝 상태에서 다시 생성하세요.

### POST /api/admin/aliases — alias 추가

| 필드 | 유형 | 필수 | 참고 |
| --- | --- | --- | --- |
| `alias` | string | 예 | 클라이언트가 요청할 모델 이름. |
| `target` | string | 예 | 요청이 라우팅되는 모델 id. |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- `alias` 누락 → `400` `missing_alias`. `target` 누락 → `400` `missing_target`.
- 성공 → `200` `{ "status": "ok", "message": "alias added" }`.
- 기존 alias를 추가하면 해당 target이 대체됩니다.

### GET /api/admin/aliases — alias 목록

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

쌍은 alias로 정렬되어 반환됩니다.

### DELETE /api/admin/aliases — alias 제거

| 필드 | 유형 | 필수 | 참고 |
| --- | --- | --- | --- |
| `alias` | string | 예 | 제거할 alias. |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- `alias` 누락 → `400` `missing_alias`.
- 알 수 없는 alias 제거는 no-op 성공 → `200` `{ "status": "ok", "message": "alias removed" }`.

## 영속화 요약

| 리소스 | 영속화? | 재시작 시 복원 |
| --- | --- | --- |
| Backends | 예 — `backend_configs` 테이블(`name` 키, 등록 시 upsert, 제거 시 삭제). | 예: 시작 시 복원. External backend는 fail-closed로 시작하고 첫 probe 라운드 후 healthy로 전환. `evernight://` URL은 시작 시 브리지를 통해 재해석. |
| Aliases | 아니요 — 인메모리 `Router.aliases`만. | 아니요. |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
