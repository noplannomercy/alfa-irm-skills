# IRM Phase 2 — 실행 구현 체크리스트 — 안전관리 보조

> 고객: 대림건설
> 유즈케이스: 안전관리 보조
> 작성일: 2026-03-27

---

## 1. LLM 에이전트 구현 체크리스트

### 1-1. 기본 구성

- [ ] **1-1-1** OpenAI API Key 설정 확인 (AWS Secrets Manager)
  ```python
  import boto3, openai
  secret = boto3.client('secretsmanager').get_secret_value(SecretId='/safety-ai/openai-key')
  openai.api_key = secret['SecretString']
  ```

- [ ] **1-1-2** GPT-4o 모델 접근 확인
  ```python
  client = openai.OpenAI()
  response = client.chat.completions.create(
      model="gpt-4o",
      messages=[{"role": "user", "content": "테스트"}],
      max_tokens=10
  )
  assert response.choices[0].message.content
  ```

- [ ] **1-1-3** GPT-4o-mini 접근 확인 (질의 분류용)

### 1-2. Function Calling 구현

- [ ] **1-2-1** 4개 도구 정의 등록 확인 (search_documents, get_weather_judgment, query_inspection_history, get_site_info)

- [ ] **1-2-2** 도구 호출 → CLI 연결 구현
  ```python
  def execute_tool(tool_name, tool_args):
      if tool_name == "search_documents":
          result = subprocess.run([
              "cli-anything-safety-search", "--json", "search",
              "--query", tool_args["query"],
              "--collection", tool_args["collection"],
              "--top-k", str(tool_args.get("top_k", 5))
          ], capture_output=True, text=True)
          return json.loads(result.stdout)
      elif tool_name == "get_weather_judgment":
          # cli-anything-weather 호출
          ...
  ```

- [ ] **1-2-3** 멀티턴 도구 호출 지원 확인 (1개 질의에서 여러 도구 순차 호출)

### 1-3. 프롬프트 구현

- [ ] **1-3-1** 시스템 프롬프트 적용 확인 (출처 필수, AI 한계 명시)

- [ ] **1-3-2** 동의어 처리 프롬프트 적용 확인
  ```
  "높이작업" → "고소작업"으로 재질의 후 검색
  ```

- [ ] **1-3-3** 법규 신뢰도 경고 문구 적용 확인
  ```
  regulation 컬렉션 사용 시 →
  "⚠️ 법규 갱신 파이프라인 구축 전. 최신 법규 개정 사항 직접 확인 권고."
  ```

- [ ] **1-3-4** 출처 표시 형식 적용 확인
  ```
  [출처: 안전관리매뉴얼_건축편.pdf - 3장 고소작업]
  [출처: 산업안전보건법 제44조 (2024.01.01 시행)]
  ```

---

## 2. FastAPI 엔드포인트 체크리스트

- [ ] **2-1** FastAPI 앱 설치 및 구동 확인
  ```bash
  pip install fastapi uvicorn
  uvicorn main:app --host 0.0.0.0 --port 8000
  curl http://localhost:8000/health
  # 기대: {"status": "ok"}
  ```

- [ ] **2-2** `/query` 엔드포인트 동작 확인
  ```bash
  curl -X POST http://localhost:8000/query \
    -H "Content-Type: application/json" \
    -d '{"query": "고소작업 안전대 기준", "user_id": "U-001"}'
  # 기대: answer + sources + confidence + response_time_ms
  ```

- [ ] **2-3** 응답 시간 측정 (목표: 5초 이내)
  - 벡터 검색 단독: 목표 2초 이내
  - GraphRAG 포함: 목표 5초 이내
  - 체인 (TBM 생성): 목표 30초 이내

- [ ] **2-4** 에러 응답 형식 확인 (HTTP 상태코드 + JSON 에러)

- [ ] **2-5** 인증 미들웨어 구현 (사내 SSO 연동 또는 API Key)

- [ ] **2-6** CORS 설정 확인 (웹 UI 연동용)

- [ ] **2-7** 요청 로깅 미들웨어 구현
  ```python
  @app.middleware("http")
  async def log_requests(request: Request, call_next):
      # 요청/응답 CloudWatch에 기록
  ```

---

## 3. 로깅 체크리스트

- [ ] **3-1** CloudWatch Logs 그룹 생성
  ```bash
  aws logs create-log-group --log-group-name /safety-ai/queries
  ```

- [ ] **3-2** 로깅 항목 전체 기록 확인 (query, user_id, sources, response_time)

- [ ] **3-3** 로그 보존 기간 설정 (30일 CloudWatch + 3년 S3)
  ```bash
  aws logs put-retention-policy \
    --log-group-name /safety-ai/queries \
    --retention-in-days 30
  ```

- [ ] **3-4** 중대재해처벌법 대비 안전 조회 이력 S3 장기 보관 구현

---

## 4. 질의 분류 체크리스트

- [ ] **4-1** 질의 유형 분류 정확도 테스트 (20개 샘플)
  - Q-01~Q-06 각 3~4건
  - 목표: 분류 정확도 90% 이상

- [ ] **4-2** 동의어 처리 테스트 케이스
  ```
  "높이작업" → Q-01 (safety_knowledge: 고소작업 처리)
  "터파기" → Q-01 (굴착작업 처리)
  "끼임 사고" → Q-02 (협착 처리)
  ```

- [ ] **4-3** 복합 질의 처리 (Q-06) 테스트
  - "크레인 법규, 매뉴얼, 사고 사례 모두 알려줘" → 교차 검색 확인

---

## 5. TBM 체인 구현 체크리스트

- [ ] **5-1** TBM 자료 생성 체인 전체 흐름 구현

- [ ] **5-2** 오늘 작업 유형 자동 감지 (현장관리 API → 오늘 예정 작업)

- [ ] **5-3** 기상 데이터 → 작업 중지 항목 자동 포함

- [ ] **5-4** 공종별 사고 사례 3건 자동 삽입

- [ ] **5-5** TBM 자료 출력 형식 (A4 1장 기준) 준수 확인

- [ ] **5-6** TBM 자료 생성 이력 로깅 (중대재해처벌법 기록)

- [ ] **5-7** 성능 목표 확인 (30초 이내 생성)

---

## 6. 기상 알림 체인 체크리스트

- [ ] **6-1** 매일 오전 6시 Cron 설정 (AWS EventBridge)
  ```
  cron(0 21 * * ? *)  # UTC 21:00 = KST 06:00
  ```

- [ ] **6-2** 8개 현장 병렬 기상 조회 구현 (asyncio)

- [ ] **6-3** 작업 중지 필요 현장 → 담당 안전팀장 알림 발송
  - 이메일 (SES) 또는 슬랙 웹훅

- [ ] **6-4** 기상 특보 발령 시 즉시 실행 (이벤트 트리거)

---

## 7. UI 구현 체크리스트 (최소)

- [ ] **7-1** 입력 화면: 텍스트 입력 + 현장 선택 드롭다운

- [ ] **7-2** 응답 화면: 답변 텍스트 + 출처 카드 (문서명, 조항, 신뢰도)

- [ ] **7-3** 기상 위젯: 상단 고정, 현재 현장별 기상 + 작업 중지 여부

- [ ] **7-4** 모바일 반응형 확인 (현장 태블릿/스마트폰 접근)

- [ ] **7-5** VPN 접속 환경에서 접근 확인

---

## 8. 보안 최종 체크리스트

- [ ] **8-1** 개인정보 입력 방지 확인 (입력창에서 이름/연락처 입력 시 경고)
- [ ] **8-2** 도면 데이터 외부 API 전송 차단 확인
- [ ] **8-3** API Key 하드코딩 없음 확인 (grep 결과 0건)
  ```bash
  grep -r "api_key\s*=" . --include="*.py" | grep -v "os.environ\|secrets"
  ```
- [ ] **8-4** RDS 접속 계정 최소 권한 확인 (읽기 전용)
- [ ] **8-5** 전송 중 암호화 확인 (TLS 1.3)
- [ ] **8-6** 감사 로그 CloudWatch 활성화 확인
