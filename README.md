# ALFA/IRM 런북 어시스턴트

> AI 컨설팅 프레임워크(ALFA + IRM)를 실전 고객 미팅에서 사용할 수 있는 Claude Code 스킬 10개

## 문서

| 문서 | 설명 |
|------|------|
| [Quick Start](doc/quick-start.md) | 처음 쓰는 사람이 5분 안에 따라하는 가이드 |
| [Skill Reference](doc/skill-reference.md) | 스킬별 입력/출력/핵심 옵션 카드형 레퍼런스 |
| [Troubleshooting](doc/troubleshooting.md) | 순서 꼬임, 의존성, 자주 묻는 질문 |
| [Scenario Walkthrough](doc/scenario-walkthrough.md) | 대본형 시나리오 — 성진오토텍 풀사이클 |

## 개요

고객 미팅 전 준비부터 미팅 후 산출물 생성, 유즈케이스 도출, IRM 기술 설계, 배포 파이프라인 설계까지 **슬래시 커맨드 10개로 풀사이클**을 자동화합니다. 각 스킬은 독립적으로 실행 가능하여, 풀사이클뿐 아니라 **필요한 부분만 단독 실행**할 수도 있습니다.

```
풀사이클:
/alfa-prep → 미팅 → /alfa-live → /alfa-wrap
  → /irm-init → /irm-search → /irm-access → /irm-exec → /irm-design → /irm-deploy → /irm-validate

단독 실행 (예시):
/irm-search  ← "컬렉션 어떻게 나오는지 보여줘"
/irm-access  ← "CLI 구성 보여줘"
```

---

## 설치

### 1. 스킬 파일 복사

```bash
# Windows
cp -r skills/* %USERPROFILE%\.claude\skills\

# macOS/Linux
cp -r skills/* ~/.claude/skills/
```

`irm-references/` 폴더도 함께 복사됩니다 (IRM 스킬이 참조하는 설계 방법론 문서).

### 2. 설치 확인

Claude Code에서 10개 슬래시 커맨드가 인식되는지 확인합니다.

```
/alfa-prep, /alfa-live, /alfa-wrap
/irm-init, /irm-search, /irm-access, /irm-exec, /irm-design, /irm-deploy, /irm-validate
```

---

## 스킬 구성 (10개)

### ALFA 스킬 — 고객 미팅

| 스킬 | 커맨드 | 역할 | 레퍼런스 |
|------|--------|------|---------|
| 미팅 준비 | `/alfa-prep` | 질문 리스트 + 사전 요청서 + 프로필 초안 | — |
| 미팅 반영 | `/alfa-live` | 프로필 업데이트 + 신호 감지 + script 저장 | — |
| 산출물 생성 | `/alfa-wrap` | 서머리, Phase 문서, 유즈케이스, IRM 투입 판단 | — |

### IRM 스킬 — 기술 설계 (분리형)

| 스킬 | 커맨드 | 역할 | 레퍼런스 |
|------|--------|------|---------|
| 리소스 분류 | `/irm-init` | 인벤토리, Q1/Q2/Q3 분류, 질의 매핑 | 실행가이드 |
| **탐색 설계** | `/irm-search` | 컬렉션, GraphRAG, 검색 전략, 임베딩 | **RAG방법론 + GraphRAG방법론** |
| **접근 설계** | `/irm-access` | CLI, REST 게이트웨이 | **CLI런북** |
| **실행+체이닝** | `/irm-exec` | 파이프라인, LLM 판단, 체이닝 플로우 | **실행가이드** |
| 종합 요약 | `/irm-design` | 위 3개 결과 취합, 아키텍처/일정 종합 | 실행가이드 |
| **배포 설계** | `/irm-deploy` | S3 구조, Forge 파싱, Cortex 설정, N8N 워크플로우 | — |
| 구현 설계 | `/irm-validate` | 체크리스트, 검증 기준, 구현 계획 | RAG+GraphRAG+CLI+워크플로우 |

**핵심: IRM Phase 1이 search/access/exec로 분리되어 있어서, 필요한 설계만 단독 실행 가능.**

---

## 워크플로우

### 풀사이클

```
0. 고객 접점 (스킬 밖)

1. /alfa-prep → 질문 리스트 + 사전 요청서 (IT팀 리소스 조사 포함)
2. 고객 발송 (스킬 밖) — IT팀 조사 시작 (병렬)
3. 고객 미팅 (스킬 밖)
4. /alfa-live → 미팅 결과 반영 (빈칸 있으면 2~4 반복)
5. /alfa-wrap → 산출물 생성 (리소스 미수령 시 수령 후 재실행)

6. /irm-init → 리소스 분류 + 질의 매핑
7. /irm-search → 컬렉션 + GraphRAG 설계
8. /irm-access → CLI + REST 설계
9. /irm-exec → 실행 + 체이닝 설계
10. /irm-design → 종합 요약 (7~9 취합)
11. /irm-deploy → 배포 설계 (S3 + Forge + Cortex + N8N)
12. /irm-validate → 구현 체크리스트 + 검증 기준

12. 구현 (스킬 밖)
13. 유즈케이스 추가 시 → 4번으로
```

### 단독 실행 (고객 즉석 질문 대응)

```bash
# "컬렉션 어떻게 나오는지 보여줘"
/irm-search

# "CLI 구성 보여줘"
/irm-access

# "파이프라인 어떻게 되는지"
/irm-exec

# "리소스 분류만 해줘"
/irm-init

# "미팅 준비만 해줘"
/alfa-prep
```

풀사이클을 다 돌리지 않아도, 고객이 궁금해하는 부분만 빠르게 설계 결과를 보여줄 수 있습니다.

---

## 폴더 구조

```
[회사명]/
├── STATUS.md
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
│   └── YYYYMMDD_[이름].md             ← /alfa-wrap
└── irm/
    └── [유즈케이스명]/
        ├── phase0_*.md                ← /irm-init
        ├── phase1_search.md           ← /irm-search
        ├── phase1_access.md           ← /irm-access
        ├── phase1_execution.md        ← /irm-exec
        ├── phase1_chaining.md         ← /irm-exec
        ├── phase1_summary.md          ← /irm-design
        └── phase2_*.md               ← /irm-validate
```

---

## 핵심 기능

### 1. 업종별 자동 분기

| | 병원 | 물류 | 보험 | 건설 |
|---|---|---|---|---|
| 미팅 | 3회 | 1회 | 2회 | 1회 |
| 거버넌스 | Heavy | Light | Heavy | Medium |
| GraphRAG | 3/4 | 0/4 | 4/4 | 승인 |
| 일정 | 12주 | 4주 | 10주 | 21주 |

### 2. 신뢰도 전파 (✅/⚠️)

리소스의 진짜/추정 여부가 IRM 끝까지 전파됩니다.

```
리소스 ✅실제 → 분류 ✅ → 설계 ✅ → 구현 ✅ (바로 착수)
리소스 ⚠️추정 → 분류 ⚠️ → 설계 ⚠️ → 구현 ⚠️ (검증 후 착수)
```

### 3. 레퍼런스 기반 설계

IRM 스킬은 실행 시 `irm-references/` 폴더의 방법론 문서를 **실제로 Read**하여 설계합니다.

| 레퍼런스 | 참조하는 스킬 |
|---------|-------------|
| RAG 설계방법론 (§1~§8) | /irm-search, /irm-validate |
| GraphRAG 설계방법론 (§9~§12) | /irm-search, /irm-validate |
| CLI 런북 | /irm-access, /irm-validate |
| 실행가이드 | /irm-init, /irm-exec, /irm-design |
| 개발워크플로우 | /irm-validate |

### 4. Phase 문서 점진적 보강

```
상태: 초안 (v1) → 부분 (v2) → 확정 (v3)
```

### 5. 사전 요청서

`/alfa-prep`에서 고객에게 보낼 숙제를 자동 생성합니다.

- **A. 담당자용** — 프로필 빈칸 채우기
- **B. IT팀용** — 문서/시스템/DB/보안 리소스 조사

### 6. 컬렉션 분리 3기준

`/irm-search`에서 컬렉션 분리 시 **3가지 기준을 각각 판단하고 근거를 명시**합니다.

1. 갱신 주기가 다른가?
2. 도메인 메타데이터가 다른가?
3. 검색 시 함께 조회되는가?

**2/3 이상 "분리"면 분리. 1개면 합침.**

### 7. 일정 추정 기준표

`/irm-design`에서 일정 추정 시 기준표 + 리스크 가산을 적용합니다.

---

## 사용 예시

### 예시 1: 풀사이클 (보험사)

```bash
/alfa-prep     # 질문 리스트 + 사전 요청서
# 미팅 1차 + 2차 (컴플라이언스 동석)
/alfa-live     # 미팅 결과 반영
/alfa-wrap     # 산출물 생성 (Phase 문서 + 유즈케이스)
/irm-init      # 리소스 분류 (8건 ✅)
/irm-search    # 3개 컬렉션 + GraphRAG 4/4 승인 + pgvector + Apache AGE
/irm-access    # 2개 CLI (uw, core)
/irm-exec      # 심사 의견 초안 + 감사로그 파이프라인
/irm-design    # 종합: 10주, 4,500만
/irm-validate  # §1~§12 체크리스트 + 검증 기준
```

### 예시 2: 단독 실행 (고객 질문 대응)

```bash
# 고객: "컬렉션이 몇 개로 나뉘는지 궁금해요"
/irm-search
# → 3기준 판단 후 컬렉션 구조 + 메타데이터 스키마 제시

# 고객: "CLI 어떻게 구성되는지만 보여주세요"
/irm-access
# → 시스템별 CLI 명령어 + REST 게이트웨이 설계 제시
```

---

## 테스트 결과

`samples/` 폴더에 5개 테스트 시나리오의 결과물이 포함되어 있습니다.

| 시나리오 | 폴더 | 특징 |
|---------|------|------|
| 병원그룹 | `samples/test-병원그룹/` | 규제 Heavy, GraphRAG 3/4, 12주 |
| 물류회사 (시뮬레이션) | `samples/test-물류회사-시뮬레이션/` | 유즈케이스 2개, 같은 고객 다른 IRM |
| 물류회사 (실제) | `samples/test-물류회사/` | 신뢰도 전파 60%→100% |
| 보험사 | `samples/test-보험사/` | GraphRAG 4/4, Apache AGE, 금융 규제 |
| 건설사 | `samples/test-건설사/` | 컬렉션 3기준, 품질 필터, 21주 |

---

## 설계 수준

| IRM Phase | 성격 | 설명 |
|-----------|------|------|
| Phase 0 | 진짜 | 실제 리소스 확인, 분류는 논리적 |
| Phase 1 | 가설 | 진짜 데이터 기반 설계, 구현 후 검증 필요 |
| Phase 2 | 가설 기반 계획 | Phase 1 가설에 따른 체크리스트/일정 |

---

## 관련 프레임워크

- **ALFA** — AI/LLM First Approach: 고객 미팅→프로필→유즈케이스 도출
- **IRM** — Intent Routing Map: 유즈케이스→행위 유형 분류→기술 설계
- **레퍼런스** — `references/` 폴더에 IRM 방법론 문서 5개 (RAG, GraphRAG, CLI, 실행가이드, 개발워크플로우)
