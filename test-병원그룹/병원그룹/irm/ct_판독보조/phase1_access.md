# CT 판독문 보조 — 접근 배치 설계

> 작성일: 2026-03-25
> 접근 질의: 4건 / 전체 14건 (29%)

## 대상 시스템

| # | 시스템명 | 접근 방식 | CLI 가능? | 우선순위 | 비고 |
|---|---------|----------|:--------:|:-------:|------|
| 1 | PACS (Infinitt) | FHIR API | Y | ★★★ | 영상 메타데이터, 판독문 원본 조회 |
| 2 | EMR (자체개발) | DB/내부 API | Y | ★★☆ | 환자 진료기록, 과거 이력 조회 |
| 3 | 판독 가이드라인 | 파일 | Y | ★☆☆ | 소견 기술 기준 참조 (정적) |

## CLI 설계

### PACS CLI
- 경계: PACS 시스템 (Infinitt FHIR API)
- 명령어:
  - `pacs patient-history --patient-id [ID] --modality CT` → 특정 환자의 CT 검사 이력 조회
  - `pacs report --study-id [ID]` → 특정 검사의 판독문 조회
  - `pacs search --date-from [날짜] --date-to [날짜] --finding [소견]` → 기간/소견별 판독문 목록 조회
- 출력: JSON 표준 형식
- 접근 모드: **읽기전용**
- 인증: FHIR 서비스 계정 (기존 EMR 권한체계 연동)

### EMR CLI
- 경계: EMR 시스템 (자체개발 내부 API/DB)
- 명령어:
  - `emr patient --patient-id [ID]` → 환자 기본 정보 조회 (가명처리 적용)
  - `emr history --patient-id [ID] --type radiology` → 영상의학과 진료 이력
  - `emr audit-log --action ai-query --user [사용자]` → AI 조회 감사로그 기록
- 출력: JSON 표준 형식
- 접근 모드: 조회=**읽기전용**, 감사로그=**쓰기**
- 인증: 기존 EMR 역할 기반 권한체계

### 가이드라인 CLI
- 경계: 판독 가이드라인 문서 (정적 파일)
- 명령어:
  - `guideline search --keyword [키워드]` → 가이드라인 내 검색
  - `guideline get --section [섹션명]` → 특정 섹션 내용 조회
- 출력: Markdown 형식
- 접근 모드: **읽기전용**

## REST 게이트웨이
- 프레임워크: FastAPI
- 노출 범위: **내부만** (원내 네트워크)
- 인증: EMR 세션 토큰 연동 (기존 인증 체계 활용)
- 엔드포인트:
  - `GET /api/pacs/patient/{id}/history` → PACS CLI 래핑
  - `GET /api/emr/patient/{id}` → EMR CLI 래핑
  - `POST /api/emr/audit` → 감사로그 기록
  - `GET /api/guideline/search` → 가이드라인 검색
