# 학습상담 보조 — 접근 구현 체크리스트

> 작성일: 2026-03-27

## CLI 구현

### cli-anything-student (RDS PostgreSQL)
- [ ] CLI 경계: 수강생 관련 테이블 (enrollments, progress, grades, students)
- [ ] 명령어 구현: enrollment list, progress get, grade list, profile get
- [ ] JSON 출력 형식 정의 (각 명령어별 스키마)
- [ ] 비식별화 함수: mask_name(), mask_email()
- [ ] setup.py + Click 프레임워크 설정
- [ ] psycopg2 데이터 접근 레이어 (읽기전용)
- [ ] 환경 변수: DATABASE_URL
- [ ] 단위 테스트: 각 명령어 정상/에러 케이스
- [ ] CLI-SPEC.md 작성 (LLM 자율 호출용)

### cli-anything-zendesk (Zendesk API)
- [ ] CLI 경계: Zendesk 티켓 조회
- [ ] 명령어 구현: ticket list, ticket get, ticket search
- [ ] JSON 출력 형식 정의
- [ ] 식별정보 마스킹 함수: mask_ticket()
- [ ] setup.py + Click 프레임워크 설정
- [ ] requests HTTP 클라이언트 (읽기전용)
- [ ] 환경 변수: ZENDESK_SUBDOMAIN, ZENDESK_EMAIL, ZENDESK_API_TOKEN
- [ ] Rate limit 처리 (429 응답 시 재시도)
- [ ] 단위 테스트: 각 명령어 정상/에러 케이스
- [ ] CLI-SPEC.md 작성

### REST 게이트웨이 (FastAPI)
- [ ] FastAPI 앱 생성
- [ ] 엔드포인트 8개 구현 (student 5 + ticket 3)
- [ ] API Key 인증 미들웨어
- [ ] 감사로그: 모든 요청 기록 (timestamp, agent_id, student_id, endpoint, status)
- [ ] 내부 네트워크 한정 배포 (EC2, 포트 제한)
- [ ] OpenAPI 문서 자동 생성 확인
- [ ] 에러 응답 표준화 (400/404/500)

## 통합 검증
- [ ] CLI → REST 엔드투엔드 테스트 (각 명령어가 REST에서도 동일 결과)
- [ ] 비식별화 확인: 원본 이름/이메일이 응답에 포함되지 않음
- [ ] 체이닝 테스트: 접근 결과가 실행 레이어에 정상 전달
