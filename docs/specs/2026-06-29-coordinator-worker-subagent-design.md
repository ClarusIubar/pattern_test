# Coordinator-Worker-Subagent Pattern Test Design

Status: draft (미확정 — pattern_test 격리 필드에서 검증 중)

## 목적

`agent-governance-runtime`의 `coordinator-worker` 실행 모드가 실제로 3계층으로
동작하는지 검증한다. 거버넌스는 아직 적용하지 않으나, 런타임 계약
(`CoordinatorOutboundBrief`, `WorkerReturnPacket`, `HandoffBriefGroup`)은 준수한다.

---

## 계층 구조

```
인간
 ↕ (전략 보고만)
Claude Coordinator
 │  CoordinatorOutboundBrief (brief_id 발행)
 ▼
Codex Worker (책임 도메인 관리자)
 ├── Subagent A → 문서 작성
 ├── Subagent B → 스크립트 생성
 ├── Subagent C → 검증
 │
 ├── [검증 실패 시] 재지시 → Subagent (재작업, Worker 루프 내부 해결)
 └── [후속 작업 / 전략 판단 필요 시] WorkerReturnPacket → Coordinator
```

### 계층별 책임

| 계층 | 역할 | 반환 대상 |
|---|---|---|
| Coordinator | brief 발행, return packet ack, 인간 보고 | 인간 |
| Worker | subagent 생성·검증·재지시, 루프 자체 완결 | Coordinator (결과/후속/전략) |
| Subagent | 실제 작업 실행 | Worker |

**Worker → Coordinator 반환 유형:**
- 작업 완료 보고 (산출물 포함)
- 후속 작업 요청 (자신의 책임 범위 밖)
- 전략 판단 요청 (방향 결정이 coordinator 권한)

Worker는 실패를 coordinator에 올리지 않는다. 책임 범위 내 재작업은 Worker가 자체 해결한다.

---

## 런타임 계약 (agent-governance-runtime 준수)

### Coordinator → Worker

```json
{
  "brief_id": "pt-001",
  "task": { "issue": "<issue-url>", "branch": "<branch>" },
  "coordinator_turn_id": "<turn-id>",
  "instruction_summary": "<4000자 이내>",
  "expected_output_spec": "<WorkerReturnPacket 형태 명시>",
  "allowed_scope": ["docs/", "scripts/"]
}
```

### Worker → Coordinator

```json
{
  "brief_id": "pt-001",
  "worker_id": "<worker-id>",
  "finish_disposition": "handoff | terminal",
  "summary": "<완료 내용 또는 후속 요청>",
  "artifacts": ["docs/README.md", "scripts/run.sh", "docs/verification-report.md"]
}
```

### Coordinator ack

`HandoffBriefGroup` — brief_id별로 묶인 패킷을 `consumed` 상태로 전환 후 인간에게 보고.

---

## 미니멀 테스트 시나리오

### 태스크: "이 레포의 구조와 목적을 설명하는 산출물 세트 작성"

**Worker 책임 도메인:** 문서·스크립트·검증 통합 산출물 생성

**Worker가 생성할 Subagent:**
- **Subagent A**: `docs/README.md` 작성 (레포 목적, 구조, 사용법)
- **Subagent B**: `scripts/run.sh` 작성 (README 내용과 정합하는 실행 스크립트)
- **Subagent C**: 검증 (README ↔ script 정합성 확인, `docs/verification-report.md` 작성)

**Worker 내부 루프:**
1. A, B 병렬 실행
2. C 실행 (A, B 산출물 기준)
3. 검증 실패 시 → 해당 subagent 재지시 (Worker 내부 해결)
4. 검증 통과 시 → WorkerReturnPacket 생성 → Coordinator 반환

**Coordinator 최종 보고 항목:**
- 완료된 산출물 목록
- 후속 작업 필요 여부
- 갭 발견 시 → `agent_bootstrap` + `agent-governance-runtime` 이슈 링크

---

## 갭 처리 기준

테스트 중 다음이 발생하면 해당 레포에 이슈를 생성한다:

| 갭 유형 | 이슈 생성 위치 |
|---|---|
| Worker가 brief_id 없이 실행됨 | agent-governance-runtime |
| Subagent 생성 계층이 Worker에서 이루어지지 않음 | agent-governance-runtime |
| Coordinator가 Worker 내부에 직접 개입함 | agent_bootstrap |
| return_transport / coordinator_ack 미이행 | agent-governance-runtime |
| Worker 루프 내 재지시가 Coordinator까지 올라옴 | agent_bootstrap |

---

## 성공 기준

- [ ] Coordinator가 brief_id를 발행하고 Worker에 전달
- [ ] Worker가 3개 subagent를 생성·실행·검증 (내부 루프 완결)
- [ ] WorkerReturnPacket이 brief_id를 유지한 채 Coordinator에 반환
- [ ] Coordinator가 HandoffBriefGroup을 ack하고 인간에게 보고
- [ ] 발견된 갭이 이슈로 포착됨

---

## 참고

- Runtime spec: `agent-governance-runtime/src/agent_governance_runtime/coordinator_handoff.py`
- Execution modes: `agent-governance-runtime/src/agent_governance_runtime/execution_modes.py`
- Models: `agent-governance-runtime/src/agent_governance_runtime/models.py`
