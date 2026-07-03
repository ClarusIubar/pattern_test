# pattern_test

coordinator-worker-subagent 3계층 패턴 테스트 필드

## 레포 목적

이 레포지토리는 coordinator-worker-subagent 3계층 패턴을 실험하고 검증하기 위한 테스트 필드입니다.

## 계층 구조

### Coordinator (최상위 계층)
전체 작업을 계획하고 brief를 발행합니다. 작업의 목표와 범위를 정의하며, worker에게 brief를 전달합니다.

### Worker (중간 계층)
Coordinator로부터 brief를 받아 subagent를 지시하고 검증합니다. 각 subagent의 작업을 조율하고 결과를 수집합니다.

### Subagent (실행 계층)
실제 작업(파일 작성, 스크립트 생성, 검증 등)을 수행합니다. Worker의 지시에 따라 구체적인 태스크를 실행합니다.

## 디렉토리 구조

```
pattern_test/
├── docs/       # 문서 파일
└── scripts/    # 실행 스크립트
```

## 사용법

- **brief_id**: pt-001
- **worker_id**: worker-A
- **subagents**:
  - subagent-A: docs 담당
  - subagent-B: scripts 담당
  - subagent-C: verification 담당

실행:

```bash
bash scripts/run.sh
```
