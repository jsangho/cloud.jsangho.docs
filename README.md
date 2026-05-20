# 코딩 규칙 인덱스 (`docs/`)

Cursor·에이전트는 **코드를 작성·수정하기 전에** 이 디렉터리에서 작업 영역에 맞는 규칙 문서를 읽는다.  
규칙은 여기(`docs/`)에만 두고, `.cursorrules`에는 **읽기 의무와 우선순위**만 둔다.

---

## 규칙 문서 목록

| 영역 | 경로 | 적용 범위 |
|------|------|-----------|
| Frontend (React/Next.js) | [`DevOps/Frontend/REACT_RULES.md`](DevOps/Frontend/REACT_RULES.md) | `frontend/` 하위 TSX·TS |
| Backend (FastAPI/Python) | [`DevOps/Backend/BACKEND_RULES.md`](DevOps/Backend/BACKEND_RULES.md) | `backend/` 하위 Python (레이어·API) |
| Backend (엔티티·PK) | [`DevOps/Backend/ENTITY_RULE.md`](DevOps/Backend/ENTITY_RULE.md) | `backend/**/models/`, 마이그레이션 |

---

## 에이전트 절차 (필수)

1. 작업 경로가 `frontend/`인지 `backend/`인지 판별한다.
2. 위 표에서 해당 **규칙 MD를 Read**한다. (채팅에서는 `@docs/DevOps/...` 멘션 가능)
3. 규칙에 맞춰 구현한다. 규칙에 없는 패턴은 임의로 도입하지 않는다.
4. 해당 영역 규칙 파일이 없거나 요구사항과 충돌하면, 구현 전에 사용자에게 확인한다.

---

## 하네스와의 관계

| 층 | 위치 | 역할 |
|----|------|------|
| 레일 | `backend/.cursorrules` (또는 향후 `frontend/.cursorrules`) | 매 세션 자동 적용, **docs/ 읽기 의무** |
| 업무 규칙 | **`docs/`** (본 디렉터리) | 스택·패턴별 상세 코딩 규칙 |
| 원칙서 | `backend/CLAUDE.md` | Karpathy 일반 행동 지침 |
| IDE 지도 | `backend/CURSOR.md` | Cursor 사용법·우선순위 |

**충돌 시 우선순위:** 사용자 명시 지침 > `docs/` 업무 규칙 > `.cursorrules` > `CLAUDE.md`
