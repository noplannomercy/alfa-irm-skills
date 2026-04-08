# cs_상담보조 — 탐색 구현 체크리스트

> 작성일: 2026-04-11

## RAG 기본 (§1~§8)

### §1 컬렉션 설계
- [ ] 컬렉션 2개: knowledge_base / consultation_history
- [ ] pgvector 확장 설치 (기존 RDS PostgreSQL)
- [ ] HNSW 인덱스 생성

### §2 메타데이터 스키마
- [ ] 공통: doc_type, category, date, source
- [ ] knowledge_base: question, section
- [ ] consultation_history: channel, resolution_type, agent_id

### §3 파싱 & 청킹
- [ ] FAQ: 구글 시트 → JSON → 1 FAQ = 1 청크
- [ ] 매뉴얼: Notion API → 마크다운 → 섹션 단위 분할
- [ ] 상담 이력: CRM API → 1 상담 = 1 청크

### §4 임베딩 모델
- [ ] Bedrock Titan Embeddings 또는 Cohere Embed v3
- [ ] 테스트: FAQ 10건 임베딩 → 유사도 검색 품질 확인

### §5 검색 전략
- [ ] 하이브리드 (벡터 + BM25), RRF
- [ ] top-k: knowledge_base 5, consultation_history 5
- [ ] 교차 컬렉션 라우팅

### §6 리랭킹
- [ ] 초기 미적용 — 파일럿 후 정확도 미달 시 추가 검토

### §7 인덱싱 범위
- [ ] knowledge_base: 최신 버전만
- [ ] consultation_history: 최근 6개월

### §8 갱신 파이프라인
- [ ] knowledge_base: FAQ 변경 시 전체 재인덱싱 (350건, 수 분)
- [ ] consultation_history: 일 1회 CRM 증분 동기화

## GraphRAG — 해당 없음
