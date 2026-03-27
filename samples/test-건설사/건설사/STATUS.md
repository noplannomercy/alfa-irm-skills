# STATUS — 대림건설 ALFA→IRM 풀사이클

> 최종 업데이트: 2026-03-27
> 프로젝트: 대림건설 안전관리 보조 AI

---

## 전체 상태: 설계 완료 / PoC 착수 준비

| 단계 | 상태 | 비고 |
|------|------|------|
| ALFA Phase 0 (Why) | ✅ 확정 (v1) | |
| ALFA Phase 1 (Assess) | ✅ 확정 (v1) | |
| ALFA Phase 2 (Govern) | ✅ 확정 (v1) | 컴플라이언스 트리거 포함 |
| IRM Phase 0 | ✅ 완료 | 리소스 8개 확정, R-06 도면 1차 제외 |
| IRM Phase 1 | ✅ 완료 | 탐색/접근/실행/체이닝 설계 완료 |
| IRM Phase 2 | ✅ 완료 | 체크리스트 + 검증 + 구현 계획 완료 |

---

## 파일 목록

### 미팅 자료

| 파일 | 상태 | 설명 |
|------|------|------|
| `meeting/prep_질문리스트.md` | ✅ | 1차 미팅 전 사전 질문 30개 |
| `meeting/prep_사전요청서.md` | ✅ | 사전 자료 요청서 (3순위 분류) |
| `meeting/1st_script.md` | ✅ | 1차 미팅 메모 (발언 기록 포함) |
| `meeting/1st_summary.md` | ✅ | 고객용 서머리 (ALFA/IRM 용어 없음) |
| `meeting/1st_internal.md` | ✅ | 내부 정리 (리스크, 기술, 영업) |

### ALFA

| 파일 | 상태 | 설명 |
|------|------|------|
| `alfa/profile.md` | ✅ 6/6 🟢 | 고객 프로필 완성 |
| `alfa/alfa_input_profile.md` | ✅ 확정 | Phase 설계 입력값 |
| `alfa/phase0_why.md` | ✅ 확정 (v1) | 도입 근거, 배경 |
| `alfa/phase1_assess.md` | ✅ 확정 (v1) | 현황 평가, 격차 분석 |
| `alfa/phase2_govern.md` | ✅ 확정 (v1) | 거버넌스, 컴플라이언스 트리거 |

### 유즈케이스

| 파일 | 상태 | 설명 |
|------|------|------|
| `usecase/20260327_안전관리보조.md` | ✅ | 유즈케이스 정의서 (리소스 위치/접근 포함) |

### IRM Phase 0

| 파일 | 상태 | 설명 |
|------|------|------|
| `irm/안전관리보조/phase0_readiness.md` | ✅ | 준비도 평가 |
| `irm/안전관리보조/phase0_resource_inventory.md` | ✅ | 리소스 인벤토리 (근거 컬럼 포함) |
| `irm/안전관리보조/phase0_classification.md` | ✅ | 3기준 판단, 신뢰도 전파 |
| `irm/안전관리보조/phase0_query_mapping.md` | ✅ | 질의 매핑, 신뢰도 전파 |

### IRM Phase 1

| 파일 | 상태 | 설명 |
|------|------|------|
| `irm/안전관리보조/phase1_search.md` | ✅ | 탐색 설계 (§1~§8 + GraphRAG §9~§12) |
| `irm/안전관리보조/phase1_access.md` | ✅ | 접근 설계 (4개 CLI 명세) |
| `irm/안전관리보조/phase1_execution.md` | ✅ | 실행 설계 (에이전트, FastAPI) |
| `irm/안전관리보조/phase1_chaining.md` | ✅ | 체이닝 설계 (TBM, 기상, 법규갱신) |
| `irm/안전관리보조/phase1_summary.md` | ✅ | 일정 추정 (기준표 + 리스크 가산) |

### IRM Phase 2

| 파일 | 상태 | 설명 |
|------|------|------|
| `irm/안전관리보조/phase2_search_checklist.md` | ✅ | §1~§12 체크리스트 |
| `irm/안전관리보조/phase2_access_checklist.md` | ✅ | 4개 CLI 구현 체크리스트 |
| `irm/안전관리보조/phase2_execution_checklist.md` | ✅ | 에이전트 구현 체크리스트 |
| `irm/안전관리보조/phase2_validation.md` | ✅ | 검증 (20개 TC, UAT, 컴플라이언스) |
| `irm/안전관리보조/phase2_implementation_plan.md` | ✅ | 구현 계획 (신뢰도 전파, 일정 기준표) |

---

## 핵심 설계 결정 요약

| 항목 | 결정 |
|------|------|
| 유즈케이스 | 안전관리 보조 (1개 집중) |
| 벡터DB | pgvector (PostgreSQL 확장, 기존 RDS 활용) |
| 그래프DB | Apache AGE (동일 PostgreSQL) |
| 컬렉션 | 3개: safety_knowledge / regulation / accident_cases |
| 임베딩 | BGE-m3-ko (로컬, 한국어 특화, 보안 대응) |
| 리랭킹 | BGE-reranker-v2-m3 (로컬) |
| GraphRAG | ✅ 적용 (동의어: 고소작업=높이작업, 법규-매뉴얼 관계) |
| 도면 (R-06) | 1차 PoC 제외 (OCR 품질 이슈, 2차 검토) |
| 일정 | 21주 (PoC 기한 10-31, 7주 여유) |
| 예산 | 3,350만원 + 예비비 650만원 = 4,000만원 |

---

## 컴플라이언스 모니터링

| 규제 | 요건 | 현재 상태 |
|------|------|---------|
| 산업안전보건법 | 안전관리 기록 5년 보존 | 설계 완료, 구현 예정 |
| 중대재해처벌법 | AI 조회 이력 보존, 출처 추적 | 설계 완료, 구현 예정 |
| 개인정보보호법 | 근로자 정보 마스킹 | 설계 완료, 전처리 단계에서 구현 |

---

## 다음 액션 (2026-04-07 기한)

| 담당 | 항목 |
|------|------|
| IT팀 | ERD, API 스펙, 도면 샘플(품질별 3장) 공유 |
| IT팀 | RDS PostgreSQL 버전 확인 + pgvector/AGE 설치 가능 여부 확인 |
| 안전팀 | SOP 우선순위 3종 지정 (고소작업/굴착/크레인 권장) |
| 안전팀 | 작업 중지 기준 문서 확인 (풍속/강수량 기준) |
| 안전팀 | 현장별 동의어 추가 목록 수집 (고소작업=높이작업 외) |
| 컨설팅팀 | 도면 보안팀 외부 전송 가능 여부 확인 결과 수령 |
| 컨설팅팀 | Phase 2-A 환경 구성 착수 (IT팀 준비 완료 후) |
