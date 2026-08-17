---
title: "Agent Cluster"
description: "다중 노드 GPU 클러스터 — CLI로 모델 가중치 다운로드, GPU 노드에서 _agent 바이너리 실행, agents.* RPC 표면을 통한 배포 구동."
---

# Agent Cluster

Arona의 배포 이야기는 두 부분으로 나뉩니다. **패널**(`arona` 서버 바이너리)은 라우팅, billing, 인증, 관리 평면을 소유합니다. 각 GPU 노드는 모델 가중치와 로컬 서빙 프로세스를 소유하는 하나의 **`_agent` 프로세스**를 실행합니다. Agent는 패널의 agent 제어 평면(`/ws/agent`)으로 장기 연결 WebSocket을 엽니다. 패널은 해당 소켓으로 `deploy` / `stop` 명령을 밀어 넣고, agent는 다운로드 진행 상황, 하트비트, 명령 결과를 위로 스트리밍합니다. 모델이 agent에서 실행되면 패널이 이를 라우팅 가능한 backend로 등록하여 `/v1/*` 및 RPC 트래픽이 도달하게 합니다 — 제어 평면은 WebSocket이고, 데이터 평면은 agent의 로컬 엔진 포트로 가는 일반 HTTP입니다.

## 모델 가중치 다운로드(CLI)

`_cli` 바이너리는 HuggingFace, ModelScope 또는 GitHub releases에서 모델 가중치를 다운로드합니다:

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **Repo 형태** — `hf:owner/repo`(기본값. 단순 `owner/repo`도 HuggingFace로 해석됨), `ms:owner/repo`(ModelScope), `gh:owner/repo[:tag]`(GitHub release, tag 선택). 긴 접두어 `huggingface:`, `modelscope:`, `github:`도 허용됩니다. 슬래시가 없는 단순 id는 Ollama 레지스트리로 해석됩니다(`packages/core/src/models/download.rs:21-28,55-86`).
- **`--filter <glob|prefix>`** — 반복 가능. Glob(또는 접두어)과 일치하는 매니페스트 파일만 다운로드됩니다. 필터가 없으면 **전체 repo**가 선택됩니다.
- **확인** — 필터 없는 다운로드는 `--yes`를 전달하지 않는 한 항상 시작 전에 `Continue? [y/N]`을 묻습니다. 필터된 다운로드는 프롬프트하지 않습니다. 선택된 총량이 2GiB 이상이면 정보 배너를 대신 출력합니다(`NO_CONFIRM_THRESHOLD`, `packages/cli/src/main.rs:12-15,
  439-464`).
- **`--out <dir>`** — 기본 대상 `~/.arona/models/<repo-id>`를 재정의합니다.
- **`--revision <rev>`** — 인라인 `:rev` 접미사(`hf:owner/repo:rev`)를 재정의합니다.
- **재개(Resume)** — 중단된 다운로드는 자동으로 재개됩니다: `.part` 파일이 유지되고 Range 요청으로 현재 길이부터 계속됩니다. 완료된 파일은 크기로 건너뛰고, 매니페스트에 다이제스트가 있으면 SHA-256으로 검증됩니다(`packages/cli/src/main.rs` `verify_sha256`/`summarize`).
- **재시도** — 네트워크 오류는 5초 간격으로 최대 3회 재시도합니다. 비네트워크 오류는 즉시 실패합니다(`packages/cli/src/main.rs:277-302`).
- **`HF_ENDPOINT`** — HuggingFace 기본 URL을 전환합니다, 예: 미러:

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

기타 CLI 명령(`packages/cli/src/main.rs:28-53`):

| 명령 | 용도 |
| --- | --- |
| `install` | 원클릭 환경 설정: 하드웨어 프로필을 감지하고 backend / 양자화 권장 사항을 출력합니다. |
| `status` | 하드웨어 프로필을 출력합니다. |
| `deploy <model>` | 모델을 로컬에서 해석하고 이미 캐시되었는지 보고합니다. |
| `download` | 모델 가중치를 다운로드합니다(위). |
| `serve` | API 서버(패널)를 시작합니다. |
| `connect <url>` | 관리 패널에 연결합니다. |
| `migrate` | 데이터베이스 마이그레이션을 실행합니다. |

## `_agent` 바이너리

`_agent`는 각 GPU 노드에서 실행되며 순수하게 환경 변수로 구성됩니다(`packages/core/src/config.rs:37-40`):

| 변수 | 기본값 | 의미 |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | 고유 노드 id. 패널이 이를 `agent_id`로 사용합니다. |
| `ARONA_PANEL_URL` | `localhost:8080` | 패널 `host:port`. Agent는 `ws://{ARONA_PANEL_URL}/ws/agent`에 연결합니다. |

전체 환경 변수 참조(패널 측 변수, 데이터베이스, 시크릿)는 [구성](./configuration.md)을 참조하세요.

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

동작:

- **제어 연결** — agent는 `ws://{ARONA_PANEL_URL}/ws/agent`(`packages/agent/src/panel.rs:23`)에 다시 연결합니다. 연결 시 `agent_name`, `gpu_info`, 이미 배포된 모델 목록을 담은 `register` 프레임을 보냅니다. 패널은 agent의 TCP 피어 주소를 `host`로 기록합니다.
- **재연결 백오프** — 1초에서 시작하여 60초 상한까지 두 배로 늘어납니다(`packages/agent/src/panel.rs:27,33-34,63`).
- **하트비트** — 30초마다 agent가 GPU 사용률, 로드된 모델 수, uptime을 보고합니다. 패널은 마지막 하트비트가 30초보다 오래되면 agent를 오프라인으로 간주합니다.
- **로컬 HTTP API** — **고정** 주소 `0.0.0.0:5790`에 바인딩됩니다. 바인딩 주소 환경 변수는 없습니다(`packages/agent/src/main.rs:109`). 패널은 이 포트와 agent의 기록된 host를 결합하여 배포된 모델의 데이터 평면 URL을 구성합니다.
- **명령** — 패널은 소켓을 통해 `deploy` / `stop` 명령을 큐에 넣습니다. `deploy` 명령은 `model_id`와 `stream_id`를 담습니다. 다운로드 진행 상황은 같은 소켓에서 `deploy_progress` 프레임으로 스트리밍되며(패널은 이를 `models.progress` SSE 알림으로 전달, [Events & Notifications](../api/events.md) 참조), 최종 `deploy_result` 프레임은 로컬 엔진 `port`와 `backend`를 보고합니다. `stop`은 `stop_result`로 응답됩니다.

`_agent`를 서비스 슈퍼바이저(systemd, malkuth, ...) 아래에서 실행하면 자동으로 재연결됩니다. 패널은 양쪽의 재시작을 허용합니다(아래 [노드 유지](#node-persistence) 참조).

## Agent 제어 평면 RPC

전체 agent 표면은 admin 게이트입니다. 모든 메서드는 유효한 JWT **및** admin 계정이 필요합니다(`validate_admin_jwt`가 `is_admin_email`을 검사합니다; `packages/core/src/gateway/rpc.rs:106-118,301-337`).

| 메서드 | Params | 반환값 |
| --- | --- | --- |
| `agents.list` | — | 클러스터 토폴로지: `id`, `name`, `host`, `status`(`online`/`offline`), GPU 요약, `models`, `last_heartbeat`, `version`, `connected_at`. |
| `agents.register` | `machine_name`, `version` | `agent_id`, `token`. |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — 노드를 제거합니다. |
| `agents.status` | `agent_id` | `online`, GPU 요약, `gpu_utilization`, `models`, `host`, `connected_at`, `last_heartbeat`. |
| `agents.deploy` | `model_id`, `agent_id?` | `{ "ok": true, "stream_id" }` — 빈 `agent_id`는 최소 부하 노드를 자동으로 대상으로 합니다. |
| `agents.stop` | `agent_id`, `model_id` | `{ "ok": true }` — 배포를 중지합니다. |

`agents.deploy`는 `stream_id`를 반환합니다. 호출 **전에** 또는 직후에 `/api/rpc/events?session=<stream_id>`를 구독하여 `models.progress` 다운로드 알림을 받으세요([Events & Notifications](../api/events.md) 참조). 전송 및 인증 세부 사항은 [JSON-RPC API](../api/jsonrpc.md)를 참조하세요.

## 배포 모델 자동 등록

`deploy_result` 프레임이 성공을 보고하면 패널은 기본 URL `http://{agent-host}:{port}`(agent의 기록된 host와 보고된 엔진 포트)로 **`agent-{model_id}`**라는 `ExternalApiBackend`를 gateway 라우터에 등록합니다(`packages/core/src/gateway/server.rs:310-366`, `packages/core/src/gateway/mod.rs:253-270`). 배포된 모델은 일반 라우팅 가능 backend가 됩니다: `/v1/chat/completions`, embeddings, RPC chat이 모두 도달하고, aliases가 적용되며, health checker가 probe합니다(backend 유형과 probing 의미는 [Backends](./backends.md) 참조).

- 동일한 모델을(예: 다른 agent에) 재배포하면 이전 backend를 대체합니다.
- 성공적인 `stop_result`는 다시 등록을 해제하며(`packages/core/src/gateway/mod.rs:274-287`), 모델 id는 해석을 중지합니다.

## 배치

명시적 `agent_id`가 없는 배포는 최소 부하 배치(`packages/core/src/gateway/tunnel.rs:214-229`)를 거칩니다: 후보는 마지막 하트비트가 30초 미만인 agents이며, **평균 GPU 사용률이 가장 낮은** 것이 선택됩니다. 텔레메트리가 없는 agent는 마지막으로 정렬되지만 선택 가능한 상태로 유지됩니다. 온라인 agent가 없으면 RPC는 `No online agents available for deployment`로 실패합니다.

라우팅 측에서 대화는 **하나의 backend에 고정**됩니다(세션 고정): 대화를 처음 제공한 backend가 기록되고 이후 턴에 재사용되므로, 런타임 KV 캐시 같은 대화별 상태가 따뜻하게 유지됩니다(`packages/core/src/routing/mod.rs:31-34,110-138`). 고정된 backend가 unhealthy가 되면 라우팅은 새 선택으로 저하되고 다시 고정합니다.

## 노드 유지

Agent 노드는 `agent_nodes` 테이블(`agent_id`, `machine_name`, `version`, `host`, `gpu_info`, `models`, `connected_at`, `last_heartbeat`; `packages/core/src/gateway/tunnel.rs:8-12`)에 유지됩니다. 패널 시작 시 영속화된 행이 복원되어 이전에 등록된 노드가 재시작에도 계속 표시됩니다. 복원된 항목은 각 agent가 WebSocket으로 재연결할 때까지 **sender 없음** 상태입니다(`packages/core/src/gateway/run.rs:74-85`). 따라서 복원되었지만 연결이 끊긴 노드에 배포하는 것은 `_agent`가 재연결될 때까지 실패합니다.

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
