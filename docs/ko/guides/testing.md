---
title: "테스트"
description: "Arona 테스트 피라미드 — 단위 테스트, 밀폐(hermetic) 통합, PostgreSQL 게이트 통합, 라이브 서버 smoke 테스트, mock 서버 및 실제 자격 증명 smoke 규율."
---

# 테스트

Arona의 테스트는 계층으로 배열되어 기본 `cargo test` 실행이 빠르고 밀폐(hermetic)이며 데이터베이스나 네트워크가 필요하지 않도록 하고, 더 무거운 스위트는 실제 와이어 표면과 실제 PostgreSQL을 사용하는 명시적 opt-in으로 만듭니다. 이 페이지는 계층, 실행 명령, 실제 자격 증명 smoke 실행에 대한 워크스페이스 규율을 설명합니다.

## 단위 테스트

커버리지의 대부분은 `packages/core/src` 내부의 일반 단위 테스트입니다: 217개의 `#[test]` / `#[tokio::test]` 함수와 `packages/agent` 및 `packages/cli`에 걸친 ~23개가 더 있습니다. 다음으로 실행합니다:

```bash
cargo test --workspace
```

네트워크 없음, 데이터베이스 없음. 주요 스위트:

- **auth.rs** — 비밀번호 정책(≥8자 AND 4가지 문자 범주 중 ≥3), 원시 INSERT/REVOKE SQL의 `::uuid` 캐스트, 요청 기본값, `false`로 대체되는 admin 플래그 읽기.
- **billing/mod.rs** — 비용 *또는* token 차원의 quota 수학, 월간 윈도우(`month_start`, `seconds_until_month_end`), rate-limit 상한(RPM *에서만* 작동, `None` = 무제한), 월간 사용량 / tier / 키 윈도우 쿼리의 SQL 형태 가드, 업스트림 보고 숫자를 선호하는 `estimate_usage`.
- **routing/mod.rs** — alias 해석, `:latest` 접미사 매칭, provider 힌트, 최소 부하 선택 및 대화 고정.
- **gateway/mod.rs** — agent backend 등록: `agent-{model_id}` 등록, 재등록 시 대체(중복이 아닌), 등록 해제 시 라우터 복원.

## 밀폐 통합(항상 실행, DB 없음)

`packages/core/tests/gateway_integration.rs`에는 데이터베이스를 건드리지 않고 실제 직렬화/계약 로직을 실행하는 세 개의 항상 실행 테스트가 있습니다:

- **A1** — JSON-RPC id 에코 직렬화 계약: 숫자, 문자열, null 요청 id가 plana의 `Id` enum을 통해 유형 충실도로 왕복합니다.
- **A2** — admin 게이트 오류 코드 계약: `AUTH_ERROR`(-32005, 익명)와 `ADMIN_REQUIRED`(-32007, 인증된 비관리자)가 구별된 상태를 유지하고, 구현 정의 범위에 있으며, plana의 코드나 billing `QUOTA_ERROR`(-32006)와 충돌하지 않습니다.
- **A3** — `estimate_usage`: 업스트림 보고 사용량이 그대로 우선합니다. 없으면 로컬 토크나이저 추정이 합이 0이 아닌 prompt/completion 개수를 생성합니다.

`packages/core/tests/smoke.rs`는 세 개의 항상 실행 테스트를 더 추가합니다: 하드웨어 감지, 모델 레지스트리 루트 경로, `MOCK_MODE=1` 아래의 구성 기본값.

## PG 게이트 통합

전체 인프로세스 gateway 스위트 — `packages/core/tests/gateway_integration.rs` —는 임의의 루프백 포트에서 완전한 axum 라우터를 돌리고, 실제 admin API를 통해 일회용 OpenAI 호환 mock 업스트림을 등록하며, reqwest로 와이어 표면을 구동합니다. `AuthManager`가 모든 경로에서(심지어 `MOCK_MODE=1`도 계정을 *데이터베이스에* 시드할 뿐) PostgreSQL과 통신하므로, 이 스위트는 `ARONA_TEST_PG=1` 뒤에 게이트되며 기본적으로 건너뜁니다. 10개 테스트:

- **T1** 등록 + 로그인 + `keys.create`/`keys.list`(목록에서 원시 키 마스킹, `arona-` 접두어).
- **T2** PostgreSQL로의 사용 기록 영속화가 있는 REST chat.
- **T3** 와이어를 통한 JSON-RPC id 에코(성공 및 오류 경로).
- **T4** `agents.list`의 admin 게이트: 익명 → `AUTH_ERROR`, 비관리자 → `ADMIN_REQUIRED`.
- **T5** 업스트림 401 → 업스트림 상세가 있는 HTTP 502 `bad_gateway`.
- **T6** 등록 시점 probe가 모델을 게시합니다(정적 모델 목록 없이 10초 내에 모델이 `GET /v1/models`에 나타남).
- **T7** `chat.send`를 통한 대화 영속화(두 턴 모두 `conversations.get`에 나타남).
- **T8** free-tier billing 게이트: 키당 10 RPM, 윈도우의 11번째 요청은 429 `rate_limit_exceeded`.
- **T9** 업스트림에서 기록된 터미널 사용량이 있는 SSE 스트림.
- **T10** 잘못된 JSON → 400. 알 수 없는 모델 → 404 `model_not_found`.

모듈 문서(gateway_integration.rs:18-26)의 일회용 PostgreSQL 원라이너로 실행합니다:

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

이것들은 일회용 테스트 컨테이너를 위한 예제 자격 증명일 뿐입니다 — 실제 데이터베이스에 절대 연결하지 마세요.

## 라이브 서버 smoke

`packages/core/tests/auth_flow.rs`는 **라이브** Arona 서버에 대해 전체 `register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` 체인을 실행하며, 배포된 인증 루프를 반영합니다. 기본적으로 `#[ignore]`됩니다 — 일반 `cargo test` 실행은 네트워크를 건드리지 않습니다. 명시적으로 실행:

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

Knobs:

- `ARONA_TEST_URL` — 기본 URL(기본값 `http://127.0.0.1:8420`).
- `ARONA_TEST_EXPECT_CHAT=1` — `POST /v1/chat/completions`가 200을 반환함을 하드 어서트합니다. 없으면 테스트는(대상 환경에 inference provider가 구성되지 않았을 수 있으므로) 인증 통과(401/403 아님)만 어서트합니다.

스위트에는 부정 테스트도 포함됩니다: 인증되지 않은 chat completion과 인증되지 않은 `GET /v1/models`가 모두 401로 거부되어야 합니다.

## Mock 서버

`scripts/mock/server.py`는 quickstart와 smoke 실행이 사용하는 aiohttp 기반 OpenAI 호환 fake입니다. `POST /v1/chat/completions`(비스트림 및 SSE), `GET /v1/models`, `GET /api/health`, `/api/rpc`의 JSON-RPC WebSocket/HTTP 표면, `/api/rpc/events`의 SSE 사이드카, 그리고 다른 서비스가 mock API key를 발견할 수 있게 반환하는 `GET /api/test-key`를 제공합니다. 기본적으로 포트 8429에서 수신합니다(`ARONA_MOCK_PORT`로 포트, `ARONA_MOCK_HOST`로 호스트 재정의). [quickstart](quickstart.md)는 실제 모델 provider 없이 end-to-end 환경을 구축하는 데 이를 사용합니다.

## 실제 자격 증명 smoke 규율

실제 provider(DeepSeek / GLM)에 대한 smoke 실행은 의도적으로 **저장소 테스트가 아닙니다** — 실제 자격 증명과 실제 비용이 필요하므로 CI나 git 트리에 있을 수 없습니다. gateway_integration 모듈 문서(gateway_integration.rs:54-55)에 문서화된 워크스페이스 관례는:

- 증거 파일은 `/mnt/work/arona-pr*-smoke.md` 아래에 있습니다 — 워크스페이스 로컬이며 git에 커밋되지 않습니다.
- 자격 증명은 환경에서만 옵니다. 예산은 작게 유지됩니다.
- inference 경로를 건드리는 각 PR은 서면 증거 기록을 받습니다.

mock 서버는 CI와 로컬 개발에서 이러한 실행의 대체물입니다. 실제 자격 증명 smoke는 릴리스 시점의 인간 단계입니다.

## CI

`.github/workflows/ci.yml`은 조직의 self-hosted runners(`[self-hosted, linux, x64, local]`)에서 `cargo fmt`, `cargo clippy`, `cargo test
--workspace`, `cargo-deny`를 실행합니다. `ci-hosted.yml`은 GitHub-hosted runners에서 동일한 검사를 미러링합니다. `.github/workflows/docs.yml`은 lagrange로 이 문서 사이트를 빌드하고 `docs/**`를 건드리는 push에서 GitHub Pages에 배포합니다.

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
