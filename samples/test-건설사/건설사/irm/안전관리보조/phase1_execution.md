# IRM Phase 1-C — 실행 확정 (Execution) — 안전관리 보조

> 고객: 대림건설
> 유즈케이스: 안전관리 보조
> 작성일: 2026-03-27

---

## 1. LLM 에이전트 설계

### 1-1. LLM 구성

| 역할 | 모델 | 용도 |
|------|------|------|
| 메인 응답 | GPT-4o | 최종 답변 생성, 컨텍스트 조립 |
| 질의 분류/라우팅 | GPT-4o-mini | 질의 유형 분류, 컬렉션 선택, 도구 선택 |
| 엔티티+트리플 추출 | GPT-4o-mini | GraphRAG 인제스트 시 엔티티 추출 |

### 1-2. 에이전트 아키텍처

```
안전관리자 입력
    ↓
[LLM: GPT-4o-mini] 질의 분류
    → Q-01 법규 → regulation 검색
    → Q-02 사례 → accident_cases 검색 + 현장API
    → Q-03 기상 → 기상API + regulation 검색
    → Q-04 절차 → safety_knowledge 검색
    → Q-05 이력 → DB 조회
    → Q-06 복합 → 교차 검색
    ↓
[CLI 호출] 해당 인터페이스 자율 호출
    ↓
[컨텍스트 조립] 검색 결과 + 그래프 관계 + 실시간 데이터
    ↓
[LLM: GPT-4o] 최종 답변 생성 (출처 포함)
    ↓
안전관리자에게 전달
```

---

## 2. 시스템 프롬프트

### 메인 응답 LLM (GPT-4o)

```
당신은 대림건설 안전관리 보조 AI입니다.

역할:
- 안전관리자의 질문에 대해 제공된 문서와 데이터를 기반으로 정확한 정보를 제공합니다.
- 모든 답변에는 출처(문서명, 조항번호)를 반드시 포함합니다.
- 법규 정보는 최신 여부를 확인하도록 주의를 당부합니다.
- AI는 참고 자료를 제공하며, 최종 안전 판단은 반드시 안전관리자가 합니다.

금지 사항:
- 제공된 문서에 없는 내용을 추측하거나 생성하지 않습니다.
- 개인 근로자 정보를 식별할 수 있는 방식으로 응답하지 않습니다.
- 법적 책임이 수반되는 확정적 판단을 내리지 않습니다.

출처 표시 형식:
[출처: {문서명} - {섹션/조항}]
```

### 질의 분류 LLM (GPT-4o-mini)

```
다음 질의를 분류하고 적절한 도구를 선택하세요.

질의 유형:
- Q-01: 법규/매뉴얼 기준 질의 → search(regulation, safety_knowledge)
- Q-02: 사고 사례 검색 → search(accident_cases) + field_api
- Q-03: 기상 조건 판단 → weather_api + search(regulation)
- Q-04: 절차/체크리스트 → search(safety_knowledge)
- Q-05: 점검/사고 이력 → db_query
- Q-06: 복합 질의 → search_cross + graphrag

동의어 처리:
- "높이작업" = "고소작업" (동일 처리)
- "터파기" = "굴착작업"
- "끼임" = "협착"
```

---

## 3. 도구(Tool) 정의

### LLM Function Calling 정의

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_documents",
            "description": "안전 문서 검색. 매뉴얼, SOP, 법규, 사고 사례 검색",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "검색 질의"},
                    "collection": {
                        "type": "string",
                        "enum": ["safety_knowledge", "regulation", "accident_cases", "cross"],
                        "description": "검색 대상 컬렉션"
                    },
                    "top_k": {"type": "integer", "default": 5},
                    "work_type": {"type": "string", "description": "작업 유형 필터 (선택)"},
                    "use_graphrag": {"type": "boolean", "default": False}
                },
                "required": ["query", "collection"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_weather_judgment",
            "description": "현장 기상 조회 및 작업 중지 판단",
            "parameters": {
                "type": "object",
                "properties": {
                    "site_id": {"type": "string"},
                    "work_type": {"type": "string", "description": "작업 유형 (크레인, 고소작업 등)"}
                },
                "required": ["site_id"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "query_inspection_history",
            "description": "안전 점검 이력 조회 (안전관리 시스템 DB)",
            "parameters": {
                "type": "object",
                "properties": {
                    "site_id": {"type": "string"},
                    "date_from": {"type": "string", "format": "date"},
                    "date_to": {"type": "string", "format": "date"},
                    "query_type": {
                        "type": "string",
                        "enum": ["inspection", "accident", "training", "pending"],
                        "default": "inspection"
                    }
                },
                "required": ["site_id"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_site_info",
            "description": "현장 정보 조회 (현장관리 시스템 API)",
            "parameters": {
                "type": "object",
                "properties": {
                    "site_id": {"type": "string"},
                    "info_type": {
                        "type": "string",
                        "enum": ["basic", "workers", "equipment"],
                        "default": "basic"
                    }
                },
                "required": ["site_id"]
            }
        }
    }
]
```

---

## 4. FastAPI 엔드포인트 설계

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel

app = FastAPI(title="대림건설 안전관리 보조 API")

class QueryRequest(BaseModel):
    query: str
    site_id: str | None = None
    work_type: str | None = None
    user_id: str  # 안전관리자 ID (로깅용)

class QueryResponse(BaseModel):
    answer: str
    sources: list[dict]
    confidence: str  # "high" / "medium" / "low"
    regulation_note: str | None  # 법규 최신 여부 경고
    response_time_ms: int

@app.post("/query", response_model=QueryResponse)
async def process_query(req: QueryRequest):
    """안전 질의응답 메인 엔드포인트"""
    # 1. 질의 분류
    # 2. 도구 호출 (CLI 또는 직접)
    # 3. 컨텍스트 조립
    # 4. LLM 답변 생성
    # 5. 로깅 (query, answer, sources, user_id)
    pass

@app.get("/health")
async def health_check():
    return {"status": "ok"}
```

---

## 5. 컨텍스트 조립 형식

```
[관련 문서]
- [안전관리매뉴얼_건축편.pdf - 3장 고소작업] 고소작업(2m 이상) 시 안전대는...
- [고소작업_SOP.pdf - 2절 작업 전 점검] 작업 전 안전대 부착 상태 확인...
- [산업안전보건법 제44조] 사업주는 근로자가 추락할 위험이 있는 장소...

[관련 법규 관계]
- 고소작업 -[동의어]-> 높이작업
- 고소작업 -[적용법규]-> 산업안전보건법 제44조
- 산업안전보건법 제44조 -[구체화]-> 안전관리매뉴얼 3장

[주의]
⚠️ 법규 정보: 현재 갱신 파이프라인 구축 전입니다. 최신 법규 개정 사항은 직접 확인을 권고합니다.
```

---

## 6. 로깅 설계

### 로깅 항목

```python
log_entry = {
    "timestamp": "2026-03-27T14:23:45",
    "user_id": "U-023",           # 안전관리자 ID
    "site_id": "SITE-001",        # 현장 ID (있는 경우)
    "query": "고소작업 안전대 기준",
    "query_type": "Q-01",
    "collections_searched": ["safety_knowledge", "regulation"],
    "graphrag_used": True,
    "answer_snippet": "고소작업(2m 이상) 시 안전대는...",
    "sources": ["안전관리매뉴얼_건축편.pdf - 3장", "산업안전보건법 제44조"],
    "response_time_ms": 2340,
    "confidence": "high"
}
```

### 로그 보존

- 저장소: AWS CloudWatch Logs (30일 보존) + S3 장기 보관 (3년)
- 중대재해처벌법 대비: 모든 안전 관련 조회 이력 보존

---

## 7. 사용자 인터페이스 요건

### UI 요건 (최소)

- **입력**: 자연어 텍스트 입력창 + 현장 선택 드롭다운
- **출력**: 답변 텍스트 + 출처 카드 (문서명+조항+PDF 링크)
- **기상 위젯**: 현재 현장 기상 + 작업 중지 여부 상시 표시
- **접근 방식**: 웹 브라우저 (모바일 반응형), VPN 필요

### 성능 요건

| 항목 | 목표 |
|------|------|
| 응답 시간 | 5초 이내 (벡터 검색 기준) |
| GraphRAG 응답 | 10초 이내 |
| 동시 사용자 | 최대 25명 (안전관리자 전원) |
| 가용성 | 99% 이상 (현장 근무 시간 기준) |
