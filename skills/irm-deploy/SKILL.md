---
name: irm-deploy
description: IRM 배포 설계. Phase 1 결과를 S3 폴더 구조, Forge 파싱 계획, Cortex 컬렉션 설정, 라우팅 맵, N8N 워크플로우로 변환한다. "배포 설계", "S3 구조", "파이프라인 설계", "irm deploy" 등의 요청에 사용.
---

# IRM 배포 설계 (Deploy Design)

> Phase 1 설계 결과를 실제 구현 인프라(S3 + Forge + Cortex + N8N)에 매핑한다.

## 선행 조건

아래 파일이 이미 존재해야 한다. 없으면 해당 스킬을 먼저 실행하라고 안내한다.

- `phase1_search.md` ← `/irm-search`에서 생성
- `phase1_access.md` ← `/irm-access`에서 생성
- `phase1_execution.md` + `phase1_chaining.md` ← `/irm-exec`에서 생성
- `phase1_summary.md` ← `/irm-design`에서 생성

또한 고객 프로필(`alfa/profile.md`)에서 인프라 정보를 참조한다.

## 핵심 원칙

- **이 스킬은 새로운 설계를 하지 않는다** — Phase 1 결과를 구현 인프라에 매핑만 한다
- **인프라 경로에 따라 분기**: AWS / G-클라우드 / 온프레미스
- **Forge는 파싱만, Cortex는 인덱싱만, S3는 파일 관리만** — 역할 혼재 금지

## 폴더 구조

```
[회사명]/
├── irm/
│   └── [유즈케이스명]/
│       ├── phase1_*.md                  ← 이미 존재
│       ├── deploy_pipeline.md           ← 이 스킬이 생성
│       └── phase2_*.md                  ← /irm-validate에서 생성
└── STATUS.md                            ← 업데이트
```

## 실행 흐름

### Step 0: Phase 1 결과 + 프로필 로드

`irm/[유즈케이스명]/phase1_*.md`와 `alfa/profile.md`를 로드한다.
- phase1_search.md → 컬렉션 구조, 임베딩 모델, 갱신 방식
- phase1_access.md → CLI 대상 시스템
- phase1_execution.md + chaining.md → 워크플로우 플로우
- phase1_summary.md → 인프라 요약
- profile.md → 인프라 경로 (AWS/G-클라우드/온프레미스)

### Step 1: 인프라 경로 판단

프로필의 인프라 정보에 따라 분기한다.

| 인프라 | 파일 저장 | 파싱 | 인덱싱 | 워크플로우 |
|--------|----------|------|--------|----------|
| AWS | S3 | Forge (ECS) | Cortex (ECS) | N8N (ECS) 또는 Step Functions |
| G-클라우드 | G-클라우드 스토리지 또는 S3(CSAP) | Forge | Cortex | N8N |
| 온프레미스 | 로컬 파일서버 | Forge (로컬) | Cortex (로컬) | N8N (로컬) |

### Step 2: 배포 설계 생성

**파일:** `irm/[유즈케이스명]/deploy_pipeline.md`

```markdown
# [유즈케이스명] — 배포 설계

> 작성일: [날짜]
> 고객: [회사명]
> 인프라 경로: [AWS / G-클라우드 / 온프레미스]

## S3 폴더 구조

```
s3://[버킷명]/[회사명]/[유즈케이스명]/
├── raw/                    ← 원본 파일 (PDF, HWP, CSV 등)
│   ├── [소스1]/            ← 리소스 인벤토리 기반 하위 폴더
│   ├── [소스2]/
│   └── ...
├── parsed/                 ← Forge 파싱 결과 (Markdown)
│   ├── [소스1]/
│   └── ...
├── indexed/                ← Cortex 인제스트 완료 로그
│   └── [컬렉션명]_[날짜].json
└── config/
    ├── routing_map.json    ← Forge→Cortex 라우팅 맵
    └── collection_config.json  ← Cortex 컬렉션 설정
```

## Forge 파싱 계획

| # | 소스명 | 파일 형식 | Forge 경로 | S3 위치 | 규모 |
|---|--------|----------|-----------|--------|------|
| 1 | [소스명] | [PDF/HWP/...] | [extract/vlm] | raw/[소스명]/ | [건수/페이지] |
| ... | ... | ... | ... | ... | ... |

### Forge 호출 설정
- 엔드포인트: [Forge URL]
- VLM 사용: [예/아니오 — HWP, 스캔 PDF 등]
- 메타 추출: [자동 — doc_type, topic, keywords]

### 청킹 전략 (소스별)

| # | 소스명 | 청킹 단위 | 청킹 도구 | 특이사항 |
|---|--------|----------|----------|---------|
| 1 | [소스명] | [조항/섹션/사례/1건=1청크] | [Docling HybridChunker/커스텀/재귀적분할] | [계층 보존, 상위 맥락 포함 등] |
| ... | ... | ... | ... | ... |

### 데이터 품질 필터 (해당 시)
- [최소 문자 수, OCR 신뢰도, 필수 메타 누락 건 제외 등]

## Cortex 컬렉션 설정

### [컬렉션명 1]
```json
{
  "collection": "[컬렉션명]",
  "embedding_model": "[모델명]",
  "vector_dimension": [차원수],
  "metadata_schema": {
    "공통": ["doc_type", "date", "source"],
    "도메인": ["[필드1]", "[필드2]"]
  },
  "index_type": "HNSW",
  "indexing_scope": "[최신형: 현행만 / 이력형: N년]",
  "search": {
    "strategy": "[hybrid / vector_only]",
    "combination": "[RRF / weighted]",
    "top_k": [N],
    "metadata_filters": ["[필터 가능 필드들]"]
  },
  "reranker": {
    "model": "[모델명 또는 null]",
    "top_k_before": [N],
    "top_k_after": [N]
  },
  "update": {
    "trigger": "[갱신 트리거]",
    "frequency": "[갱신 주기]",
    "method": "[전체 재인덱싱 / 증분]"
  }
}
```

### [컬렉션명 2]
(동일 구조)

### 임베딩 모델 상세
- 모델: [모델명]
- 로컬/API: [로컬(GPU) / Bedrock / API]
- 한국어 성능: [검증 여부, 대안 모델]
- **주의: 모델 변경 시 전체 재인덱싱 필요 — 초기 선정 신중하게**

### 교차 컬렉션 검색 (해당 시)
- 대상 질의: [어떤 질의가 여러 컬렉션을 교차 검색하는지]
- 라우팅 방식: [질의 분석 → 대상 컬렉션 결정 프롬프트]
- 교차 리랭킹: [각 컬렉션 결과 합친 후 리랭킹 1회]

## 라우팅 맵 (Forge → Cortex)

| Forge 메타 조건 | → Cortex 컬렉션 | 비고 |
|----------------|----------------|------|
| doc_type = [값] | [컬렉션명] | [설명] |
| doc_type = [값] | [컬렉션명] | [설명] |
| ... | ... | ... |

```json
{
  "routes": [
    {
      "condition": {"doc_type": "[값]"},
      "target_collection": "[컬렉션명]",
      "extra_metadata": {"[키]": "[값]"}
    }
  ]
}
```

## GraphRAG 배포 (해당 시 — GraphRAG 승인된 경우만)

- DB: Apache AGE (Cortex와 같은 PostgreSQL)
- 노드 유형: [Phase 1 search에서 정의한 엔티티 목록]
- 엣지 유형: [Phase 1 search에서 정의한 관계 목록]
- 시드 데이터:
  - 소스: [어떤 문서/데이터에서 초기 관계 추출]
  - 방법: [자동 LLM 추출 / 수동 매핑 / 기존 데이터(매핑 엑셀 등)]
  - 수동 검증: [필요 여부 + 검증 주체]
- 추출 파이프라인: Cortex 인제스트 시 `extract=true` (이후 문서에서 자동 트리플 추출)
- 질의 확장 방식: [Term→MEANS→Entity 등 구체적 경로]
- 멀티홉: [기본 N-hop]
- 갱신 동기화: [벡터+그래프 동시 갱신 방식]

## 접근 레이어 배포

| CLI | 대상 시스템 | 배포 방식 | 비고 |
|-----|-----------|----------|------|
| [cli-xxx] | [시스템명] | [N8N HTTP노드 / Lambda / 직접] | [메모] |

### 비식별화/필터링 (해당 시)
- 대상: [어떤 시스템의 어떤 필드]
- 방식: [화이트리스트/마스킹/제거]
- 위치: [프록시/CLI 내부/N8N 노드]

## N8N 워크플로우 설계

### 워크플로우 1: 데이터 인제스트 (S3 → Forge → Cortex)
```
[트리거] S3 파일 업로드
    ↓
[노드1] Forge /convert 호출
    ↓
[노드2] 라우팅 맵 참조 → 대상 컬렉션 결정
    ↓
[노드3] Cortex /ingest 호출 (메타데이터 포함)
    ↓
[노드4] indexed/ 로그 기록
```

### 워크플로우 2: 메인 체이닝 (사용자 질의 처리)
```
[트리거] 사용자 입력
    ↓
[노드1] [Phase 1 chaining 패턴 1 — 접근/탐색/실행 순서대로]
    ↓
...
[노드N] 응답 반환 + 감사로그
```

### 워크플로우 3: 갱신 (해당 시)
```
[트리거] [스케줄/이벤트]
    ↓
[노드1] [소스에서 변경 감지]
    ↓
[노드2] Forge 재파싱
    ↓
[노드3] Cortex 재인덱싱
```

## 감사/로그 (해당 시)
- 감사로그 저장: [위치]
- AI 감사 리포트: [빈도]
- 감사 대상: [모든 AI 호출 / 접근 조회 / 차이 로그]

## 배포 순서

```
1. S3 폴더 구조 생성
2. Forge 배포 (ECS/로컬)
3. Cortex 배포 + 컬렉션 생성
4. 원본 파일 S3 업로드 → 인제스트 워크플로우 실행
5. GraphRAG 시드 데이터 로드 (해당 시)
6. CLI/접근 레이어 배포
7. N8N 메인 워크플로우 배포
8. 엔드투엔드 테스트
```
```

### Step 3: STATUS.md 업데이트

IRM 진행 상태에 deploy 완료를 추가한다.

## 주의 사항

- **이 스킬은 새로운 설계를 하지 않는다** — Phase 1 결과를 인프라에 매핑만
- 인프라 경로(AWS/G-클라우드/온프레미스)에 따라 파일 저장/배포 방식이 달라짐
- Forge에 폴더/프로젝트 개념을 넣지 않는다 — S3가 파일 관리
- 라우팅 맵은 JSON으로 생성하여 N8N에서 참조 가능하게
- 비식별화/감사로그 요구사항은 Phase 2 GOVERN에서 가져온다
- GraphRAG 배포는 Cortex 인제스트의 extract=true 옵션으로 처리
