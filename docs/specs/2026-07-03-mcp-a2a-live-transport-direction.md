# A2A Live Transport 구현 방향 (스키마 아키텍처 기반 상세 설계)

Date: 2026-07-03 (v2 — 스키마 검증 후 전면 재작성)
Status: draft (미확정 — pattern_test에만 기록)
Baseline: agent-governance-runtime v0.1.1 / Codex app-server protocol v2 (codex 0.142.5)

검증된 사실 위에서 작성함:
- `codex app-server`(stdio)는 **Windows에서 동작 확인** (initialize 응답,
  `platformOs: windows`). Unix 전용은 `daemon`/`remote-control` lifecycle 헬퍼뿐.
- 스키마 전체 표면 추출 완료 (`codex app-server generate-json-schema`).

---

## 1. Codex app-server 스키마 아키텍처 분석

### 1.1 프로토콜 구조 — 3층

```
[1] JSON-RPC envelope    JSONRPCRequest / Response / Notification / Error
[2] Lifecycle 도메인      Thread → Turn → Item  (핵심 객체 모델)
[3] Operation 도메인      Fs/* Command/* McpServer/* Config/* Account/*
                          Skills/* Review/* WindowsSandbox/*
```

### 1.2 핵심 객체 모델: Thread → Turn → Item

```
Thread  (영속, resume/fork/rollback 가능, cwd·sandbox·모델·MultiAgentMode 보유)
  └── Turn  (한 번의 지시 사이클: 시작 → 스트리밍 → 완료)
        └── Item  (agentMessage | commandExecution | fileChange |
                   mcpToolCall | reasoning | plan ...)
```

**이 모델이 A2A 설계의 전부를 결정한다:**

| 스키마 요소 | 의미 | A2A에서의 역할 |
|---|---|---|
| `Thread/start` | worker 스레드 생성 (cwd, baseInstructions, sandbox, approvalPolicy, MultiAgentMode) | **Worker 프로비저닝** |
| `Turn/start` | 스레드에 입력을 넣고 실행 사이클 시작 | **brief 송신 = Turn 하나** |
| `Turn/steer` | 실행 중인 turn에 방향 수정 주입 | **Coordinator발 재지시 (실행 중)** |
| `Turn/interrupt` | turn 중단 | **취소/회수** |
| `Turn/completed` (notif) | turn 종료 + 최종 결과 | **유일한 터미널 신호 = return packet 추출 시점** |
| `Turn/started`, `Item/*` (notif) | 진행 스트림 | 관찰용 (비권위, bounded journal로만 소비) |
| `Turn/diff/updated`, `Turn/plan/updated` | 변경 diff / plan 갱신 | readback 보조 증거 |
| `Thread/injectItems` | turn 밖에서 스레드에 항목 주입 | 다음 turn 전 컨텍스트 보강 |
| `Thread/resume` / `fork` / `rollback` | 스레드 수명 관리 | 재시도·분기 실험·복구 |
| `Review/start` | 네이티브 코드 리뷰 사이클 | Worker의 검증 subagent 대체 경로 |
| `MultiAgentMode` (none/explicitRequestOnly/proactive) | worker의 subagent 위임 정책 | **3계층 스위치** |
| `Thread/status/changed`, `Thread/tokenUsage/updated` | 상태·비용 스트림 | lease/예산 관찰 |
| `WindowsSandbox/*` | Windows 샌드박스 셋업 | Windows가 프로토콜 1급 시민이라는 증거 |

### 1.3 아키텍처적 함의

1. **Turn이 brief 경계다.** 런타임의 `CoordinatorOutboundBrief` 1건 = `Turn/start` 1건.
   brief_id↔turnId 대응이 성립하므로 `HandoffEventKey`의 dedup 차원이 프로토콜
   레벨에서 자연 지원된다.
2. **터미널 신호가 명시적이다.** `Turn/completed`만이 종료다. Item 스트림은 아무리
   그럴듯해도 비권위 — 이는 런타임의 "raw output은 권위가 아니다" 원칙과 정확히 동형.
3. **재지시가 두 종류로 분리된다.**
   - Worker 내부 재지시(subagent 재작업): `MultiAgentMode=proactive`로 Codex 내부에서 해소
   - Coordinator발 재지시: 실행 중이면 `Turn/steer`, 완료 후면 새 `Turn/start`
4. **알림 스트림은 이벤트 소싱이다.** 폴링이 필요 없다(TSK-052-11 정합).
   단, 전량 저장은 bounded 원칙 위반 — 프로젝터가 선별 투영해야 한다.
5. **stdio 스폰이 데몬을 대체한다.** `daemon`은 여러 클라이언트가 한 서버를 공유하기
   위한 편의 기능일 뿐. bootstrap이 워커 채널마다 자식 프로세스를 소유하면
   lifecycle 문제가 사라지고 Windows 제약도 사라진다.

---

## 2. 런타임 계약 ↔ 스키마 바인딩 매핑

| 런타임 계약 (구현 완료) | app-server 스키마 | 바인딩 방법 |
|---|---|---|
| `WorkerBrief` / `CoordinatorOutboundBrief` | `Turn/start` params.input | brief를 envelope 텍스트로 렌더링해 input item으로 투입 |
| `brief_id` | turnId + Thread metadata | `Thread/metadata/update`로 brief_id 스탬프, 이벤트 키에 turnId 병기 |
| `allowed_scope` | `ThreadStartParams.cwd` + `sandbox` + `approvalPolicy` | 스코프를 프롬프트 계약이 아니라 **샌드박스로 강제** (write-set = workspace-write + cwd) |
| `governance_level` | `baseInstructions` / `developerInstructions` | 거버넌스 지시를 스레드 생성 시 고정 주입 |
| `WorkerReturnPacket` / `PortableReturnPayload` | `Turn/completed` + 마지막 agentMessage Item | 최종 메시지의 typed JSON 블록 파싱 → payload 검증 |
| `HandoffEventKey` (brief_id 차원) | turnId + threadId | 프로토콜 ID와 런타임 ID 이중 기록 |
| `HandoffReceipt` 승격 | (스키마 없음 — 우리 소유) | bootstrap store에서 promoted/pending-evidence/blocked/untyped 결정 |
| `HandoffBriefGroup.ack_state` | (스키마 없음 — 우리 소유) | pending→acked→consumed 전이는 coordinator route-queue 소관 |
| `TransportCapabilityReadback` | `initialize` 응답 (`platformOs`, `userAgent`) | 스폰+handshake 성공/실패를 typed readback으로 |
| Worker 내부 subagent 루프 | `MultiAgentMode=proactive`, `Review/start` | 런타임 코드 불필요 — 설정과 brief 계약으로 해소 |
| changed_files readback | `Turn/diff/updated` | diff 알림을 bounded 요약으로 투영 |
| 예산/lease 관찰 | `Thread/tokenUsage/updated` | 토큰 사용 초과 시 `Turn/interrupt` 트리거 가능 |

**경계 유지**: 런타임(agent-governance-runtime)은 이 표의 왼쪽만 소유하고 계속
transport-opaque로 남는다. 오른쪽 바인딩 전체가 bootstrap 신규 모듈이다.
runtime에는 코드 변경이 필요 없다 — 이것이 SDD의 "bootstrap consumes runtime;
runtime does not import bootstrap" 방향과 일치.

---

## 3. 채널 결정 (v1 문서에서 수정)

| | v1 판단 | v2 판단 (스키마 검증 후) |
|---|---|---|
| 기본 채널 | CODEX_MCP_SERVER (동기 tool call) | **CODEX_APP_SERVER (stdio 스폰)** |
| 이유 | Windows에서 app-server 불가로 오판 | stdio 스폰은 Windows 동작 확인. MCP tool call은 Turn 스트림·steer·interrupt·다중 워커가 없어 A2A 요건 미달 |
| CODEX_MCP_SERVER | 기본 | 단순 폴백 (단발 위임, 알림 불필요할 때) |
| ISSUE_ARTIFACT | 폴백 | 최후 폴백 유지 (pt-001 검증됨) |

A2A의 요건 — 비동기 진행 관찰, 실행 중 재지시, 중단, 워커 N개 동시 운용 —
은 app-server 스키마만 충족한다. MCP 서버 채널은 coordinator를 tool call 동안
블로킹시키므로 다중 워커 orchestration에 부적합.

---

## 4. 구현 컴포넌트 (agent_bootstrap 신규 모듈)

```
bootstrap_codex_appserver_client.py      [L0] JSON-RPC 클라이언트
  - stdio 자식 프로세스 스폰/종료 (워커 채널당 1프로세스)
  - request/response 상관 (id), notification 디스패치
  - initialize handshake → TransportCapabilityReadback 생성

bootstrap_worker_channel_supervisor.py   [L1] 채널 수명 관리
  - Thread/start로 worker 프로비저닝 (cwd, sandbox, MultiAgentMode=proactive,
    baseInstructions=거버넌스 계약)
  - Thread/metadata/update로 brief_id·task_id 스탬프
  - 프로세스 사망/Thread/closed 감지 → 채널 DEGRADED readback

bootstrap_brief_turn_binder.py           [L2] 발신 바인딩
  - WorkerBrief → Turn/start input 렌더링 (echo 계약 포함:
    "최종 메시지는 PortableReturnPayload JSON 블록으로 종료, brief_id echo")
  - Coordinator발 재지시 → 실행 중이면 Turn/steer, 완료 후면 새 Turn/start
  - 취소 → Turn/interrupt

bootstrap_turn_event_projector.py        [L2] 수신 투영
  - 알림 스트림 → 선별 투영 (bounded):
    Turn/started → brief 상태 transmitted→streaming
    Turn/diff/updated → changed_files 요약 갱신
    Thread/tokenUsage/updated → 예산 관찰, 임계 초과 시 interrupt 신호
    Turn/completed → return 추출 트리거
  - HandoffEventKey(brief_id, turnId 병기)로 route-queue 이벤트 생성
  - Item/* delta는 저장하지 않음 (bounded 원칙; 필요 시 Thread/read로 재조회)

bootstrap_return_packet_extractor.py     [L3] 승격
  - Turn/completed 시 최종 agentMessage에서 JSON 블록 파싱
  - PortableReturnPayload 검증 (runtime 검증기 호출, 재구현 금지)
  - HandoffReceipt 결정:
    brief_id echo 일치 + evidence 충족 → promoted
    payload 유효하나 증거 부족     → pending-evidence
    blocker 존재                 → blocked
    파싱 불가                    → untyped (오류가 아닌 typed 차단 상태)
  - untyped 시 자동 재지시 1회 (Turn/start: "echo 계약 위반, 형식 준수 재반환")
    → 재실패 시 blocked 확정

bootstrap_transport_capability_probe.py  [L0] 탐침
  - codex.exe 경로 glob 해석 (해시 디렉토리 자동 변경 대응 — 하드코딩 금지)
  - 스폰 + initialize + Thread/start(ephemeral) + 폐기 스모크
  - 채널별 TransportCapabilityReadback 산출
```

의존 방향: L3 → L2 → L1 → L0, 전 계층이 runtime 계약을 소비만 함.

---

## 5. Brief 수명 상태 머신

```
drafted ──Turn/start──▶ transmitted ──Turn/started──▶ streaming
                                                        │
                              ┌─── Turn/steer (재지시) ──┤
                              ▼                         │
                           streaming ◀──────────────────┘
                                                        │
                                                 Turn/completed
                                                        ▼
                                                    returned
                                                        │ extractor
              ┌──────────────┬──────────────┬───────────┤
              ▼              ▼              ▼           ▼
          promoted    pending-evidence   blocked     untyped
              │              │              │           │ 재지시 1회
              │              │              │           └──▶ streaming | blocked
              ▼
        acked ──▶ consumed        (HandoffBriefGroup — coordinator 소관,
                                   consumed 후에만 downstream 허용)
```

실패 경로 (fail-closed):
- 프로세스 사망 / `Thread/closed` 수신 → brief는 `blocked(channel-lost)`,
  채널 readback DEGRADED. `Thread/resume`으로 복구 시도는 새 brief로 취급.
- `Error` notification → bounded 요약과 함께 blocked.
- 토큰 예산 초과 → `Turn/interrupt` → blocked(budget-exceeded).
- 어떤 경로도 자동으로 GitHub 완료 권위를 만들지 않음 —
  `state="finish-applied"`조차 TSK-006 `FinishReadbackDecision` 없이는 비권위.

---

## 6. 시퀀스 — 3계층 정상 사이클

```
Coordinator                bootstrap 채널               Codex worker (app-server)
    │  WorkerBrief(pt-002)      │                            │
    ├───────────────────────────▶ Thread/start(cwd, proactive)│
    │                           ├────────────────────────────▶ threadId
    │                           │ Turn/start(brief envelope)  │
    │                           ├────────────────────────────▶
    │                           │◀─ Turn/started ─────────────┤
    │                           │◀─ Item/* (subagent 위임,    │  ← worker가 내부에서
    │                           │    커밋, 검증...)            │    subagent 생성/검증/재지시
    │                           │◀─ Turn/diff/updated ────────┤    (proactive 모드, 알림은
    │                           │◀─ Turn/completed ───────────┤     채널 내부에서만 소비)
    │                           │ extractor: payload 파싱     │
    │                           │ HandoffReceipt=promoted     │
    │◀── route-queue event ─────┤                            │
    │  ack → consumed           │                            │
    │  (GitHub readback으로     │                            │
    │   완료 권위 별도 확인)      │                            │
```

pt-001의 GAP-01(서브에이전트 알림이 coordinator로 누출)은 이 구조에서 원천 제거:
subagent lifecycle이 Codex 프로세스 내부에 있고, coordinator에 도달하는 것은
채널이 투영한 route-queue 이벤트뿐이다.

---

## 7. 플랫폼 매트릭스 (검증 기반, v1에서 수정)

| 채널 | win32 | unix | 근거 |
|---|---|---|---|
| CODEX_APP_SERVER (stdio 스폰) | **available** | available | initialize 응답 실측 (win32) |
| CODEX_APP_SERVER (daemon 공유) | unavailable | available | daemon lifecycle Unix 전용 (실측) |
| CODEX_MCP_SERVER | available | available | mcp-server 기동 확인 (win32) |
| ISSUE_ARTIFACT | available | available | pt-001 검증 |
| CLI_SDK (`codex exec`) | available | available | CLI 표면 확인 |

---

## 8. 단계 분해 (이슈화 대기 — 미확정)

| 단계 | 내용 | 완료 기준 |
|---|---|---|
| P1 | L0: appserver_client + capability probe | win32/unix에서 스폰→initialize→Thread/start(ephemeral)→폐기 스모크 + TransportCapabilityReadback 산출 |
| P2 | L1: channel supervisor | Thread 프로비저닝/메타데이터 스탬프/사망 감지 테스트 |
| P3 | L2: brief_turn_binder + event_projector | brief→Turn 발신, 알림→bounded 투영, steer/interrupt 경로 테스트 |
| P4 | L3: return_packet_extractor | promoted/pending/blocked/untyped 4상태 + untyped 재지시 1회 테스트 |
| P5 | pt-002 실검증 (pattern_test) | §6 시퀀스 전체를 win32에서 완주, GAP-01 부재 증명 |
| P6 | coordinator route-queue 접합 + TSK-053 배선 | ack 전이와 downstream 게이트 통합 |

각 단계는 fixture 우선(런타임 테스트 원칙과 동일) — L0만 실프로세스 스모크,
L1~L3은 녹화된 알림 스트림 fixture로 단위 테스트.

---

## 9. 리스크

| 리스크 | 완화 |
|---|---|
| app-server가 experimental — 스키마 파손 가능 | probe가 `generate-json-schema` 스냅샷을 버전 스탬프와 함께 보관, 필드 드리프트 감지 시 typed 차단 |
| 최종 메시지 JSON 블록 파싱의 취약성 | echo 계약을 baseInstructions에 고정 + untyped 재지시 1회 + Turn/diff는 독립 증거로 교차 확인 |
| MultiAgentMode 미노출/무시 가능성 | P5에서 subagent 생성 여부를 Item 스트림으로 실측; 미동작 시 worker가 순차 자체 수행해도 계약은 동일 (성능 갭으로만 기록) |
| 알림 폭주로 bounded 위반 | projector가 화이트리스트 알림만 투영, Item delta 무저장 |
| codex.exe 자동 업데이트로 경로/버전 변동 | glob 해석 + initialize의 버전 문자열을 capability readback에 기록 |
| 프로세스 누수 (워커당 1 자식 프로세스) | supervisor가 소유권 보유, brief 종결 시 Thread/archive + 프로세스 종료, 시작 시 고아 스캔 |
```
