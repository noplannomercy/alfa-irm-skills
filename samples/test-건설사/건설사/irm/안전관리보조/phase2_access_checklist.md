# IRM Phase 2 — 접근 구현 체크리스트 — 안전관리 보조

> 고객: 대림건설
> 유즈케이스: 안전관리 보조
> 작성일: 2026-03-27

---

## CLI 구성요소 구현 체크리스트

### A. cli-anything-safety-search

**탐색 통합 CLI (벡터 검색 + GraphRAG)**

- [ ] **A-1** 패키지 구조 생성
  ```
  agent-harness/
  ├── setup.py
  └── cli_anything/
      └── safety_search/
          ├── __init__.py
          ├── safety_search_cli.py
          └── core/
              ├── __init__.py
              ├── storage.py      # pgvector 연결
              ├── graphrag.py     # AGE 그래프 탐색
              └── logger.py
  ```

- [ ] **A-2** setup.py 작성
  ```python
  from setuptools import setup, find_namespace_packages
  setup(
      name="cli-anything-safety-search",
      version="0.1.0",
      packages=find_namespace_packages(include=["cli_anything.*"]),
      install_requires=["click", "psycopg2-binary", "pgvector", "FlagEmbedding"],
      entry_points={
          "console_scripts": [
              "cli-anything-safety-search=cli_anything.safety_search.safety_search_cli:cli"
          ]
      },
  )
  ```

- [ ] **A-3** 환경변수 설정 확인
  ```bash
  export SAFETY_DB_URL="postgresql://safety_ai_reader:${PW}@rds.internal:5432/safety_mgmt"
  export EMBED_MODEL_PATH="/opt/models/BGE-m3-ko"
  export RERANKER_MODEL_PATH="/opt/models/bge-reranker-v2-m3"
  ```

- [ ] **A-4** 단일 컬렉션 검색 커맨드 동작 확인
  ```bash
  PYTHONIOENCODING=utf-8 cli-anything-safety-search --json search \
    --query "고소작업 안전대 기준" \
    --collection safety_knowledge \
    --top-k 5
  # 기대: JSON 응답, results 배열, source_file 포함
  ```

- [ ] **A-5** 메타데이터 필터 동작 확인
  ```bash
  cli-anything-safety-search --json search \
    --query "안전 조치" --collection accident_cases \
    --work-type "크레인" --top-k 3
  # 기대: work_type='크레인' 문서만 반환
  ```

- [ ] **A-6** 교차 컬렉션 검색 동작 확인
  ```bash
  cli-anything-safety-search --json search-cross \
    --query "고소작업 법규 사고 사례" \
    --collections "safety_knowledge,regulation,accident_cases" \
    --use-graphrag true --top-k 5
  # 기대: 3개 컬렉션 통합 결과 + 그래프 관계 포함
  ```

- [ ] **A-7** 동의어 처리 확인
  ```bash
  cli-anything-safety-search --json search \
    --query "높이작업 기준" --collection safety_knowledge
  # 기대: "고소작업" 관련 결과 반환 (동의어 처리)
  ```

- [ ] **A-8** CLI-SPEC.md 작성 및 배치
  ```
  위치: cli_anything/safety_search/spec/CLI-SPEC.md
  내용: 커맨드 목록, 파라미터, 응답 형식, 사용 예시
  ```

- [ ] **A-9** 설치 및 최종 검증
  ```bash
  cd agent-harness && pip install -e .
  cli-anything-safety-search --help
  cli-anything-safety-search --json search --query "테스트" --collection safety_knowledge
  ```

---

### B. cli-anything-safety-db

**안전관리 시스템 DB 조회 CLI**

- [ ] **B-1** 읽기 전용 DB 계정 발급 확인 (IT팀)
  ```sql
  -- IT팀 실행 확인
  SELECT usename FROM pg_user WHERE usename = 'safety_ai_reader';
  ```

- [ ] **B-2** DB 연결 테스트
  ```bash
  PYTHONIOENCODING=utf-8 cli-anything-safety-db --json inspection history \
    --site-id SITE-001 \
    --date-from "2026-01-01" --date-to "2026-03-27"
  ```

- [ ] **B-3** SQL 인젝션 방지 확인 (파라미터 바인딩)
  ```python
  # 올바른 방식 (파라미터 바인딩)
  cur.execute(
      "SELECT * FROM inspection_history WHERE site_id = %s AND date >= %s",
      (site_id, date_from)
  )
  # 금지: f"WHERE site_id = '{site_id}'" (인젝션 취약점)
  ```

- [ ] **B-4** 개인정보 필드 마스킹 확인
  ```python
  # 응답에서 worker_name → "○○○" 치환
  # 응답에서 phone → "마스킹" 치환
  ```

- [ ] **B-5** 점검 이력 조회 동작 확인 (JSON 응답)

- [ ] **B-6** 미완료 시정조치 조회 동작 확인

- [ ] **B-7** CLI-SPEC.md 작성

---

### C. cli-anything-field-api

**현장관리 시스템 REST API CLI**

- [ ] **C-1** API Key 환경변수 확인
  ```bash
  echo $FIELD_API_KEY  # 빈 값이면 안 됨
  ```

- [ ] **C-2** 현장 목록 조회 테스트
  ```bash
  cli-anything-field-api --json sites list
  # 기대: 8개 현장 목록 반환
  ```

- [ ] **C-3** 현장 인력/장비 조회 테스트
  ```bash
  cli-anything-field-api --json workers list --site-id SITE-001
  cli-anything-field-api --json equipment list --site-id SITE-001 --type "crane"
  ```

- [ ] **C-4** API 타임아웃 + 재시도 구현 확인 (3회, 1s 간격)

- [ ] **C-5** API 에러 처리 확인 (4xx, 5xx, 네트워크 오류)

- [ ] **C-6** VPN 환경에서 접근 가능 여부 확인

- [ ] **C-7** CLI-SPEC.md 작성

---

### D. cli-anything-weather

**기상청 API + 작업 중지 판단 CLI**

- [ ] **D-1** 공공데이터포털 API Key 발급 및 설정 확인
  ```bash
  echo $KMA_API_KEY  # 빈 값이면 안 됨
  ```

- [ ] **D-2** 현장 좌표 → 기상청 격자 변환 함수 구현
  ```python
  def latlon_to_grid(lat, lon):
      """위경도 → 기상청 격자 (X, Y) 변환"""
      # 기상청 좌표 변환 알고리즘 적용
  ```

- [ ] **D-3** 현장별 좌표 목록 등록 (8개 현장)
  ```python
  SITE_COORDS = {
      "SITE-001": {"name": "서울 강남 현장", "lat": 37.5, "lon": 127.0},
      # ... 8개 현장
  }
  ```

- [ ] **D-4** 기상 조회 테스트
  ```bash
  cli-anything-weather --json forecast --site-id SITE-001 --hours 6
  ```

- [ ] **D-5** 작업 중지 판단 로직 구현 및 테스트
  ```python
  WORK_STOP_CRITERIA = {
      "크레인": {"wind_speed": 10.0},  # m/s
      "고소작업": {"wind_speed": 10.0},
      "전기작업": {"precipitation": 1.0},  # mm/h
  }
  ```

- [ ] **D-6** 작업 중지 판단 테스트 (풍속 12m/s → 크레인 중지 권고 확인)

- [ ] **D-7** 기상 API 실패 시 폴백 (마지막 성공 데이터 캐시) 구현

- [ ] **D-8** 전체 현장 일괄 기상 조회 동작 확인
  ```bash
  cli-anything-weather --json sites-summary
  ```

- [ ] **D-9** CLI-SPEC.md 작성

---

## 공통 체크리스트

- [ ] **COM-1** 모든 CLI `--json` 플래그 동작 확인 (LLM 호출 시 항상 사용)
- [ ] **COM-2** 모든 CLI 에러 응답 JSON 형식 확인
  ```json
  {"error": "에러 메시지"}
  ```
- [ ] **COM-3** AWS Secrets Manager에서 API Key 로드 구현 (하드코딩 금지)
  ```python
  import boto3
  def get_secret(secret_name):
      client = boto3.client('secretsmanager', region_name='ap-northeast-2')
      return client.get_secret_value(SecretId=secret_name)
  ```
- [ ] **COM-4** 로깅 구현 확인 (모든 CLI 호출 이력 기록)
- [ ] **COM-5** 4개 CLI 모두 PATH에 설치 완료 확인
  ```bash
  which cli-anything-safety-search
  which cli-anything-safety-db
  which cli-anything-field-api
  which cli-anything-weather
  ```
- [ ] **COM-6** 통합 테스트: 5개 시나리오 전체 실행 (Q-01~Q-05 각 1건)
