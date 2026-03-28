# 학습상담 보조 — 탐색 구현 체크리스트

> 작성일: 2026-03-27
> GraphRAG: 불필요 (§9~§12 스킵)

## RAG 기본 (§1~§8)

### §1 컬렉션 설계
- [ ] 컬렉션 분리 기준 확정: 3개 (faq_and_rules / course_content / consultation_history)
- [ ] faq_and_rules: FAQ 150건 + 환불규정 10p → pgvector 테이블 생성
- [ ] course_content: 교안 PDF 3,000 + 자막 SRT 2,800 → pgvector 테이블 생성
- [ ] consultation_history: Zendesk 티켓 (최근 1년) → pgvector 테이블 생성
- [ ] 벡터 저장소: pgvector (기존 RDS에 extension 활성화)
- [ ] 인덱스: HNSW (전 컬렉션)

### §2 메타데이터 스키마
- [ ] 공통 필드: doc_id, doc_type, source, created_at
- [ ] faq_and_rules 도메인: category, question_type, rule_section
- [ ] course_content 도메인: course_id, course_title, category, level, instructor, content_type, chapter
- [ ] consultation_history 도메인: ticket_id, ticket_date, inquiry_type, resolution_status, agent_id
- [ ] 추출 방식 확정: course 메타 = S3 경로 파싱, inquiry_type = LLM 분류, rule_section = 정규식

### §3 문서 파싱 & 청킹
- [ ] FAQ: 구글독스 export → 1 Q&A 쌍 = 1 청크 (커스텀 전처리)
- [ ] 환불규정 PDF: Docling + HybridChunker (조항 단위)
- [ ] 강의 교안 PDF: Docling + HybridChunker (섹션/페이지 단위), PoC 50~100개
- [ ] 강의 자막 SRT: 커스텀 전처리 (5~10분 단위 병합), course_id + chapter 메타 부착
- [ ] Zendesk 티켓: 1 티켓 = 1 청크, 수강생 식별정보 마스킹
- [ ] 품질 필터: 교안 50자 미만 페이지 제외, 자막 10초 미만 제외, 티켓 20자 미만 제외, 미해결 티켓 제외

### §4 임베딩 모델
- [ ] 모델: OpenAI text-embedding-3-small (PoC)
- [ ] API 키 설정 + 연결 확인
- [ ] 벡터 차원: 1536
- [ ] 테스트 임베딩: FAQ 10건 + 교안 3건으로 품질 확인
- [ ] 전환 계획: BGE-m3-ko (1024차원, 로컬) — GPU 확보 후

### §5 검색 전략
- [ ] faq_and_rules: 벡터 검색 단독, top-5
- [ ] course_content: 하이브리드 (벡터+BM25), RRF, top-10
- [ ] consultation_history: 하이브리드 (벡터+BM25), RRF, top-10
- [ ] 라우팅 프롬프트: 질의 의도 → 컬렉션 선택 규칙 정의
- [ ] 메타데이터 필터링: category, course_title, doc_type 기반

### §6 리랭킹
- [ ] faq_and_rules: 미적용 (소규모)
- [ ] course_content: FlashRank 적용 (top-20 → top-5)
- [ ] consultation_history: FlashRank 적용 (top-20 → top-5)
- [ ] FlashRank 설치 + CPU 동작 확인

### §7 인덱싱 범위
- [ ] faq_and_rules: 전수 (최신형), 구버전 FAQ 수정 시 삭제→재삽입
- [ ] course_content: PoC 50~100개 (최신형), 폐강 강의 제외
- [ ] consultation_history: 최근 1년 (이력형), 미해결 티켓 제외

### §8 갱신 파이프라인
- [ ] faq_and_rules: 수동 트리거 (FAQ 변경 시 관리자 실행), 삭제→재파싱→재임베딩
- [ ] course_content: 일배치 (S3 이벤트 감지), 신규/수정 강의만 증분 처리
- [ ] consultation_history: 일배치 (Zendesk API), 전일 종결 티켓 추가
- [ ] doc_id 체계: faq_{id}, rule_{section}, course_{id}_pdf_{page}, course_{id}_srt_{seg}, zendesk_{ticket_id}
- [ ] 청크 ID: {collection}_{doc_id}_{chunk_index}
