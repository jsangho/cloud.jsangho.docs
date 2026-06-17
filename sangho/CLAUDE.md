# 백엔드 행동 지침 (sangho)

> **메인 규칙.** 충돌 시 이 문서가 `sangho/.cursorrules`보다 우선한다.  
> 루트 `CLAUDE.md`가 최상위다.

**스택:** FastAPI · Python 3.10+ · PostgreSQL (Neon) · SQLAlchemy 2.0 async · SQLModel · Alembic · Uvicorn

---

## 1. 구현 전 사고

- 새 엔드포인트 → `main.py` include 목록 · 기존 `*_router.py` prefix 먼저 확인
- 새 ORM 모델 → `_claude/ENTITY_RULE.md` 반드시 먼저 읽기
- 기존 패턴(titanic 앱 구조) 먼저 Read 후 동일 구조로 구현
- 불분명하면 멈추고 무엇이 헷갈리는지 구체적으로 질문

---

## 2. 헥사고날 레이어 규칙

```
domain/           ← 순수 파이썬. 프레임워크 import 절대 금지
app/              ← UseCase(Protocol) · DTO · Interactor
adapter/inbound/  ← FastAPI router · Pydantic schema
adapter/outbound/ ← SQLAlchemy PG repo · 외부 API 클라이언트
dependencies/     ← FastAPI Depends 팩토리
```

의존성 방향: `adapter → app → domain`. 역방향 즉시 거부.  
도메인 레이어가 FastAPI · SQLAlchemy 를 import 하면 구현 전에 멈추고 사용자에게 확인.

---

## 3. 경로 규칙

| 위치 | import 시작점 |
|------|--------------|
| `sangho/apps/<앱>/` 내부 | 앱명부터 (`from kayfabe.domain.entities...`) |
| core 패키지 | `jsangho.core.` 로 시작 (`from jsangho.core.matrix.grid_oracle_database_manager import get_db`) |

---

## 4. 네이밍 컨벤션

| 레이어 | 파일명 패턴 | 클래스/변수 |
|--------|------------|------------|
| Router | `{resource}_router.py` | 변수: `{resource}_router = APIRouter(prefix=...)` |
| Schema | `{resource}_schema.py` | `{Resource}Schema`, `{Resource}Response` |
| UseCase port | `{resource}_use_case.py` | `{Resource}UseCase` (Protocol) |
| Interactor | `{resource}_interactor.py` | `{Resource}Interactor` |
| Repository port | `{resource}_repository.py` | `{Resource}Repository` (Protocol) |
| PG Repository | `{resource}_pg_repository.py` | `{Resource}PgRepository` |
| Provider | `{resource}_provider.py` | `get_{resource}_use_case()`, `get_{resource}_repository()` |
| DTO | `{resource}_dto.py` | `{Resource}Dto` |
| ORM | `{resource}_orm.py` | `{Resource}Orm` |
| Mapper | `{resource}_mapper.py` | `{Resource}Mapper` |

---

## 5. DB 엔티티 규칙 (요약)

> 전체 규칙: `vault/sangho/ENTITY_RULE.md`

- PK: 반드시 `id: int`, auto-increment (UUID · 복합 PK 금지)
- SQLModel: `id: Optional[int] = Field(default=None, primary_key=True)`
- SQLAlchemy: `id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)`
- 비즈니스 식별자(email, login_id): `unique=True, index=True` 별도 컬럼 (PK 대체 불가)
- FK: `{참조테이블}_id: int` 형식

---

## 6. 앱 현황

| 앱 | 도메인 | API Prefix | 상태 |
|----|--------|-----------|------|
| `titanic` | ML 학습 실습 (crew/passenger) | `/titanic` | 안정 (구조 템플릿) |
| `kayfabe` | WWE 예측 · 랭킹 · 챔피언십 · 타이틀 히스토리 | `/ple`, `/rankings`, `/records`, `/championship`, `/title-history` | 운영 |
| `user` | 인증 · 프로필 | `/users` | 운영 |
| `imitation_game` | 학습용 | — | 최소 |
| `inception` | 학습용 | — | 최소 |
| `social_network` | 소셜 | — | 플레이스홀더 |

---

## 7. 검증 체크리스트

| 작업 | 검증 방법 |
|------|----------|
| 엔드포인트 추가 | `http://127.0.0.1:8000/docs` Swagger UI에서 확인 |
| DB 모델 변경 | `alembic revision --autogenerate -m "..."` → `alembic upgrade head` |
| 전체 동작 | `curl http://127.0.0.1:8000/{prefix}/{endpoint}` 200 응답 |
| Docker | `docker compose up --build -d backend` 정상 기동 후 로그 확인 |

---

## 8. 머신러닝 데이터 분석 원칙

### Categorical — 카테고리로 묶이는 데이터

| 척도 | 설명 | 예시 |
|------|------|------|
| **nominal** (명목) | 순서 없이 이름으로만 구분 | 청팀 · 홍팀 · 백팀 |
| **ordinal** (순서) | 자료 간 서열(순서)이 존재 | 이길 가능성: 1.매우낮음 ~ 5.매우높음 |

### Quantitative — 숫자로 셀 수 있는 데이터

| 척도 | 설명 | 예시 |
|------|------|------|
| **interval** (등간) | 기준점 없이 일정한 측정 구간 (배수 비교 불가) | 시간대, 온도, pH |
| **ratio** (비율) | 절대적 원점(0) 기준 — 배수 비교 가능 | 나이, 금액, 몸무게 |

> **선택 기준:** 배수 표현이 가능하면 ratio, 순서만 있으면 ordinal, 이름뿐이면 nominal.

---

## 9. async def vs def 선택 기준

메소드가 `await`할 대상이 있는지로만 판단한다.

| 성격 | 형태 | 예시 |
|------|------|------|
| I/O-bound (DB 쿼리, 네트워크, LLM 호출) | `async def` | `introduce_myself`, `chat` |
| CPU-bound (형태소 분석, 수치 연산) | `def` | `analyze_intent`, `train_model` |

**`async def`를 붙여도 CPU 작업은 비블로킹이 되지 않는다.** 코루틴이 될 뿐이며 이벤트 루프를 그대로 점유한다. `async` 표시가 붙어 있어서 비블로킹인 것처럼 보이는 게 더 위험하다.

Kiwi 등 CPU 작업이 실제로 무거워 이벤트 루프 블로킹이 문제가 되면, `async def`로 바꾸는 게 아니라 호출 측에서 스레드풀에 넘긴다.

```python
result = await asyncio.to_thread(use_case.analyze_intent, question)
```

---

## 관련 문서

| 문서 | 역할 |
|------|------|
| `CLAUDE.md` (루트) | 아키텍처 · 행동 규칙 (최상위) |
| `_claude/ENTITY_RULE.md` | DB 엔티티 필수 규칙 |
| `_claude/CLAUDE.md` | 백엔드 구현 세부 지침 |
| [`apps/kayfabe/_docs/CLAUDE.md`](apps/kayfabe/_docs/CLAUDE.md) | Kayfabe 도메인 · CQRS · ERD |
| [`apps/titanic/_docs/CLAUDE.md`](apps/titanic/_docs/CLAUDE.md) | Titanic 앱 구조 (템플릿) |
