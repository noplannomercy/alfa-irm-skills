# 학습상담 보조 — 실행 배치 설계

> 작성일: 2026-03-27
> 실행(확정) 질의: 2건 / 전체 14건 (14%)
> 실행(유연) 질의: 1건 / 전체 14건 (7%)
> 참조: IRM 실행가이드 v10

---

## 실행(확정) — 결정적 파이프라인

### 1. 환불 판단 파이프라인

> 질의 #7: "결제한 지 7일 됐는데 환불 가능한가요?"

**파이프라인:**

```
Step 1: [탐색] faq_and_rules 컬렉션 → 환불 규정 해당 조항 검색
        → 입력: "환불 기준" / 필터: doc_type = "rule_clause"
        → 출력: {rule_text, rule_section}

Step 2: [접근] cli-anything-student → 수강생 결제/수강 정보 조회
        → 입력: student_id
        → 호출: enrollment list --student-id <ID>
        → 출력: {enrolled_at, payment_date, course_id, progress_pct}

Step 3: [판단] 규칙 기반 → 환불 가능 여부 결정
        → 입력: rule_text + enrollment_data
        → 규칙 적용 (아래 판단 로직)
        → 출력: {eligible: true/false, reason, refund_type, refund_amount_pct}

Step 4: [응답] 상담사에게 판단 결과 + 근거 제공
        → 출력: 환불 가능 여부 + 적용 규정 조항 + 계산 근거
```

**판단 로직 (규칙 기반):**

```python
def judge_refund(rule: dict, enrollment: dict) -> dict:
    days_since_payment = (today - enrollment["payment_date"]).days
    progress_pct = enrollment.get("progress_pct", 0)

    # 환불규정 PDF 기반 규칙 (PoC 시 규정 분석 후 확정)
    # 예시 규칙 — 실제 규정 수령 후 구체화
    if days_since_payment <= 7 and progress_pct < 10:
        return {
            "eligible": True,
            "refund_type": "full",
            "refund_amount_pct": 100,
            "reason": f"결제 후 {days_since_payment}일, 진도 {progress_pct}% — 전액 환불 대상",
            "rule_section": rule["rule_section"]
        }
    elif days_since_payment <= 30 and progress_pct < 50:
        return {
            "eligible": True,
            "refund_type": "partial",
            "refund_amount_pct": 50,  # 규정에 따라 조정
            "reason": f"결제 후 {days_since_payment}일, 진도 {progress_pct}% — 부분 환불 대상",
            "rule_section": rule["rule_section"]
        }
    else:
        return {
            "eligible": False,
            "refund_type": None,
            "refund_amount_pct": 0,
            "reason": f"결제 후 {days_since_payment}일, 진도 {progress_pct}% — 환불 불가",
            "rule_section": rule["rule_section"]
        }
```

**성공/실패 경계:**

| Step | 성공 | 실패 시 대응 |
|------|------|-----------|
| Step 1 | 환불 규정 조항 1건 이상 검색됨 | "환불 규정을 찾을 수 없습니다. 담당자에게 확인하세요." |
| Step 2 | 수강생 결제 정보 조회 성공 | "수강생 정보를 조회할 수 없습니다. 수강생 ID를 확인하세요." |
| Step 3 | 규칙 조건 중 하나에 매칭 | 매칭 실패 시 "규정 범위 밖입니다. 담당자에게 에스컬레이션하세요." |

---

### 2. 강의 변경 판단 파이프라인

> 질의 #8: "다른 강의로 변경하고 싶다는데 가능한가요?"

**파이프라인:**

```
Step 1: [탐색] faq_and_rules 컬렉션 → 변경 규정 해당 조항 검색
        → 입력: "강의 변경 기준" / 필터: doc_type = "rule_clause"
        → 출력: {rule_text, rule_section}

Step 2: [접근] cli-anything-student → 수강생 현재 수강 상태 조회
        → 입력: student_id
        → 호출: enrollment list --student-id <ID>
        → 출력: {course_id, enrolled_at, status, progress_pct}

Step 3: [판단] 규칙 기반 → 변경 가능 여부 결정
        → 입력: rule_text + enrollment_data
        → 규칙 적용
        → 출력: {eligible: true/false, reason, conditions}

Step 4: [응답] 상담사에게 판단 결과 + 근거 제공
```

**판단 로직:**

```python
def judge_course_change(rule: dict, enrollment: dict) -> dict:
    days_since_enrollment = (today - enrollment["enrolled_at"]).days
    progress_pct = enrollment.get("progress_pct", 0)
    status = enrollment.get("status", "")

    if status == "completed":
        return {
            "eligible": False,
            "reason": "이수 완료된 강의는 변경 불가",
            "rule_section": rule["rule_section"]
        }
    elif progress_pct > 50:
        return {
            "eligible": False,
            "reason": f"진도 {progress_pct}% — 50% 초과 시 변경 불가 (규정)",
            "rule_section": rule["rule_section"]
        }
    else:
        return {
            "eligible": True,
            "reason": f"진도 {progress_pct}%, 변경 가능",
            "conditions": "차액 정산 필요 시 별도 안내",
            "rule_section": rule["rule_section"]
        }
```

---

## 실행(유연) — 판단 기반 프로세스

### 3. 학습 추천 프로세스

> 질의 #9: "이 수강생 진도가 많이 느린데, 어떤 강의를 추천하면 좋을까요?"

**프로세스:**

```
Step 1: [접근] cli-anything-student → 수강생 이력/진도 전체 조회
        → 호출: progress get --student-id <ID>
        → 출력: [{course_id, progress_pct, last_accessed, score}]

Step 2: [접근] cli-anything-student → 수강생 성적 조회
        → 호출: grade list --student-id <ID>
        → 출력: [{course_id, score, grade}]

Step 3: [LLM 판단] 학습 패턴 분석
        → 입력: 진도 데이터 + 성적 데이터
        → LLM 프롬프트:
          "이 수강생의 학습 패턴을 분석하세요:
           - 어떤 분야에 강/약한가?
           - 진도가 느린 이유 추정 (난이도? 흥미?)
           - 수강 패턴 (집중형? 분산형?)"
        → 출력: {strengths, weaknesses, pattern_analysis}

Step 4: [탐색] course_content 컬렉션 → 관련 강의 검색
        → 입력: LLM 분석 결과 기반 검색 쿼리
        → 필터: level = [약점 보완 수준], category = [관심 분야]
        → 출력: [{course_id, course_title, level, category}]

Step 5: [LLM 판단] 추천 강의 선정 + 사유 생성
        → 입력: 패턴 분석 + 검색된 강의 목록
        → 출력: [{recommended_course, reason, priority}]

Step 6: [사람 확인] 상담사가 추천 결과 검토 후 수강생에게 안내
        → Human-in-the-loop: 상담사가 추천 적절성 판단
        → 상담사가 수정/보완 후 최종 안내
```

**판단 포인트:** Step 3 (패턴 분석), Step 5 (추천 선정)
**Human-in-the-loop:** Step 6 — 상담사가 최종 검토 및 수정

**고정화 조건:**
- 동일 패턴(진도 30% 미만 + 특정 카테고리)에서 동일 추천이 10회 연속 발생
- → 해당 패턴을 규칙으로 정의하여 실행(확정)으로 전환 검토
- 예: "프로그래밍 입문 강의 진도 30% 미만이면 → '프로그래밍 기초 다지기' 강의 자동 추천"

**LLM 프롬프트 템플릿:**

```
당신은 학습 상담 어시스턴트입니다.

## 수강생 학습 데이터
{progress_data}
{grade_data}

## 분석 요청
1. 이 수강생의 강점/약점 분야를 파악하세요
2. 진도가 느린 강의의 원인을 추정하세요 (난이도/흥미/시간 부족)
3. 보완할 수 있는 강의를 추천하되, 수강생 수준에 맞는 난이도를 선택하세요

## 출력 형식
- 강점: [분야]
- 약점: [분야]
- 추천 강의: [강의명] — 사유: [왜 이 강의인지]
- 우선순위: [높음/중간/낮음]

## 주의
- 추천은 상담사가 최종 확인합니다 (보조 수단)
- 확실하지 않은 분석에는 "확인 필요" 표시
```

---

## 실행 레이어 요약

| 절차 | 유형 | 리소스 | 판단 방식 | HitL | 근거 |
|------|------|--------|----------|:----:|:----:|
| 환불 판단 | 확정 | #6a, #6b, #4a | 규칙 기반 (규정 PDF) | ✗ | ✅ |
| 강의 변경 판단 | 확정 | #6a, #6b, #4a | 규칙 기반 (규정 PDF) | ✗ | ✅ |
| 학습 추천 | 유연 | #4a, #4b, #2 | LLM 판단 | ✓ (Step 6) | ✅ |

---

## PoC 구현 우선순위

| 우선순위 | 절차 | 근거 |
|:-------:|------|------|
| 1 | 환불 판단 | 문의의 20% 차지, 규칙 명확, 즉시 구현 가능 |
| 2 | 강의 변경 판단 | 환불과 동일 규정 기반, 환불 구현 후 확장 용이 |
| 3 | 학습 추천 | LLM 판단 필요, 품질 검증에 시간 소요, PoC 후반부 |
