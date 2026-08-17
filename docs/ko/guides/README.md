---
title: "Arona"
description: "AI 모델의 자체 배포 및 원격 관리 플랫폼 — gateway, backends, billing, agents, memory."
---

# Arona

**AI 모델의 자체 배포 및 원격 관리 플랫폼입니다.**

Arona는 Rust(axum)로 작성된 **순수 backend** 플랫폼입니다. 자체 하드웨어에서 실행하는 모델을 위한 OpenAI 호환 모델 gateway이자 관리 평면(management plane)입니다. `/v1/*` OpenAI 호환 REST API, JSON-RPC 2.0 관리 평면(`/api/rpc`), agent 제어 평면(`/ws/agent`) 및 `/docs`의 Swagger UI를 제공합니다.

**번들된 웹 대시보드와 번들 CLI chat은 없습니다.** chat + 관리 UI는 [shittim-chest](https://github.com/celestia-island/shittim-chest)에 있으며, RPC 표면을 통해 Arona와 통신합니다. Arona는 서버 측(라우팅, billing, 인증, 모델 배포, agents, memory)에 집중합니다.

## 기능 매트릭스

| 영역 | Arona가 제공하는 것 |
| --- | --- |
| **대화 코어** | OpenAI 호환 `chat.completions`(스트리밍 + 비스트리밍), `embeddings`, `models` 목록. 터미널 `[DONE]` 청크와 최종 청크의 실제 사용량이 있는 스트리밍. |
| **Backends** | Admin이 등록한 업스트림: `external`(모든 OpenAI 호환 HTTP API), `ollama`, CEP `engine`(WebSocket), `minimax-cloud` 비디오, 산업/엣지 서비스로의 `evernight://` 브리지 URL. |
| **인증** | JWT access/refresh 쌍(15분 / 7일), SHA-256 해시로 저장되는 API keys `arona-{uuid}`, 세 가지 admin 등급, 비밀번호 정책, 이중 트랙 rate limiting. |
| **Billing 및 사용량** | 시드된 tier(free / pro / enterprise), 모든 채널의 요청별 usage records, plana 가격표, 프로젝트별 quota 범위, 429 + `Retry-After`. |
| **모델 관리** | 모델 다운로드(`hf:` / `ms:` / `gh:` 소스), `_agent` GPU 노드 배포, 배포된 모델의 라우팅 가능한 backend 자동 등록. |
| **Realtime 및 멀티모달** | 전이중 `realtime.*` 세션, `engine.invoke` 인지/제어 채널, 비동기 비디오 생성 작업(MiniMax cloud). |
| **Agent 클러스터** | GPU 노드가 `/ws/agent`로 연결, 최소 부하 배치, 세션 고정, 재시작 간 노드 유지. |
| **Memory gateway** | entelecheia Philia를 통한 장기 memory: 회상 주입(recall injection), 에피소드 writeback, 명시적 저하. |
| **운영** | Health probes, `RUST_LOG` 추적, 업스트림 오류 매핑(502 vs 500), 정상 종료(graceful shutdown), 시작 시 자동 마이그레이션. |

## 포지셔닝

Arona는 **gateway + 플랫폼**입니다. 모델 트래픽을 backend로 라우팅하고, 모델을 GPU agents에 배포하며, 모든 것을 계량합니다.

- **pi와의 비교** — pi는 모델과 대화하는 CLI 어시스턴트입니다. arona에는 CLI chat이 없습니다. Arona는 pi(및 기타 도구)가 통신하는 *대상* 플랫폼입니다.
- **one-api / new-api와의 비교** — 이들은 모델 provider 앞에 서는 API-key gateway입니다. arona는 **모델 배포**(가중치 다운로드, agents에서 실행), 완전한 관리 RPC 평면, billing tier와 memory를 추가합니다.
- **LiteLLM과의 비교** — gateway 동료입니다. arona는 추가로 gateway 뒤에 있는 모델의 배포 수명 주기(deployment lifecycle)를 소유합니다.

## 여기서 시작하기

- [빠른 시작](quickstart.md) — 내장 mock 업스트림으로 end-to-end.
- [구성](configuration.md) — 모든 환경 변수.
- [배포](deployment.md) — bare metal, systemd, Docker, supervision.
- [Backends](backends.md) — backend 유형, URL 의미 및 probing.
- [OpenAI 호환 REST API](../api/openai-rest.md) — `/v1/*`.
- [JSON-RPC API](../api/jsonrpc.md) — 전체 관리 평면.

## 저장소 구조

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

웹 대시보드는 이 저장소에서 제거되었으며 이제 [shittim-chest](https://github.com/celestia-island/shittim-chest)(chest #291)에 있습니다.

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
