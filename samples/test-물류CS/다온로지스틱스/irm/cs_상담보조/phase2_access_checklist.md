# cs_상담보조 — 접근 구현 체크리스트

> 작성일: 2026-04-11

## CLI 구현

### cli-crm (CRM 상담 이력 조회)
- [ ] 기존 CRM REST API 문서 확인
- [ ] 명령어 구현:
  - [ ] `crm consultation get --id`
  - [ ] `crm consultation list --customer --limit`
  - [ ] `crm consultation stats --date --category`
- [ ] JSON 출력 형식 통일
- [ ] **개인정보 마스킹** 옵션 (--mask-pii)
- [ ] setup.py + Click
- [ ] 단위 테스트

## REST 게이트웨이 — 불필요 (기존 CRM API 사용)

## 통합 검증
- [ ] cli-crm → RAG 검색 연결 테스트
- [ ] 개인정보 마스킹 정상 동작 확인
