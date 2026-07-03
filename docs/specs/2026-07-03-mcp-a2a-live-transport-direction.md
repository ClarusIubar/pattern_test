# MCP A2A Live Transport 구현 방향

Date: 2026-07-03
Status: draft (미확정 — pattern_test에만 기록, 거버넌스/런타임 반영 전 검토용)
Baseline: agent-governance-runtime v0.1.1 (TSK-005 CLOSED, TSK-006 closeable-milestone)

---

## 1. 전제 정정 (2026-06-29 status-check 대비)

| 이전 판단 | 정정 |
|---|---|
| MCP wrapping 미구현 | **계약 레벨은 구현 완료.** TSK-005-01~05가 `AgentExecutionChannel`, `TransportCapabilityReadback`, `WorkerBrief`, `PortableReturnPayload`, `HandoffReceipt`를 릴리즈함 |
| A2A transport 미구현 | **의도된 non-goal.** TSK-005 부모 이슈에 "live external A2A network protocol / worker spawning / Codex subagent lifecycle은 비목표"로 명시. 이번 단계가 바로 그 live transport 구현 |
| Windows에서 Codex 제어 불가 | **부분 정정.** `app-server daemon` lifecycle만 Unix 전용. `codex mcp-server`(stdio)는 Windows에서 동작 확인됨 |

## 2. 이미 존재하는 두 층

### 2.1 런타임 계약층 (agent-governance-runtime, 구현 완료)

```
CoordinatorOutboundBrief ──┐            (TSK-001: brief_id 사이클)
WorkerReturnPacket         ├─ 실행 계약
HandoffBriefGroup          ┘
AgentExecutionChannel ─────┐            (TSK-005: 채널/수송 계약)
TransportCapabilityReadback│
WorkerBrief                ├─ 수송 계약
PortableReturnPayload      │
HandoffReceipt             ┘
CompletionReceipt ─────────┐            (TSK-006: 완료 계약)
FinishReadbackDecision     ┘
```

핵심 규칙 (코드로 강제됨):
- 채널 분류: portable = `CODEX_MCP_SERVER`, `CODEX_APP_SERVER`, `ISSUE_ARTIFACT`, `CLI_SDK` /
  fast-path(비권위) = `CODEX_THREAD_TOOL`, `CODEX_SUBAGENT`
- fast-path 채널은 `TransportStatus.AVAILABLE`로 보고되면 blocker (권위로 승격 금지)
- 모든 채널이 `requires_typed_payload=True`, `raw_output_authority=False`
- raw MCP content / threadId / subagent 직접 출력은 `HandoffReceipt` 승격
  (promoted | pending-evidence | blocked | untyped) 없이는 아무것도 아님

### 2.2 Codex 스키마층 (app-server JSON-RPC v2, 로컬 확인)

- `Thread/start` (`ThreadStartParams`): `cwd`, `baseInstructions`, `developerInstructions`,
  `model`, `sandbox`, `approvalPolicy`, **`MultiAgentMode`**
- `ThreadInjectItems`: 실행 중 스레드에 항목 주입 (재지시 경로)
- `ThreadRead` / `ThreadList` / `ThreadResume` / `ThreadFork`
- 알림: `ThreadStartedNotification`, `ThreadStatusChangedNotification`,
  `AgentMessageDeltaNotification`, `ThreadTokenUsageUpdatedNotification`
- **`MultiAgentMode`**: `none` | `explicitRequestOnly` | `proactive`
  → Codex worker가 **자체 subagent를 네이티브로 생성**하는 스위치.
  3계층의 "Worker 내부 subagent 루프"는 새로 만드는 게 아니라 이 모드를 켜는 것

## 3. 채널 아키텍처 결정

```
                       Claude Coordinator
                             │
              ┌──────────────┼───────────────────┐
              ▼              ▼                   ▼
   [1] CODEX_MCP_SERVER  [2] CODEX_APP_SERVER  [3] ISSUE_ARTIFACT
       stdio, 전 플랫폼      JSON-RPC, Unix 전용     GitHub 이슈, 전 플랫폼
       동기 요청/응답        스레드 생명주기+알림       비동기, 사람 개입 가능
       ── 기본 채널 ──      ── 승격 채널 ──          ── 폴백 채널 ──
              │
              ▼
        Codex Worker Thread (MultiAgentMode=proactive)
              │  Worker 자체 판단으로 생성/검증/재지시
              ├── Codex Subagent (문서)     ┐
              ├── Codex Subagent (스크립트)  ├ CODEX_SUBAGENT fast-path
              └── Codex Subagent (검증)     ┘ raw 출력은 비권위
              │
              ▼
        Worker가 subagent 출력을 evidence_refs로 집계
              │
              ▼
        PortableReturnPayload (brief_id echo) → HandoffReceipt 승격
```

- **기본 채널 = `CODEX_MCP_SERVER`**: `codex mcp-server`(stdio)를 Claude Code MCP 서버로
  등록. Windows 포함 전 플랫폼. 데몬 불필요. pt-001에서 확인된 "완료 알림이 Coordinator로
  새는 문제"가 구조적으로 없음 — MCP 호출은 동기이며 subagent lifecycle이 Codex 프로세스
  내부에 격리됨
- **승격 채널 = `CODEX_APP_SERVER`**: 스레드 재개/포크/주입/알림 스트림이 필요할 때.
  Unix에서 먼저 살리고, Windows는 upstream 데몬 lifecycle 지원 전까지
  `TransportCapabilityReadback(status=UNAVAILABLE, platform="win32")`로 typed 차단
- **폴백 채널 = `ISSUE_ARTIFACT`**: pt-001에서 이미 검증한 GitHub 이슈 brief/return 경로.
  transport 장애 시에도 거버넌스 사이클이 멈추지 않는 최후 경로

## 4. 구현 단계

### Phase A — Transport Capability Probe (bootstrap)

새 모듈 `bootstrap_transport_capability_probe.py`:
- 각 채널을 실제로 탐침: `codex mcp-server` 기동 가능 여부, app-server 소켓 연결,
  `gh` 인증 상태
- 결과를 런타임 `TransportCapabilityReadback`으로 생성 (bootstrap은 소비만, 스키마 발명 금지)
- 플랫폼별 기대값 고정:

| 채널 | win32 | unix |
|---|---|---|
| CODEX_MCP_SERVER | available | available |
| CODEX_APP_SERVER | unavailable (daemon lifecycle) | available |
| ISSUE_ARTIFACT | available | available |
| CLI_SDK | available (`codex exec`) | available |

- 주의: Codex 바이너리 경로가 자동 업데이트로 해시 디렉토리 변경됨
  (`bin/aec6b7c6.../codex.exe` → `bin/ea1c6031.../codex.exe` 확인). 경로 하드코딩 금지,
  탐침 시점에 glob 해석

### Phase B — WorkerBrief MCP 바인딩 어댑터 (bootstrap)

새 모듈 `bootstrap_codex_mcp_channel.py`:
1. **등록**: Claude Code MCP 설정에 `codex mcp-server` 등록
   (`command: <resolved codex.exe>, args: ["mcp-server"]`)
2. **발신**: 런타임 `WorkerBrief`(brief_id, issue_url, task_id, scope_id, instruction,
   expected_output, channel=CODEX_MCP_SERVER, governance_level)를 MCP tool 호출의
   프롬프트 envelope로 렌더링. envelope에 echo 계약 포함:
   "최종 메시지는 반드시 PortableReturnPayload JSON 블록으로 끝나야 하며
   brief_id를 그대로 echo한다"
3. **cwd/모드 주입**: `-c` config override로 워크스페이스 cwd, sandbox,
   `MultiAgentMode` 상당 설정 전달
4. **수신**: MCP tool 결과(content)를 `PortableReturnPayload`로 파싱.
   파싱 실패 = `HandoffReceipt(untyped)` — 오류가 아니라 typed 차단 상태
5. **승격**: payload 검증 → `HandoffReceipt` 상태 결정
   - brief_id echo 일치 + evidence_refs 유효 → `promoted`
   - 내용은 있으나 증거 부족 → `pending-evidence`
   - blocker 존재 → `blocked`

### Phase C — Worker 관리형 Subagent (설정, 코드 아님)

- Worker 스레드 기동 시 `MultiAgentMode=proactive` (또는 mcp-server 채널의 동등 config)
- WorkerBrief instruction에 subagent 위임 계약 명시:
  "하위 작업은 subagent에 위임하라. subagent 원출력은 반환하지 말고,
  검증·재지시 후 집계 결과만 evidence_refs로 승격하라"
- pt-001 갭(GAP-01)은 여기서 해소: subagent 완료 통지가 Codex 프로세스 내부에서
  소비되므로 Coordinator로 새지 않음
- `CODEX_SUBAGENT` 채널 정책이 이미 fast-path 비권위로 고정되어 있으므로
  런타임 변경 불필요

### Phase D — app-server 승격 채널 (Unix 먼저, Windows 대기)

새 모듈 `bootstrap_codex_app_server_channel.py`:
1. Unix: `codex app-server daemon start` → `app-server proxy`로 JSON-RPC
   `Thread/start` (cwd, baseInstructions=WorkerBrief envelope)
2. `ThreadStatusChangedNotification` 수신 → `HandoffEventKey`(brief_id 포함)로
   coordinator route-queue 이벤트화 — 폴링 제거 (TSK-052-11 정합)
3. 재지시: `ThreadInjectItems`로 실행 중 Worker에 후속 brief 주입
4. Windows: probe가 unavailable을 기록하고 CODEX_MCP_SERVER로 자동 폴백.
   upstream 지원 추가 시 probe만 갱신되면 채널이 자동 개방됨

### Phase E — 거버넌스 배선 (agent_bootstrap, TSK-053 축)

- coordinator route-queue가 `HandoffReceipt(promoted)`만 소비:
  ack 전이 `pending → acked → consumed` 후 downstream 허용
- `WorkerReturnPacket.state="finish-applied"`는 GitHub readback
  (`CompletionReceipt`/`FinishReadbackDecision`) 없이는 완료 권위가 아님 — TSK-006 계약 소비
- `context start`/`finish` CLI 배선은 기존 TSK-053 로드맵 유지

## 5. 검증 계획 (pattern_test에서 pt-002)

pt-001과 동일한 미니멀 태스크(문서/스크립트/검증)를 **채널만 바꿔** 재실행:

1. **pt-002a**: CODEX_MCP_SERVER 채널로 WorkerBrief 발신 → Codex worker가
   proactive 모드로 subagent 3개 생성 → PortableReturnPayload 수신 → promoted 승격
2. **pt-002b**: 동일 brief를 ISSUE_ARTIFACT 폴백으로 재실행 — 채널 교체가
   계약을 깨지 않음을 증명
3. **성공 기준**:
   - Coordinator 세션에 subagent 완료 통지가 도달하지 않음 (GAP-01 해소 증명)
   - brief_id가 발신→echo→receipt까지 유지
   - raw MCP content가 어디서도 권위로 사용되지 않음 (HandoffReceipt 경유 강제)
   - Windows에서 전체 사이클 완주
4. 갭 발견 시: 계약 갭은 runtime, 배선 갭은 bootstrap, transport 갭은 pattern_test에
   먼저 기록 후 확정되면 이슈화

## 6. 리스크

| 리스크 | 완화 |
|---|---|
| Codex 바이너리 해시 경로 변동 | probe 시점 glob 해석, 경로 비영속 |
| mcp-server tool 스키마가 Codex 릴리즈마다 변동 | Phase A probe에 tool 스키마 스냅샷 포함, 변동 시 typed 차단 |
| MultiAgentMode 설정이 mcp-server 채널에 미노출일 가능성 | pt-002a에서 최우선 검증; 미노출이면 app-server 채널로 3계층 검증을 이관하고 upstream 이슈화 |
| Worker가 echo 계약을 어기고 자유 텍스트 반환 | untyped receipt로 차단 → 재지시 1회 → 그래도 실패면 blocked |
| Windows app-server 공백 장기화 | 기본 채널이 MCP stdio이므로 사이클 자체는 비차단 |
