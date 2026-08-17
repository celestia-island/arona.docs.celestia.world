---
title: "구성"
description: "Arona 서버가 읽는 모든 환경 변수에 대한 참조(기본값과 의미 포함)."
---

# 구성

Arona는 **전적으로 환경 변수**로 구성되며, 프로세스 시작 시 한 번 읽힙니다(`packages/core/src/config.rs`의 `Config::load`, 일부는 처음 사용 시 읽힘). 구성 파일은 없습니다. 변수를 변경한 후 서버를 재시작해야 적용됩니다.

이 페이지는 서버 코드가 읽는 모든 항목에 대한 참조로, 관심사별로 그룹화되어 있습니다. Mock 전용 및 런타임 변수도 완전성을 위해 포함됩니다.

## 참조 표

| 변수 | 기본값 | 용도 |
| --- | --- | --- |
| `DATABASE_URL` | *(필수)* | PostgreSQL 연결 URL. |
| `JWT_SECRET` | *(mock 모드 외 필수)* | JWT 서명에 사용되는 시크릿. |
| `ARONA_HOST` | `0.0.0.0` | 바인딩 주소(`SHITTIM_CHEST_HOST`로 대체됨). |
| `ARONA_PORT` | `8420` | 바인딩 포트(`SHITTIM_CHEST_PORT`로 대체됨). |
| `ARONA_DATA_DIR` | 설정 안 됨 | 로컬 데이터 디렉터리. |
| `ARONA_ADMIN_TOKEN` | 설정 안 됨 | `/api/admin/*` 및 admin RPC 메서드용 bearer token. |
| `ARONA_REGISTRATION_OPEN` | `0` | Truthy 값(`1`/`true`/`yes`/`on`)이 가입을 엽니다. |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | 키별 분당 인메모리 요청 제한. `0`이면 모든 요청을 차단합니다. |
| `MOCK_MODE` | 설정 안 됨 | 존재(어떤 값이든)하면 개발용 mock 모드를 활성화합니다. |
| `MOCK_SEED_PATH` | 설정 안 됨 | Mock 모드에서 실행되는 원시 SQL 시드 파일. |
| `ARONA_MEMORY_URL` | 설정 안 됨 | Philia memory gateway WebSocket URL. |
| `ARONA_MEMORY_TOKEN` | 설정 안 됨 | Memory gateway용 token. |
| `ARONA_MEMORY_WRITEBACK` | `true` | 완료된 chat 턴을 memory에 다시 기록(writeback). `true`/`false`를 허용합니다(다른 값은 기본값으로 대체됨). |
| `ARONA_AGENT_NAME` | `arona-agent` | GPU 노드 agent 식별자. |
| `ARONA_PANEL_URL` | `localhost:8080` | Agent가 연결하는 위치(`ws://<panel_url>/ws/agent`). |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | `evernight://` backend URL용 로컬 evernight agent. |
| `ARONA_MISTRALRS` | 설정 안 됨 | 존재하면 Gguf 모델 플랜에 대해 Mistral.rs 엔진을 강제합니다. |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | 생성된 llama.cpp 모델 서버가 바인딩하는 인터페이스. |
| `HF_ENDPOINT` | `https://huggingface.co` | 모델 다운로드용 Hugging Face 기본 URL. |
| `GITHUB_TOKEN` | 설정 안 됨 | GitHub 모델 레지스트리용 액세스 token. |
| `RUST_LOG` | 설정 안 됨 | 추적 필터, 예: `info` 또는 `arona=debug,info`. |

## 필수 변수

### `DATABASE_URL`

PostgreSQL 연결 URL. **필수**: 비어 있으면 서버가 `FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.`로 종료하고, `migrate` CLI 하위 명령은 실행을 거부합니다. 스키마는 시작 시 `serve`가 자동으로 생성/업데이트합니다.

### `JWT_SECRET`

`auth.login`과 `auth.register`가 발급하는 access/refresh JWT 쌍을 서명하는 데 사용되는 시크릿. **프로덕션에서 필수**: 코드에 개발용 대체값(`dev-secret-not-for-production-use-only-32chars`)이 포함되어 있지만, `MOCK_MODE=1`이 아닌 한 서버는 이 값을 사용한 시작을 거부합니다:

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

길고 무작위한 값을 사용하세요(예: `openssl rand -hex 32`).

## 서버

### `ARONA_HOST` / `ARONA_PORT`

Gateway의 바인딩 주소와 포트. 레거시 호환성을 위해 `SHITTIM_CHEST_HOST` / `SHITTIM_CHEST_PORT`로 대체됩니다. 최종 기본값은 `0.0.0.0:8420`입니다.

### `ARONA_DATA_DIR`

선택적 로컬 데이터 디렉터리. 임시 위치가 필요한 구성 요소를 위해 앱 상태에 전달됩니다. 기본값은 설정되지 않음.

## 보안 및 액세스 제어

### `ARONA_ADMIN_TOKEN`

`/api/admin/*` HTTP 라우트(`POST/GET/DELETE
/api/admin/backends`, `/api/admin/aliases`)와 `billing.plan.set` / `billing.video.pricing.set` RPC 메서드를 보호하는 bearer token. **설정되지 않은** 경우 해당 라우트는 모두 `Admin access required`(401)로 거부합니다 — 기본값이 없습니다. 서버 시작 전에 강력한 무작위 값으로 설정하세요.

### `ARONA_REGISTRATION_OPEN`

`auth.register`를 통한 셀프 서비스 가입을 엽니다. Truthy 값은 정확히 `1`, `true`, `yes`, `on`(대소문자 구분 없음)입니다. 그 외의 모든 값 — `0`, `false`, `off` 또는 설정되지 않았거나 빈 변수 포함 — 은 닫힌 상태를 유지합니다. 이 플래그는 시작 시 한 번 읽힙니다. **첫 번째 등록 사용자는 항상 허용**되며(가입이 닫혀 있어도) admin이 됩니다.

### `ARONA_API_RATE_LIMIT_RPM`

인증된 모든 `/v1/*` 요청(chat, embeddings, video, models)에 적용되는 키별 인메모리 슬라이딩 윈도우 rate limit(분당 요청 수)으로, API-key 해시(또는 JWT를 허용하는 `GET /v1/models`의 경우 `u:<email>` 라벨)를 키로 사용합니다. RPC 평면은 이 리미터의 적용을 받지 않습니다 — `/v1/*` 인증 추출기만 호출자입니다. 기본값 `60`. 값은 프로세스 전역 `OnceLock`에 한 번 파싱됩니다. **값이 `0`이면 모든 요청을 차단합니다** — 검사가 `entry.len() >= rpm`이므로 `0`이면 어떤 요청도 통과할 수 없습니다. 이것은 gateway 전역 제한이며, billing tier가 그 위에 자체 키별 제한을 부과합니다.

## 개발

### `MOCK_MODE`

개발용 mock 모드. **존재**로 활성화됩니다 — 검사는 `std::env::var("MOCK_MODE").is_ok()`이므로 *어떤* 값(설정된 `0`이나 빈 값 포함)이든 활성화합니다. 다음을 수행합니다:

- `JWT_SECRET` 요구 사항을 해제합니다(내장 개발 시크릿이 허용됨);
- 4개의 데모 계정을 시드합니다(`demiurge@celestia.world`, `momoi@celestia.world`, `midori@celestia.world`, `yuzu@celestia.world`, 비밀번호 `33550336`);
- 리스너를 바인딩하기 전에 시드가 완료될 때까지 기다립니다.

프로덕션에서는 절대 사용하지 마세요.

### `MOCK_SEED_PATH`

Mock 모드에서만, 내장 계정 시드 대신 실행되는 원시 SQL 파일을 가리킵니다(문장은 `;`로 분리, `--` 주석은 건너뜀). 파일을 읽을 수 없으면 내장 시드가 대체로 사용됩니다.

## Memory gateway

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

장기 memory gateway(entelecheia Philia)용 구성. `ARONA_MEMORY_URL`과 `ARONA_MEMORY_TOKEN`이 모두 설정되고 비어 있지 않은 경우에만 memory가 **완전히 활성화**됩니다. 활성화된 경우:

- 완료된 chat 턴이 회상되어 컨텍스트로 주입되며,
- `ARONA_MEMORY_WRITEBACK`(기본값 `true`)은 완료된 턴이 memory 서비스에 다시 기록되는지 여부를 제어합니다. `0` 또는 `false`는 writeback을 비활성화합니다.

Memory 실패는 chat을 차단하지 않습니다. 결과 상태는 `X-Arona-Memory` 응답 헤더(`enabled` / `disabled` / `offline`)에 반영됩니다.

## Agent 식별자 및 클러스터

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

GPU 노드 agent 바이너리(`_agent`)의 식별자: `ARONA_AGENT_NAME`(기본값 `arona-agent`)은 agent의 이름/id로 패널에 보고되고, `ARONA_PANEL_URL`(기본값 `localhost:8080`)은 agent가 연결하는 위치입니다(`ws://<panel_url>/ws/agent`).

Agent의 자체 HTTP API는 `0.0.0.0:5790`에 바인딩되도록 **하드코딩**되어 있습니다 — 이에 대한 바인딩 주소 환경 변수는 없습니다.

### `ARONA_AGENT_BIND_ADDR`

Agent가 Gguf 모델을 배포할 때 **생성된 llama.cpp 모델 서버**가 바인딩하는 인터페이스로, 다른 머신에서 엔진에 도달할 수 있게 합니다(예: `0.0.0.0`). 기본값은 `127.0.0.1`입니다. 이것은 agent HTTP API 바인딩(`0.0.0.0:5790`에 고정)이 *아님*에 유의하세요.

## Evernight 브리지

### `ARONA_EVERNIGHT_URL`

`evernight://` backend URL을 로컬 TCP 포워드로 해석하는 데 사용되는 로컬 evernight agent의 WebSocket URL. 기본값 `ws://127.0.0.1:3001/ws`.

## 모델 런타임 및 다운로드

### `ARONA_MISTRALRS`

존재(어떤 값이든)하면 기본적으로 llama.cpp를 사용할 Gguf 모델 플랜에 대해 Mistral.rs 엔진을 강제합니다. `MOCK_MODE`와 같은 존재 의미론.

### `HF_ENDPOINT`

Hugging Face 모델 다운로드(`hf:` 소스)용 기본 URL, 기본값 `https://huggingface.co`. huggingface.co에 도달할 수 없을 때 `https://hf-mirror.com` 같은 미러로 설정하세요. 모델 다운로더가 읽으며, 끝의 슬래시는 제거됩니다.

### `GITHUB_TOKEN`

GitHub 모델 레지스트리(`gh:` 소스)가 API 액세스에 사용하는 액세스 token. 기본값은 설정되지 않음. 없으면 GitHub API rate limit이 적용됩니다.

## 로깅

### `RUST_LOG`

`tracing_subscriber`가 시작 시 적용하는 표준 추적 필터, 예: `info` 또는 `arona=debug,info`. 일반적인 `RUST_LOG` 의미론을 따릅니다(`error`/`warn`/`info`/`debug`/`trace`, 대상별 재정의).

## 기본값 한눈에 보기

| 설정 | 기본값 |
| --- | --- |
| 바인딩 주소 / 포트 | `0.0.0.0:8420` |
| 키별 API rate limit | 60 RPM |
| Agent 이름 | `arona-agent` |
| Panel URL | `localhost:8080` |
| Memory writeback | 켜짐 |
| 가입 | 닫힘 |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
