# 입출고 관리 보조 — 탐색 구현 체크리스트

> 작성일: 2026-05-15

## RAG 기본 (§1~§8)

### §1 컬렉션 설계
- [ ] `logistics_docs` 컬렉션 (매뉴얼+절차서+기준표)
- [ ] `defect_history` 컬렉션 (이상 건 처리 이력)
- [ ] 분리 기준 확정 (갱신주기/메타데이터 차이)

### §2 메타데이터 스키마
- [ ] logistics_docs: doc_type, center_id, category
- [ ] defect_history: defect_type, center_id, product, date, handler

### §3 문서 파싱 & 청킹
- [ ] 매뉴얼/절차서: 섹션별 청킹
- [ ] 이상 이력: 건별 1청크 (제목+내용+처리결과)

### §4 임베딩 모델
- [ ] Titan Embeddings V2 (Bedrock, CS와 공유)

### §5 검색 전략
- [ ] 하이브리드 (벡터 + BM25) — 품목 코드/센터 ID 키워드 매칭 중요
- [ ] top-k = 10 (이상 이력은 유사 건 다수 필요)

### §6 리랭킹
- [ ] Cohere Rerank (Bedrock API) — 용어 불일치로 노이즈 예상

### §7 인덱싱 범위
- [ ] logistics_docs: 전체 (최신형)
- [ ] defect_history: 3년치 (이력형)

### §8 갱신 파이프라인
- [ ] logistics_docs: 수동 (문서 변경 시)
- [ ] defect_history: 배치 (일 1회, WMS에서 추출)

## GraphRAG 추가 (§9~§12)

### §9 인프라
- [ ] Neo4j Community 설치 (AWS EC2)
- [ ] 그래프 스키마: 이상유형, 품목, 센터, 담당자

### §10 엔티티/트리플 추출
- [ ] 동의어 관계 구축: 파손↔훼손↔깨짐↔damage↔broken
- [ ] 센터별 용어 매핑 수집 (물류운영팀장 감수)
- [ ] LLM 기반 신규 동의어 자동 탐지

### §11 그래프 검색 통합
- [ ] 질의 확장: "파손" 검색 → "훼손","깨짐" 등 자동 확장
- [ ] 크로스 컬렉션: GraphRAG → defect_history 검색

### §12 갱신 동기화
- [ ] 신규 이상 건 기록 시 → 벡터 인덱싱 + 그래프 엔티티 갱신
- [ ] 신규 동의어 발견 시 → 그래프 엣지 추가
