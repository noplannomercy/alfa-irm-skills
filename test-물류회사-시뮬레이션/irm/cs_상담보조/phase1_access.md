# CS 상담 보조 — 접근 배치 설계

> 작성일: 2026-03-26
> 접근 질의: 4건

## 대상 시스템

| # | 시스템명 | 접근 방식 | CLI 가능? | 우선순위 | 비고 |
|---|---------|----------|:--------:|:-------:|------|
| 1 | CS DB (MySQL) | DB 직접 조회 | Y | ★★★ | 상담 이력, 주문 정보 |
| 2 | CJ대한통운 API | REST API | Y | ★★★ | 배송 추적 |
| 3 | 한진 API | REST API | Y | ★★★ | 배송 추적 |
| 4 | 롯데 API | REST API | Y | ★★★ | 배송 추적 |

## CLI 설계

### delivery CLI (택배사 3곳 통합)
- 경계: 배송 추적 (택배사 통합)
- 명령어:
  - `delivery track --tracking-number [번호]` → 배송 현황 (택배사 자동 감지)
  - `delivery status-map --carrier [택배사] --code [코드]` → 통합 상태 코드로 변환
- 출력: JSON 표준 형식 (통합 상태 코드)
- 핵심: **상태 코드 매핑 테이블** 내장 (CJ "배달중" = 한진 "배송중" = 롯데 "이동중" → 통합 "DELIVERING")

### cs CLI (CS DB 조회)
- 경계: CS 상담 시스템
- 명령어:
  - `cs history --customer-id [ID]` → 상담 이력 조회
  - `cs order --order-id [주문번호]` → 주문 정보
  - `cs stats --date-range [기간]` → 통계 (미배송, 클레임 등)
- 출력: JSON 표준 형식
- 접근 모드: 읽기전용
- 개인정보: 출력 시 자동 마스킹 적용

## REST 게이트웨이
- 프레임워크: FastAPI
- 노출 범위: 내부만 (CS 시스템 내 호출)
- 인증: 기존 CS 시스템 세션 토큰 연동
