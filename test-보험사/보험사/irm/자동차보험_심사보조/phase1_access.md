# 자동차보험 심사 보조 — 접근 배치 설계

> 작성일: 2026-04-03
> 접근 질의: 2건
> 설계 신뢰도: ✅ (ERD/API 스펙 실제 확인)
> 참조: IRM_CLI_구성요소_런북_v10

## 대상 시스템

| # | 시스템명 | 접근 방식 | CLI 가능? | 우선순위 | 근거 | 비고 |
|---|---------|----------|:--------:|:-------:|:----:|------|
| 1 | 심사 이력 DB | PostgreSQL 직접 | Y | ★★★ | ✅ | ERD 확인, applications/reviews/decisions |
| 2 | 보험 코어 API | REST API | Y | ★★★ | ✅ | 스펙 수령, 계약/보험료/지급이력 |

## CLI 설계 (CLI 런북 참조)

### uw CLI (심사 시스템)
- 경계: 심사 시스템 (PostgreSQL)
- 명령어:
  - `uw case --case-id [ID]` → 심사 건 상세 (요청/심사/결정)
  - `uw history --contract-id [ID]` → 계약별 심사 이력
  - `uw similar --accident-type [유형] --product auto` → 유사 사례 목록 (최근 N건)
  - `uw stats --date-range [기간] --product auto` → 심사 통계 (승인/거절/보류 비율)
- 출력: JSON 표준 형식
- 접근 모드: 읽기전용
- **개인정보 자동 가명처리**: 출력 시 이름 마스킹, 주민번호 제거

### core CLI (보험 코어)
- 경계: 보험 코어 시스템 (REST API)
- 명령어:
  - `core contract --contract-id [ID]` → 계약 정보
  - `core premium --contract-id [ID]` → 보험료 정보
  - `core claims --contract-id [ID]` → 지급 이력 (⚠️ 통계/건수만, 상세는 신용정보라 비노출)
- 출력: JSON 표준 형식
- 접근 모드: 읽기전용
- **신용정보 제한**: claims는 건수/총액만, 개별 지급 상세 비노출

## REST 게이트웨이
- 프레임워크: FastAPI
- 노출: 내부만 (심사 시스템 내)
- 인증: 심사역 계정 + 역할 기반 접근제어
- 감사로그: 모든 API 호출 기록
