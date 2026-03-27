# IRM Phase 2 — 구현 계획 (Implementation Plan) — 안전관리 보조

> 고객: 대림건설
> 유즈케이스: 안전관리 보조
> 작성일: 2026-03-27
> 신뢰도 전파: Phase 0 분류 → Phase 1 설계 → Phase 2 구현 계획
> 일정 기준표: 컬렉션 규모별 + GraphRAG 복잡도별 리스크 가산 적용

---

## 1. 신뢰도 전파 요약 (Phase 0 → Phase 2)

| 단계 | safety_knowledge | regulation | accident_cases |
|------|-----------------|-----------|----------------|
| Phase 0 원본 신뢰도 | H | H (단, 갱신 지연) | H (단, 전처리 필요) |
| Phase 0 컬렉션 신뢰도 | H | **M** (갱신 파이프라인 미구축) | **M** (마스킹+전처리 미완) |
| Phase 2 목표 신뢰도 | H (유지) | **H 승급** (갱신 파이프라인 구축 후) | **H 승급** (전처리 완료 후) |
| 구현 우선순위 | 3순위 (기반) | **1순위** (법규 위반 리스크) | **2순위** (TBM 핵심 기능) |

---

## 2. 단계별 구현 계획

### Phase 2-A: 환경 구성 (1주)

**목표**: AWS 인프라 + PostgreSQL 확장 설치 완료

| 작업 | 담당 | 결과물 |
|------|------|--------|
| RDS PG 버전 확인 + pgvector 설치 | IT팀 인프라 | pgvector 동작 확인 |
| Apache AGE 설치 (Docker 기반 개발 환경 우선) | IT팀 인프라 | AGE 동작 확인 |
| 3개 컬렉션 테이블 생성 | IT팀 데이터 | safety_knowledge, regulation, accident_cases 테이블 |
| safety_graph 그래프 공간 생성 | IT팀 데이터 | AGE 그래프 공간 |
| BGE-m3-ko 모델 로컬 설치 | IT팀 데이터 | /opt/models/BGE-m3-ko/ |
| AWS Secrets Manager 키 등록 | IT팀 인프라 | OpenAI, 현장API, KMA API Key |
| 읽기 전용 DB 계정 발급 | IT팀 인프라 | safety_ai_reader 계정 |

**완료 기준**: `SELECT version(); \dx` → pgvector + AGE 모두 표시

---

### Phase 2-B: 데이터 전처리 (2주)

**목표**: 인덱싱 가능한 형태로 모든 데이터 준비

**우선 순위 1: 사고 사례집 (accident_cases)**

```python
# 1단계: Excel 파싱 + 개인정보 마스킹
import pandas as pd
import re

df = pd.read_excel('/docs/accident/사고사례집.xlsx')
# 이름 마스킹
df['근로자'] = df['근로자'].apply(lambda x: re.sub(r'[가-힣]{2,4}', '○○○', str(x)))
# 연락처 마스킹
df['연락처'] = df['연락처'].apply(lambda x: '마스킹')

# 2단계: 1사고 = 1청크 형식으로 변환
def accident_to_chunk(row):
    return f"""사고유형: {row['사고유형']}
발생일: {row['발생일']}
공종: {row['공종']}
발생원인: {row['원인']}
조치사항: {row['조치']}
재발방지: {row['재발방지']}"""
```

검증: 180건 전체 처리 + 마스킹 확인 + root_cause LLM 추출 정확도 80%+

**우선 순위 2: 법규집 (regulation)**

```python
# Docling으로 조항 단위 청킹
from docling.document_converter import DocumentConverter
from docling.chunking import HybridChunker

doc = DocumentConverter().convert('/docs/regulation/산업안전보건법.pdf').document
# 조항 단위 청크 경계: "제○○조" 패턴
```

검증: 조항번호 추출 정확도 + 구버전 제거 확인

**우선 순위 3: 매뉴얼+SOP+체크리스트 (safety_knowledge)**

```python
# 4종 매뉴얼 + 30종 SOP + 20종 체크리스트 파싱
for file_path in safety_files:
    doc = DocumentConverter().convert(file_path).document
    chunks = list(chunker.chunk(dl_doc=doc))
```

---

### Phase 2-C: 인덱싱 (2주)

**목표**: 3개 컬렉션 전체 임베딩 + HNSW 인덱싱 완료

**순서**: regulation → accident_cases → safety_knowledge

```python
from FlagEmbedding import BGEM3FlagModel
import psycopg2
from pgvector.psycopg2 import register_vector

model = BGEM3FlagModel('dragonkue/BGE-m3-ko', use_fp16=True)

def index_chunks(chunks, table_name, meta_extractor):
    conn = psycopg2.connect(SAFETY_DB_URL)
    register_vector(conn)
    cur = conn.cursor()

    for batch in batched(chunks, batch_size=32):
        texts = [c['content'] for c in batch]
        embeddings = model.encode(texts)['dense_vecs']

        for chunk, embedding in zip(batch, embeddings):
            meta = meta_extractor(chunk)
            cur.execute(
                f"INSERT INTO {table_name} (content, embedding, ...) VALUES (%s, %s, ...)",
                (chunk['content'], embedding.tolist(), *meta.values())
            )
    conn.commit()
```

**완료 기준**:
- safety_knowledge: 4,000~6,000 청크 확인
- regulation: 6,000~10,000 청크 확인
- accident_cases: 180건 = 180~360 청크 (PDF 상세 포함 시)
- HNSW 인덱스 생성 완료 (3개 테이블)

**신뢰도 업데이트**:
- regulation: 인덱싱 완료 + 구버전 제거 확인 후 → **H 승급**
- accident_cases: 마스킹 검증 + 전처리 완료 후 → **H 승급**

---

### Phase 2-D: GraphRAG 구축 (2주)

**목표**: safety_graph 동의어+관계 구축

**D-1: 동의어 노드 수동 등록 (안전팀 협의 후)**

```python
# 안전팀 워크숍에서 확정한 동의어 목록
SYNONYMS = [
    ("고소작업", "높이작업"),
    ("고소작업", "고공작업"),
    ("굴착작업", "터파기작업"),
    ("협착", "끼임"),
    # ... 추가 동의어
]

for term1, term2 in SYNONYMS:
    # AGE에 동의어 관계 노드 + 엣지 생성
    create_synonym_edge(term1, term2)
```

**D-2: 문서별 엔티티+트리플 자동 추출**

```python
# GPT-4o-mini로 건설 안전 도메인 트리플 추출
SAFETY_EXTRACT_PROMPT = """
건설 안전 도메인에서 엔티티와 관계를 추출해.
엔티티 유형: 작업유형, 법규, 위험요인, 안전조치
관계: 인과/조건/적용/예방 관계만 추출
규칙:
- from ≠ to (자기참조 금지)
- 관계명 동사형 (영향받다, 유발하다, 적용되다, 예방하다)
- 엔티티 10개 이내, 트리플 10개 이내
"""
```

**완료 기준**:
- "높이작업" 검색 → "고소작업" 관련 결과 반환 확인
- 법규→매뉴얼 2홉 탐색 동작 확인
- 고아 노드 0건 확인

---

### Phase 2-E: CLI 개발 (2주)

**목표**: 4개 CLI 구성요소 개발 + 테스트 + 설치

개발 우선순위:
1. `cli-anything-safety-search` (핵심)
2. `cli-anything-weather` (기상 판단)
3. `cli-anything-safety-db` (이력 조회)
4. `cli-anything-field-api` (현장 정보)

**완료 기준**: 각 CLI `--help` 동작 + 기본 커맨드 JSON 응답 확인

---

### Phase 2-F: 에이전트 + API 개발 (3주)

**목표**: LLM 에이전트 + FastAPI 엔드포인트 + 로깅 구현

```
주 1: 기본 Q&A 파이프라인 (질의분류→검색→응답)
주 2: GraphRAG 통합 + 교차 검색
주 3: 기상 판단 체인 + TBM 체인
```

**완료 기준**: TC-01~TC-10 통과 (20개 중 10개 기본 시나리오)

---

### Phase 2-G: UI + 체이닝 (2주)

**목표**: 최소 웹 UI + TBM/기상 체인 완성

---

### Phase 2-H: 법규 갱신 파이프라인 (1주)

**목표**: regulation 컬렉션 즉시 갱신 자동화

```python
# 법령정보 사이트 변경 감지 → 구버전 삭제 → 신버전 인덱싱
def update_regulation(new_pdf_path):
    # 1. 기존 청크 삭제 (같은 law_name)
    cur.execute("DELETE FROM regulation WHERE law_name = %s", (law_name,))
    # 2. 신버전 파싱 + 인덱싱
    index_regulation(new_pdf_path)
    # 3. GraphRAG 관계 업데이트
    # 4. 안전팀장 알림
```

**완료 기준**: regulation 신뢰도 **M → H 승급** 확인

---

### Phase 2-I: QA + 사용자 테스트 (3주)

**목표**: TC-01~TC-20 전체 통과 + UAT 완료

| 주차 | 내용 |
|------|------|
| 1주 | 20개 TC 전체 실행 + 버그 수정 |
| 2주 | UAT (안전관리자 10명, 1주 사용) |
| 3주 | UAT 피드백 반영 + 재검증 |

---

### Phase 2-J: PoC 준비 + 발표 (1주)

**목표**: 성공 기준 측정 + 발표 자료 준비

---

## 3. 전체 일정 (기준표 적용)

| 단계 | 기간 | 시작 | 완료 | 비고 |
|------|------|------|------|------|
| A. 환경 구성 | 1주 | 2026-04-07 | 2026-04-14 | - |
| B. 데이터 전처리 | 2주 | 2026-04-14 | 2026-04-28 | 사고사례 마스킹 포함 |
| C. 인덱싱 | 2주 | 2026-04-28 | 2026-05-12 | ~15,000청크, 소규모 기준 |
| D. GraphRAG | 2주 | 2026-05-12 | 2026-05-26 | 중간 복잡도 +1주 포함 |
| E. CLI 개발 | 2주 | 2026-05-26 | 2026-06-09 | - |
| F. 에이전트+API | 3주 | 2026-06-09 | 2026-06-30 | - |
| G. UI+체이닝 | 2주 | 2026-06-30 | 2026-07-14 | - |
| H. 갱신 파이프라인 | 1주 | 2026-07-14 | 2026-07-21 | regulation H 승급 |
| I. QA+UAT | 3주 | 2026-07-21 | 2026-08-11 | 안전관리자 10명 |
| J. PoC 준비 | 1주 | 2026-08-11 | 2026-08-18 | - |
| **총계** | **21주** | **2026-04-07** | **2026-09-08** | 기한 10-31보다 7주 여유 |

**리스크 가산 근거**:
- 기준 소규모 인덱싱: 1~2주 → 2주 적용
- GraphRAG 중간 복잡도: +1주 적용
- IT팀 50% 가용: +2주 적용 (A~C 단계 병렬 처리 제한)
- 법규 갱신 파이프라인 신규: +1주 적용
- 사고 사례 전처리: +1주 적용

---

## 4. 역할 분담

| 단계 | IT팀 백엔드 | IT팀 인프라 | IT팀 데이터 | 외부 컨설턴트 |
|------|-----------|-----------|-----------|------------|
| A | - | ✅ 주도 | ✅ 지원 | ✅ 가이드 |
| B | - | - | ✅ 주도 | ✅ 가이드 |
| C | - | - | ✅ 주도 | ✅ 지원 |
| D | ✅ 지원 | - | ✅ 주도 | ✅ 주도 |
| E | ✅ 주도 | - | ✅ 지원 | ✅ 가이드 |
| F | ✅ 주도 | - | - | ✅ 주도 |
| G | ✅ 주도 | - | - | ✅ 가이드 |
| H | - | - | ✅ 주도 | ✅ 가이드 |
| I | ✅ 지원 | - | - | ✅ 주도 |

---

## 5. 예산 집행 계획

| 단계 | 항목 | 금액 |
|------|------|------|
| A | AWS 인프라 셋업 | 50만원 |
| A~J | AWS EC2+RDS 운영 (7개월) | 350만원 |
| B~C | OpenAI API (인덱싱 GPT-4o-mini) | 50만원 |
| F~J | OpenAI API (GPT-4o 응답) | 250만원 |
| A~J | 외부 컨설팅 | 2,500만원 |
| A | 모델 서버 (GPU EC2, 단기) | 100만원 |
| 기타 | 도구, 라이선스 | 50만원 |
| **합계** | | **3,350만원** |
| 예비비 (16%) | | **650만원** |
| **총 예산** | | **4,000만원** |
