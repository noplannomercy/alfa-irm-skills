# Quick Start — 5분 안에 따라하기

> 처음 쓰는 사람이 가상 고객 하나로 전체 흐름을 체험하는 가이드.

## 사전 준비

1. 스킬 9개가 설치되어 있어야 함 (README.md 참고)
2. 저장할 폴더를 정한다 (예: `~/test-고객사`)

## 전체 흐름 (10단계)

```
ALFA 단계 (고객 대응)
  ① /alfa-prep   → 미팅 전 준비
  ② /alfa-live   → 1차 미팅 결과 입력
  ③ /alfa-wrap   → 1차 산출물 생성 (고객 회신)
  ④ /alfa-live   → 2차 미팅 결과 입력
  ⑤ /alfa-wrap   → 2차 산출물 생성 (유즈케이스 정의서)

IRM 단계 (기술 설계)
  ⑥ /irm-init     → 리소스 분류 + 질의 매핑
  ⑦ /irm-search   → 탐색(RAG/GraphRAG) 설계
  ⑧ /irm-access   → 접근(CLI/REST) 설계
  ⑨ /irm-exec     → 실행+체이닝 설계
  ⑩ /irm-design   → 종합 요약
  ⑪ /irm-validate → 구현 체크리스트 + 검증 기준
```

---

## Step by Step

### ① /alfa-prep — 미팅 전 준비

```
/alfa-prep

ABC전자라는 가전제품 제조사. 직원 500명.
고객센터에서 AI 도입 검토 중. 팀장이 관심 있다고 함.
```

**나오는 것:**
- `meeting/prep_질문리스트.md` — 미팅에서 물어볼 질문
- `meeting/prep_사전요청서.md` — 고객에게 보낼 숙제
- `alfa/profile.md` — 프로필 초안 (빈칸 많음)
- `STATUS.md` — 진행 현황

**실제 시나리오에서는:** 사전요청서를 고객에게 이메일로 발송.

### ② /alfa-live — 1차 미팅 결과

```
/alfa-live

ABC전자 1차 미팅 메모
참석: 김부장, 이과장
- 고객센터 상담사 30명, 월 상담 2만건
- 반복 질문이 많아서 AI로 자동 응답하고 싶음
- FAQ 문서 500건, 상담 이력 DB 있음
- 클라우드(AWS) 사용 중
- 예산 미정, 일단 가능성 확인 차원
```

**나오는 것:** 프로필 업데이트 + 신호 감지 + STATUS 업데이트

### ③ /alfa-wrap — 1차 산출물

```
/alfa-wrap
```

입력 없이 실행. 기존 폴더에서 자동으로 읽어감.

**나오는 것:**
- `meeting/1st_summary.md` — **고객에게 보내는 서머리**
- `meeting/1st_internal.md` — 내부 정리
- `alfa/phase0_why.md` 등 Phase 문서 (초안)

**실제 시나리오에서는:** 1st_summary.md를 고객에게 회신.

### ④⑤ 2차 미팅 → /alfa-live → /alfa-wrap

유즈케이스가 좁혀지면 (예: "고객센터 자동응답 보조") 유즈케이스 정의서가 생성됨.

### ⑥ /irm-init — 리소스 분류

```
/irm-init
```

유즈케이스 정의서를 자동으로 찾아서 Q1→Q2→Q3 분류 실행.

**나오는 것:**
- `phase0_resource_inventory.md` — 리소스 목록
- `phase0_classification.md` — 행위 유형 분류
- `phase0_query_mapping.md` — 예상 질의 매핑

### ⑦⑧⑨ Phase 1 설계 3종

```
/irm-search   ← 탐색(RAG) 설계
/irm-access   ← 접근(CLI) 설계
/irm-exec     ← 실행+체이닝 설계
```

순서대로 실행. 각각 Phase 0 결과를 자동으로 읽어감.

### ⑩ /irm-design — 종합

```
/irm-design
```

⑦⑧⑨ 결과를 취합. **아키텍처 + 예상 일정** 생성.

### ⑪ /irm-validate — 구현 설계

```
/irm-validate
```

**구현 체크리스트 + 검증 기준 + 구현 계획** 생성. IRM 완료.

---

## 팁

- **미팅 횟수는 유동적.** 1회로 다 파악되면 live 1번 + wrap 1번이면 됨. 빈칸 많으면 2~3회.
- **wrap은 미팅마다 돌려도 되고, 다 끝나고 한번에 돌려도 됨.** 실제로는 미팅마다 돌려서 고객에게 서머리 회신하는 게 좋음.
- **IRM 스킬은 단독 실행도 가능.** "컬렉션 어떻게 나오는지만 보여줘" → `/irm-search`만 실행.
- **고객용 문서에는 ALFA/IRM/GraphRAG 등 내부 용어가 절대 나오지 않음.** summary만 고객에게 보내면 됨.

---

## 실제 테스트 결과

`samples/` 폴더에 6개 시나리오 결과물이 있음:

| 시나리오 | 업종 | 특징 |
|---------|------|------|
| test-제조사 | 자동차 부품 | 온프레미스, GraphRAG 3/4, 12주 |
| test-병원그룹 | 의료 | 규제 Heavy, 12주 |
| test-보험사 | 금융 | GraphRAG 4/4, 10주 |
| test-건설사 | 건설 | 컬렉션 3기준, 21주 |
| test-물류회사 | 물류 | Light, 4주 |
| test-에듀테크 | 교육 | Light~Medium |
