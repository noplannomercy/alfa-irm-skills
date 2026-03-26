# CT 판독문 보조 — 접근 구현 체크리스트

> 작성일: 2026-03-25

## CLI 구현

### PACS CLI (★★★ 최우선)
- [ ] CLI 경계: PACS 시스템 (Infinitt FHIR API)
- [ ] 명령어 목록:
  - `pacs patient-history --patient-id <ID> --modality CT` → 환자 CT 검사 이력
  - `pacs report --study-id <ID>` → 특정 검사 판독문
  - `pacs search --date-from <날짜> --date-to <날짜> --finding <소견>` → 기간/소견별 목록
- [ ] JSON 출력 형식 정의 (환자ID/검사ID/판독문/날짜/부위)
- [ ] setup.py + Click 프레임워크 설정
- [ ] FHIR 서비스 계정 설정 (기존 EMR 권한체계 연동)
- [ ] 접근 모드: **읽기전용** 강제 (쓰기 명령어 없음)
- [ ] 단위 테스트: FHIR 응답 모킹, 정상/오류 케이스 각 5건
- [ ] 감사 로그 연동: 모든 조회 시 EMR 감사로그에 기록

### EMR CLI (★★☆)
- [ ] CLI 경계: EMR 시스템 (자체개발 내부 API/DB)
- [ ] 명령어 목록:
  - `emr patient --patient-id <ID>` → 환자 기본 정보 (가명처리 적용)
  - `emr history --patient-id <ID> --type radiology` → 영상의학과 진료 이력
  - `emr audit-log --action ai-query --user <사용자> --target <대상> --query <질의>` → 감사로그 기록
- [ ] JSON 출력 형식 정의 (환자정보/이력/로그ID)
- [ ] setup.py + Click 프레임워크 설정
- [ ] DB 접속 설정 (자체개발 EMR DB 커넥션)
- [ ] 접근 모드: 조회=읽기전용, `audit-log`=쓰기
- [ ] 가명처리 자동 적용: `emr patient` 출력 시 PHI 마스킹 강제
- [ ] 단위 테스트: 정상 조회/가명처리/감사로그 기록 각 5건

### 가이드라인 CLI (★☆☆)
- [ ] CLI 경계: 판독 가이드라인 문서 (정적 Markdown/PDF)
- [ ] 명령어 목록:
  - `guideline search --keyword <키워드>` → 가이드라인 내 검색
  - `guideline get --section <섹션명>` → 특정 섹션 내용
- [ ] 출력: Markdown 형식
- [ ] 가이드라인 파일 위치 설정 (로컬 경로)
- [ ] 단위 테스트: 검색 정확도, 섹션 조회 정상 동작

## REST 게이트웨이
- [ ] 프레임워크: **FastAPI**
- [ ] 엔드포인트 정의:
  - `GET /api/pacs/patient/{id}/history?modality=CT` → PACS CLI 래핑
  - `GET /api/pacs/report/{study_id}` → 판독문 조회
  - `GET /api/pacs/search?date_from=&date_to=&finding=` → 판독문 검색
  - `GET /api/emr/patient/{id}` → 환자 정보 (가명처리)
  - `GET /api/emr/history/{id}?type=radiology` → 진료 이력
  - `POST /api/emr/audit` → 감사로그 기록
  - `GET /api/guideline/search?keyword=` → 가이드라인 검색
- [ ] 인증: EMR 세션 토큰 연동 (기존 인증 체계 활용)
- [ ] 노출 범위: **내부만** (원내 네트워크, 외부 접근 차단)
- [ ] API 문서: OpenAPI 자동 생성 (Swagger UI)
- [ ] 레이트 리밋: 사용자당 60req/min

## 통합 검증
- [ ] CLI → REST 엔드투엔드: 각 CLI 명령어가 REST로 정상 래핑되는지 확인
- [ ] 체이닝 테스트: 접근(PACS 조회) → 탐색(판독문 검색) 연결 동작 확인
- [ ] 에러 처리: FHIR 타임아웃, EMR DB 연결 실패 시 적절한 에러 반환
- [ ] 감사로그: 모든 API 호출에 대해 감사로그 자동 기록 확인
