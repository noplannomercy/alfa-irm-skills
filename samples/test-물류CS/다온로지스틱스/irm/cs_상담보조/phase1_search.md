# cs_상담보조 — 탐색 배치 설계

> 작성일: 2026-04-11
> 탐색 질의: 8건 / 전체 12건 (67%)
> 참조: RAG 설계방법론 v10

## 데이터 소스 분석

| # | 소스명 | 형태 | 갱신 주기 | 이력형/최신형 | 규모 | 근거 |
|---|--------|------|----------|-------------|------|:----:|
| 1 | FAQ | 구글 시트 | 수시 (3개월 내 업데이트) | 최신형 | 300건 | ✅ |
| 2 | 상담 매뉴얼 | Notion | 비정기 | 최신형 | 50건 | ✅ |
| 3 | 상담 이력 | CRM(PostgreSQL) | 실시간 | 이력형 | 2년 50만건 | ✅ |

## 컬렉션 설계 (§1 참조)

### 분리 기준 판단

#### [FAQ + 매뉴얼] vs [상담 이력]
| 기준 | 판단 | 근거 |
|------|------|------|
| 갱신 주기 | **분리** | FAQ/매뉴얼은 비정기 개정(최신형), 상담 이력은 실시간(이력형) |
| 메타데이터 | **분리** | FAQ는 category/question, 이력은 channel/customer/resolution |
| 질의 패턴 | **분리** | "반품 방법?" vs "이전에 어떻게 답했지?" — 거의 교차 안 됨 |
| **판단** | **3/3 → 분리** | |

#### [FAQ] vs [매뉴얼]
| 기준 | 판단 | 근거 |
|------|------|------|
| 갱신 주기 | 통합 | 둘 다 비정기 |
| 메타데이터 | 통합 | 둘 다 category/doc_type으로 커버 |
| 질의 패턴 | 통합 | "반품 어떻게?" → FAQ+매뉴얼 동시 검색이 자연스러움 |
| **판단** | **0/3 → 통합** | |

### 컬렉션 구조

| 컬렉션명 | 포함 소스 | 메타데이터 | 인덱싱 범위 | 갱신 방식 |
|----------|----------|-----------|-----------|----------|
| **knowledge_base** | FAQ + 상담 매뉴얼 | doc_type, category | 최신형 — 최신 버전만 | 수시 (FAQ 변경 시 재인덱싱) |
| **consultation_history** | 상담 이력 | channel, category, resolution_type | 이력형 — 6개월 | 일 1회 CRM 동기화 |

### 벡터 저장소 선택
- 선택: **pgvector** — 근거: 이미 AWS RDS PostgreSQL 사용 중, 확장만 추가. 소규모(350건 + 이력).
- 인덱스: **HNSW** — 범용 최적

## 메타데이터 스키마 (§2 참조)

### 공통 메타데이터
```json
{
  "doc_type": "string",    // faq / manual / consultation
  "category": "string",    // delivery / return / claim / pricing / booking
  "date": "date",
  "source": "string"       // 원본 URL/ID
}
```

### knowledge_base 도메인 메타데이터
```json
{
  "question": "string",    // FAQ 질문 원문 (FAQ만 해당)
  "section": "string"      // 매뉴얼 섹션명 (매뉴얼만 해당)
}
```

### consultation_history 도메인 메타데이터
```json
{
  "channel": "string",         // kakaotalk / phone / email
  "resolution_type": "string", // resolved / escalated / pending
  "agent_id": "string"         // 상담사 ID (개인정보 아님)
}
```

## 파싱 & 청킹 (§3 참조)

| 컬렉션 | 데이터 형태 | 도구 | 전략 |
|-------|-----------|------|------|
| knowledge_base (FAQ) | 구글 시트 | 커스텀 전처리 | 1 FAQ = 1 청크 (질문+답변) |
| knowledge_base (매뉴얼) | Notion | Notion API → 마크다운 → 재귀적 분할 | 섹션 단위 |
| consultation_history | CRM 구조화 | 커스텀 전처리 | 1 상담 = 1 청크 |

## 임베딩 모델 (§4 참조)
- 선택: **Bedrock — Amazon Titan Embeddings** 또는 **Cohere Embed v3**
- 근거: AWS 환경, API 기반, 클라우드 OK (보안 제한 없음). 한국어 지원.
- 대안: BGE-m3-ko 로컬 — 비용 절감 필요 시

## 검색 전략 (§5 참조)
- 방식: **하이브리드 (벡터 + BM25)** — FAQ에 고유명사(집하, 착불 등) 많음
- 결합: **RRF**
- top-k: knowledge_base 5건, consultation_history 5건
- 교차 컬렉션: "이 문의 답변 만들어줘" → 양쪽 동시 검색

## 리랭킹 (§6 참조)
- 적용 여부: **미적용** (초기) — 소규모, 2개 컬렉션, 검색 품질 충분할 가능성. 파일럿 후 정확도 미달 시 추가.

## 인덱싱 범위 (§7 참조)
| 컬렉션 | 유형 | 범위 |
|-------|------|------|
| knowledge_base | 최신형 | 최신 버전만. FAQ 수정 시 해당 건 재인덱싱. |
| consultation_history | 이력형 | 최근 6개월 (반복 패턴 분석에 충분) |

## 갱신 파이프라인 (§8 참조)
| 컬렉션 | 트리거 | 방식 |
|-------|--------|------|
| knowledge_base | FAQ 시트 변경 감지 or 주 1회 | 전체 재인덱싱 (350건 — 소요 수 분) |
| consultation_history | 일 1회 스케줄 | CRM 증분 동기화 (신규 건만) |

## GraphRAG 판단

| # | 조건 | 충족 | 근거 |
|---|------|:----:|------|
| 1 | 동음이의어/용어 불일치 | ✗ | 물류 용어 통일됨 |
| 2 | 인과관계 질의 >20% | ✗ | 0/12 (0%) |
| 3 | RAG 적중률 목표 미달 | ✗ | 미구축 — 해당 없음 |
| 4 | 동일 엔티티 다중 맥락 | ✗ | 없음 |
| | **합계** | **0/4** | |

→ 결정: **불필요**
