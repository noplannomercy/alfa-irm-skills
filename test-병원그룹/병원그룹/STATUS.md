# 병원그룹 — 진행 현황

> 최종 업데이트: 2026-03-25

## 현재 단계: IRM Phase 0~2 완료 — 구현 착수 가능 (CT 판독문 보조)
## 다음 작업: 구현 착수 (선행 조건: IRB 승인 + 샘플 데이터 확보)

## 미팅 이력
| 회차 | 일시 | 참석자 | 핵심 결과 |
|------|------|--------|----------|
| (미팅 전) | — | — | 질문 리스트 준비 완료 |
| 1차 | 2026-03-25 | 미기재 | 인프라 온프레미스 확정, 유즈케이스 2개 확인, GraphRAG 신호 감지 |
| 2차 | 2026-03-25 | 의료정보보안팀장, IRB 담당자, 영상의학과 과장 | 규제/거버넌스 확정, 예산 5천만, PoC 스코핑 합의 |

## Phase 진행
- [x] Phase 0 WHY — CTO 스폰서, 예산 5천만, CT 판독 보조 확정
- [x] Phase 1 ASSESS — 온프레미스+A100, PACS(Infinitt/FHIR), 인력 3명
- [x] Phase 2 GOVERN — 가명처리, IRB 간편심의, 보조수단 판정, RACI
- [ ] Phase 3 BUILD — 4월 중순 시작 예정
- [x] Phase 4 USE CASE — CT 판독 보조 확정, 유즈케이스 정의서 작성 완료
- [ ] Phase 5 SCALE — PoC 이후 (MRI 확장, 분원 확장, 간호기록 요약)

## 프로필 채움 현황
| # | 항목 | 상태 |
|---|------|------|
| 1 | 회사 개요 | 🟢 충분 |
| 2 | AI/LLM 현황 | 🟢 충분 |
| 3 | 인프라 | 🟢 충분 |
| 4 | 규제 환경 | 🟢 충분 |
| 5 | 원하는 것 | 🟢 충분 |
| 6 | 제약사항 | 🟢 충분 |

## 감지 신호
- 🔔 **#GraphRAG** — 의사마다 표현 차이 + 이전 PoC 실패 원인 → 의료 용어 특화 핵심
- 🔔 **#on-premise** — 데이터 외부 반출 절대 불가 → 로컬 LLM 필수
- 🔔 **#poc-history** — 작년 챗봇 PoC 실패 (도메인 특화 부족) → 차별점 확보 필수
- 🔔 **#2-phase** — 장기적 환자 설명용 확장 희망 → 현재는 내부 전용 유지

## 산출물 현황
| 문서 | 파일 | 상태 |
|------|------|------|
| 질문 리스트 | meeting/prep_질문리스트.md | ✅ |
| 1차 미팅 서머리 | meeting/1st_summary.md | ✅ |
| 1차 내부 정리 | meeting/1st_internal.md | ✅ |
| 2차 미팅 서머리 | meeting/2nd_summary.md | ✅ |
| 2차 내부 정리 | meeting/2nd_internal.md | ✅ |
| Phase 0 WHY | alfa/phase0_why.md | ✅ |
| Phase 1 ASSESS | alfa/phase1_assess.md | ✅ |
| Phase 2 GOVERN | alfa/phase2_govern.md | ✅ |
| ALFA 입력 프로필 | alfa/alfa_input_profile.md | ✅ |
| 유즈케이스 정의서 | usecase/20260325_ct_판독보조.md | ✅ |
| IRM 투입 판단 | irm/ct_판독보조/phase0_readiness.md | ✅ |
| IRM 리소스 인벤토리 | irm/ct_판독보조/phase0_resource_inventory.md | ✅ |
| IRM 리소스 분류 | irm/ct_판독보조/phase0_classification.md | ✅ |
| IRM 질의 매핑 | irm/ct_판독보조/phase0_query_mapping.md | ✅ |
| IRM 탐색 배치 | irm/ct_판독보조/phase1_search.md | ✅ |
| IRM 접근 배치 | irm/ct_판독보조/phase1_access.md | ✅ |
| IRM 실행 배치 | irm/ct_판독보조/phase1_execution.md | ✅ |
| IRM 체이닝 설계 | irm/ct_판독보조/phase1_chaining.md | ✅ |
| IRM Phase 1 요약 | irm/ct_판독보조/phase1_summary.md | ✅ |
| IRM 탐색 구현 체크리스트 | irm/ct_판독보조/phase2_search_checklist.md | ✅ |
| IRM 접근 구현 체크리스트 | irm/ct_판독보조/phase2_access_checklist.md | ✅ |
| IRM 실행 구현 체크리스트 | irm/ct_판독보조/phase2_execution_checklist.md | ✅ |
| IRM 검증 기준 | irm/ct_판독보조/phase2_validation.md | ✅ |
| IRM 구현 계획 | irm/ct_판독보조/phase2_implementation_plan.md | ✅ |

## 다음 마일스톤
| 일정 | 항목 |
|------|------|
| 4월 초 | IRB 간편심의 승인 |
| 4월 초~중순 | 가명처리 파이프라인 + 샘플 데이터 준비 |
| 4월 중순 | PoC 개발 시작 + IRM Phase 0 투입 |
| 6월 말 | 파일럿 완료 + MRI 확장 판단 |
