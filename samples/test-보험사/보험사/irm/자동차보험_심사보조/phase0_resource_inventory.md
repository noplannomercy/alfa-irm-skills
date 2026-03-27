# 자동차보험 심사 보조 — 리소스 인벤토리

> 작성일: 2026-04-03
> 고객: 미래에셋손해보험
> 소스: usecase/20260403_자동차보험_심사보조.md

## 리소스 목록

| # | 리소스명 | 유형 | 위치/접근 방법 | 근거 | 규모 |
|---|---------|------|-------------|:----:|------|
| 1 | 심사 가이드라인 | 문서 | \\fileserver\insurance\guidelines\심사가이드라인_v3.pdf | ✅ | PDF 120p, 상품별 |
| 2 | 약관집 (자동차) | 문서 | \\fileserver\insurance\terms\자동차보험약관_2026.pdf | ✅ | PDF ~100p |
| 3 | FAQ | 문서 | \\fileserver\insurance\faq\심사FAQ_v2.xlsx | ✅ | 200항목 |
| 4 | 심사 이력 DB | 시스템 | PostgreSQL rds-uw.ap-northeast-2 / underwriting.applications, reviews, decisions | ✅ | 5년 60만건 (자동차 ~27만건) |
| 5 | 보험 코어 API | 시스템 | https://core-api.internal/v2 (스펙: s3://docs/core-api-spec.yaml) | ✅ | 계약/보험료/지급이력 |
| 6 | OCR 청구서 | 시스템 | s3://insurance-ocr/claims/ | ✅ | 2년 16만건 (품질 혼재) |
| 7 | 심사 절차서 | 절차 | \\fileserver\insurance\procedures\심사절차_v5.pdf | ✅ | 5단계, 상품별 분기 |
| 8 | 재심사 절차서 | 절차 | \\fileserver\insurance\procedures\재심사절차_v2.pdf | ✅ | 3단계 |

### 근거 기준
- ✅ **실제**: IT팀 제공, 스펙/ERD/파일 확인 완료

## 리소스 유형별 요약
- 문서/파일: 3건 (✅3)
- 시스템: 3건 (✅3)
- 절차/매뉴얼: 2건 (✅2)
- **합계: 8건 — 신뢰도: ✅8건 (100%)**
