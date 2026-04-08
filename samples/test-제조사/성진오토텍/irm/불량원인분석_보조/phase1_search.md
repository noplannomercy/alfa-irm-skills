# 불량원인분석_보조 — 탐색 배치 설계

> 작성일: 2026-04-24
> 탐색 질의: 7건 / 전체 13건 (54%)
> 참조: RAG 설계방법론 v10, GraphRAG 설계방법론 v10

## 데이터 소스 분석

| # | 소스명 | 형태 | 갱신 주기 | 이력형/최신형 | 규모 | 근거 |
|---|--------|------|----------|-------------|------|:----:|
| 1 | 불량 이력 (MES) | 구조화 DB | 실시간 발생 | 이력형 | 3년 15만건 | ✅ |
| 2 | 검사기준서 | PDF 문서 | 비정기 (개정 시) | 최신형 | 약 200건 | ✅ |
| 3 | 작업표준서 | HWP 문서 | 비정기 (개정 시) | 최신형 | 약 300건 | ✅ |
| 4 | 과거 8D 보고서 | HWP 문서 | 불량 발생 시 | 이력형 | 미확인 | ⚠️ |

## 컬렉션 설계 (§1 참조)

### 분리 기준 판단 (3기준 각각 판단+근거 필수)

#### [검사기준서+작업표준서] vs [불량 이력+8D 보고서]
| 기준 | 판단 | 근거 |
|------|------|------|
| 갱신 주기 | **분리** | 기준서는 비정기 개정(최신형), 불량 이력은 실시간 발생(이력형) |
| 메타데이터 | **분리** | 기준서는 doc_type/part_no/revision, 불량 이력은 defect_code/line/severity |
| 질의 패턴 | **분리** | "이 부품 검사 기준?" vs "과거 유사 불량?" — 거의 교차하지 않음 |
| **판단** | **3/3 → 분리** | |

#### [검사기준서] vs [작업표준서]
| 기준 | 판단 | 근거 |
|------|------|------|
| 갱신 주기 | 통합 | 둘 다 비정기 개정 |
| 메타데이터 | 통합 | 둘 다 doc_type/part_no/process_step — 유사 |
| 질의 패턴 | **분리** | "검사 기준?" vs "작업 절차?" — 질의 의도가 다름 |
| **판단** | **1/3 → 통합** | doc_type 메타데이터로 필터링 가능 |

#### [불량 이력] vs [과거 8D 보고서]
| 기준 | 판단 | 근거 |
|------|------|------|
| 갱신 주기 | 통합 | 둘 다 불량 발생 시 |
| 메타데이터 | **분리** | 불량 이력은 defect_code/line/count, 8D는 root_cause/action/result |
| 질의 패턴 | **분리** | "유사 불량 건?" vs "어떤 대책을 세웠었지?" — 검색 의도 다름 |
| **판단** | **2/3 → 분리** | |

### 컬렉션 구조

| 컬렉션명 | 포함 소스 | 메타데이터 | 인덱싱 범위 | 갱신 방식 |
|----------|----------|-----------|-----------|----------|
| **quality_standards** | 검사기준서 + 작업표준서 | doc_type, part_no, process_step, revision | 최신형 — 최신 버전만 | 비정기 (개정 시 교체) |
| **defect_history** | 불량 이력 (MES 추출) | defect_code, line, product, severity, plant | 이력형 — 3년+ | 주기적 (일 1회 MES 동기화) |
| **investigation_reports** | 과거 8D 보고서 | root_cause, action, defect_code, result | 이력형 — 전수 | 불량 건 종결 시 추가 |

### 벡터 저장소 선택 (§1 제품 비교표 참조)
- 선택: **pgvector** — 근거: 온프레미스 환경, 소~중규모(500문서 + 15만건 이력), PostgreSQL 단일 DB로 관리 단순, GraphRAG(Apache AGE)와 같은 DB 사용
- 인덱스: **HNSW** — 근거: 15만건 수준, 범용 최적, 정확도+속도 밸런스

## 메타데이터 스키마 (§2 참조)

### 공통 메타데이터 (전 컬렉션)
```json
{
  "doc_type": "string",       // inspection_standard / work_standard / defect_record / 8d_report
  "date": "date",             // 작성일/발생일
  "source": "string",         // 원본 위치 (파일경로/MES ID)
  "plant": "string"           // 화성/아산
}
```

### quality_standards 도메인 메타데이터
```json
{
  "part_no": "string",         // 부품번호
  "process_step": "string",    // 공정 단계
  "revision": "string",        // 문서 버전
  "file_format": "string"      // pdf / hwp
}
```

### defect_history 도메인 메타데이터
```json
{
  "defect_code": "string",     // 불량코드 (원본 — 공장별 상이)
  "defect_code_std": "string", // 표준화 불량코드 (매핑 엑셀 기반)
  "line": "string",            // 생산 라인
  "product": "string",         // 제품명
  "severity": "string"         // critical / major / minor
}
```

### investigation_reports 도메인 메타데이터
```json
{
  "root_cause": "string",      // 원인 분류 (LLM 추출)
  "action_taken": "string",    // 대책 요약 (LLM 추출)
  "defect_code_std": "string", // 표준화 불량코드
  "effectiveness": "string"    // 대책 효과 (effective / partial / ineffective)
}
```

## 파싱 & 청킹 (§3 참조)

| 컬렉션 | 데이터 형태 | 도구 | 전략 |
|-------|-----------|------|------|
| quality_standards | PDF + HWP | Docling + HybridChunker (PDF), **HWP 전용 파서 필요** (HWP → 텍스트 변환 후 Docling) | 섹션/조항 단위, merge_peers=True |
| defect_history | MES 구조화 데이터 | 커스텀 전처리 | 1건 = 1청크. 메타데이터는 MES 필드 직접 매핑 |
| investigation_reports | HWP 문서 | **HWP 전용 파서** → Docling | 1보고서 = 1~3청크 (요약/원인/대책) |

**HWP 파싱 리스크**: 오픈소스 HWP 파서(pyhwp, python-hwp 등)는 품질 불안정. 샘플 3건으로 반드시 테스트 후 전략 확정.

## 임베딩 모델 (§4 참조)

| 기준 | 판단 |
|------|------|
| 한국어 | 필수 (품질 문서/불량 기록 전부 한국어) |
| 로컬/API | **로컬 필수** (데이터 외부 전송 금지 — OEM 보안) |
| 선택 | **BGE-m3-ko** (dragonkue/BGE-m3-ko) |
| 근거 | 한국어 파인튜닝, 로컬 무료, Dense+Sparse 지원(하이브리드 검색), 1024차원, 8K 컨텍스트 |
| 대안 | KURE-v1 (성능 약간 상위, 검증 필요) |

## 검색 전략 (§5 참조)

- 방식: **하이브리드 (벡터 + BM25)** — 근거: 불량코드/부품번호 등 고유명사 검색(BM25) + 의미 검색(벡터) 모두 필요
- 결합: **RRF** (초기 — 가중치 튜닝 불필요, 안정적)
- top-k: quality_standards 5건, defect_history 10건, investigation_reports 5건
- 교차 컬렉션: "이 불량의 원인 분석해줘" → defect_history + quality_standards + investigation_reports 동시 검색 → 합친 후 리랭킹
- 메타데이터 필터링: plant, defect_code_std, part_no로 범위 좁힘

## 리랭킹 (§6 참조)

- 적용 여부: **적용** — 근거: 교차 컬렉션 검색, 불량코드 용어 불일치로 false positive 예상
- 리랭커 모델: **BGE-reranker-v2-m3** (로컬) — 근거: 로컬 필수(보안), GPU 보유, 한국어 지원, Apache 2.0
- 파이프라인: 하이브리드 검색 top-20 → 리랭킹 → 최종 top-5 → LLM 전달

## 인덱싱 범위 (§7 참조)

| 컬렉션 | 유형 | 범위 | 근거 |
|-------|------|------|------|
| quality_standards | 최신형 | 최신 revision만. 개정 시 구버전 삭제. | 구버전 기준으로 검사하면 오류 |
| defect_history | 이력형 | 3년+, severity=critical은 기간 무관 전수 | 패턴 분석에 장기 이력 필요 |
| investigation_reports | 이력형 | 전수 (기간 제한 없음) | 과거 원인-대책 패턴이 핵심 자산 |

## 갱신 파이프라인 (§8 참조)

| 컬렉션 | 트리거 | 방식 | doc_id |
|-------|--------|------|--------|
| quality_standards | 문서 개정 시 (수동 트리거) | 구버전 삭제 → 신버전 인덱싱 | 파일경로 + revision |
| defect_history | 일 1회 스케줄 | MES 증분 동기화 (신규 건만 추가) | MES 불량 ID |
| investigation_reports | 8D 보고서 종결 시 (수동 트리거) | 신규 추가 | 보고서 파일명 |

## GraphRAG 판단

| # | 조건 | 충족 | 근거 |
|---|------|:----:|------|
| 1 | 동음이의어/용어 불일치 빈번 | **✓** | "B-crack" vs "브레이크디스크_균열", "피로파괴" vs "크랙" — 200코드 중 60개(30%) 불일치 |
| 2 | 인과관계 질의 >20% | **✓** | 3/13 (23%) — "원인 분석", "유사 건 원인", "피로파괴와 크랙 관계" |
| 3 | RAG 적중률 목표 미달 | ✗ | 아직 RAG 미구축 — 구축 후 평가 (보류) |
| 4 | 동일 엔티티 다중 맥락 | **✓** | 같은 불량 현상이 공장별(화성/아산) + 제품군별(브레이크/서스펜션) 다르게 표현 |
| | **합계** | **3/4** | |

→ 결정: **승인 (3/4 충족)**

## GraphRAG 구성 (§9~§12 참조)

- 기술 스택: **pgvector + Apache AGE** (같은 PostgreSQL 16)
- 노드 (엔티티 유형):
  - `DefectType` — 불량 유형 (크랙, 기포, 치수불량 등)
  - `DefectCode` — 불량코드 (B-crack, 브레이크디스크_균열 등)
  - `Part` — 부품 (브레이크 디스크, 서스펜션 암 등)
  - `Process` — 공정 (주조, 가공, 열처리 등)
  - `RootCause` — 근본 원인 (원재료 결함, 공정 온도 이탈 등)
- 엣지 (관계 유형):
  - `DefectCode` -[MEANS]→ `DefectType` (동의어 관계 — 매핑 엑셀 시드)
  - `DefectType` -[OCCURS_IN]→ `Process` (어떤 공정에서 발생)
  - `DefectType` -[AFFECTS]→ `Part` (어떤 부품에 영향)
  - `DefectType` -[CAUSED_BY]→ `RootCause` (인과관계)
  - `RootCause` -[RESOLVED_BY]→ `Action` (대책 관계 — 8D 보고서에서 추출)
- 활용:
  - 질의 확장: "크랙" 검색 → MEANS 관계로 "B-crack", "브레이크디스크_균열", "피로파괴" 모두 검색
  - 인과 추적: 불량유형 → 근본원인 → 과거 대책 체인 2-hop 탐색
- 시드 데이터: 불량코드 매핑 엑셀(60쌍) → DefectCode-MEANS-DefectType 초기 구축
- 추출: 8D 보고서에서 LLM 기반 엔티티+트리플 자동 추출 (§10)
