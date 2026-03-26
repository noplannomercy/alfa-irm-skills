# CS 상담 보조 — 접근 구현 체크리스트

> 작성일: 2026-03-26

## CLI 구현

### delivery CLI (택배사 통합)
- [ ] 택배사 3곳 API 연동 (CJ, 한진, 롯데)
- [ ] 상태 코드 매핑 테이블 구현 (각 택배사 코드 → 통합 코드)
- [ ] `delivery track --tracking-number` 명령어
- [ ] `delivery status-map --carrier --code` 명령어
- [ ] JSON 표준 출력
- [ ] 택배사 자동 감지 (송장번호 패턴 기반)
- [ ] API 장애 시 폴백 처리
- [ ] 단위 테스트 (택배사별 5건)

### cs CLI (CS DB 조회)
- [ ] MySQL 연결 설정
- [ ] `cs history --customer-id` 명령어
- [ ] `cs order --order-id` 명령어
- [ ] `cs stats --date-range` 명령어
- [ ] JSON 표준 출력
- [ ] **개인정보 자동 마스킹** (출력 시)
- [ ] 단위 테스트 (명령어별 5건)

### REST 게이트웨이
- [ ] FastAPI 엔드포인트 정의
- [ ] CS 시스템 세션 토큰 연동
- [ ] 내부 네트워크 제한
- [ ] API 문서 자동 생성

## 통합 검증
- [ ] delivery + cs CLI 엔드투엔드 테스트
- [ ] 체이닝 테스트 (배송 조회 → FAQ 검색 → 답변 생성)
