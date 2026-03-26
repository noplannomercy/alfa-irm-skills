# CS 상담 보조 — 접근 구현 체크리스트

> 작성일: 2026-04-15
> 신뢰도: ✅ API 스펙/ERD 실제 확인

## delivery CLI (택배사 통합)
- [ ] CJ대한통운 연동 (REST, API Key, 12종 코드)
- [ ] 한진 연동 (REST, OAuth2, 9종 코드)
- [ ] 롯데 연동 (REST, API Key, 10종 코드)
- [ ] 상태 코드 매핑 테이블 (12+9+10 → 통합 8종)
- [ ] 택배사 자동 감지 (송장번호 패턴)
- [ ] 한진 500건/일 캐싱 (5분)
- [ ] `delivery track` + `delivery status-map` 명령어
- [ ] 단위 테스트 (택배사별 5건 = 15건)

## cs CLI (CS DB)
- [ ] MySQL 연결 (RDS, consultations/customers/orders)
- [ ] `cs history` 명령어 + 개인정보 자동 마스킹
- [ ] `cs order` 명령어
- [ ] `cs stats` 명령어
- [ ] 마스킹 규칙 적용 (이름 홍*동, 전화 010-****-5678, 주소 시/구)
- [ ] 단위 테스트 (명령어별 5건 = 15건)

## REST 게이트웨이
- [ ] FastAPI 엔드포인트
- [ ] CS 세션 토큰 인증
- [ ] 내부 네트워크 제한
