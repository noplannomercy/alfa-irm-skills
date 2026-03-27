# IRM Phase 1-B — 접근 설계 (Access) — 안전관리 보조

> 고객: 대림건설
> 유즈케이스: 안전관리 보조
> 작성일: 2026-03-27

---

## 1. 접근 레이어 개요

탐색(Phase 1-A)에서 설계한 컬렉션과 실시간 데이터 소스에 접근하기 위한 인터페이스를 정의한다. 실행 레이어(Phase 1-C)에서 LLM이 이 인터페이스를 자율 호출한다.

### 접근 방식 분류

| 접근 유형 | 대상 | 인터페이스 |
|---------|------|---------|
| 벡터 검색 | safety_knowledge, regulation, accident_cases | pgvector 쿼리 API |
| 그래프 탐색 | safety_graph (Apache AGE) | Cypher 쿼리 API |
| DB 조회 | R-07 안전관리 시스템 | SQL 파라미터 쿼리 |
| REST API 호출 | R-08 현장관리 API, R-09 기상청 API | HTTP 클라이언트 |

---

## 2. CLI 구성요소 설계

IRM v10 원칙: 모든 접근 레이어는 LLM이 자율 호출할 수 있는 CLI로 노출한다.

### 2-1. 탐색 CLI: `cli-anything-safety-search`

```
역할: 벡터 검색 + 그래프 탐색 통합 인터페이스
```

#### 커맨드 목록

```bash
# 단일 컬렉션 검색
cli-anything-safety-search --json search \
  --query "고소작업 안전대 기준" \
  --collection safety_knowledge \
  --top-k 7 \
  --work-type "고소작업"

# 법규 검색
cli-anything-safety-search --json search \
  --query "크레인 작업 풍속 기준" \
  --collection regulation \
  --top-k 5

# 사고 사례 검색
cli-anything-safety-search --json search \
  --query "크레인 전도 사고" \
  --collection accident_cases \
  --top-k 5 \
  --accident-type "전도" \
  --year-from 2021 --year-to 2025

# 교차 컬렉션 검색 (GraphRAG 포함)
cli-anything-safety-search --json search-cross \
  --query "고소작업 관련 법규 매뉴얼 사고 사례 전부" \
  --collections "safety_knowledge,regulation,accident_cases" \
  --use-graphrag true \
  --top-k 5
```

#### 응답 형식

```json
{
  "query": "고소작업 안전대 기준",
  "collection": "safety_knowledge",
  "results": [
    {
      "doc_id": "sk_001_023",
      "content": "고소작업(2m 이상) 시 안전대는...",
      "source_file": "안전관리매뉴얼_건축편.pdf",
      "chapter": "3장 고소작업",
      "work_type": "고소작업",
      "score": 0.923,
      "graph_context": [
        "고소작업 -[동의어]-> 높이작업",
        "고소작업 -[적용법규]-> 산업안전보건법 제44조"
      ]
    }
  ],
  "confidence": "high",
  "search_type": "hybrid_graphrag"
}
```

### 2-2. DB 조회 CLI: `cli-anything-safety-db`

```
역할: 안전관리 시스템 DB 실시간 조회
접속: rds.internal:5432/safety_mgmt (VPC 내, 읽기 전용)
```

#### 커맨드 목록

```bash
# 현장 점검 이력 조회
cli-anything-safety-db --json inspection history \
  --site-id SITE-001 \
  --date-from "2026-01-01" --date-to "2026-03-27"

# 사고 이력 조회
cli-anything-safety-db --json accident history \
  --site-id SITE-001 \
  --year 2025

# 교육 이력 조회
cli-anything-safety-db --json training history \
  --worker-id "W001" \
  --date-from "2025-01-01"

# 미완료 시정조치 조회
cli-anything-safety-db --json inspection pending \
  --site-id SITE-001
```

#### 응답 형식

```json
{
  "query_type": "inspection_history",
  "site_id": "SITE-001",
  "date_range": {"from": "2026-01-01", "to": "2026-03-27"},
  "count": 12,
  "records": [
    {
      "inspection_id": "INS-2026-001",
      "date": "2026-03-15",
      "work_type": "크레인",
      "result": "적합",
      "findings": "안전대 미착용 1건 적발, 시정 완료"
    }
  ]
}
```

### 2-3. 현장 API CLI: `cli-anything-field-api`

```
역할: 현장관리 시스템 REST API 조회
엔드포인트: https://api.daelim.internal/field/v1/
인증: API Key (환경변수 FIELD_API_KEY)
```

#### 커맨드 목록

```bash
# 현장 목록 조회
cli-anything-field-api --json sites list

# 현장 상세 조회
cli-anything-field-api --json sites get --site-id SITE-001

# 현장 인력 조회
cli-anything-field-api --json workers list --site-id SITE-001

# 현장 장비 조회
cli-anything-field-api --json equipment list --site-id SITE-001 --type "crane"
```

### 2-4. 기상 API CLI: `cli-anything-weather`

```
역할: 기상청 단기예보 조회 + 작업 중지 판단
엔드포인트: https://apis.data.go.kr/1360000/VilageFcstInfoService2.0/
인증: API Key (환경변수 KMA_API_KEY)
```

#### 커맨드 목록

```bash
# 현장 기상 조회 (현장 좌표 자동 변환)
cli-anything-weather --json forecast \
  --site-id SITE-001 \
  --hours 6

# 작업 중지 판단
cli-anything-weather --json check-work-stop \
  --site-id SITE-001 \
  --work-type "크레인"

# 전체 현장 기상 요약
cli-anything-weather --json sites-summary
```

#### 응답 형식 (작업 중지 판단)

```json
{
  "site_id": "SITE-001",
  "site_name": "서울 강남 현장",
  "forecast_time": "2026-03-27T14:00:00",
  "weather": {
    "wind_speed": 12.3,
    "precipitation": 0.0,
    "temperature": 8.5,
    "condition": "맑음"
  },
  "work_stop_judgment": {
    "crane_work": {
      "judgment": "중지 권고",
      "reason": "풍속 12.3m/s > 기준 10m/s",
      "regulation": "산업안전보건법 제38조 (크레인 풍속 기준)"
    },
    "high_altitude_work": {
      "judgment": "가능 (주의)",
      "reason": "풍속 기준 이하이나 강풍 경향 모니터링 필요"
    }
  }
}
```

---

## 3. 접근 권한 설계

### 3-1. DB 접근 권한

```sql
-- 읽기 전용 계정 생성 (IT팀 실행)
CREATE USER safety_ai_reader WITH PASSWORD '${SECURE_PASSWORD}';
GRANT SELECT ON inspection_history TO safety_ai_reader;
GRANT SELECT ON accident_history TO safety_ai_reader;
GRANT SELECT ON training_history TO safety_ai_reader;
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM safety_ai_reader;
GRANT SELECT ON inspection_history, accident_history, training_history TO safety_ai_reader;
```

### 3-2. API Key 관리

```bash
# AWS Secrets Manager에 저장 (IT팀 구성)
aws secretsmanager create-secret \
  --name /safety-ai/field-api-key \
  --secret-string '{"api_key": "${FIELD_API_KEY}"}'

aws secretsmanager create-secret \
  --name /safety-ai/kma-api-key \
  --secret-string '{"api_key": "${KMA_API_KEY}"}'
```

### 3-3. 네트워크 접근

| 접근 대상 | 방식 | VPC 내부/외부 |
|---------|------|------------|
| RDS PostgreSQL | VPC 내 직접 | 내부 |
| 현장관리 API | VPN + 사내 네트워크 | 내부 |
| 기상청 API | 인터넷 (공공) | 외부 (허용) |
| OpenAI API | 인터넷 | 외부 (도면 제외, 텍스트만) |

---

## 4. 에러 처리 및 폴백

| 시나리오 | 처리 방법 |
|---------|---------|
| DB 연결 실패 | 에러 반환 + "시스템 일시 장애" 안내, 수동 조회 안내 |
| 현장관리 API 타임아웃 | 3회 재시도 (1s 간격) → 실패 시 마지막 캐시 데이터 사용 |
| 기상청 API 실패 | 기상청 대기 중 메시지 + 마지막 성공 데이터 + 타임스탬프 표시 |
| 벡터 검색 결과 없음 | "관련 문서 없음" 반환 + 유사 키워드 제안 |
| GraphRAG 탐색 실패 | 벡터 검색 단독으로 폴백 |

---

## 5. CLI-SPEC.md 명세 (요약)

```markdown
---
name: cli-anything-safety-search
command: cli-anything-safety-search
description: 대림건설 안전관리 보조 — 매뉴얼/법규/SOP/사고사례 통합 검색 (GraphRAG 포함)
flags:
  - --json   # LLM 호출 시 항상 사용
---

## Command Groups

### search
- `search --query Q --collection C --top-k N [--work-type W]` — 컬렉션 검색
- `search-cross --query Q --collections C1,C2 [--use-graphrag]` — 교차 검색

## Usage Examples

    # 법규 검색
    cli-anything-safety-search --json search \
      --query "고소작업 안전대 기준" \
      --collection regulation

    # TBM 사고 사례 검색
    cli-anything-safety-search --json search \
      --query "크레인 전도 사고" \
      --collection accident_cases \
      --top-k 3
```
