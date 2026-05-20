# Python / FastAPI 백엔드 코딩 규칙 (Maestro)

이 문서는 **Cursor / 에이전트**가 `backend/` 작업 시 따를 구현 규칙입니다.  
`backend/.cursorrules` 및 `.cursor/rules/backend-coding.mdc`와 연동됩니다.

---

## 0. 구현 전 필수

1. 본 문서(`BACKEND_RULES.md`)를 읽고 작업 범위에 맞는 섹션을 적용한다.
2. 도메인·맥락이 필요하면 `docs/` 하위 관련 문서를 추가로 읽는다 (예: `docs/타이타닉개발/`, `docs/maestro개발/`).
3. `backend/.cursorrules`의 하네스(단순성, 정밀한 수정)와 **모순되지 않게** 코드를 쓴다.

---

## 1. 디렉터리·실행

| 항목 | 규칙 |
|------|------|
| 앱 루트 | `backend/apps/` (`PYTHONPATH=apps`) |
| 엔트리 | `main.py` → `uvicorn main:app` |
| 로컬 실행 | `backend/start-backend.ps1` (포트 **8000**, 이 터미널에 로그) |
| 환경 변수 | `backend/.env` (`matrix.app.keymaker` 부트스트랩) |
| 비밀 | `.env`·키를 커밋하지 않는다 |

---

## 2. 레이어 아키텍처 (필수)

신규 API·도메인 로직은 **secom** 모듈과 동일한 계층을 따른다.

```text
main.py (라우트·트랜잭션 경계)
  → *Controller*  (요청 단위 조립, 로깅)
    → *Service*    (비즈니스 규칙, HTTPException)
      → *Repository* (SQLAlchemy 쿼리·persist)
        → *Model*   (ORM 엔티티)
```

| 계층 | 역할 | 금지 |
|------|------|------|
| `main.py` | 라우트 등록, `DbSession` 주입, `commit`/`rollback` | 비즈니스 규칙·SQL 직접 작성 |
| Controller | Service 호출, 레이어 완료 로그 | DB 접근, 복잡한 분기 |
| Service | 검증, 중복 체크, `HTTPException` | raw SQL, 라우트 정의 |
| Repository | `select`/`add`/`flush` | HTTP 응답 결정 |
| schemas | Pydantic 요청·응답 DTO | ORM 대체용 남용 |

**예시 모듈:** `backend/apps/secom/app/` (`controllers/`, `services/`, `repositories/`, `models/`, `schemas/`).

---

## 3. 데이터베이스

- **비동기:** `AsyncSession`, `DbSession` 의존성 (`database` / `db.session`).
- **트랜잭션:** 쓰기 API는 라우트에서 `try` → 성공 시 `await session.commit()`, `HTTPException`·기타 예외 시 `await session.rollback()`.
- **설정 없음:** `is_database_configured()`가 false면 `503`과 명한 `detail`.
- **테이블 초기화:** lifespan / `secom.app.db_init` 패턴 유지.
- **엔티티 PK:** 모든 테이블은 정수 자동 증가 기본 키, 컬럼명·필드명 `id` 통일 → `docs/DevOps/Backend/ENTITY_RULE.md`.

---

## 4. API·에러·로깅

- 요청/응답: **Pydantic** `BaseModel`, `response_model` 명시.
- 클라이언트 오류: `HTTPException` + 한국어 `detail` (기존 톤 유지).
- 로깅: 모듈별 `logger = logging.getLogger(__name__)`, 레이어 완료 시 `[ClassName] action 레이어 완료` 형식.
- `main.py` 미들웨어: 요청 `→` / 응답 `←` 로그 유지.

---

## 5. 보안·인증 (secom)

- 비밀번호: `secom.app.services.password_hasher` (`hash_password` / `verify_password`)만 사용.
- 평문 비밀번호 저장·로그 출력 금지.

---

## 6. 기존 앱 모듈

| 모듈 | 용도 |
|------|------|
| `secom` | 회원가입·로그인·사용자 |
| `titanic` | 수업용 CSV·모델 API (`JamesController` 등) |
| `doro` | 도로 데이터 API |
| `matrix` | `keymaker` (Gemini·env) |

새 기능이 titanic/doro에 속하면 해당 `app/` 패키지에 두고, `main.py`는 import·라우트만 추가한다.

---

## 7. 코드 스타일

- Python 3.10+ 타입 힌트 (`User | None` 등).
- import: 표준 라이브러리 → 서드파티 → 로컬 (`apps` 기준 패키지명).
- 요청 범위 밖 리팩터·포맷 일괄 변경 금지 (`backend/.cursorrules` §3).
- 테스트·스크립트는 요청 시에만 추가.

---

## 8. 체크리스트 (PR·커밋 전)

- [ ] 로직이 올바른 레이어에 있는가
- [ ] DB 쓰기 경로에 `commit`/`rollback`이 있는가
- [ ] `.env`·시크릿이 diff에 없는가
- [ ] `http://127.0.0.1:8000/docs`에서 엔드포인트가 기대대로 동작하는가

---

## 9. Cursor에 붙여넣을 프롬프트 (수동용)

```text
@backend/.cursorrules 와 @docs/DevOps/Backend/BACKEND_RULES.md 를 먼저 읽고 backend 코드를 작성하세요.
secom 레이어(Controller → Service → Repository)를 따르고, main.py에는 라우트와 트랜잭션만 두세요.
```

---

## 10. 참고 경로

| 항목 | 경로 |
|------|------|
| 에이전트 하네스 | `backend/.cursorrules` |
| 상세 Karpathy 지침 | `backend/CLAUDE.md` |
| Cursor 규칙 (자동) | `.cursor/rules/backend-coding.mdc` |
| 회원 API 예시 | `backend/apps/main.py`, `secom/app/` |
