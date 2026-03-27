# 미래에셋손해보험 — 진행 현황

> 최종 업데이트: 2026-04-03

## 현재 단계: IRM Phase 0~2 완료 — 구현 착수 가능
## 다음 작업: 구현 착수 (10주, 6월 초 완료 예정)

## 미팅 이력
| 회차 | 일시 | 참석자 | 핵심 결과 |
|------|------|--------|----------|
| 1차 | 2026-03-27 | CTO, IT팀장, 심사팀장 | 현황 파악, 리소스 100%, 심사보조 1순위 |
| 2차 | 2026-04-03 | +컴플라이언스팀장, 준법감시인 | 규제 확정, PoC 범위(자동차보험) 합의 |

## Phase 진행
- [x] Phase 0 WHY — 확정 (v1)
- [x] Phase 1 ASSESS — 확정 (v1), 리소스 100%
- [x] Phase 2 GOVERN — 확정 (v1), 컴플라이언스 확정
- [ ] Phase 3 BUILD — 10주 예정
- [x] Phase 4 USE CASE — 자동차보험 심사 보조 IRM 완료
- [ ] Phase 5 SCALE — 손해사정 2차, 고객 안내 장기

## IRM 결과 요약
| 항목 | 결과 |
|------|------|
| 설계 신뢰도 | **✅ 100% — 실제 데이터 + 레퍼런스 참조** |
| GraphRAG | **승인 (4/4 만점)** |
| 그래프DB | **Apache AGE (PostgreSQL 확장)** ← 레퍼런스 기반 |
| 벡터DB | **pgvector (기존 PostgreSQL 확장)** |
| 컬렉션 | 3개 (uw_guidelines, uw_cases, claim_docs) |
| CLI | 2개 (uw, core) |
| 일정 | 10주 |
| 예산 | ~4,500만 (5천만 내) |

## 핵심 규제 결정
- 보조수단성: 충족
- 신용정보: LLM 투입 금지 (통계만)
- 가명처리: 이름/주민번호/주소/연락처
- 감사로그: AI 결과 + 심사역 판단 함께 보관

## 감지 신호
- 🔔 **#GraphRAG (4/4)** — 약관 관계 복잡 + 심사역 판단 불일치 + 인과관계 29%
- 🔔 **#gov-heavy** — 금감원, 7대 원칙, 신용정보법
- 🔔 **#poc-history** — OCR+RPA 실패
- 🔔 **#2-phase** — 고객 안내 장기

## 산출물 현황 (25개)

### ALFA
| 문서 | 상태 |
|------|:----:|
| meeting/ (prep 2개 + 1st 3개 + 2nd 3개) | ✅ |
| alfa/ (profile, input, phase0/1/2) | ✅ |
| usecase/20260403_자동차보험_심사보조.md | ✅ |

### IRM — 자동차보험 심사 보조
| 문서 | 상태 |
|------|:----:|
| phase0 (readiness, inventory, classification, query_mapping) | ✅ |
| phase1 (search, access, execution, chaining, summary) | ✅ |
| phase2 (search/access/execution checklist, validation, implementation_plan) | ✅ |
