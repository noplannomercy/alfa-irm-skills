# 불량원인분석_보조 — 접근 배치 설계

> 작성일: 2026-04-24
> 접근 질의: 4건 / 전체 13건 (31%)
> 참조: CLI 런북 v10

## 대상 시스템

| # | 시스템명 | 접근 방식 | CLI 가능? | 우선순위 | 근거 | 비고 |
|---|---------|----------|:--------:|:-------:|:----:|------|
| 1 | MES (미라콤) | DB — ODBC 직접 조회 | Y | ★★★ | ✅ | 불량 이력 조회, 불량 상세, 추이 분석. REST API 없음. |
| 2 | 불량코드 매핑 | 엑셀 파일 (룩업) | Y | ★★★ | ✅ | 화성↔아산 코드 변환. 매핑 엑셀 수령 완료. |

## CLI 설계

### MES CLI (`cli-mes`)
- 경계: MES 시스템 (불량 관련 데이터 조회)
- 접근: ODBC → pyodbc 또는 psycopg2 (MES DB 종류에 따라)
- 명령어:
  - `mes defect get --id <불량ID>` → 불량 건 상세 조회
  - `mes defect search --code <불량코드> --plant <공장> --from <시작일> --to <종료일>` → 불량 이력 검색
  - `mes defect trend --code <불량코드> --period <기간>` → 불량 발생 추이 (월별/주별 집계)
  - `mes defect list --line <라인> --severity <등급> --limit <N>` → 특정 라인 최근 불량 목록
- 출력: JSON 표준 형식
- 접근 모드: **읽기전용**
- 보안:
  - 감사로그: 모든 조회 시 사용자/일시/쿼리 기록 (Phase 2 GOVERN 요구사항)
  - MES 접속 정보: 환경변수로 관리 (하드코딩 금지)

### 불량코드 매핑 CLI (`cli-defectcode`)
- 경계: 불량코드 매핑 테이블 (엑셀 → JSON 변환 후 사용)
- 접근: JSON 파일 룩업 (엑셀을 JSON으로 1회 변환)
- 명령어:
  - `defectcode lookup --code <코드> --from-plant <공장>` → 다른 공장의 대응 코드 반환
  - `defectcode list --plant <공장>` → 해당 공장 전체 코드 목록
  - `defectcode unmatched` → 매핑 안 된 코드 목록 (불일치 현황)
- 출력: JSON 표준 형식
- 접근 모드: **읽기전용**
- 비고: GraphRAG의 MEANS 관계 시드 데이터와 동일 소스. CLI는 단건 룩업용, GraphRAG는 질의 확장용.

## REST 게이트웨이

- 프레임워크: **FastAPI**
- 노출 범위: **내부만** (사내 서버, 외부 노출 없음)
- 인증: 사내 네트워크 기반 — API Key (초기), 향후 SSO 연동 검토
- 감사로그: 모든 요청 로깅 (사용자, 일시, 엔드포인트, 파라미터)
- 엔드포인트:
  - `GET /api/defect/{id}` → 불량 상세
  - `GET /api/defect/search` → 불량 이력 검색 (쿼리 파라미터)
  - `GET /api/defect/trend` → 불량 추이
  - `GET /api/defectcode/lookup` → 코드 매핑
- OpenAPI 문서 자동 생성 (`/docs`)

## CLI 우선순위 정리

| CLI | 우선순위 | 근거 |
|-----|:-------:|------|
| cli-mes | ★★★ (즉시) | 체이닝 1단계 — 불량 건 조회가 모든 분석의 시작점 |
| cli-defectcode | ★★★ (즉시) | 체이닝 초기 — 코드 변환 없이는 교차 검색 불가 |

## 주의사항
- MES ODBC 접속 시 쿼리 성능 확인 필요 — 15만건 테이블 풀스캔 방지 (인덱스 확인)
- 감사로그는 Phase 2 GOVERN 요구사항 (보안담당 확인)
- cli-defectcode의 매핑 엑셀은 수동 업데이트 → 갱신 주기 고객과 합의 필요
