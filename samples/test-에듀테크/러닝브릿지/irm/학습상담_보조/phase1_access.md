# 학습상담 보조 — 접근 배치 설계

> 작성일: 2026-03-27
> 접근 질의: 3건 / 전체 14건 (21%)
> 참조: CLI 런북 v10

---

## 대상 시스템

| # | 시스템명 | 접근 방식 | CLI 가능? | 우선순위 | 근거 | 비고 |
|---|---------|----------|:--------:|:-------:|:----:|------|
| 1 | 수강생 DB | 관계형 DB (RDS PostgreSQL) | Y | ★★★ | ✅ | 수강이력/진도/성적 조회. 비식별화 필수 |
| 2 | Zendesk CS | 외부 API (REST) | Y | ★★ | ✅ | 특정 수강생 상담 이력 조회. API 키 필요 |

### 접근 질의 매핑

| 질의 | 대상 시스템 | 용도 |
|------|-----------|------|
| #5 "이 수강생의 현재 학습 진도가 어떻게 되나요?" | 수강생 DB | 진도 조회 |
| #6 "이 수강생이 이전에 어떤 문의를 했었나요?" | Zendesk CS | 상담 이력 조회 |
| #13 "이 수강생이 들은 강의 목록과 성적을 알려주세요" | 수강생 DB | 수강이력/성적 조회 |

---

## CLI 설계

### 1. 수강생 DB CLI (`cli-anything-student`)

- **경계**: RDS PostgreSQL (수강생 관련 테이블)
- **접근 방식**: psycopg2 직접 연결
- **접근 모드**: 읽기전용
- **보안**: 비식별화 적용 (이름 마스킹, 이메일 도메인만 노출)

#### 명령어

```
cli-anything-student --json enrollment list --student-id <ID>
  → 수강생의 수강 강의 목록 반환
  → 출력: [{course_id, course_title, enrolled_at, status}]

cli-anything-student --json progress get --student-id <ID> [--course-id <CID>]
  → 수강생의 학습 진도 반환 (강의 지정 시 해당 강의만)
  → 출력: [{course_id, course_title, progress_pct, last_accessed, chapters_completed}]

cli-anything-student --json grade list --student-id <ID>
  → 수강생의 성적 목록 반환
  → 출력: [{course_id, course_title, score, grade, completed_at}]

cli-anything-student --json profile get --student-id <ID>
  → 수강생 기본 정보 반환 (비식별화 적용)
  → 출력: {student_id, name_masked, email_domain, enrolled_courses_count, join_date}
```

#### 비식별화 규칙

```python
def mask_name(name: str) -> str:
    """홍길동 → 홍*동"""
    if len(name) <= 1:
        return "*"
    return name[0] + "*" * (len(name) - 2) + name[-1]

def mask_email(email: str) -> str:
    """user@example.com → ***@example.com"""
    domain = email.split("@")[-1]
    return f"***@{domain}"
```

#### 데이터 접근 레이어

```python
import psycopg2, os
from contextlib import contextmanager

DATABASE_URL = os.environ.get("DATABASE_URL", "postgresql://user:pass@rds-endpoint:5432/learning_bridge")

@contextmanager
def _conn():
    conn = psycopg2.connect(DATABASE_URL)
    try:
        yield conn
    finally:
        conn.close()

def get_enrollments(student_id: str):
    sql = """
        SELECT course_id, course_title, enrolled_at, status
        FROM enrollments e
        JOIN courses c ON e.course_id = c.id
        WHERE e.student_id = %s
        ORDER BY enrolled_at DESC
    """
    with _conn() as conn:
        cur = conn.cursor()
        cur.execute(sql, [student_id])
        cols = [d[0] for d in cur.description]
        return [dict(zip(cols, row)) for row in cur.fetchall()]
```

---

### 2. Zendesk CS CLI (`cli-anything-zendesk`)

- **경계**: Zendesk API (상담 티켓)
- **접근 방식**: REST API (requests)
- **접근 모드**: 읽기전용
- **보안**: 수강생 식별 정보 마스킹 후 반환

#### 명령어

```
cli-anything-zendesk --json ticket list --requester-id <ID> [--limit <N>]
  → 특정 수강생의 상담 이력 반환 (최근 N건, 기본 10)
  → 출력: [{ticket_id, subject, status, created_at, description_masked}]

cli-anything-zendesk --json ticket get --ticket-id <ID>
  → 특정 티켓 상세 반환
  → 출력: {ticket_id, subject, status, created_at, description_masked, comments_masked}

cli-anything-zendesk --json ticket search --query <QUERY> [--limit <N>]
  → 키워드 기반 티켓 검색 (Zendesk Search API)
  → 출력: [{ticket_id, subject, status, created_at, excerpt_masked}]
```

#### 데이터 접근 레이어

```python
import requests, os

ZENDESK_SUBDOMAIN = os.environ.get("ZENDESK_SUBDOMAIN", "learningbridge")
ZENDESK_EMAIL = os.environ.get("ZENDESK_EMAIL", "")
ZENDESK_API_TOKEN = os.environ.get("ZENDESK_API_TOKEN", "")
BASE_URL = f"https://{ZENDESK_SUBDOMAIN}.zendesk.com/api/v2"

def _get(path, params=None):
    r = requests.get(
        f"{BASE_URL}{path}",
        params=params,
        auth=(f"{ZENDESK_EMAIL}/token", ZENDESK_API_TOKEN),
    )
    r.raise_for_status()
    return r.json()

def get_tickets_by_requester(requester_id: str, limit: int = 10):
    data = _get(f"/users/{requester_id}/tickets/requested.json")
    tickets = data.get("tickets", [])[:limit]
    return [mask_ticket(t) for t in tickets]
```

---

## REST 게이트웨이

### 개요
- **프레임워크**: FastAPI
- **노출 범위**: 내부만 (상담사 보조 도구 전용)
- **인증**: API Key (내부 서비스 간 통신)
- **감사로그**: 모든 접근 요청 기록 (student_id, 요청 시각, 요청 유형)

### 엔드포인트

```
GET  /api/v1/students/{student_id}/enrollments     → 수강 목록
GET  /api/v1/students/{student_id}/progress         → 학습 진도
GET  /api/v1/students/{student_id}/progress/{course_id}  → 특정 강의 진도
GET  /api/v1/students/{student_id}/grades           → 성적 목록
GET  /api/v1/students/{student_id}/profile          → 기본 정보 (비식별화)
GET  /api/v1/students/{student_id}/tickets          → 상담 이력 (Zendesk)
GET  /api/v1/tickets/{ticket_id}                    → 티켓 상세
GET  /api/v1/tickets/search?q={query}               → 티켓 검색
```

### 아키텍처

```
상담사 보조 UI
      │
      ▼
┌─────────────────┐
│  FastAPI Gateway │ ← API Key 인증 + 감사로그
│  (내부 네트워크) │
└──────┬──────────┘
       │
  ┌────┴────┐
  │         │
  ▼         ▼
┌──────┐  ┌────────┐
│ RDS  │  │Zendesk │
│(PgSQL)│  │ API    │
└──────┘  └────────┘
```

### 감사로그 스키마

```python
# 모든 접근 요청 기록
{
    "timestamp": "2026-03-27T10:30:00Z",
    "agent_id": "counselor_01",         # 상담사 ID
    "student_id": "STU_12345",          # 조회 대상
    "endpoint": "/api/v1/students/STU_12345/progress",
    "method": "GET",
    "response_status": 200
}
```

---

## 환경 변수

```bash
# 수강생 DB
DATABASE_URL=postgresql://reader:password@rds-endpoint:5432/learning_bridge

# Zendesk
ZENDESK_SUBDOMAIN=learningbridge
ZENDESK_EMAIL=api@learningbridge.com
ZENDESK_API_TOKEN=xxxxx

# Gateway
API_KEY=internal-service-key
```

---

## CLI-SPEC.md 요약 (LLM 컨텍스트용)

LLM이 자율 호출할 때 참조하는 명세:

```markdown
## cli-anything-student
수강생 DB 조회 CLI. 읽기전용. 비식별화 적용.
- enrollment list --student-id <ID>  → 수강 목록
- progress get --student-id <ID>     → 학습 진도
- grade list --student-id <ID>       → 성적
- profile get --student-id <ID>      → 기본 정보

## cli-anything-zendesk
Zendesk 상담 이력 조회 CLI. 읽기전용. 마스킹 적용.
- ticket list --requester-id <ID>    → 상담 이력
- ticket get --ticket-id <ID>        → 티켓 상세
- ticket search --query <QUERY>      → 키워드 검색
```

---

## 체이닝 연동 (Phase 0 체이닝 패턴 지원)

| 체이닝 질의 | 접근 단계 | CLI 호출 |
|------------|----------|---------|
| #7 환불 문의 | 수강생 결제일 조회 | `student enrollment list --student-id <ID>` |
| #8 변경 문의 | 수강생 현재 수강 상태 조회 | `student enrollment list --student-id <ID>` |
| #9 학습 추천 | 수강이력/진도 조회 | `student progress get --student-id <ID>` |

접근 결과는 JSON으로 반환되어 실행 레이어(`/irm-exec`)에 입력으로 전달된다.
