# 할부심사_규정검색 — 탐색 구현 체크리스트

> 작성일: 2026-04-21

## RAG 기본 (§1~§8)

### §1 컬렉션
- [ ] regulations (여신규정+자동차특약+리스특약 — product_type 메타로 구분)
- [ ] guidelines (심사 가이드라인 — 해석+사례)
- [ ] review_history (심사 이력 — 비식별화 후)
- [ ] pgvector HNSW 인덱스

### §2 메타데이터
- [ ] 공통: doc_type, product_type, date, source
- [ ] regulations: chapter, section, article, paragraph, revision
- [ ] guidelines: case_type, interpretation_target, author
- [ ] review_history: product_type, decision(approved/rejected/conditional), anonymized

### §3 파싱 & 청킹
- [ ] **규정집 2,000p: Forge(PDF→MD) + 조항 단위 청킹** ← 프로젝트 성패
  - [ ] 편→장→조→항 계층 구조 보존
  - [ ] 상위 헤딩 컨텍스트 포함 (HybridChunker contextualize)
  - [ ] PoC 잔여 PDF 재파싱 (품질 검증)
- [ ] 가이드라인 200p: Forge(PDF+HWP→MD) → 사례 단위
- [ ] 심사 이력: CBS → 비식별화 → 1건 = 1청크

### §4 임베딩
- [ ] Bedrock Titan V2 또는 Cohere Embed v3
- [ ] 한국어 법률/규정 용어 테스트 (조항 번호, 법률 용어)
- [ ] 대안: BGE-m3-ko 로컬 (Bedrock 한국어 품질 미달 시)

### §5~§6 검색 + 리랭킹
- [ ] 하이브리드 + RRF
- [ ] regulations top-10 (3규정 합본), guidelines 5, review_history 5
- [ ] 리랭킹 적용 (상품별 정확도 핵심)

### §7~§8 인덱싱 + 갱신
- [ ] regulations: 최신형, 분기 1회 전체 재인덱싱 — **구버전 삭제 필수**
- [ ] review_history: 이력형 2년, 월 1회 비식별화 동기화

## GraphRAG (§9~§12)

### §9 인프라
- [ ] Apache AGE (RDS PostgreSQL)
- [ ] 그래프 스키마: Regulation, Article, Product, Topic, Term

### §10 엔티티/트리플 추출
- [ ] 규정집 목차에서 편/장/조 구조 추출 (시드)
- [ ] 조항별 product_type 매핑 (APPLIES_TO) — **PoC 실패 해결 핵심**
- [ ] 조항별 topic 매핑 (COVERS)
- [ ] LLM 추출 품질 검증: 샘플 20조항 수동 대조

### §11 그래프 검색 통합
- [ ] 질의 → Topic 매핑 → COVERS 역탐색 → Article → APPLIES_TO로 상품 필터
- [ ] 2-hop: Topic → Article → Regulation (어떤 규정의 몇 조인지)

### §12 갱신 동기화
- [ ] 규정 개정 시: 해당 Article 노드/엣지 업데이트 + 벡터 재인덱싱
- [ ] SUPERSEDES 관계 추가 (구조항→신조항)
