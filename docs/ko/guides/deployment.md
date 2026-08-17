---
title: "배포"
description: "릴리스 자산, 스크립트 또는 소스에서 arona-server를 설치하고, systemd로 bare metal, Docker Compose 또는 malkuth 감독 하에서 실행하며, 안전하게 업그레이드합니다."
---

# 배포

Arona는 `_cli` 크레이트에서 빌드된 **단일 자체 포함(self-contained) Rust 바이너리**로 제공됩니다. 릴리스 워크플로우는 이를 `arona-server-<tag>-linux-x86_64`(`.github/workflows/release.yml`)로 게시하며, 동일한 바이너리가 두 역할을 모두 수행합니다:

- **API 서버** — `serve`(gateway, `migrate`는 스키마 마이그레이션을 명시적으로 적용);
- **모델 도구** — `install`(하드웨어 감지), `status`, `deploy
  <model>`, `download <repo>`, `connect <panel-url>`.

서버는 PostgreSQL이 필요합니다. 스키마는 시작 시 자동으로 생성되고 마이그레이션되므로, 배포는 대부분 "바이너리를 가져와 데이터베이스에 연결하고 실행"입니다. End-to-end 안내는 [빠른 시작](./quickstart.md)부터 시작하고, 프로덕션 구성은 여기로 돌아오세요.

## 요구 사항

- **Linux x86_64** — 사전 빌드된 릴리스 자산용. Rust 1.91+가 지원하는 모든 플랫폼에서 소스로 빌드할 수 있습니다.
- **PostgreSQL** — Docker Compose 예제는 `postgres:16-alpine`을 사용합니다. 최신 버전이면 모두 작동합니다. 서버는 `DATABASE_URL` 없이는 시작을 거부합니다.
- **`curl`, `python3`, `ca-certificates`** — install 스크립트를 사용하는 경우(Debian/RedHat에서는 스크립트가 apt/dnf로 직접 설치).
- 바이너리를 위한 쓰기 가능한 위치(예: `/usr/local/bin`)와, 모델 다운로드를 실행한다면 `ARONA_DATA_DIR` 아래 모델 캐시용 디스크 공간.

## 설치

### 릴리스 자산에서

원하는 태그의 자산을 다운로드하고 실행 가능하게 만듭니다:

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### install 스크립트에서

저장소에는 `scripts/install.sh`가 포함되어 있습니다: GitHub API에서 최신 릴리스 태그를 확인하고(`ARONA_VERSION` 재정의를 존중), 일치하는 자산을 다운로드하여 기본적으로 `~/.local/bin`의 `arona-server`로 설치하며(`ARONA_BIN_DIR`로 디렉터리 재정의), 다음 단계를 출력합니다:

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### 소스에서

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli`는 릴리스 워크플로우와 Dockerfile이 빌드하는 것과 정확히 동일합니다.

## 구성

모든 구성은 환경 변수를 통해 이루어집니다. [구성 참조](./configuration.md)가 모든 변수를 문서화하며, 저장소 루트의 `.env.example`은 작동하는 시작점입니다. 최소 집합:

| 변수 | 필수? | 용도 |
| --- | --- | --- |
| `DATABASE_URL` | 예 | PostgreSQL 연결 문자열. 설정되지 않으면 서버가 종료됩니다. |
| `JWT_SECRET` | 예 | Token 서명 시크릿. `MOCK_MODE=1`이 아니면 내장 개발 시크릿으로 실행을 거부합니다. |
| `ARONA_ADMIN_TOKEN` | 강력 권장 | `/api/admin/*` 라우트용 공유 bearer token. 없으면 해당 라우트가 항상 401을 반환합니다. |

선택 사항: `ARONA_HOST`(기본값 `0.0.0.0`), `ARONA_PORT`(기본값 `8420`), `ARONA_REGISTRATION_OPEN`(truthy `1`/`true`/`yes`/`on` — 가입을 엽니다. 첫 번째 등록 사용자가 admin이 됨), `ARONA_DATA_DIR`(모델 캐시 루트), `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`(memory gateway), `ARONA_API_RATE_LIMIT_RPM`(키별 rate limit) 및 `RUST_LOG`(추적 필터).

## 데이터베이스 마이그레이션

`serve`는 시작 시 데이터베이스에 연결하고 보류 중인 모든 스키마 마이그레이션을 적용합니다(`init_database` → `Migrator::up`). 따라서 별도의 배포 단계가 없습니다. 마이그레이션을 명시적으로 적용하려면 — 예를 들어 첫 시작 전에 검증하기 위해 — 다음을 실행하세요:

```bash
arona-server migrate
```

데이터베이스 사용자는 대상 스키마에 대한 **`CREATE` 권한**이 필요합니다. 시작 마이그레이션이 테이블을 생성하기 때문입니다. 스키마 외의 데이터 마이그레이션은 없습니다. 데이터베이스 *자체가* 상태이므로 업그레이드 전에 백업하세요([Upgrade and backup](#upgrade-and-backup) 참조).

## systemd로 bare metal 실행

예시 unit 파일(`/etc/systemd/system/arona.service`). **아래의 모든 시크릿 값은 플레이스홀더입니다 — 사용 전에 `CHANGE_ME`를 교체하세요:**

```ini
[Unit]
Description=Arona API server
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=arona
Environment=DATABASE_URL=postgres://arona:CHANGE_ME@127.0.0.1:5432/arona
Environment=JWT_SECRET=CHANGE_ME
Environment=ARONA_ADMIN_TOKEN=CHANGE_ME
Environment=ARONA_HOST=0.0.0.0
Environment=ARONA_PORT=8420
Environment=RUST_LOG=info
# Optional:
# Environment=ARONA_REGISTRATION_OPEN=1
# Environment=ARONA_MEMORY_URL=ws://127.0.0.1:8424/ws
# Environment=ARONA_MEMORY_TOKEN=CHANGE_ME
# Environment=ARONA_MEMORY_WRITEBACK=1
# Environment=ARONA_DATA_DIR=/var/lib/arona
ExecStart=/usr/local/bin/arona-server serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

그런 다음 활성화하고 시작합니다:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

저장소 루트에는 두 개의 서비스를 가진 `docker-compose.yml`이 포함되어 있습니다:

- **`arona`** — `Dockerfile`에서 이미지를 빌드하고(tag `arona:local`), `${ARONA_PORT:-8420}:8420`을 게시하며, `DATABASE_URL`, `JWT_SECRET`, `ARONA_ADMIN_TOKEN`을 요구합니다 — `.env`에서 누락된 항목이 있으면 Compose가 `:?` 스타일 오류로 즉시 실패합니다. Healthcheck는 `curl -fsS http://127.0.0.1:8420/readyz`를 실행합니다.
- **`postgres`** — `postgres:16-alpine`이며 플레이스홀더 전용 자격 증명(`POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`; 기본 비밀번호는 `change-me` — 재정의 필요)과 named volume `arona-pgdata`를 사용합니다.

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

번들된 postgres 서비스는 Compose 네트워크 내부의 호스트 `postgres`에서 접근할 수 있습니다. `Dockerfile`은 `rust:1.91-slim-bookworm` 빌더에서 `_cli`와 `_agent`를 빌드하고, `debian:bookworm-slim` 런타임에 `ca-certificates`, `curl`, `python3`을 설치하며, 두 바이너리를 복사하고, 포트 8420(서버)과 5790(번들 `_agent`의 로컬 API)을 노출하며, entrypoint로 `arona-server serve`를 실행합니다.

## malkuth로 감독되는 배포(워크스페이스 관례)

이 워크스페이스에서 arona는 **malkuth** 감독 하에 `arona-malkuth.service`로 실행됩니다. 이 패턴은 여기에 배포되는 모든 서비스에 적용됩니다:

- malkuth는 `arona-server serve` 프로세스를 감독합니다 — 프로세스를 생성하고, probe하고, 실패 시 재시작하며, 종료 전에 연결을 드레인합니다.
- 각 감독 서비스는 서비스별 **info 포트**와 서비스별 **proxy 포트**를 통해서만 노출됩니다. 서비스 자체는 `127.0.0.1`에 바인딩되며 네트워크에서 직접 도달할 수 없습니다.
- 슈퍼바이저는 `--watch <deployment-path>`로 시작됩니다. 해당 경로의 바이너리가 변경되면 — 예: 업그레이드가 새 파일을 복사 — malkuth는 **롤링 재시작**을 수행합니다. 한 번에 한 인스턴스씩, 먼저 연결을 드레인합니다.

운영상의 결과:

- 슈퍼바이저 proxy 뒤에서 실행할 때 `ARONA_HOST=127.0.0.1`로 바인딩하세요. Proxy가 유일한 네트워크 노출 진입점입니다.
- 업그레이드는 "새 바이너리를 배포 경로에 복사"입니다. Watcher가 롤링 재시작을 트리거합니다. 감독 unit을 명시적으로 재시작할 수도 있습니다.
- 슈퍼바이저의 health check를 `/readyz`(또는 `/api/health`)로 지정하세요. [Health probes](#health-probes) 참조.

## Health probes

서버는 두 개의 비인증 health 계열을 노출합니다(둘 다 [operations 가이드](./operations.md)에서도 다룸):

- `GET /healthz`, `GET /readyz`(별칭) 및 `GET /v1/health`는 `200`과 함께 `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`을 반환합니다.
- `GET /api/health`는 plana `HealthResponse` 형태를 반환합니다: `status`, `version`, `kind`, `uptime`(초), `network`, `build_hash`, `engine_version`.

로드 밸런서, 슈퍼바이저, 컨테이너 healthcheck는 `/readyz`를 가리키세요. Uptime과 network 세부 정보가 필요하면 `/api/health`를 사용하세요.

## 업그레이드 및 백업

- **이전 바이너리를 보관하세요.** `arona-server`를 덮어쓰기 전에 `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak`로 복사해 두면, 새 버전이 시작에 실패할 경우 즉시 바이너리를 롤백할 수 있습니다.
- **PostgreSQL을 백업하세요.** 데이터베이스가 모든 상태(backends, agent nodes, users, keys, conversations, usage)를 보유하며, 유일한 자동 스키마 변경은 시작 마이그레이션입니다. 업그레이드 전에 매번 `arona` 데이터베이스를 `pg_dump` 하세요.
- **데이터베이스 사용자에게 `CREATE` 권한이 필요합니다.** 마이그레이션이 시작 시 실행되므로, 읽기 전용 역할은 서버를 부팅할 수 없습니다.
- **정상적으로 종료하세요.** 서버는 SIGINT/SIGTERM에서 진행 중인 연결을 드레인하므로, `kill -9`보다 `systemctl restart arona` 또는 슈퍼바이저 재시작을 선호하세요.

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
