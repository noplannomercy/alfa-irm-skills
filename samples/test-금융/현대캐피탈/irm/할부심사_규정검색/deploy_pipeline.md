# 할부심사_규정검색 — 배포 설계

> 작성일: 2026-04-21
> 고객: 현대캐피탈
> 인프라 경로: AWS (금융 리전, Bedrock CSAP 인증)

## S3 폴더 구조

```
s3://hyundai-capital-ai/할부심사_규정검색/
├── raw/                              ← 원본 파일
│   ├── regulations/                  ← 여신규정 + 자동차특약 + 리스특약
│   │   ├── 여신규정_v2026Q1.pdf
│   │   ├── 자동차금융특약_v2026Q1.pdf
│   │   └── 리스특약_v2026Q1.pdf
│   ├── guidelines/                   ← 심사 가이드라인
│   │   ├── 심사가이드라인.pdf
│   │   └── 심사가이드라인_부록.hwp
│   └── history/                      ← 심사 이력 (비식별화 CSV)
│       └── review_history_anonymized_202604.csv
├── parsed/                           ← Forge 파싱 결과
│   ├── regulations/
│   │   ├── 여신규정_v2026Q1.md
│   │   ├── 자동차금융특약_v2026Q1.md
│   │   └── 리스특약_v2026Q1.md
│   └── guidelines/
│       ├── 심사가이드라인.md
│       └── 심사가이드라인_부록.md
├── indexed/                          ← Cortex 인제스트 로그
│   ├── regulations_20260501.json
│   ├── guidelines_20260501.json
│   └── review_history_20260501.json
└── config/
    ├── routing_map.json
    └── collection_config.json
```

## Forge 파싱 계획

| # | 소스명 | 파일 형식 | Forge 경로 | 청킹 전략 | S3 위치 |
|---|--------|----------|-----------|----------|--------|
| 1 | 여신규정 | PDF (~800p) | extract | 조항 단위 (편→장→조→항 보존) | raw/regulations/ |
| 2 | 자동차금융 특약 | PDF (~600p) | extract | 조항 단위 | raw/regulations/ |
| 3 | 리스 특약 | PDF (~400p) | extract | 조항 단위 | raw/regulations/ |
| 4 | 심사 가이드라인 (본문) | PDF (~180p) | extract | 사례 단위 (1사례=1~2청크) | raw/guidelines/ |
| 5 | 심사 가이드라인 (부록) | HWP (~20p) | vlm (semantic) | 사례 단위 | raw/guidelines/ |

### Forge 호출 설정
- 엔드포인트: `http://forge:8003/convert` (ECS 내부)
- VLM 사용: HWP 파일만 (5번) — extract로 HWP 처리 안 되면 VLM semantic 경로
- 메타 추출: 자동 (doc_type, topic, keywords, language, summary)
- 주의: PoC 잔여 PDF 전환본 **사용하지 않음** — 원본 PDF로 재파싱

## Cortex 컬렉션 설정

### regulations
```json
{
  "collection": "regulations",
  "embedding_model": "bedrock/amazon.titan-embed-text-v2",
  "metadata_schema": {
    "공통": ["doc_type", "date", "source", "product_type"],
    "도메인": ["chapter", "section", "article", "paragraph", "revision"]
  },
  "index_type": "HNSW",
  "search_strategy": "hybrid",
  "reranker": "bge-reranker-v2-m3",
  "top_k": 10,
  "update_trigger": "분기 정기 개정 + 수시",
  "update_frequency": "분기 1회 전체 재인덱싱"
}
```

### guidelines
```json
{
  "collection": "guidelines",
  "embedding_model": "bedrock/amazon.titan-embed-text-v2",
  "metadata_schema": {
    "공통": ["doc_type", "date", "source"],
    "도메인": ["case_type", "interpretation_target", "author"]
  },
  "index_type": "HNSW",
  "search_strategy": "hybrid",
  "top_k": 5,
  "update_trigger": "변경 시",
  "update_frequency": "수시"
}
```

### review_history
```json
{
  "collection": "review_history",
  "embedding_model": "bedrock/amazon.titan-embed-text-v2",
  "metadata_schema": {
    "공통": ["doc_type", "date", "source"],
    "도메인": ["product_type", "decision", "anonymized"]
  },
  "index_type": "HNSW",
  "search_strategy": "hybrid",
  "top_k": 5,
  "update_trigger": "월 1회",
  "update_frequency": "CBS → 비식별화 → 증분"
}
```

### 임베딩 모델 주의
- Bedrock Titan V2가 한국어 법률/규정 용어에서 품질 미달 시 → **BGE-m3-ko 로컬(ECS)로 전환**
- **파일럿 1주차에 샘플 30건 임베딩 테스트 필수** — 전환 시 전체 재인덱싱

## 라우팅 맵 (Forge → Cortex)

| Forge 메타 조건 | → Cortex 컬렉션 | extra_metadata |
|----------------|----------------|----------------|
| doc_type = "rulebook" AND source contains "여신규정" | regulations | product_type: "공통" |
| doc_type = "rulebook" AND source contains "자동차" | regulations | product_type: "자동차할부" |
| doc_type = "rulebook" AND source contains "리스" | regulations | product_type: "리스" |
| doc_type = "manual" OR doc_type = "guide" | guidelines | — |
| doc_type = "consultation" | review_history | anonymized: true |

```json
{
  "routes": [
    {
      "condition": {"doc_type": "rulebook", "source_contains": "여신규정"},
      "target_collection": "regulations",
      "extra_metadata": {"product_type": "공통"}
    },
    {
      "condition": {"doc_type": "rulebook", "source_contains": "자동차"},
      "target_collection": "regulations",
      "extra_metadata": {"product_type": "자동차할부"}
    },
    {
      "condition": {"doc_type": "rulebook", "source_contains": "리스"},
      "target_collection": "regulations",
      "extra_metadata": {"product_type": "리스"}
    },
    {
      "condition": {"doc_type": ["manual", "guide"]},
      "target_collection": "guidelines",
      "extra_metadata": {}
    },
    {
      "condition": {"doc_type": "consultation"},
      "target_collection": "review_history",
      "extra_metadata": {"anonymized": true}
    }
  ]
}
```

## GraphRAG 배포

- DB: Apache AGE (Cortex와 같은 RDS PostgreSQL)
- 스키마:
  - 노드: Regulation, Article, Product, Topic, Term
  - 엣지: BELONGS_TO, APPLIES_TO, COVERS, MEANS, SUPERSEDES
- 시드 데이터:
  - 규정집 목차 → Article 노드 + BELONGS_TO(Regulation) 관계
  - 조항별 상품 적용 → APPLIES_TO(Product) 관계 — **수동 검증 필요**
  - 주제 매핑 → COVERS(Topic) 관계
- 추출 파이프라인: Cortex 인제스트 시 `extract=true` (가이드라인/이력에서 추가 트리플)
- **APPLIES_TO 매핑이 PoC 실패 해결 핵심** — 심사기획팀 3명 검증 세션 필요

## 접근 레이어 배포

| CLI | 대상 시스템 | 배포 방식 | 비고 |
|-----|-----------|----------|------|
| cli-cbs | CBS (Oracle, REST API) | N8N HTTP Request 노드 | 오토에버 API 세팅 |

### 신용정보 차단 (프록시)
- 대상: CBS REST API 응답
- 방식: **화이트리스트** (허용 필드만 통과)
- 허용 필드: product_type, vehicle_info, document_status, dealer_info, application_date
- 차단 필드: credit_score, delinquency_history, income, debt_ratio (+ 신규 필드 기본 차단)
- 위치: N8N Function 노드 (Forge/Cortex 전 단계)
- 검증: 샘플 50건 전수 확인 → 신용정보 통과 0건

### 감사로그
- 모든 CBS 조회: 사용자/일시/심사ID/조회항목 기록
- 모든 AI 호출: 입력/출력/타임스탬프/사용자 기록
- 저장: RDS 별도 테이블 (audit_logs)
- AI 감사 리포트: 분기 1회 자동 생성

## N8N 워크플로우 설계

### 워크플로우 1: 규정 인제스트 (S3 → Forge → Cortex)
```
[트리거] S3 raw/regulations/ 파일 업로드
    ↓
[노드1] Forge /convert 호출 (extract 경로)
    ↓
[노드2] 라우팅 맵 참조 → regulations 컬렉션 + product_type 메타
    ↓
[노드3] Cortex /ingest 호출 (extract=true → GraphRAG 트리플 추출)
    ↓
[노드4] S3 indexed/ 로그 기록
```

### 워크플로우 2: 심사 의견서 초안 (메인 체이닝)
```
[트리거] 심사역 질의 입력
    ↓
[노드1] CBS REST API → 신용정보 차단 프록시(Function) → 심사 건 정보
    ↓
[노드2] Cortex /recall (regulations + GraphRAG 질의 확장)
         → product_type 필터로 해당 상품 조항만
    ↓
[노드3] Cortex /search (guidelines) — 병렬
[노드4] Cortex /search (review_history) — 병렬
    ↓
[노드5] Bedrock Claude → 의견서 초안 생성
         → "AI 보조 활용" + "참조: 자동차금융특약 제X조 제X항"
    ↓
[노드6] 감사로그 기록
    ↓
[노드7] 심사역 UI 반환 (검토/수정/확정 + 차이 로그)
```

### 워크플로우 3: 분기 규정 갱신
```
[트리거] 분기 1회 스케줄 (1월/4월/7월/10월)
    ↓
[노드1] S3 raw/regulations/ 신규 버전 감지
    ↓
[노드2] Cortex 기존 regulations 컬렉션 삭제
    ↓
[노드3] Forge 재파싱 → Cortex 재인덱싱 (전체)
    ↓
[노드4] GraphRAG SUPERSEDES 관계 업데이트
    ↓
[노드5] indexed/ 로그 기록
```

## 배포 순서

```
1. S3 버킷 + 폴더 구조 생성 (오토에버)
2. Forge ECS 배포 (우리)
3. Cortex ECS 배포 + 컬렉션 3개 생성 (우리)
4. Bedrock 활성화 + 임베딩 모델 테스트 (오토에버 + 우리)
5. 규정집 원본 S3 업로드 → 워크플로우 1 실행 → 인제스트
6. GraphRAG 시드: 목차 구조 + APPLIES_TO 매핑 (우리 + 심사기획팀 검증)
7. 신용정보 차단 프록시 배포 + 검증 50건 (우리 + 컴플라이언스)
8. N8N 워크플로우 2(메인) + 3(갱신) 배포 (우리)
9. 감사로그 테이블 + AI 감사 리포트 (우리 + 오토에버)
10. 엔드투엔드 테스트
```

### 오토에버 vs 우리 (배포 기준)

| 단계 | 오토에버 | 우리 |
|------|---------|------|
| 1. S3 구조 | ✅ | — |
| 2. Forge 배포 | ECS 인프라 | 컨테이너 + 설정 |
| 3. Cortex 배포 | ECS + RDS | 컨테이너 + 컬렉션 설정 |
| 4. Bedrock | 활성화 + IAM | 임베딩 테스트 |
| 5. 인제스트 | — | ✅ |
| 6. GraphRAG | — | ✅ (심사팀 검증) |
| 7. 프록시 | — | ✅ (컴플라이언스 검증) |
| 8. N8N | ECS 인프라 | 워크플로우 설계+구현 |
| 9. 감사 | DB 테이블 | 로직 + 리포트 |
| 10. 테스트 | 공동 | 공동 |
