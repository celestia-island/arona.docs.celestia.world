---
title: "빠른 시작"
description: "내장 mock 업스트림을 사용한 Arona end-to-end 안내: migrate, serve, backend 등록, API key 생성 및 chat."
---

# 빠른 시작

이 가이드는 **내장 mock 업스트림**을 사용하여 단일 머신에서 Arona를 완전한 end-to-end로 설정하는 과정을 안내합니다. 실제 모델 가중치, GPU 또는 외부 API 계정이 필요하지 않습니다. 이 가이드를 마치면 다음을 갖추게 됩니다:

- 실행 중인 Arona gateway(`/v1/*` OpenAI 호환 REST API와 `/api/rpc` JSON-RPC 관리 평면),
- `external` backend로 등록된 mock 업스트림,
- 사용자 계정과 API key,
- mock에 대한 비스트리밍 **및** 스트리밍 chat 턴,
- `usage.list`를 통해 확인할 수 있는 사용량 기록.

## 사전 요구 사항

- **Rust 툴체인**(저장소 루트의 `rust-toolchain.toml` 참조).
- **Python 3** 및 `aiohttp` — mock 업스트림에만 필요합니다(`pip install aiohttp`).
- **실행 중인 PostgreSQL** 인스턴스와 해당 연결 URL.

## 1. 환경 설정

Arona는 **프로세스 시작 시** 환경 변수에서 구성을 읽습니다. 두 가지가 필수입니다: `DATABASE_URL`과 `JWT_SECRET` — 이 변수 없이는 서버가 시작을 거부합니다(`MOCK_MODE=1`이 아닌 경우). `ARONA_ADMIN_TOKEN`은 강력히 권장됩니다. 없으면 모든 `/api/admin/*` 라우트가 401을 반환합니다.

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

이 변수들은 프로세스 시작 시 한 번만 읽힙니다. 변경하면 서버를 재시작하세요. 전체 변수 참조는 [구성](configuration.md)을 참조하세요.

## 2. 마이그레이션 및 서버 시작

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

새 데이터베이스에는 `serve`만으로 충분합니다. 시작 시 자동으로 마이그레이션합니다. 서버는 기본적으로 `0.0.0.0:8420`에 바인딩됩니다(`ARONA_HOST` / `ARONA_PORT`로 재정의).

## 3. mock 업스트림 시작

두 번째 터미널에서:

```bash
python3 scripts/mock/server.py
```

mock은 기본적으로 `127.0.0.1:8429`에서 수신하는 aiohttp 서버입니다(`ARONA_MOCK_PORT`가 포트를 재정의). 시작 시 API key를 출력하며, `{"api_key": ..., "base_url": ...}`를 반환하는 `GET /api/test-key`도 제공합니다. `gpt-5.5`를 포함한 여러 모델 id를 노출하며, 일반 및 스트리밍 chat completions 모두에 응답합니다.

출력된 키를 캡처합니다:

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. mock을 external backend로 등록

Backend는 admin HTTP API를 통해 등록됩니다:

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

backend는 등록 즉시 probe되며 약 1~2초 내에 healthy 상태로 전환됩니다. probe가 완료되기 전까지는 fail-closed인 "아직 probe되지 않음(not probed yet)" 상태로 유지됩니다(아래 문제 해결 상자 참조). 구성은 영속화되므로 backend는 재시작 후에도 유지됩니다.

## 5. 계정 등록 및 로그인

계정은 JSON-RPC 평면인 `POST /api/rpc`에 존재합니다. `ARONA_REGISTRATION_OPEN=1`이 설정되어 있으므로 `auth.register`는 열려 있으며, 첫 번째 등록 사용자가 admin이 됩니다.

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

비밀번호는 최소 8자 이상이어야 하며 **또한** 4가지 문자 범주(대문자, 소문자, 숫자, 특수 문자) 중 최소 3가지를 포함해야 합니다. 그런 다음 로그인하여 JWT 쌍을 얻습니다:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

응답에서 `access_token`을 내보냅니다:

```bash
export JWT="<access_token from the login response>"
```

## 6. API key 생성

`keys.create`는 JWT로 인증되며 **전체** `arona-{uuid}` 시크릿을 정확히 한 번 반환합니다. 데이터베이스에는 SHA-256 해시만 저장되므로 지금 복사하세요:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. Chat(비스트리밍)

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

OpenAI 스타일의 completion 객체 — `choices[0].message`와 `usage` 블록 — 를 반환받습니다.

## 8. Chat(스트리밍)

동일한 엔드포인트에 `"stream": true`를 지정하면 server-sent events로 응답합니다. token당 하나의 `data:` 청크이며, 마지막에 `data: [DONE]` 청크로 종료됩니다:

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. 사용량 확인

모든 chat 턴은 키의 접두어 아래에 usage 행을 기록합니다. JWT로 조회합니다:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

위에서 만든 `gpt-5.5` 요청에 대한 레코드가 하나 이상 표시되어야 합니다.

## 문제 해결

- **`No backend available for model: gpt-5.5`(HTTP 404, `code:
  model_not_found`)** — 해당 모델 id를 제공하는 등록된 backend가 없습니다. backend가 등록되지 않았거나(또는 `models` 목록에 id가 없는 경우), 등록 호출이 실패한 경우입니다. `GET /api/admin/backends`(admin token)로 확인하세요.
- **`all backends unhealthy`(HTTP 500, `backend_error`)** — 모델에 대해 backend *는* 등록되어 있지만 healthy 후보가 없습니다. 갓 등록된 external backend는 fail-closed "아직 probe되지 않음" 상태로 시작하며, 등록 시점의 probe가 완료되면(~1~2초 후) healthy로 전환됩니다. 이 시간 동안 chat을 하면 이 오류가 발생합니다. 잠시 후 다시 시도하거나, mock이 실제로 `127.0.0.1:8429`에서 실행 중인지 확인하세요.
- **`/v1/*`에서 HTTP 401** — `Authorization` 헤더가 없으면 `Missing Authorization header. Use: Bearer <api_key>`가 발생하고, 알 수 없는 키는 `Invalid API key`가 발생합니다. `$AR_KEY`(전체 시크릿, 접두어 아님)를 다시 확인하세요.
- **`/api/admin/*`에서 HTTP 401 `Admin access required`** — bearer token이 `ARONA_ADMIN_TOKEN`과 일치하지 않거나, 변수가 설정되지 않은 경우입니다(그러면 라우트가 항상 거부). 설정한 후 서버를 재시작하세요.
- **`auth.register`가 "Registration is closed"로 실패** — `ARONA_REGISTRATION_OPEN`이 truthy가 아니면 가입이 비활성화됩니다. 서버 시작 **전에** `ARONA_REGISTRATION_OPEN=1`을 설정하거나(시작 시 읽힘), 가장 첫 번째 사용자가 되세요 — 첫 번째 등록 사용자는 항상 허용되며 admin이 됩니다.
- **HTTP 429 rate limits** — 세 가지 독립적인 제한이 발동할 수 있습니다:
  - 키별 인메모리 제한, 기본 60 RPM(`ARONA_API_RATE_LIMIT_RPM`) → `Rate limit exceeded. Try again later.`;
  - free billing tier의 키별 10 RPM 제한 → `Retry-After: 60` 헤더가 있는 429;
  - free tier의 월간 $1 / 100k-token quota → 다음 billing 기간을 가리키는 `Retry-After`가 있는 429.

## 다음 단계

- [구성](configuration.md) — 모든 환경 변수.
- [Backends](backends.md) — backend 유형, URL 의미 및 probing.
- [배포](deployment.md) — bare metal, systemd, Docker.
- [OpenAI 호환 REST API](../api/openai-rest.md) — 전체 `/v1/*` 표면.
- [JSON-RPC API](../api/jsonrpc.md) — 위에서 사용한 관리 평면.

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
