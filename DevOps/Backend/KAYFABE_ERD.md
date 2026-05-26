# Kayfabe ERD

WWE PLE(프리미엄 라이브 이벤트) 예측·결과를 Neon Postgres에 저장하는 Kayfabe 도메인 스키마입니다.  
ORM: `backend/apps/kayfabe/app/models/ple_model.py` · 회원 FK: `secom` `users` 테이블.

물리 PK·`id` 인조키 규칙은 [`ENTITY_RULE.md`](ENTITY_RULE.md)를 따릅니다. 본 문서의 **논리 ER**은 비즈니스 식별자·식별관계 기준이며, ORM은 조인 편의를 위해 모든 테이블에 `id`를 둡니다.

---

## PK · 인조키 정책

| 테이블 | 논리 PK (비즈니스) | 물리 PK (ORM·Neon) | `id` 인조키 |
|--------|-------------------|-------------------|-------------|
| `ple_events` | `slug` | `id` + `slug` UK | **불필요** (논리). 물리는 ENTITY_RULE로 `id` 유지 |
| `ple_matches` | `(event_id, match_key)` | `id` + UK `(event_id, match_key)` | **불필요** (논리). FK `event_id`가 부모 식별자 |
| `ple_match_pick` (교차) | `(match_id, client_id)` | 물리 테이블 `ple_predictions` · `id` + UK | 경기·브라우저 참가를 연결. `user_id`는 PK 밖 FK |
| `users` (Secom) | `id` | `id` | **필요** (외부 도메인) |

- **논리:** 자식 행은 부모 없이 존재할 수 없으면 PK에 부모 FK(또는 slug)가 포함되는 **식별관계**로 모델링한다.
- **물리:** [`ENTITY_RULE.md`](ENTITY_RULE.md)에 따라 **모든 테이블**에 `int id` autoincrement PK를 둔다. 비즈니스 중복 방지는 **UK**로 처리한다.

---

## 식별 · 비식별 관계

| 부모 | 자식 | 유형 | 설명 |
|------|------|------|------|
| `ple_events` | `ple_matches` | **식별** | 경기는 이벤트 없이 정의되지 않음. 논리 PK에 `event_id`(또는 `slug`) 포함 |
| `ple_matches` | `ple_match_pick` | **식별** | 픽(예측)은 경기 없이 없음. 논리 PK `(match_id, client_id)` |
| `users` | `ple_match_pick` | **비식별** | `user_id`는 PK에 **미포함**. **신규 예측 API는 `userId` 필수** · DB는 레거시 NULL 허용 · 회원 삭제 시 SET NULL |

| 부모 → 자식 | 유형 | Mermaid (중간) | 렌더 |
|-------------|------|----------------|------|
| `ple_events` → `ple_matches` | 식별 | `\|\|--o{` | **실선** · 자식 0건 허용 |
| `ple_matches` → `ple_match_pick` | 식별 | `\|\|--o{` | **실선** · 자식 0건 허용 |
| `users` → `ple_match_pick` | 비식별 | `\|\|..o{` + `..` | **점선** · 1:N (M:N 아님) |

```text
식별:     자식.PK ⊃ 부모 FK(또는 slug)   CASCADE              Mermaid 중간: --
비식별:   자식.FK only, PK 밖           users 삭제 → SET NULL   Mermaid 중간: ..
카디널리티: o{ = 0건 이상 (||--o{)  ·  |{ = 1건 이상 (||--|{, 본 스키마 미사용)
비식별 관계: `\|\|..o{` (중간 `..`) → 점선
```

> `users`와 `ple_match_pick`만 보면 M:N처럼 보일 수 있으나, **둘 사이에 직접 선을 두지 않고** 가운데 **교차 엔티티**를 두면 `경기 1:N 교차` + `회원 1:N 교차`로 해소됩니다.

---

## 교차 엔티티 (`ple_match_pick`)

`users` ↔ `ple_matches`는 **직접 연결하지 않습니다.**  
한 회원이 여러 경기에, 한 경기에 여러 회원이 참여하면 **논리적 M:N**이 되므로, 가운데 **교차(연관) 엔티티 `ple_match_pick`** 이 필요합니다.

| 구분 | 설명 |
|------|------|
| 논리 이름 | `ple_match_pick` — 「한 경기 + 한 참가(브라우저)」당 승자 예측 1건 |
| 물리 테이블 | **`ple_predictions`** (별도 junction 테이블 없음, 1테이블이 교차 엔티티 역할) |
| 경기 쪽 | `match_id` FK · UK `(match_id, client_id)` → 경기에 **식별** |
| 회원 쪽 | `user_id` FK → 회원에 **비식별** (PK 아님) · **저장·조회 upsert 키: `(match_id, user_id)`** |
| M:N 해소 | `ple_matches` **1:N** `ple_match_pick` **N:1** `users` (한 회원·여러 경기 픽) |

```text
ple_events ══1:N══► ple_matches ══1:N══► ple_match_pick ··· N:1 ··· users
           (실선)              (실선)         (교차·물리 ple_predictions)
                                              신규 예측: userId 필수 (API)
```

| 저장·조회 기준 | 키 | 용도 |
|----------------|-----|------|
| 논리 UK (브라우저) | `(match_id, client_id)` | 기기·세션 단위 중복 방지 |
| 논리 UK (회원) | `(match_id, user_id)` | 로그인 회원 경기당 1픽 · **upsert·`myPick` 조회** |
| API | `userId` + `clientId` | 예측 POST 시 `userId` **필수** (401) |

---

## ER 다이어그램

Neon·SQLAlchemy 실제 스키마(`ple_model.py`)와 **로그인 필수 예측** API 정책을 반영합니다.

> **Obsidian 편집 시 보기:** 소스 모드에서는 Mermaid가 코드만 보입니다. **라이브 프리뷰**(기본 편집기) 또는 아래 임베드로 렌더됩니다. `설정 → 편집기 → 기본 편집기`가 **라이브 프리뷰**인지 확인하세요 (`docs/.obsidian/app.json` → `legacyEditor: false`).

### 물리 `erDiagram` (임베드 · 편집 화면용)

![[KAYFABE_ERD-erDiagram]]

동일 다이어그램 소스: [[KAYFABE_ERD-erDiagram]]

### 1) 논리 — M:N 해소 (교차 엔티티)

`users`와 `ple_matches` 사이에 **직선을 두지 않습니다.** 가운데 `ple_match_pick`이 **유일한 연결 지점**입니다.

```mermaid
flowchart TB
    users[("users<br/>Secom 회원")]
    matches[("ple_matches<br/>경기")]
    pick[("ple_match_pick<br/>교차 엔티티<br/>물리: ple_predictions")]

    matches ==>|"1 : 0..N · 식별<br/>match_id"| pick
    users -.->|"1 : 0..N · 비식별<br/>user_id"| pick

    mnHint["users ↔ ple_matches<br/>직접 연결 없음 (M:N은 교차로만)"]
    mnHint -.-> users
    mnHint -.-> matches

    style pick fill:#1a3d2e,stroke:#4ade80,color:#e2e8f0
    style matches fill:#3b2f4a,stroke:#a78bfa,color:#e2e8f0
    style users fill:#44403c,stroke:#a8a29e,color:#e2e8f0
```

| 논리 관계 | 표기 | 설명 |
|-----------|------|------|
| `ple_matches` → `ple_match_pick` | **1 : 0..N 식별** | 경기당 0건 이상 픽 · `match_id`가 논리 UK 일부 |
| `users` → `ple_match_pick` | **1 : 0..N 비식별** | 회원당 0건 이상 픽 · `user_id`는 FK만 |
| `users` ↔ `ple_matches` | **(없음)** | M:N은 교차 엔티티로만 해소 |

### 2) 물리 — `erDiagram` (Neon 테이블)

| 논리 ER | 물리 테이블 | 논리 PK (UK) | 물리 PK |
|---------|-------------|---------------|---------|
| `ple_events` | `ple_events` | `slug` | `id` |
| `ple_matches` | `ple_matches` | `(event_id, match_key)` | `id` |
| **`ple_match_pick`** | **`ple_predictions`** | `(match_id, client_id)` | `id` |
| `users` (Secom) | `users` | `id` | `id` |

```mermaid
erDiagram
    ple_events ||--o{ ple_matches : event_id
    ple_matches ||--o{ ple_match_pick : match_id
    users ||..o{ ple_match_pick : user_id

    ple_events {
        bigint id PK
        string slug UK
        string label
        int month
        int year
        string status
        datetime finished_at
    }

    ple_matches {
        bigint id PK
        bigint event_id FK
        string match_key
        string title
        string format
        string status
        string winner_pick
        string ai_pick
        int point_value
        datetime finished_at
    }

    ple_match_pick {
        bigint id PK
        bigint match_id FK
        string client_id
        bigint user_id FK
        string pick
        datetime created_at
    }

    users {
        bigint id PK
        string login_id UK
        string nickname
        string email UK
    }
```

> 교차 엔티티 `ple_match_pick` = 물리 `ple_predictions`. 이벤트 단위 `status` / `finished_at`는 **`ple_events`만** 사용 (`ple_results` 없음). UK는 아래 **제약 조건** 참고.

**선·관계 읽는 법**

| 연결 | Mermaid | ER |
|------|---------|-----|
| `ple_events` → `ple_matches` | `\|\|--o{` + `--` | 1:0..N **식별** |
| `ple_matches` → `ple_match_pick` | `\|\|--o{` + `--` | 1:0..N **식별** |
| `users` → `ple_match_pick` | `\|\|..o{` + `..` | 1:0..N **비식별** (M:N 아님) |

**애플리케이션:** `POST …/predict` · `…/predictions/batch` → body `userId` **필수** · upsert·`myPick` → `(match_id, user_id)`.

**UK:** `ple_matches` · `uq_ple_event_match_key` → `(event_id, match_key)` · `ple_predictions` · `uq_ple_prediction_match_client` → `(match_id, client_id)` · `uq_predictions_match_user` → `(match_id, user_id)` (`user_id` **NOT NULL**일 때 경기당 회원 1건)

---

## 카디널리티

| 관계 | ER 표기 | 유형 | Mermaid | 의미 |
|------|---------|------|---------|------|
| `ple_events` → `ple_matches` | **1 : 0..N** | 식별 | `\|\|--o{` 실선 | 한 PLE에 0건 이상 경기 · sync 전 경기 없음 가능 |
| `ple_matches` → `ple_match_pick` | **1 : 0..N** | 식별 | `\|\|--o{` 실선 | 한 경기에 0건 이상 픽 · 예측 전 0건 가능 |
| (경기 + `client_id`) → 픽 | **1 : 1** | — | — | 브라우저당 경기 1회 (`uq_ple_prediction_match_client`) |
| (경기 + `user_id`) | **1 : 1** | — | — | 로그인·`user_id` NOT NULL 시 경기당 1회 (`uq_predictions_match_user`) |
| `users` → `ple_match_pick` | **1 : 0..N** | 비식별 | 점선 · `\|\|..o{` | 회원당 0건 이상 픽 · **신규 `userId` 필수** |
| `users` ↔ `ple_matches` | **없음** | — | — | **교차 엔티티**로만 연결 |

---

## 제약 조건

| 이름 | 테이블 | 규칙 |
|------|--------|------|
| `uq_ple_event_match_key` | `ple_matches` | `(event_id, match_key)` 유일 |
| `uq_ple_prediction_match_client` | `ple_predictions` | `(match_id, client_id)` 유일 |
| `uq_predictions_match_user` | `ple_predictions` | `(match_id, user_id)` 유일 (`user_id` NOT NULL인 행만 적용) |
| FK CASCADE | `ple_matches` → `ple_events` | 이벤트 삭제 시 경기·예측 연쇄 삭제 |
| FK CASCADE | `ple_predictions` → `ple_matches` | 경기 삭제 시 예측 연쇄 삭제 |
| FK SET NULL | `ple_predictions.user_id` → `users` | 회원 삭제 시 `user_id`만 NULL |
| API | predict · predict batch | body `userId` 필수 · 미제공·무효 id → **401** |

---

## 예측 흐름 (로그인 필수)

```mermaid
sequenceDiagram
    participant U as users
    participant F as Frontend
    participant API as FastAPI
    participant P as ple_match_pick

    U->>F: 로그인 (localStorage)
    F->>API: GET /ple/{slug}?user_id=
    API->>P: user_id로 myPick 조회
    F->>API: POST /ple/{slug}/predictions/batch (userId, clientId)
    API->>API: users.id 검증
    API->>P: upsert (match_id, user_id)
```

---

## 테이블 · 필드 설명

### `ple_events` (`PleEventModel`)

| 필드 | 타입 | 키 | 설명 |
|------|------|-----|------|
| id | bigint | PK (물리) | ENTITY_RULE 인조키 |
| slug | varchar(64) | UK · **논리 PK** | URL·프론트 식별자 |
| label | varchar(120) | | 표시 이름 |
| month | int | | PLE 월별 순서 |
| year | int | | 연도 |
| status | varchar(20) | | `upcoming` · `live` · `finished` |
| finished_at | timestamptz | | 이벤트 종료 시각 |
| created_at | timestamptz | | 생성 |
| updated_at | timestamptz | | 갱신 |

### `ple_matches` (`PleMatchModel`)

| 필드 | 타입 | 키 | 설명 |
|------|------|-----|------|
| id | bigint | PK (물리) | ENTITY_RULE 인조키 |
| event_id | bigint | FK · **논리 PK 일부** | `ple_events.id` |
| match_key | varchar(80) | **논리 PK 일부** | 프론트 카드 `id` |
| title | varchar(200) | | 경기 제목 |
| format | varchar(20) | | `singles` · `multi` |
| card_variant | varchar(10) | | `sideA` · `sideB` |
| sort_order | int | | 카드 정렬 |
| card_json | text | | 선수·배당 JSON |
| status | varchar(20) | | `scheduled` · `live` · `finished` |
| winner_pick | varchar(20) | | 방송 결과 |
| winner_name | varchar(200) | | 승자 표시명 |
| ai_pick | varchar(20) | | AI 예측 |
| ai_pick_name | varchar(200) | | AI 예측 표시명 |
| ai_correct | boolean | | AI 정답 여부 |
| point_value | int | | 순위 가중치 |
| finished_at | timestamptz | | 결과 확정 시각 |
| created_at | timestamptz | | 생성 |
| updated_at | timestamptz | | 갱신 |

### `ple_match_pick` · 물리 `ple_predictions` (`PlePredictionModel`)

논리 **교차 엔티티**. ORM·Neon 테이블명은 `ple_predictions`입니다.

| 필드 | 타입 | 키 | 설명 |
|------|------|-----|------|
| id | bigint | PK (물리) | ENTITY_RULE 인조키 |
| match_id | bigint | FK · **논리 PK 일부** | `ple_matches.id` |
| client_id | varchar(64) | **논리 PK 일부** · UK | 기기 식별 · API와 함께 전송 |
| user_id | bigint | FK (비식별) · UK `(match_id, user_id)` | `users.id` · **신규 예측 API 필수** · DB nullable(레거시) |
| pick | varchar(20) | | `left` · `right` · `"0"`… |
| created_at | timestamptz | | 예측 시각 |

### `users` (Secom, 참조)

| 필드 | 타입 | 키 | 설명 |
|------|------|-----|------|
| id | bigint | PK | 외부 도메인 인조키 |
| login_id | varchar | UK | 로그인 ID |
| nickname | varchar | | 닉네임 |
| email | varchar | UK | 이메일 |

---

## 상태 값

| 구분 | 값 | 용도 |
|------|-----|------|
| 이벤트 status | upcoming, live, finished | PLE 전체 |
| 경기 status | scheduled, live, finished | 개별 매치 |
| pick / winner_pick | left, right, 0..n | singles / multi |

---

## `card_json` 구조 (요약)

| format | 주요 키 |
|--------|---------|
| singles | `left`, `right`, `bookmakerDecimal` |
| multi | `competitors[]`, `bookmakerDecimal` |

---

## 레이어드 구조 (`backend/apps/kayfabe`)

| 레이어 | 경로 | 역할 |
|--------|------|------|
| Controller | `app/controllers/ple_controller.py` | API 진입 |
| Service | `app/services/ple_service.py` | 동기화·보드·예측 |
| Repository | `app/repositories/ple_repository.py` | Neon CRUD |
| Model | `app/models/ple_model.py` | ORM |
| Schema | `app/schemas/ple_schema.py` | DTO |

흐름: **Controller → Service → Repository → Neon**

---

## HTTP API (`main.py`)

| 메서드 | 경로 | DB 영향 |
|--------|------|---------|
| GET | `/ple/events` | `ple_events` |
| GET | `/ple/{slug}` | events + matches · `?user_id=` 시 회원 `myPick` |
| POST | `/ple/{slug}/sync-from-client` | events·matches upsert |
| POST | `/ple/{slug}/predictions/batch` | `ple_predictions` · **`userId` 필수** |
| POST | `/ple/{slug}/matches/{match_key}/predict` | `ple_predictions` · **`userId` 필수** |
| POST | `/ple/{slug}/matches/{match_key}/result` | `ple_matches` 결과 |
| GET | `/ple/{slug}/live` | SSE · `user_id` 쿼리 optional |
| POST | `/ple/link-predictions` | **410** (폐기) |
| GET | `/rankings` | `user_id` NOT NULL 픽만 집계 |

---

## 프론트 연동

| 화면 | 경로 | API |
|------|------|-----|
| PLE 목록·예측 | `/ple`, `/ple/[slug]` | sync, predict(batch), live · **로그인 후 예측** |
| 결과 등록 | `/results`, `/results/[slug]` | sync, result |

---

## 3NF · ER 체크리스트

- [ ] 논리 PK: `slug` · `(event_id, match_key)` · `(match_id, client_id)` — 교차 엔티티 `ple_match_pick`
- [ ] 물리: 모든 테이블 `id` PK · 교차 엔티티 = `ple_predictions` ([`ENTITY_RULE.md`](ENTITY_RULE.md))
- [ ] `ple_events` → `ple_matches` → `ple_match_pick` **식별관계**
- [ ] `users` → `ple_match_pick` **1:0..N 비식별** (`||..o{` 점선 · API `userId` 필수)
- [ ] `ple_events` → `ple_matches` · `ple_matches` → `ple_match_pick` **1:0..N 식별** (`||--o{` 실선)
- [ ] UK: `uq_ple_event_match_key`, `uq_ple_prediction_match_client`, `uq_predictions_match_user`
