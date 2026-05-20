# Backend 코딩 규칙 (Cursor)

`backend/` 아래 FastAPI·SQLAlchemy 코드를 작성·리팩터링할 때 이 문서를 따른다.

---

## 1. 레이어 구조

모듈별로 `apps/<모듈>/app/` 아래 계층을 유지한다.

| 계층 | 역할 | 예시 |
|------|------|------|
| `controllers/` | HTTP 진입·요청/응답 조립, Service 호출 | `UserController` |
| `services/` | 비즈니스 로직, 트랜잭션 단위 오케스트레이션 | `UserService` |
| `repositories/` | DB 접근·쿼리 | `UserRepository` |
| `models/` | SQLAlchemy/SQLModel ORM 모델 (`id` PK 규칙은 [`ENTITY_RULE.md`](ENTITY_RULE.md)) | `UserModel` |
| `schemas/` | Pydantic 요청/응답·검증 | `UserSchema` |

- **Controller**에 DB 쿼리·비밀번호 해시 등 도메인 로직을 넣지 않는다.
- **Service**가 Repository를 호출한다. Controller가 Repository를 직접 부르지 않는다.
- 라우트 등록·미들웨어는 `apps/main.py`에서 통합한다.

---

## 2. 의존성·세션

- DB 세션은 `AsyncSession`을 생성자/함수 인자로 주입한다.
- 모듈 간 import는 기존 패턴을 따른다 (예: `from database import ...`, `from secom.app...`).

---

## 3. 로깅

- 계층 경계에서 `LAYER_LOG`로 진입/퇴장을 남긴다 (기존 `UserController` 스타일).
- 비밀번호·토큰 원문은 로그에 남기지 않는다.

---

## 4. API·에러

- 요청/응답 body는 Pydantic `schemas/`로 정의한다.
- HTTP 예외는 FastAPI 관례(`HTTPException` 등)를 따르고, 메시지는 클라이언트가 이해할 수 있게 쓴다.
- 요청 범위 밖 엔드포인트·미들웨어를 추가하지 않는다.

---

## 5. 변경 범위

- 요청과 무관한 리팩터링·포맷 일괄 수정을 하지 않는다.
- 새 기능은 기존 모듈(`secom`, `kayfabe` 등) 구조를 복제해 확장한다.

---

## 6. 저장소 내 예시

- 사용자·인증: `backend/apps/secom/app/`
- PLE: `backend/apps/kayfabe/app/`
- 앱 진입점: `backend/apps/main.py`

---

## 7. Cursor 에이전트 지시문

```text
backend/ 코드를 수정하기 전에 docs/DevOps/Backend/BACKEND_RULES.md 를 읽고,
Controller → Service → Repository 레이어를 지켜 구현해 주세요.
models/·테이블 추가 시 docs/DevOps/Backend/ENTITY_RULE.md (int PK id) 도 따릅니다.
docs/README.md 인덱스도 확인하세요.
```
