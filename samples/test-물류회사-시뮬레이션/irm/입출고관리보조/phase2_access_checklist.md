# 입출고 관리 보조 — 접근 구현 체크리스트

> 작성일: 2026-05-15

## CLI 구현

### wms CLI
- [ ] MySQL 연결 설정 (기존 RDS)
- [ ] `wms inbound` 명령어 + 테스트
- [ ] `wms outbound` 명령어 + 테스트
- [ ] `wms stock` 명령어 + 테스트
- [ ] `wms defect-list` 명령어 + 테스트
- [ ] `wms defect-record` 명령어 (쓰기) + 테스트
- [ ] `wms defect-stats` 명령어 + 테스트
- [ ] JSON 표준 출력
- [ ] 단위 테스트 (명령어별 5건 = 30건)

### REST 게이트웨이
- [ ] 기존 FastAPI에 wms 엔드포인트 추가
- [ ] 센터별 담당자 인증
- [ ] API 문서 갱신

## 통합 검증
- [ ] wms CLI 엔드투엔드 테스트
- [ ] 체이닝 테스트 (이상 건 조회 → 유사 검색 → 처리 제안)
