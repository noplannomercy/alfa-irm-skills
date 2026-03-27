# 입출고 관리 보조 — 접근 배치 설계

> 작성일: 2026-05-15
> 접근 질의: 3건

## 대상 시스템

| # | 시스템명 | 접근 방식 | CLI 가능? | 우선순위 | 비고 |
|---|---------|----------|:--------:|:-------:|------|
| 1 | WMS (입출고 이력) | DB (MySQL) | Y | ★★★ | 바코드 스캔 기반 |
| 2 | WMS (재고 현황) | DB (MySQL) | Y | ★★★ | 실시간 재고 |

## CLI 설계

### wms CLI
- 경계: 창고관리시스템 (WMS)
- 명령어:
  - `wms inbound --center [센터ID] --date [날짜]` → 입고 현황
  - `wms outbound --center [센터ID] --date [날짜]` → 출고 현황
  - `wms stock --product [품목코드]` → 재고 조회
  - `wms defect-list --center [센터ID] --date-range [기간]` → 이상 건 목록
  - `wms defect-record --center [센터ID] --type [유형] --product [품목] --desc [내용]` → 이상 건 기록 (쓰기)
  - `wms defect-stats --date-range [기간]` → 이상 건 통계
- 출력: JSON 표준 형식
- 접근 모드: 읽기 + 쓰기 (이상 건 기록)

## REST 게이트웨이
- 기존 FastAPI 게이트웨이에 엔드포인트 추가
- 인증: 센터별 담당자 계정
