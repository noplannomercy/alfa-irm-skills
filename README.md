# ALFA/IRM 런북 어시스턴트

> AI 컨설팅 프레임워크(ALFA + IRM)를 실전 고객 미팅에서 사용할 수 있는 Claude Code 스킬 6개

## 개요

고객 미팅 전 준비부터 미팅 후 산출물 생성, 유즈케이스 도출, IRM 기술 설계까지 **슬래시 커맨드 6개로 풀사이클**을 자동화합니다.

```
/alfa-prep → 미팅 → /alfa-live → /alfa-wrap → /irm-init → /irm-design → /irm-validate
```

---

## 설치

### 1. 스킬 파일 복사

`skills/` 폴더의 6개 디렉토리를 Claude Code 스킬 경로에 복사합니다.

```bash
# Windows
cp -r skills/* %USERPROFILE%\.claude\skills\

# macOS/Linux
cp -r skills/* ~/.claude/skills/
```

### 2. 설치 확인

Claude Code에서 슬래시 커맨드가 인식되는지 확인합니다.

```
/alfa-prep
/alfa-live
/alfa-wrap
/irm-init
/irm-design
/irm-validate
```

6개가 모두 스킬 목록에 나타나면 설치 완료입니다.

---

## 워크플로우

```
0. 고객 접점 (스킬 밖)
   └─ 미팅, 이메일, 소개서 등으로 고객 정보 획득

1. /alfa-prep — 질문 리스트 + 사전 요청서
   ├─ 고객 정보 입력 → 6항목 프로필 초안
   ├─ 빈칸 기반 질문 리스트 생성 (meeting/prep_질문리스트.md)
   ├─ 사전 요청서 생성 (meeting/prep_사전요청서.md)
   │   ├─ A. 담당자용 질의 (프로필 빈칸)
   │   └─ B. IT팀용 리소스 조사 요청
   └─ STATUS.md + 폴더 구조 생성

2. 질문 리스트 정제 + 고객 발송 (스킬 밖)
   └─ IT팀 리소스 조사 시작 (병렬)

3. 고객 미팅 (스킬 밖)

4. /alfa-live — 미팅 결과 반영
   ├─ 미팅 메모/녹음 텍스트 입력
   ├─ 프로필 업데이트 + 신호 감지
   ├─ 회차별 script 저장
   └─ 빈칸 있으면 → 2~4 반복

5. /alfa-wrap — 산출물 생성
   ├─ 회차별 고객용 서머리 + 내부 정리
   ├─ ALFA Phase 문서 (phase0_why, phase1_assess, phase2_govern)
   ├─ ALFA 입력 프로필 확정
   ├─ 유즈케이스 정의서 (usecase/YYYYMMDD_이름.md)
   ├─ IRM 투입 판단
   └─ 리소스 미수령 시 → 수령 후 재실행하여 보강

6. /irm-init — 리소스 분류 + 질의 매핑
   ├─ 유즈케이스 정의서에서 리소스 로드 (✅실제/⚠️추정 구분)
   ├─ Q1(위치)/Q2(절차)/Q3(결정적) 분류
   ├─ 예상 질의 매핑 + 행위 유형 분포
   ├─ GraphRAG 사전 신호
   └─ 신뢰도 전파 시작

7. /irm-design — 배치 설계
   ├─ 탐색: 컬렉션 설계 + GraphRAG 4조건 판단
   ├─ 접근: CLI + REST 설계
   ├─ 실행(확정): 파이프라인 설계
   ├─ 실행(유연): LLM + Human-in-the-loop
   ├─ 체이닝 설계
   └─ 신뢰도 전파 (Phase 0 → Phase 1)

8. /irm-validate — 구현 설계 + 검증
   ├─ 행위 유형별 구현 체크리스트
   ├─ 검증 기준 (정량)
   ├─ PoC 범위 + 성공 기준
   ├─ 구현 계획 (순서, 일정, 인력, 예산)
   └─ ✅먼저 착수, ⚠️검증 후 착수 우선순위

9. 구현 (스킬 밖)

10. 유즈케이스 추가 시 → 4번으로
```

---

## 스킬 상세

### ALFA 스킬 (고객 미팅)

| 스킬 | 커맨드 | 입력 | 출력 |
|------|--------|------|------|
| 미팅 준비 | `/alfa-prep` | 고객 정보 (아는 만큼) | 질문 리스트, 사전 요청서, 프로필 초안 |
| 미팅 반영 | `/alfa-live` | 미팅 메모/녹음 텍스트 | 프로필 업데이트, 신호 감지, script 저장 |
| 산출물 생성 | `/alfa-wrap` | 프로필 + script | 서머리, Phase 문서, 유즈케이스, IRM 판단 |

### IRM 스킬 (기술 설계)

| 스킬 | 커맨드 | 입력 | 출력 |
|------|--------|------|------|
| 리소스 분류 | `/irm-init` | 유즈케이스 정의서 | 인벤토리, 분류, 질의 매핑 |
| 배치 설계 | `/irm-design` | Phase 0 결과 | 탐색/접근/실행 설계, 체이닝 |
| 구현 설계 | `/irm-validate` | Phase 1 결과 | 체크리스트, 검증 기준, 구현 계획 |

---

## 폴더 구조

각 고객별로 아래 구조가 자동 생성됩니다.

```
[회사명]/
├── STATUS.md                          ← 현재 상태 한눈에
├── meeting/
│   ├── prep_질문리스트.md             ← /alfa-prep
│   ├── prep_사전요청서.md             ← /alfa-prep
│   ├── 1st_script.md                  ← /alfa-live
│   ├── 1st_summary.md                 ← /alfa-wrap (고객용)
│   ├── 1st_internal.md                ← /alfa-wrap (내부용)
│   └── ...
├── alfa/
│   ├── profile.md                     ← /alfa-prep → /alfa-live 업데이트
│   ├── alfa_input_profile.md          ← /alfa-wrap (확정)
│   ├── phase0_why.md                  ← /alfa-wrap (버전 관리)
│   ├── phase1_assess.md               ← /alfa-wrap (버전 관리)
│   └── phase2_govern.md               ← /alfa-wrap (버전 관리)
├── usecase/
│   └── YYYYMMDD_[이름].md             ← /alfa-wrap (계속 추가 가능)
└── irm/
    └── [유즈케이스명]/
        ├── phase0_resource_inventory.md   ← /irm-init
        ├── phase0_classification.md       ← /irm-init
        ├── phase0_query_mapping.md        ← /irm-init
        ├── phase1_search.md               ← /irm-design
        ├── phase1_access.md               ← /irm-design
        ├── phase1_execution.md            ← /irm-design
        ├── phase1_chaining.md             ← /irm-design
        ├── phase1_summary.md              ← /irm-design
        ├── phase2_search_checklist.md     ← /irm-validate
        ├── phase2_access_checklist.md     ← /irm-validate
        ├── phase2_execution_checklist.md  ← /irm-validate
        ├── phase2_validation.md           ← /irm-validate
        └── phase2_implementation_plan.md  ← /irm-validate
```

---

## 핵심 기능

### 1. 업종별 자동 분기

같은 스킬이 업종에 따라 다른 결과를 냅니다.

| | 병원 (의료) | 물류 |
|---|---|---|
| 미팅 횟수 | 3회 | 1회 |
| 거버넌스 | Heavy | Light |
| GraphRAG | 승인 (3/4) | 불필요 (0/4) |
| 일정 | 12주 | 4주 |

### 2. 신뢰도 전파 (✅/⚠️)

리소스의 진짜/추정 여부가 IRM 끝까지 전파됩니다.

```
리소스 ✅실제 → 분류 ✅ → 질의 ✅ → 설계 ✅ → 구현 ✅ (바로 착수)
리소스 ⚠️추정 → 분류 ⚠️ → 질의 ⚠️ → 설계 ⚠️ → 구현 ⚠️ (검증 후 착수)
```

### 3. Phase 문서 점진적 보강

ALFA Phase 문서는 미팅 회차마다 점진적으로 채워집니다.

```
상태: 초안 (v1) → 부분 (v2) → 확정 (v3)
```

변경 이력이 버전별로 기록되어 "왜 바뀌었는지" 추적 가능합니다.

### 4. 사전 요청서

`/alfa-prep`에서 고객에게 보낼 숙제를 자동 생성합니다.

- **A. 담당자용** — 프로필 빈칸 채우기 질의
- **B. IT팀용** — 문서/시스템/DB/보안 리소스 조사

미팅 전 발송하여 IT팀 조사를 선행시키면 IRM 투입까지 시간이 단축됩니다.

### 5. 유즈케이스 확산

하나의 고객에서 유즈케이스를 계속 추가할 수 있습니다. 각 유즈케이스는 독립적으로 IRM을 돌리되, 기존 인프라 재사용을 자동 판단합니다.

```
CS 상담 보조 (1차) → 인프라 전부 신규
입출고 관리 보조 (2차) → Bedrock/pgvector 재사용, Neo4j만 신규
```

---

## 사용 예시

### 예시 1: 물류회사 CS 상담 보조

```bash
# 1. 미팅 준비
/alfa-prep
> 저장 위치: c:\workspace\customers
> 중견 물류회사, 직원 800명, CS 하루 2000건, AI 효율화 지시

# 2. 미팅 후 결과 입력
/alfa-live
> (미팅 메모 붙여넣기)

# 3. 산출물 생성
/alfa-wrap

# 4. IRM 실행 (리소스 수령 후)
/irm-init
/irm-design
/irm-validate
```

### 예시 2: 병원그룹 CT 판독 보조

```bash
# 1~3. ALFA (미팅 3회 — 규제 Heavy)
/alfa-prep → 미팅1 → /alfa-live → 미팅2 → /alfa-live → 미팅3 → /alfa-live → /alfa-wrap

# 4. IRM
/irm-init    # GraphRAG 3/4 신호 감지
/irm-design  # GraphRAG 승인, 온프레미스 설계
/irm-validate # 12주 구현 계획
```

---

## 테스트 결과

이 레포에 3개 테스트 시나리오의 결과물이 포함되어 있습니다.

| 시나리오 | 폴더 | 파일 수 | 특징 |
|---------|------|:------:|------|
| 병원그룹 | `test-병원그룹/` | 24 | 규제 Heavy, GraphRAG 승인, 12주 |
| 물류회사 (시뮬레이션) | `test-물류회사-시뮬레이션/` | 33 | 유즈케이스 2개, 같은 고객 다른 IRM |
| 물류회사 (실제) | `test-물류회사/` | 26 | 신뢰도 전파, 리소스 60%→100% 보강 |

---

## 설계 수준

| IRM Phase | 성격 | 설명 |
|-----------|------|------|
| Phase 0 | 진짜 | 실제 리소스 확인, 분류는 논리적 |
| Phase 1 | 가설 | 진짜 데이터 기반 설계, 구현 후 검증 필요 |
| Phase 2 | 가설 기반 계획 | Phase 1 가설에 따른 체크리스트/일정 |

입력이 진짜면 설계도 **진짜 데이터 기반 가설**입니다. 전부 추정이었던 시뮬레이션과는 다릅니다.

---

## 관련 프레임워크

- **ALFA** — AI/LLM First Approach: 고객 미팅→프로필→유즈케이스 도출
- **IRM** — Intent Routing Map: 유즈케이스→행위 유형 분류→기술 설계

이 스킬은 두 프레임워크를 Claude Code 슬래시 커맨드로 자동화한 것입니다.
