# Coordinator-Worker-Subagent 패턴 상태 확인

Date: 2026-06-29
Status: 탐색 완료 (미확정 — 이 레포에만 기록)

## 목적

coordinator-worker-subagent 3계층 패턴이 현재 거버넌스/런타임 위에서 동작하는지
상태 확인. 미확정 내용이므로 pattern_test에만 기록하고 거버넌스/런타임 레포에는
반영하지 않는다.

---

## 확인된 레이어 현황

```
Claude Coordinator (Claude Code CLI)
  │
  │  [구현됨] 런타임 계약
  │  CoordinatorOutboundBrief → WorkerReturnPacket → HandoffBriefGroup
  │  (agent-governance-runtime/coordinator_handoff.py)
  │
  │  [미구현] Transport 레이어
  │  codex app-server JSON-RPC → MCP wrapping → A2A
  │
  ▼
Codex Worker Thread (app-server proxy 경유 예정)
  │
  │  [미구현] Worker 내부 subagent 루프
  │  Worker가 foreground subagent를 동기 호출하는 메커니즘
  │
  ▼
Subagents (실제 작업 실행)
```

---

## 레이어별 상세

### 런타임 계약 — 구현됨

- `CoordinatorOutboundBrief`: brief_id 발행, instruction_summary, expected_output_spec
- `WorkerReturnPacket`: brief_id echo, state, implementation_delta, finish_evidence
- `HandoffBriefGroup`: brief_id별 packet 묶음, ack_state 전이
- `execution_mode_policy`: COORDINATOR_WORKER 모드에서 brief_id_required=True
- 참고: `agent-governance-runtime/docs/sdd/coordinator-worker-handoff-contract.md`

### Codex app-server JSON-RPC — 스키마 존재, 연결 미구현

- 스키마 위치: `codex app-server generate-json-schema`로 확인
- `Thread/startRequest` (ThreadStartParams: cwd, baseInstructions, model 등)
- `ThreadInjectItemsParams`: 실행 중인 스레드에 메시지 주입
- 제약: `app-server daemon` 및 `remote-control`이 현재 **Unix 전용** (Windows 미지원)
- Windows에서는 `\\.\pipe\codex-ipc`가 존재하나 app-server proxy 연결 거부

### MCP wrapping / A2A — 미구현

- Coordinator(Claude Code)가 Codex 스레드를 도구로 호출하는 MCP wrapping 없음
- A2A(Agent-to-Agent) transport 레이어 미구현
- SDD 명시: "Codex send_to_message_thread() or any vendor-specific transport is opaque
  to this contract" — transport는 런타임 계약 밖이며 agent_bootstrap TSK-053 영역

### Worker 내부 subagent 루프 — 동작 불가

- Claude Code Agent tool로 Worker를 생성하면 Worker 내부 subagent 완료 알림이
  Worker가 아닌 Coordinator 세션으로 라우팅됨
- Worker self-contained subagent 루프 불가 (foreground 동기 호출 메커니즘 없음)
- 이번 테스트에서 Coordinator가 직접 subagent를 디스패치하는 방식으로 우회

---

## 이번 테스트에서 실제로 검증된 것

- 런타임 계약 형태(brief_id, WorkerReturnPacket) 준수 여부 — **형태 검증 완료**
- GitHub issue를 transport로 사용하는 경우 brief 전달 가능 — **확인**
- Claude Code Agent tool로 3계층 흉내는 가능하나 진짜 3계층은 아님 — **확인**

## 진짜 3계층이 동작하려면 필요한 것

1. `codex app-server` JSON-RPC → MCP wrapping (Windows 포함)
2. MCP를 통한 A2A transport 구현
3. Worker 내부 foreground subagent 호출 메커니즘
4. 위 선행 이슈 정리 후 처리 예정 (현재 deferred)

---

## 산출물 (이번 테스트)

- Brief 이슈: https://github.com/ClarusIubar/pattern_test/issues/1
- WorkerReturnPacket 코멘트: https://github.com/ClarusIubar/pattern_test/issues/1#issuecomment-4828993107
- PR: https://github.com/ClarusIubar/pattern_test/pull/2
- 브랜치: worker/pt-001
- 산출물: docs/README.md, scripts/run.sh, docs/verification-report.md
