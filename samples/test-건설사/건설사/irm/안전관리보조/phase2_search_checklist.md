# IRM Phase 2 — 탐색 구현 체크리스트 — 안전관리 보조

> 고객: 대림건설
> 유즈케이스: 안전관리 보조
> 작성일: 2026-03-27
> 참조: IRM_RAG_설계방법론_v10.md §1~§8, IRM_GraphRAG_설계방법론_v10.md §1~§4

---

## §1. 컬렉션 설계 체크리스트

- [ ] **1-1** PostgreSQL RDS 버전 확인 (PG 15 이상 권장)
  ```sql
  SELECT version();
  ```
- [ ] **1-2** pgvector 확장 설치
  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  SELECT * FROM pg_extension WHERE extname = 'vector';
  -- 확인: vector | 0.8.x
  ```
- [ ] **1-3** 3개 컬렉션 테이블 생성
  ```sql
  -- safety_knowledge
  CREATE TABLE safety_knowledge (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding VECTOR(1024),  -- BGE-m3-ko 차원
    doc_type VARCHAR(20),
    work_type VARCHAR(50),
    date DATE,
    source_file VARCHAR(255),
    specialty VARCHAR(20),
    doc_category VARCHAR(20),
    chapter VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW()
  );

  -- regulation
  CREATE TABLE regulation (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding VECTOR(1024),
    doc_type VARCHAR(20) DEFAULT 'regulation',
    work_type VARCHAR(50),
    date DATE,
    source_file VARCHAR(255),
    law_name VARCHAR(100),
    article_no VARCHAR(50),
    effective_date DATE,
    regulation_type VARCHAR(50),
    created_at TIMESTAMPTZ DEFAULT NOW()
  );

  -- accident_cases
  CREATE TABLE accident_cases (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding VECTOR(1024),
    doc_type VARCHAR(20) DEFAULT 'accident',
    work_type VARCHAR(50),
    date DATE,
    source_file VARCHAR(255),
    accident_type VARCHAR(50),
    severity VARCHAR(20),
    root_cause VARCHAR(100),
    year INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
  );
  ```
- [ ] **1-4** HNSW 인덱스 생성 (임베딩 후 생성)
  ```sql
  CREATE INDEX ON safety_knowledge USING hnsw (embedding vector_cosine_ops);
  CREATE INDEX ON regulation USING hnsw (embedding vector_cosine_ops);
  CREATE INDEX ON accident_cases USING hnsw (embedding vector_cosine_ops);
  ```
- [ ] **1-5** 컬렉션별 레코드 수 확인 (인덱싱 후)
  - safety_knowledge: 목표 4,000~6,000 청크
  - regulation: 목표 6,000~10,000 청크
  - accident_cases: 목표 1,500~2,500 청크

---

## §2. 메타데이터 스키마 체크리스트

- [ ] **2-1** 공통 메타데이터 4개 필드 전 컬렉션 존재 확인 (doc_type, work_type, date, source_file)
- [ ] **2-2** safety_knowledge 도메인 메타 (specialty, doc_category, chapter) 추출 검증
  - 자동 추출 (규칙 기반): 헤딩 텍스트에서 챕터명 추출
  - 샘플 10건 검토 — 정확도 80% 이상 확인
- [ ] **2-3** regulation 도메인 메타 (law_name, article_no, effective_date) 추출 검증
  - law_name: "산업안전보건법", "시행령" 등 패턴 매칭
  - article_no: "제○○조" 패턴 추출 (regex: `제\d+조`)
  - 샘플 10건 검토
- [ ] **2-4** accident_cases 도메인 메타 (accident_type, severity, root_cause, year) 추출 검증
  - accident_type, severity: Excel 컬럼에서 직접 추출 (소스 시스템)
  - root_cause: LLM(GPT-4o-mini) 추출 — 샘플 20건 정확도 검토 (목표 80% 이상)
  - year: 날짜 컬럼에서 자동 추출
- [ ] **2-5** 개인정보 마스킹 적용 확인
  - 사고 사례에서 근로자 이름 → "○○○" 치환 확인
  - 연락처 → "마스킹" 처리 확인

---

## §3. 문서 파싱 및 청킹 체크리스트

- [ ] **3-1** Docling 설치 및 버전 확인
  ```bash
  pip install docling
  python -c "import docling; print(docling.__version__)"
  ```
- [ ] **3-2** BGE-m3-ko 토크나이저 로드 확인
  ```python
  from transformers import AutoTokenizer
  tokenizer = AutoTokenizer.from_pretrained("dragonkue/BGE-m3-ko")
  print(tokenizer.model_max_length)  # 8192 확인
  ```
- [ ] **3-3** HybridChunker 설정 확인
  ```python
  from docling.chunking import HybridChunker
  chunker = HybridChunker(tokenizer=tokenizer, merge_peers=True)
  ```
- [ ] **3-4** 안전관리 매뉴얼 샘플 파싱 테스트 (R-01)
  - 파싱 성공 여부
  - 청크 수 예상 범위 내 (200p → 약 400~800 청크)
  - 챕터 경계 인식 여부
- [ ] **3-5** 법규집 파싱 테스트 (R-03)
  - 조항 경계 인식 여부 (제○○조 = 청크 경계)
  - 표 직렬화 (마크다운 형태) 확인
- [ ] **3-6** 사고 사례 Excel 전처리 테스트 (R-04)
  ```python
  # 1 사고 = 1 청크 형식
  content = f"""
  사고유형: {row['사고유형']}
  발생원인: {row['원인']}
  조치사항: {row['조치']}
  재발방지: {row['재발방지']}
  """
  ```
  - 180건 전체 처리 성공 여부
  - 개인정보 마스킹 적용 확인

---

## §4. 임베딩 모델 체크리스트

- [ ] **4-1** BGE-m3-ko 모델 로드 및 추론 테스트
  ```python
  from FlagEmbedding import BGEM3FlagModel
  model = BGEM3FlagModel('dragonkue/BGE-m3-ko', use_fp16=True)
  embeddings = model.encode(["고소작업 안전대 기준"])
  assert embeddings['dense_vecs'].shape[1] == 1024
  ```
- [ ] **4-2** 임베딩 차원 확인: 1024 (pgvector VECTOR(1024) 일치)
- [ ] **4-3** 한국어 샘플 임베딩 품질 확인
  - "고소작업"과 "높이작업" 코사인 유사도 > 0.8 (동의어 기대)
  - "고소작업"과 "굴착작업" 코사인 유사도 < 0.5 (비관련어 기대)
- [ ] **4-4** 배치 처리 속도 확인 (15,000 청크 기준)
  - CPU: 예상 3~4시간
  - GPU (g4dn.xlarge): 예상 30~40분
- [ ] **4-5** 임베딩 모델 로컬 저장 경로 설정 (외부 전송 금지 정책)
  ```
  저장 경로: /opt/models/BGE-m3-ko/
  서버: EC2 인스턴스 로컬 (S3 캐시 금지)
  ```

---

## §5. 검색 전략 체크리스트

- [ ] **5-1** pgvector 벡터 검색 쿼리 테스트
  ```sql
  SELECT id, content, source_file,
         embedding <=> %s::vector AS distance
  FROM safety_knowledge
  WHERE work_type = '고소작업'
  ORDER BY distance
  LIMIT 7;
  ```
- [ ] **5-2** BM25 하이브리드 검색 구현 확인
  - pgvector + tsvector 조합 또는 BGE-m3-ko Sparse 벡터 활용
  - RRF 결합 로직 구현
  ```python
  def reciprocal_rank_fusion(vector_results, bm25_results, k=60):
      scores = {}
      for rank, (id, *rest) in enumerate(vector_results):
          scores[id] = scores.get(id, 0) + 1 / (k + rank + 1)
      for rank, (id, *rest) in enumerate(bm25_results):
          scores[id] = scores.get(id, 0) + 1 / (k + rank + 1)
      return sorted(scores.items(), key=lambda x: x[1], reverse=True)
  ```
- [ ] **5-3** 메타데이터 필터링 테스트
  - work_type 필터: "크레인" → 크레인 관련 문서만 반환 확인
  - accident_type 필터: "추락" → 추락 사고만 반환 확인
- [ ] **5-4** 교차 컬렉션 검색 구현
  - 3개 컬렉션 병렬 검색
  - 결과 통합 후 리랭킹 1회 수행 (컬렉션별 따로 리랭킹 금지)
- [ ] **5-5** 동의어 처리 확인 ("높이작업" 입력 → "고소작업" 결과 반환)

---

## §6. 리랭킹 체크리스트

- [ ] **6-1** BGE-reranker-v2-m3 모델 로드
  ```python
  from FlagEmbedding import FlagReranker
  reranker = FlagReranker('BAAI/bge-reranker-v2-m3', use_fp16=True)
  ```
- [ ] **6-2** 리랭킹 파이프라인 구현
  ```python
  # top-20 후보 → 리랭킹 → top-5 반환
  candidates = search_hybrid(query, top_k=20)
  scores = reranker.compute_score([[query, c['content']] for c in candidates])
  reranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)[:5]
  ```
- [ ] **6-3** 리랭킹 전/후 품질 비교 (10개 샘플 질의)
  - 관련성 향상 확인 (안전팀장 검토)

---

## §7. 인덱싱 범위 체크리스트

- [ ] **7-1** safety_knowledge: 구버전 존재 여부 확인 후 제거
- [ ] **7-2** regulation: 구버전 법령 제거 확인
  ```sql
  -- 구버전 법령 조회
  SELECT source_file, effective_date, COUNT(*)
  FROM regulation
  GROUP BY source_file, effective_date
  ORDER BY effective_date DESC;
  ```
- [ ] **7-3** accident_cases: 5년치(2021~2025) 전체 포함 확인
  ```sql
  SELECT year, COUNT(*) FROM accident_cases GROUP BY year ORDER BY year;
  -- 기대: 2021~2025 각 연도별 데이터 존재
  ```
- [ ] **7-4** 드래프트/미승인 문서 제외 확인 (source_file 목록 검토)

---

## §8. 갱신 파이프라인 체크리스트

- [ ] **8-1** regulation 즉시 갱신 파이프라인 구현
  - 법령정보 사이트 변경 감지 스크립트
  - 구버전 삭제 → 신버전 삽입 순서 검증
  - 갱신 완료 후 안전팀장 알림 발송
- [ ] **8-2** safety_knowledge 주 1회 갱신 배치
  - 파일 변경 감지 (해시 비교)
  - 변경된 파일만 재인덱싱
- [ ] **8-3** accident_cases 사고 발생 시 추가 파이프라인
  - 신규 사고 사례 → 마스킹 → 파싱 → 인덱싱 → 엔티티 추출 (AGE)
- [ ] **8-4** 갱신 모니터링 알림 설정
  - 인덱싱 실패 시 즉시 알림 (CloudWatch)
  - regulation 30분 초과 지연 시 알림

---

## §9. GraphRAG 인프라 구성 체크리스트

- [ ] **9-1** Apache AGE 설치 확인
  ```sql
  CREATE EXTENSION IF NOT EXISTS age;
  LOAD 'age';
  SET search_path = ag_catalog, "$user", public;
  \dx
  -- 확인: age | 1.5.0 | ag_catalog
  ```
- [ ] **9-2** 그래프 공간 생성
  ```sql
  SELECT * FROM ag_catalog.create_graph('safety_graph');
  ```
- [ ] **9-3** 기본 엔티티 노드 사전 등록 (동의어)
  ```
  고소작업 ←→ 높이작업
  굴착작업 ←→ 터파기작업
  협착 ←→ 끼임
  ```

---

## §10. 엔티티+트리플 추출 파이프라인 체크리스트

- [ ] **10-1** GPT-4o-mini 안전 도메인 추출 프롬프트 설계
  ```
  건설 안전 도메인 엔티티 추출 규칙:
  - 작업유형 (고소작업, 굴착, 크레인...)
  - 법규 (산업안전보건법 제○조...)
  - 위험요인 (추락, 협착, 감전...)
  - 안전조치 (안전대, 흙막이...)
  관계 추출: 인과/조건/동의어만 (단순 언급 금지)
  ```
- [ ] **10-2** MERGE 패턴 구현 (AGE 1.5.0 기준)
  ```python
  # MATCH → 존재 확인 → CREATE/UPDATE 분기
  ```
- [ ] **10-3** doc_id 정확 매칭 구현 (부분 매칭 오류 방지)
  ```
  doc_ids = '{id}' OR STARTS WITH '{id},' OR ENDS WITH ',{id}' OR CONTAINS ',{id},'
  ```
- [ ] **10-4** 엔티티 추출 품질 검증 (샘플 10개 문서)
  - 엔티티 10개 이내 준수
  - 자기참조 (from = to) 없음 확인
  - 관계명 동사형 확인

---

## §11. 그래프 검색 통합 체크리스트

- [ ] **11-1** 통합 검색 함수 구현
  ```python
  def safety_graphrag_search(query, top_k=5):
      # 1. 쿼리 임베딩 → pgvector 검색
      # 2. 각 doc_id로 AGE 엔티티 조회
      # 3. 2홉 멀티홉 탐색
      # 4. 관련문서 + 관계체인 컨텍스트 조립
  ```
- [ ] **11-2** 2홉 탐색 Cypher 쿼리 테스트
  ```sql
  SELECT * FROM cypher('safety_graph', $$
      MATCH (a:entity {name: '고소작업'})-[r1]-(b:entity)-[r2]-(c:entity)
      WHERE a <> c
      RETURN a.name, type(r1), b.name, type(r2), c.name
      LIMIT 5
  $$) AS (a agtype, r1 agtype, b agtype, r2 agtype, c agtype);
  ```
- [ ] **11-3** 동의어 탐색 확인
  - "높이작업" 검색 → "고소작업" 관련 문서 반환 확인
- [ ] **11-4** 단일 엔드포인트 `POST /search` 구현 (벡터+그래프 통합)
- [ ] **11-5** 컨텍스트 조립 형식 확인
  ```
  [관련 문서] ...
  [관련 관계] 고소작업 -[동의어]-> 높이작업 ...
  ```

---

## §12. GraphRAG 갱신 파이프라인 체크리스트

- [ ] **12-1** 신규 사고 사례 추가 시 pgvector + AGE 동시 갱신 구현
- [ ] **12-2** 법규 갱신 시 AGE 엣지 업데이트 (법규-매뉴얼 관계 재구성)
- [ ] **12-3** 고아 노드 정리 스크립트
  ```sql
  SELECT * FROM cypher('safety_graph', $$
      MATCH (n:entity) WHERE n.doc_ids = '' OR n.doc_ids IS NULL
      DETACH DELETE n
  $$) AS (result agtype);
  ```
- [ ] **12-4** 정합성 확인 (pgvector doc_id ↔ AGE entity.doc_ids 대조)
  - 주 1회 자동 실행
  - 괴리 발견 시 알림
