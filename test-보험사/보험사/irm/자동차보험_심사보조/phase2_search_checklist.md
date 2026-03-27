# 자동차보험 심사 보조 — 탐색 구현 체크리스트

> 작성일: 2026-04-03
> 신뢰도: ✅ 전부 실제 데이터 기반
> 참조: RAG 설계방법론 v10 §1~§8, GraphRAG 설계방법론 v10 §1~§4

## RAG 기본 (§1~§8)

### §1 컬렉션 설계
- [ ] 3개 컬렉션: uw_guidelines, uw_cases, claim_docs
- [ ] pgvector (기존 PostgreSQL 확장)
- [ ] HNSW 인덱스 (27만건 규모)

### §2 메타데이터 스키마
- [ ] uw_guidelines: doc_type, product, section, keyword
- [ ] uw_cases: case_type, decision, reviewer, accident_type, date
- [ ] claim_docs: claim_type, date, product, status

### §3 문서 파싱 & 청킹
- [ ] 가이드라인/약관: 섹션별 청킹 (조항 단위)
- [ ] FAQ: 항목별 1청크
- [ ] 심사 이력: 건별 1청크 (요청+심사의견+결정)
- [ ] OCR 청구서: 건별 1청크 (품질 필터 적용)

### §4 임베딩 모델
- [ ] Titan Embeddings V2 (Bedrock)

### §5 검색 전략
- [ ] 하이브리드 (벡터 + BM25) — 약관 조항 번호 키워드 중요
- [ ] RRF 결합
- [ ] 크로스 컬렉션 라우팅 (질의 유형별)

### §6 리랭킹
- [ ] Cohere Rerank (Bedrock API)
- [ ] 3개 컬렉션 크로스 검색 시 적용

### §7 인덱싱 범위
- [ ] uw_guidelines: 전체 (최신형)
- [ ] uw_cases: 자동차 5년치 27만건 (이력형)
- [ ] claim_docs: 2년치 16만건 (이력형, 품질 필터)

### §8 갱신 파이프라인
- [ ] uw_guidelines: 수동 (약관/가이드라인 변경 시)
- [ ] uw_cases: 배치 일 1회 (신규 심사 완료 건)
- [ ] claim_docs: 배치 (청구 접수 시)

## GraphRAG (§9~§12) — 승인 (4/4)

### §9 인프라 (GraphRAG 설계방법론 §1)
- [ ] Apache AGE 설치 (기존 PostgreSQL 16에 확장)
- [ ] pgvector + Apache AGE 같은 DB 인스턴스
- [ ] 그래프 스키마: 약관조항, 사고유형, 심사기준, 심사결과

### §10 엔티티+트리플 추출 (GraphRAG 설계방법론 §2)
- [ ] LLM 기반 추출 (Claude)
- [ ] 관계 유형: references(조항간), applies_to(사고→약관), based_on(결과→기준), synonym(동의어)
- [ ] 추출 품질 검증: 심사팀장 감수

### §11 그래프 검색 통합 (GraphRAG 설계방법론 §3)
- [ ] 질의 확장: 동의어/관련 조항 자동 확장
- [ ] 멀티홉 관계 추적 (2-hop: 사고유형→약관→면책)
- [ ] 컨텍스트 어셈블리: [관련 문서] + [관련 관계]

### §12 갱신 동기화 (GraphRAG 설계방법론 §4)
- [ ] 약관 변경 시: 벡터 재인덱싱 + 그래프 관계 갱신
- [ ] 신규 심사 건: 벡터 추가 + 사고유형→결과 엣지 추가
- [ ] 삭제 시: 벡터 삭제 + 관련 엣지 삭제 + 고아 노드 정리
