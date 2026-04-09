# 세금과태료_민원상담보조 — 탐색 구현 체크리스트

> 작성일: 2026-04-18

## RAG 기본 (§1~§8)

### §1 컬렉션
- [ ] tax_knowledge (FAQ 150건 + 매뉴얼 30건 + 법령)
- [ ] tax_consultation (민원 이력 — 비식별화 후)
- [ ] pgvector HNSW 인덱스

### §2 메타데이터
- [ ] 공통: doc_type, tax_type, date, source
- [ ] tax_knowledge: question, section, law_article
- [ ] tax_consultation: case_type, channel, status (비식별화 후)

### §3 파싱 & 청킹
- [ ] FAQ: DB → JSON → 1 FAQ = 1 청크
- [ ] 매뉴얼: **HWP → Forge → Markdown** → 섹션 단위
- [ ] 법령: 법제처 API → 조항 단위
- [ ] 이력: 새올 DB → 비식별화 → 1건 = 1청크

### §4 임베딩
- [ ] BGE-m3-ko (로컬) 또는 CSAP 인증 임베딩
- [ ] 한국어 행정 용어 테스트 필수

### §5~§6 검색 + 리랭킹
- [ ] 하이브리드 + RRF + BGE-reranker-v2-m3

### §7~§8 인덱싱 + 갱신
- [ ] tax_knowledge: 최신형, FAQ 변경 시 재인덱싱
- [ ] tax_consultation: 이력형 2년, 주 1회 동기화 (비식별화 후)

## GraphRAG (§9~§12)
- [ ] Apache AGE 스키마 (TaxType, FineType, Exemption, Term, Law)
- [ ] 엣지: MEANS, HAS_SUBTYPE, EXEMPT_BY, GOVERNED_BY
- [ ] FAQ+매뉴얼에서 엔티티 추출 (시드)
- [ ] 질의 확장: "세금 깎아달라" → 감면 유형 자동 확장

## 비식별화 파이프라인
- [ ] 주민번호 마스킹 규칙
- [ ] 소득정보 마스킹 규칙
- [ ] 이름/전화번호 제거 규칙
- [ ] 인덱싱 전 비식별화 검증 (누락 0건)
