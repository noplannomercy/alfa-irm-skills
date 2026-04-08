# 불량원인분석_보조 — 접근 구현 체크리스트

> 작성일: 2026-04-24

## CLI 구현

### cli-mes (MES 불량 조회)
- [ ] CLI 경계 확정: MES 시스템 (불량 관련 읽기전용)
- [ ] ODBC 접속 설정 (pyodbc — 환경변수 관리)
- [ ] 명령어 구현:
  - [ ] `mes defect get --id` (불량 상세)
  - [ ] `mes defect search --code --plant --from --to` (이력 검색)
  - [ ] `mes defect trend --code --period` (발생 추이)
  - [ ] `mes defect list --line --severity --limit` (최근 목록)
- [ ] JSON 출력 형식 통일
- [ ] 감사로그 연동 (사용자/일시/쿼리 기록)
- [ ] setup.py + Click 프레임워크 설정
- [ ] 단위 테스트 작성 (각 명령어)
- [ ] MES 쿼리 성능 확인 (15만건 인덱스 유무)

### cli-defectcode (불량코드 매핑)
- [ ] 매핑 엑셀 → JSON 변환 스크립트
- [ ] 명령어 구현:
  - [ ] `defectcode lookup --code --from-plant` (코드 변환)
  - [ ] `defectcode list --plant` (전체 코드 목록)
  - [ ] `defectcode unmatched` (미매핑 코드)
- [ ] JSON 출력 형식 통일
- [ ] 단위 테스트 작성
- [ ] 매핑 엑셀 갱신 시 JSON 재생성 절차 정의

### REST 게이트웨이
- [ ] FastAPI 프로젝트 구성
- [ ] 엔드포인트 4개:
  - [ ] GET /api/defect/{id}
  - [ ] GET /api/defect/search
  - [ ] GET /api/defect/trend
  - [ ] GET /api/defectcode/lookup
- [ ] API Key 인증 설정
- [ ] 감사로그 미들웨어 (모든 요청)
- [ ] OpenAPI 문서 자동 생성 확인 (/docs)
- [ ] 내부 네트워크 바인딩 (0.0.0.0 아닌 내부 IP)

## 통합 검증
- [ ] CLI → REST 엔드투엔드 테스트
- [ ] 체이닝 테스트: cli-mes → cli-defectcode → RAG 검색 연결
- [ ] 감사로그 정상 기록 확인
