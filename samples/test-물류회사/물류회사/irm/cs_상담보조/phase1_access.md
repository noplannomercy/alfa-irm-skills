# CS 상담 보조 — 접근 배치 설계

> 작성일: 2026-04-15
> 접근 질의: 5건
> 설계 신뢰도: ✅ (API 스펙/ERD 실제 확인)

## 대상 시스템

| # | 시스템명 | 접근 방식 | CLI 가능? | 우선순위 | 근거 | 비고 |
|---|---------|----------|:--------:|:-------:|:----:|------|
| 1 | CS DB (MySQL) | DB 직접 | Y | ★★★ | ✅ | ERD 확인, consultations/customers/orders |
| 2 | CJ대한통운 | REST API | Y | ★★★ | ✅ | API Key, 12종 상태코드, 1000건/일 |
| 3 | 한진 | REST API | Y | ★★★ | ✅ | OAuth2, 9종 상태코드, 500건/일 |
| 4 | 롯데 | REST API | Y | ★★★ | ✅ | API Key, 10종 상태코드, 2000건/일 |

## CLI 설계

### delivery CLI (택배사 3곳 통합)
- 경계: 배송 추적 통합
- 명령어:
  - `delivery track --tracking-number [번호]` → 배송 현황 (택배사 자동 감지)
  - `delivery status-map --carrier [택배사] --code [코드]` → 통합 코드 변환
- 상태 코드 매핑: CJ 12종 + 한진 9종 + 롯데 10종 → 통합 8종 (RECEIVED, PICKED_UP, IN_TRANSIT, OUT_FOR_DELIVERY, DELIVERING, DELIVERED, FAILED, RETURNED)
- 인증: CJ/롯데 API Key, 한진 OAuth2 → CLI 내부에서 처리
- 호출 제한: 한진 500건/일이 병목 → 캐싱 적용 (배송 상태는 분 단위 변경)

### cs CLI (CS DB 조회)
- 경계: CS 상담 시스템
- 명령어:
  - `cs history --customer-id [ID]` → 상담 이력 (개인정보 자동 마스킹)
  - `cs order --order-id [번호]` → 주문 정보
  - `cs stats --date-range [기간]` → 통계
- 마스킹 규칙 (phase2_govern에서 확정):
  - 이름: 홍*동
  - 전화번호: 010-****-5678
  - 주소: 시/구까지만

## REST 게이트웨이
- 프레임워크: FastAPI
- 노출: 내부만
- 인증: CS 시스템 세션 토큰
