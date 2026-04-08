# 불량원인분석_보조 — 탐색 구현 체크리스트

> 작성일: 2026-04-24

## RAG 기본 (§1~§8)

### §1 컬렉션 설계
- [ ] 컬렉션 3개 확정: quality_standards / defect_history / investigation_reports
- [ ] pgvector 테이블 3개 생성 (PostgreSQL 16)
- [ ] HNSW 인덱스 생성 (각 컬렉션)

### §2 메타데이터 스키마
- [ ] 공통 필드 정의: doc_type, date, source, plant
- [ ] quality_standards 도메인: part_no, process_step, revision, file_format
- [ ] defect_history 도메인: defect_code, defect_code_std, line, product, severity
- [ ] investigation_reports 도메인: root_cause, action_taken, defect_code_std, effectiveness
- [ ] defect_code_std 추출 방식: 불량코드 매핑 엑셀에서 자동 변환

### §3 문서 파싱 & 청킹
- [ ] PDF 파싱: Docling + HybridChunker (검사기준서 200건)
- [ ] **HWP 파싱: 전용 파서 선정 + 샘플 3건 품질 테스트** ← 리스크
- [ ] HWP → 텍스트 변환 후 Docling 파이프라인 투입
- [ ] MES 데이터: 1건 = 1청크, 메타데이터 MES 필드 직접 매핑
- [ ] 8D 보고서: 1보고서 = 1~3청크 (요약/원인/대책 분리)
- [ ] 토크나이저 정렬: BGE-m3-ko 토크나이저 사용

### §4 임베딩 모델
- [ ] 모델: BGE-m3-ko (dragonkue/BGE-m3-ko) — 로컬 배포
- [ ] RTX 4090에서 추론 성능 확인
- [ ] 벡터 차원: 1024
- [ ] 테스트: 샘플 문서 10건 임베딩 → 유사도 검색 품질 확인

### §5 검색 전략
- [ ] 하이브리드 검색 (벡터 + BM25) 구성
- [ ] RRF 결합 설정
- [ ] top-k: quality_standards 5, defect_history 10, investigation_reports 5
- [ ] 교차 컬렉션 라우팅: 질의 분석 → 대상 컬렉션 결정 프롬프트
- [ ] 메타데이터 필터링: plant, defect_code_std, part_no

### §6 리랭킹
- [ ] BGE-reranker-v2-m3 로컬 배포 (RTX 4090)
- [ ] 파이프라인: top-20 → 리랭킹 → top-5
- [ ] 교차 컬렉션 결과 합친 후 리랭킹 1회

### §7 인덱싱 범위
- [ ] quality_standards: 최신 revision만, 개정 시 구버전 삭제
- [ ] defect_history: 3년+, severity=critical 기간 무관 전수
- [ ] investigation_reports: 전수 (기간 제한 없음)

### §8 갱신 파이프라인
- [ ] quality_standards: 문서 개정 시 수동 트리거 (구버전 삭제 → 신버전)
- [ ] defect_history: 일 1회 MES 증분 동기화 (cron)
- [ ] investigation_reports: 8D 종결 시 수동 트리거
- [ ] doc_id 매핑 체계: 파일경로+revision / MES불량ID / 보고서파일명

## GraphRAG 추가 (§9~§12)

### §9 인프라
- [ ] Apache AGE 설치 (pgvector와 같은 PostgreSQL 16)
- [ ] 그래프 스키마 정의:
  - 노드: DefectType, DefectCode, Part, Process, RootCause, Action
  - 엣지: MEANS, OCCURS_IN, AFFECTS, CAUSED_BY, RESOLVED_BY

### §10 엔티티/트리플 추출
- [ ] 매핑 엑셀 시드: DefectCode→MEANS→DefectType 60쌍 초기 로드
- [ ] 8D 보고서 LLM 추출 설정 (gpt-4o-mini 또는 로컬 LLM)
- [ ] 제약조건: 자기참조 금지, 동사형 관계, 인과관계만
- [ ] 추출 품질 검증: 샘플 5건 수동 대조

### §11 그래프 검색 통합
- [ ] 질의 확장: "크랙" → MEANS 관계로 "B-crack", "브레이크디스크_균열" 등 자동 확장
- [ ] 멀티홉 경로 쿼리: DefectType→CAUSED_BY→RootCause (2-hop)
- [ ] 컨텍스트 어셈블리: [관련 문서 청크] + [그래프 관계 정보] 결합

### §12 갱신 동기화
- [ ] 벡터 + 그래프 동기 갱신 (8D 보고서 추가 시)
- [ ] 문서 삭제 시: 벡터 삭제 + 관련 엣지 삭제 + 고아 노드 정리
- [ ] 매핑 엑셀 갱신 시: MEANS 관계 업데이트
