# 자동차보험 심사 보조 — Phase 1 종합 요약

> 작성일: 2026-04-03
> 고객: 미래에셋손해보험
> **설계 신뢰도: ✅ 100% — 리소스 전부 실제, 레퍼런스 참조 설계**
> **참조: RAG 설계방법론 v10, GraphRAG 설계방법론 v10, CLI 런북 v10**

## 아키텍처 개요

### 행위 유형별 배치

| 행위 유형 | 비율 | 배치 방식 | 핵심 구성요소 | 신뢰도 |
|----------|------|----------|-------------|:------:|
| 탐색 | 57% | **GraphRAG** + 하이브리드 RAG | 3개 컬렉션 (pgvector), Apache AGE 그래프 | ✅ |
| 접근 | 14% | CLI + REST | uw CLI (PostgreSQL), core CLI (REST API) | ✅ |
| 실행(확정) | 14% | 결정적 파이프라인 | 가명처리, 감사로그, 재심사 절차 | ✅ |
| 실행(유연) | 14% | LLM + Human-in-the-loop | 심사 의견 초안, 할증 판단 보조 | ✅ |

### GraphRAG 판단
- 결정: **승인 (4/4 만점)**
- 핵심: 약관 조항 간 관계 추적 (면책→특약→할증)

### 인프라 요약
- LLM: **Bedrock Claude**
- 임베딩: **Titan Embeddings V2**
- 벡터DB: **pgvector** (기존 PostgreSQL 확장, HNSW 인덱스)
- 그래프DB: **Apache AGE** (같은 PostgreSQL 16에서 — GraphRAG 설계방법론 §1)
- REST: **FastAPI** (내부)
- CLI: **2개** (uw, core)

### 기존 인프라 재사용
- PostgreSQL (심사 DB) → pgvector + Apache AGE 확장만 추가
- AWS 인프라 전부 재사용
- 신규: Bedrock 개통 + PostgreSQL 확장(pgvector, Apache AGE) + FastAPI

## 전체 아키텍처 플로우

```
심사역 (심사 시스템) → FastAPI 게이트웨이
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
      [탐색]        [접근]        [실행]
    GraphRAG        uw CLI        가명처리 PL
    (Apache AGE)    core CLI      감사로그 PL
    질의 확장          ↓          재심사 PL
       ↓          JSON 반환      LLM 의견 초안
    하이브리드 RAG                LLM 할증 판단
    (pgvector)                      ↓
    3개 컬렉션                 심사역 확인
       ↓                    (Human Gate)
    리랭킹                       ↓
       ↓                    감사로그 기록
    LLM 응답 + 판단 근거        (AI+심사역 함께)
```

## 예상 일정

| 단계 | 기간 | 내용 |
|------|------|------|
| 인프라 | 1주 | Bedrock 개통, pgvector+Apache AGE 확장, FastAPI |
| 탐색 (RAG) | 3주 | 3개 컬렉션 구축, 파싱/임베딩/검색, 하이브리드+리랭킹 |
| GraphRAG | 2주 | 약관 관계 그래프 구축, 질의 확장, 관계 추적 |
| 접근 (CLI) | 1주 | uw CLI + core CLI + REST 래핑 |
| 실행 | 2주 | 가명처리, 감사로그, 심사 의견 초안, 할증 판단 |
| 통합 | 1주 | 체이닝, 엔드투엔드, 심사역 피드백 |
| **합계** | **10주** | |

## 이전 테스트와 비교 (레퍼런스 참조 효과)

| 항목 | 이전 (병원, 레퍼런스 없음) | 이번 (보험, 레퍼런스 있음) |
|------|----------------------|----------------------|
| 그래프DB | Neo4j Community | **Apache AGE (같은 PostgreSQL)** |
| 벡터DB | Milvus/Qdrant | **pgvector (기존 PostgreSQL 확장)** |
| 인프라 추가 | Neo4j 별도 설치 | **PostgreSQL 확장만 (pgvector+AGE)** |
| 설계 근거 | 스킬 내장 지식 | **§1~§12 참조 명시** |

## 리스크

| 리스크 | 영향 | 대응 |
|--------|------|------|
| 약관 관계 그래프 초기 품질 | 잘못된 조항 관계 추적 | 심사팀장 감수, 약관 전문가 검토 |
| OCR 데이터 품질 혼재 | claim_docs 검색 노이즈 | 전처리 (품질 점수 필터링, 최소 길이) |
| 심사역 수용도 | AI 보조 불신 | 초기 "검색만" 기능으로 시작, 점진적 확대 |
| 금감원 검사 | 판단 근거 미비 | 감사로그 완전성 검증 (누락 0건) |

## 다음 단계
→ /irm-validate로 Phase 2 (구현 설계 + 검증) 진행
