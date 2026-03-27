# 자동차보험 심사 보조 — 접근 배치 설계 (/irm-access 단독)

> 작성일: 2026-03-27
> 접근 질의: 2/14건 (14%)
> 설계 신뢰도: ✅ (ERD/API 스펙 실제 확인)
> 참조: CLI 런북 v10

---

## 대상 시스템

| # | 시스템명 | 접근 방식 | CLI 가능? | 우선순위 | 근거 | 비고 |
|---|---------|----------|:--------:|:-------:|:----:|------|
| 4 | 심사 이력 DB | PostgreSQL 직접 | Y | ★★★ | ✅ | rds-uw.ap-northeast-2, ERD 확인 |
| 5 | 보험 코어 API | REST API | Y | ★★★ | ✅ | https://core-api.internal/v2, 스펙 확인 |

---

## CLI 설계 (CLI 런북 Path A: 신규)

### uw CLI (심사 시스템)

- **경계**: 심사 시스템 (PostgreSQL rds-uw)
- **명령어**:
  - `uw case --case-id [ID]` → 심사 건 상세 (요청/심사의견/결정)
  - `uw history --contract-id [ID]` → 계약별 심사 이력 (최근 N건)
  - `uw similar --accident-type [유형] --product auto --limit [N]` → 유사 사례 목록
  - `uw stats --date-range [시작~끝] --product auto` → 심사 통계 (승인/거절/보류)
- **출력**: JSON 표준 형식
- **접근 모드**: 읽기전용
- **보안**:
  - 개인정보 자동 가명처리 (이름 마스킹, 주민번호 제거 — Phase 2 GOVERN 기준)
  - 심사역 계정으로만 접근
- **구현**: Click 프레임워크 + setup.py (CLI 런북 공통 패턴)
- **테스트**: 명령어별 정상 5건 + 오류 5건 = 40건

### core CLI (보험 코어)

- **경계**: 보험 코어 시스템 (REST API)
- **명령어**:
  - `core contract --contract-id [ID]` → 계약 정보 (상품/기간/보험료)
  - `core premium --contract-id [ID]` → 보험료 상세 (할증/할인 포함)
  - `core claims --contract-id [ID]` → 지급 이력 (**건수/총액만**, 개별 상세 비노출)
- **출력**: JSON 표준 형식
- **접근 모드**: 읽기전용
- **보안**:
  - **신용정보 제한**: `core claims`는 건수/총액 통계만 반환, 개별 지급 상세는 차단
  - 근거: Phase 2 GOVERN — 보험금 지급 이력 = 신용정보 → LLM 직접 투입 금지
- **구현**: Click 프레임워크 + requests (REST 호출)
- **테스트**: 명령어별 정상 5건 + 오류 5건 = 30건

---

## CLI-SPEC.md 생성 계획

각 CLI에 대해 `CLI-SPEC.md`를 생성하여 LLM이 자율 호출할 수 있도록 한다 (CLI 런북 참조).

```
cli-specs/
├── uw-cli-spec.md      ← 심사 시스템 CLI 명세
└── core-cli-spec.md    ← 코어 시스템 CLI 명세
```

---

## REST 게이트웨이

- **프레임워크**: FastAPI
- **노출 범위**: 내부만 (심사 시스템 내)
- **인증**: 심사역 계정 + 역할 기반 접근제어 (RBAC)
- **감사로그**: 모든 API 호출 기록 (CloudWatch)
  - 기록 항목: 호출자, 명령어, 파라미터, 타임스탬프, 응답 상태
  - 금감원 검사 대비: AI 조회 이력 보존 (Phase 2 GOVERN)
- **엔드포인트**:
  - `POST /uw/case` → uw case
  - `POST /uw/history` → uw history
  - `POST /uw/similar` → uw similar
  - `POST /uw/stats` → uw stats
  - `POST /core/contract` → core contract
  - `POST /core/premium` → core premium
  - `POST /core/claims` → core claims (신용정보 제한 적용)

---

## 주의 사항

- 이 설계는 **접근(CLI+REST)만** 다룸
- 탐색(컬렉션/RAG)은 `/irm-search`에서, 실행(파이프라인)은 `/irm-exec`에서 별도
- 신용정보 제한은 CLI 레벨에서 강제 (API 레벨이 아닌 CLI 출력에서 차단)
- 가명처리 규칙은 Phase 2 GOVERN 문서 참조
