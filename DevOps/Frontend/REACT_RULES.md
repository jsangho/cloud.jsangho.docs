# React 프론트엔드 규칙 (Cursor)

프론트엔드 React/Next.js 코드를 작성·리팩터링할 때 이 문서를 따른다.

**목표**

- `useState`를 남발하지 않고, 폼 입력은 DOM/`FormData`에 두며, UI 상태만 객체 하나로 묶는다.
- **사용자 입력값(PII·비밀번호 등)을 UI·로그·알림에 노출하지 않는다.**

---

## 1. 원칙

| 구분 | 권장 | 비권장 |
|------|------|--------|
| 폼 입력값 | `name` + uncontrolled input, 제출 시 `FormData` | 필드마다 `useState` |
| UI 상태 (모드, 로딩 등) | 관련 필드를 **하나의 `useState` 객체**로 묶기 | `useState` 5~10개 나열 |
| 부분 갱신 | `patchState({ key: value })` 패턴 | 필드마다 `setX` 함수 |
| 사용자 알림 | 고정 문구·인라인 UI (토스트 등) | `alert`에 폼 값·API 원문을 그대로 넣기 |

`useState`가 많을수록 리렌더 범위가 넓어지고, setter 이름·초기값이 흩어져 유지보수가 어렵다.  
**입력값은 state에 넣지 말고**, 화면 전환·제출 중 같은 **UI 전용 상태만** 객체로 압축한다.

---

## 2. 참조 패턴 (폼 제출)

```tsx
async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();

  const form = e.currentTarget;
  const formData = new FormData(form);
  const formProps = Object.fromEntries(formData.entries());

  const userId = String(formProps.userId ?? "").trim();
  const password = String(formProps.password ?? "");
  // 필요 시 nickname, email 등 동일하게 String(formProps.xxx ?? "")
}
```

- 각 `<input>`에 반드시 `name` 속성을 둔다.
- controlled로 바꿀 특별한 이유(실시간 검증 UI, 외부 state 동기화 등)가 없으면 `value` + `onChange` + `useState` per field를 쓰지 않는다.

---

## 3. 참조 패턴 (UI 상태 압축)

```tsx
type PageState = {
  mode: "login" | "signup";
  isSubmitting: boolean;
};

const initialState: PageState = {
  mode: "login",
  isSubmitting: false,
};

export default function LoginPage() {
  const [state, setState] = useState<PageState>(initialState);

  const patchState = (patch: Partial<PageState>) =>
    setState((prev) => ({ ...prev, ...patch }));

  // 사용 예: patchState({ isSubmitting: true });
  // 사용 예: patchState({ mode: "login" });
}
```

- 타입 이름은 `XxxPageState`, `initialState`처럼 컴포넌트 단위로 명확히 한다.
- 서로 무관한 도메인(예: 모달 열림 + 테이블 페이지네이션 + 폼 모드)은 **한 객체에 억지로 넣지 말고** 역할별로 나눈다. “압축”은 **같은 화면·같은 concern** 안에서만 적용한다.

---

## 4. 입력값 노출 금지

폼에서 읽은 값·API 응답·검증 오류를 **그대로** 사용자에게 보여주지 않는다. (스크린 공유·브라우저 히스토리·스크린샷 유출 방지)

### 금지 예

```tsx
// ❌ 폼 변수를 alert/화면에 삽입
alert(`가입 완료\n닉네임: ${nickname}\n이메일: ${email}`);

// ❌ 서버 detail/message를 검증 없이 노출 (입력값이 echo 될 수 있음)
alert(error.message);

// ❌ console에 비밀번호·토큰
console.log("signup", { password, userId });
```

### 허용 예

```tsx
// ✅ 고정 문구만
alert("회원가입이 완료되었습니다.");

// ✅ 필드 종류만 안내 (실제 값 없음)
alert("회원가입 입력란을 모두 작성해 주세요.");
alert("비밀번호와 비밀번호 확인이 일치하지 않습니다.");

// ✅ 타임아웃 등 입력과 무관한 시스템 메시지
alert(
  `서버 응답이 ${requestTimeoutMs / 1000}초 이상 걸려 요청을 중단했습니다.`
);

// ✅ API 실패 시 사전에 정의한 일반 메시지 (원문 미사용)
alert("회원가입에 실패했습니다.");
alert("로그인에 실패했습니다.");
```

### API 오류 처리

- `parseApiError`·`error.message`·`data.detail`을 **그대로 `alert`에 넣지 않는다.** (중복 ID, 이메일 형식 등에 사용자 입력이 포함될 수 있음)
- 디버깅은 개발 환경에서만 `console.error`로 하고, **민감 필드는 마스킹**한다.
- 성공 시에도 `data.message`에 사용자 식별 정보가 올 수 있으면 **고정 성공 문구**만 쓴다.

### 리팩터 절차

1. `alert(`, `confirm(`, `prompt(` 검색
2. 템플릿 리터럴·변수 삽입·`error.message` 직접 노출 여부 확인
3. 입력값·API 원문을 제거하고 §4 허용 패턴으로 교체 (또는 인라인 UI state로 전환)

---

## 5. 저장소 내 예시

로그인/회원가입: `frontend/app/login/page.tsx`  
- 폼: `FormData` + `Object.fromEntries`  
- UI: `LoginPageState` + `patchState`  
- 알림: 고정 문구만 (`nickname`/`email` 등 폼 변수 미삽입)

---

## 6. Cursor 에이전트 지시문 — useState·FormData (복사·@멘션용)

아래 블록은 사용자가 매번 타이핑하지 않아도 되도록, **이 파일을 @로 첨부하거나 Project Rules에 포함**할 때 쓰는 표준 프롬프트이다.

```text
React에서 useState는 많이 사용하면 안 됩니다.

다음 코드를 참조하여, 여러 개의 useState를 하나의 useState 객체로 압축하고,
폼 입력값은 useState 대신 FormData로 읽도록 변경해 주세요.

const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  const formData = new FormData(e.currentTarget);
  const formProps = Object.fromEntries(formData.entries());
};

추가 규칙:
- input에는 name 속성을 부여하고, 제출 핸들러에서 formProps로 값을 읽는다.
- UI 전용 상태(모드, isSubmitting 등)만 타입이 있는 state 객체 + patchState(Partial<T>)로 묶는다.
- 요청 범위 밖 리팩터링은 하지 않는다.
- 상세 규칙은 docs/DevOps/Frontend/REACT_RULES.md 를 따른다.
```

---

## 7. Cursor 에이전트 지시문 — 입력값 노출 제거 (복사·@멘션용)

아래 블록은 **“input 데이터가 노출되는 코드 제거”** 요청을 매번 치지 않아도 되도록 쓰는 표준 프롬프트이다.

```text
docs/DevOps/Frontend/REACT_RULES.md §4(입력값 노출 금지)를 따르세요.

프론트엔드에서 사용자 입력값(아이디, 이메일, 닉네임, 비밀번호 등)이
노출되는 코드를 모두 제거해 주세요.

대상 예:
- alert / confirm / prompt 에 폼 변수·API 응답 원문을 넣은 경우
- alert(`...${nickname}...`) 처럼 템플릿 리터럴로 입력값을 보여주는 경우
- catch 블록에서 error.message 를 그대로 alert 하는 경우 (서버가 입력값을 echo 할 수 있음)

교체 원칙:
- 성공·실패는 사전에 정의한 고정 한글 문구만 사용한다.
- 타임아웃(AbortError)처럼 입력과 무관한 메시지는 예외적으로 유지할 수 있다.
- 요청 범위 밖 리팩터링은 하지 않는다.
- 상세 규칙은 docs/DevOps/Frontend/REACT_RULES.md 를 따른다.
```

---

## 8. Cursor에서 이 규칙 켜기

1. **인덱스:** [`docs/README.md`](../../README.md) — 하네스 §0에서 `docs/` 필수 참조  
2. **채팅에서 @멘션:** `@docs/DevOps/Frontend/REACT_RULES.md`  
3. **백엔드 레일:** `backend/.cursorrules`가 구현 전 `docs/` 읽기를 강제함  
4. **프론트 작업 시:** `frontend/` 아래 TSX를 수정할 때 이 파일을 기본 컨텍스트로 포함

---

## 9. 체크리스트 (리뷰·리팩터 후)

**상태·폼**

- [ ] 텍스트 입력마다 `useState`가 있지 않은가?
- [ ] 제출 시 `FormData` / `formProps`로 값을 읽는가?
- [ ] 남은 `useState`는 UI concern끼리만 객체로 묶였는가?
- [ ] `patchState`로 부분 업데이트하는가?
- [ ] `name` 없는 폼 필드가 없는가?

**입력값 노출**

- [ ] `alert`/`confirm`에 폼 변수·`error.message`·`data.detail` 원문이 없는가?
- [ ] 성공 메시지에 닉네임·이메일·ID 등 PII가 포함되지 않는가?
- [ ] `console.log`에 비밀번호·토큰이 없는가?
