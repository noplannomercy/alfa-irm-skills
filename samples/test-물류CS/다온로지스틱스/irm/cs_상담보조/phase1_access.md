# cs_상담보조 — 접근 배치 설계

> 작성일: 2026-04-11
> 접근 질의: 2건 / 전체 12건 (17%)
> 참조: CLI 런북 v10

## 대상 시스템

| # | 시스템명 | 접근 방식 | CLI 가능? | 우선순위 | 근거 | 비고 |
|---|---------|----------|:--------:|:-------:|:----:|------|
| 1 | CRM | REST API (이미 존재) | Y | ★★★ | ✅ | 상담 이력 조회, 고객별 이력 |

## CLI 설계

### CRM CLI (`cli-crm`)
- 경계: CRM 시스템 (상담 이력 조회)
- 접근: REST API (기존 API 활용)
- 명령어:
  - `crm consultation get --id <상담ID>` → 특정 상담 건 조회
  - `crm consultation list --customer <고객ID> --limit <N>` → 고객별 이력
  - `crm consultation stats --date <날짜> --category <카테고리>` → 일별 통계
- 출력: JSON 표준 형식
- 접근 모드: **읽기전용**
- 보안: 개인정보 포함 → 응답에서 개인정보 필터링(이름/전화/주소 마스킹) 옵션

## REST 게이트웨이

기존 CRM REST API가 이미 있으므로 **별도 게이트웨이 불필요.** CLI가 기존 API를 래핑.

## 주의사항
- CRM API 응답에 개인정보 포함 → LLM에 전달 전 필터링 필요
- AWS 환경이므로 CLI → CRM API 접근에 네트워크 이슈 없음
