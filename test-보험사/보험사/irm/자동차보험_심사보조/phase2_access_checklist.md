# 자동차보험 심사 보조 — 접근 구현 체크리스트

> 작성일: 2026-04-03
> 신뢰도: ✅
> 참조: CLI 런북 v10

## uw CLI (심사 DB)
- [ ] PostgreSQL 연결 (rds-uw.ap-northeast-2)
- [ ] `uw case --case-id` 명령어
- [ ] `uw history --contract-id` 명령어
- [ ] `uw similar --accident-type --product auto` 명령어
- [ ] `uw stats --date-range --product auto` 명령어
- [ ] 개인정보 자동 가명처리 (출력 시)
- [ ] JSON 표준 출력
- [ ] 단위 테스트 (명령어별 5건 = 20건)

## core CLI (보험 코어 API)
- [ ] REST API 연동 (https://core-api.internal/v2)
- [ ] `core contract --contract-id` 명령어
- [ ] `core premium --contract-id` 명령어
- [ ] `core claims --contract-id` 명령어 (⚠️ 건수/총액만, 상세 비노출)
- [ ] 신용정보 제한 로직 (지급 상세 차단)
- [ ] JSON 표준 출력
- [ ] 단위 테스트 (명령어별 5건 = 15건)

## REST 게이트웨이
- [ ] FastAPI 엔드포인트
- [ ] 심사역 계정 + 역할 기반 인증
- [ ] 모든 API 호출 감사로그
- [ ] 내부 네트워크 제한
