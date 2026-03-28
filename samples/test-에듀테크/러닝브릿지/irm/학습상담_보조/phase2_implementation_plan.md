# 학습상담 보조 — 구현 계획

> 작성일: 2026-03-27
> 고객: 러닝브릿지

## 구현 범위 요약

| 구성요소 | 신규/재사용 | 우선순위 | 신뢰도 | 예상 기간 |
|---------|:---------:|:-------:|:------:|----------|
| pgvector extension | 신규 (기존 RDS 확장) | ★★★ | ✅ | 0.5주 |
| Bedrock 개통 + 임베딩 API | 신규 | ★★★ | ✅ | 0.5주 |
| 비식별화 파이프라인 | 신규 | ★★★ | ✅ | 0.5주 |
| faq_and_rules 컬렉션 | 신규 | ★★★ | ✅ | 0.5주 |
| course_content 컬렉션 (PoC 50~100개) | 신규 | ★★★ | ✅ | 0.5주 |
| consultation_history 컬렉션 (3개월) | 신규 | ★★ | ✅ | 1주 |
| CLI: cli-anything-student | 신규 | ★★★ | ✅ | 0.5주 |
| CLI: cli-anything-zendesk | 신규 | ★★ | ✅ | 0.5주 |
| FastAPI Gateway | 신규 | ★★★ | ✅ | 0.5주 |
| 환불 판단 파이프라인 | 신규 | ★★★ | ✅ | 0.5주 |
| 강의 변경 판단 파이프라인 | 신규 | ★★ | ✅ | 0.5주 |
| 학습 추천 프로세스 (LLM+HitL) | 신규 | ★ | ✅ | 1주 |
| 체이닝 오케스트레이터 | 신규 | ★★★ | ✅ | 0.5주 |
| 통합 테스트 + 파일럿 | 신규 | ★★★ | ✅ | 1.5주 |

**✅확정 항목 100%** — 추정 항목 없음, 전체 즉시 착수 가능

---

## 구현 순서 (의존성 기반)

```
Phase A: 인프라 (W1, 1주)
  ├─ pgvector extension 활성화
  ├─ Bedrock 개통 + API 키 설정
  ├─ OpenAI Embedding API 연결
  ├─ 비식별화 파이프라인 (mask_name, mask_email)
  ├─ FastAPI Gateway 기본 배포
  └─ CLI 2개 설치 (student, zendesk)

Phase B: 탐색 — faq_and_rules (W2, 0.5주)
  ├─ FAQ 150건 파싱 + 환불규정 10p 파싱
  ├─ 임베딩 + 인덱싱
  ├─ 벡터 검색 확인
  └─ 의존: Phase A

Phase C: 실행(확정) (W2, 0.5주) ← Phase B와 병렬 가능
  ├─ 환불 판단 규칙 함수 구현
  ├─ 강의 변경 판단 규칙 함수 구현
  └─ 의존: Phase A (CLI)

Phase D: 탐색 — course_content (W3, 0.5주)
  ├─ 교안+자막 50~100개 파싱 + 임베딩
  ├─ 하이브리드 검색 + FlashRank 리랭킹 설정
  └─ 의존: Phase A

Phase E: 탐색 — consultation_history (W4, 1주)
  ├─ Zendesk API → 3개월분 추출 + 마스킹 + 임베딩
  ├─ 하이브리드 검색 + 리랭킹 설정
  └─ 의존: Phase A

Phase F: 실행(유연) + 체이닝 (W4, 1주) ← Phase E와 병렬 가능
  ├─ 학습 추천 LLM 프롬프트 + HitL UI
  ├─ 체이닝 오케스트레이터 (라우팅 + 패턴 A/B)
  └─ 의존: Phase B + C + D

Phase G: 통합 + 파일럿 (W5~W6, 2주)
  ├─ 엔드투엔드 테스트 (시나리오 8건)
  ├─ 상담사 3~5명 파일럿 운영 (2주)
  ├─ 검색 품질 모니터링 + 피드백 수렴
  ├─ 평가 보고서 + 확대 판단
  └─ 의존: Phase F
```

### 간트 차트 (주차별)

```
W1  ████████████████  Phase A: 인프라 + CLI + Gateway
W2  ████████ ████████ Phase B: faq_and_rules │ Phase C: 실행(확정)
W3  ████████████████  Phase D: course_content
W4  ████████ ████████ Phase E: consultation_history │ Phase F: 실행(유연)+체이닝
W5  ████████████████  Phase G: 파일럿 운영
W6  ████████████████  Phase G: 평가 + 확대 판단
```

---

## 리스크

| 리스크 | 영향 | 확률 | 대응 |
|--------|------|:----:|------|
| FAQ 비구조화 | 파싱 재작업 +0.5주 | 중 | W1에서 수령 즉시 구조 확인. 비구조적이면 수동 정리 |
| 교안 PDF 이미지 중심 | 텍스트 추출 불가 → 콘텐츠 커버리지 저하 | 중 | 샘플 파싱 테스트 (W1). 이미지 중심 교안은 PoC 제외 |
| 환불 규정 해석 모호 | 판단 정확도 저하 | 낮 | 학습운영팀과 W2에서 규정 해석 확정 미팅 |
| Zendesk Rate Limit | 대량 추출 시간 초과 | 낮 | 페이징 + 야간 배치 처리 |
| 상담사 파일럿 참여 저조 | 충분한 피드백 확보 불가 | 중 | 학습운영팀장 협조 요청, 인센티브 검토 |

---

## 다른 유즈케이스와의 관계

| 항목 | 이 유즈케이스 (학습상담 보조) | 2차 유즈케이스 (수강생 대면 AI) | 공유 가능 |
|------|--------------------------|--------------------------|:--------:|
| pgvector | 동일 RDS | 동일 RDS | ✓ |
| faq_and_rules | 공유 (FAQ+규정 동일) | 공유 | ✓ |
| course_content | 공유 (강의 콘텐츠 동일) | 공유 | ✓ |
| consultation_history | 내부 참조용 | 제외 (수강생에게 노출 불가) | ✗ |
| CLI: student | 비식별화 적용 | 비식별화 강화 필요 | ✓ (보강) |
| CLI: zendesk | 내부 전용 | 제외 | ✗ |
| 환불/변경 판단 | 상담사 검토 | 자동 안내 → 거버넌스 Medium | ✓ (보강) |
| 학습 추천 | HitL (상담사) | 자동 추천 → 거버넌스 재검토 | ✓ (보강) |

**2차 유즈케이스 투입 시**: 인프라 + 2개 컬렉션 재사용으로 **구축 기간 ~40% 절감** 예상. 단, 거버넌스 Medium 격상 + 오답 책임 + 비식별화 강화 필요.

---

## IRM 완료 상태

- [x] Phase 0: 리소스 분류 ✅ (6건, 100% 확인)
- [x] Phase 1: 배치 설계 ✅ (탐색+접근+실행+체이닝)
- [x] Phase 2: 구현 설계 ✅ (체크리스트 3종 + 검증 기준)
- [x] 검증 기준 정의 ✅ (시나리오 8건 + 정량 기준)
- [x] → **구현 착수 가능**

---

## 구현 워크플로우 (개발 워크플로우 v10 참조)

```
Phase 2 구현:
  feature 브랜치 생성
  → superpowers:executing-plans (Phase A~F 순서)
  → superpowers:test-driven-development (각 Phase별)
  (버그 시) superpowers:systematic-debugging

Phase 3 검증+출하:
  superpowers:verification-before-completion
  → /review → /qa
  → /ship → /retro
```
