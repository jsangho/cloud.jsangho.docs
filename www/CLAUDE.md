# 프론트엔드 행동 지침 (www)

> **메인 규칙.** 충돌 시 이 문서가 `www/.cursorrules`보다 우선한다.  
> 루트 `CLAUDE.md`가 최상위다.

**스택:** Next.js 16 (App Router) · React 19 · TypeScript 5.7 · Tailwind CSS 4 · Radix UI · shadcn/ui

---

## 1. 구현 전 사고

- 새 fetch 전 → `lib/api.ts` · `lib/*-api.ts` 기존 패턴 · 백엔드 prefix 먼저 확인
- 같은 도메인 컴포넌트 먼저 Read (예: PLE 추가 시 `components/ple-event-grid.tsx` 확인)
- `"use client"` 여부 · `NEXT_PUBLIC_*` env 사용은 기존 페이지와 맞춤
- 불분명하면 멈추고 질문

---

## 2. 컴포넌트 원칙

- **서버 컴포넌트 기본.** 인터랙션 · 훅이 필요할 때만 `"use client"` 추가
- 새 UI 원형(primitive) → `components/ui/` (Radix 기반 shadcn 패턴)
- 도메인 컴포넌트 → `components/{domain}-{feature}.tsx`
- 파일명: kebab-case (`ple-event-grid.tsx`)

---

## 3. 상태 관리

- 페이지 단위 상태: `useState<PageState>({...})` 단일 객체 + `patchState()` 헬퍼
- 폼 입력: `FormData` (비제어), 필드별 `useState` 남발 금지
- 전역 상태: Context API (서버→클라이언트 단방향만)
- useState 객체 압축 → `agent.md` (사용자 명시 요청 시만)

---

## 4. API 연동 패턴

```typescript
// lib/{domain}-api.ts 패턴
const BASE = process.env.NEXT_PUBLIC_API_BASE_URL // http://127.0.0.1:8000

export async function fetchXxx(): Promise<XxxType> {
  const res = await fetch(`${BASE}/{prefix}/{endpoint}`)
  if (!res.ok) throw new Error(await res.text())
  return res.json()
}
```

- 타임아웃: `AbortController` + `signal` 필요 시만 추가
- 에러: `throw new Error(...)` → 호출부에서 `try/catch`

---

## 5. 라우트 구조

| 경로 | 역할 | 백엔드 앱 |
|------|------|----------|
| `/` | PLE 이벤트 대시보드 | kayfabe |
| `/ple/[slug]` | 이벤트 상세 · 예측 | kayfabe |
| `/rankings` | 유저 예측 랭킹 | kayfabe |
| `/records/[name]` | 선수별 타이틀 히스토리 | kayfabe |
| `/results/[slug]` | 경기 결과 | kayfabe |
| `/championship` | 현재 챔피언 현황 | kayfabe |
| `/lesson/titanic` | Titanic ML 실습 | titanic |
| `/login` | 로그인 | user |
| `/my-info` | 내 정보 | user |

---

## 6. 검증 체크리스트

| 작업 | 검증 방법 |
|------|----------|
| API 연동 | 브라우저 Network 탭 200 응답 확인 |
| UI 변경 | 대상 라우트에서 실제 동작 확인 |
| 빌드 | `pnpm build` 에러 없음 |
| 타입 | `tsc --noEmit` 에러 없음 |

---

## 관련 문서

| 문서 | 역할 |
|------|------|
| `CLAUDE.md` (루트) | 아키텍처 · 행동 규칙 (최상위) |
| `sangho/CLAUDE.md` | 백엔드 API prefix · 엔드포인트 목록 |
| `sangho/apps/kayfabe/_docs/CLAUDE.md` | Kayfabe 전체 HTTP API 명세 |
| [`_claude/REACT_RULES.md`](_claude/REACT_RULES.md) | useState · FormData · 입력값 노출 금지 규칙 |
| `agent.md` | useState 객체 압축 패턴 |
