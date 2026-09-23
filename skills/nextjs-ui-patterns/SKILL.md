---
name: nextjs-ui-patterns
description: "Next.js App Router + Tailwind CSS + shadcn/ui UI 패턴 라이브러리. 레이아웃/라우트 그룹, 폼-Server Action 연동, Server/Client Component 경계, 데이터 페칭 전략을 제공하는 react-frontend-dev 확장 스킬. 'App Router 레이아웃', 'Server Component', 'Client Component', 'shadcn 패턴', '폼 검증 화면', 'Next.js 데이터 페칭' 등 Next.js/React 프론트엔드 설계 시 사용한다. 단, 백엔드 로직(Server Action 내부 구현)이나 Vue/Svelte 등 non-React 프론트엔드는 이 스킬의 범위가 아니다."
---

# Next.js UI Patterns — App Router + Tailwind + shadcn/ui 패턴

react-frontend-dev 에이전트가 Next.js App Router 기반 화면을 구현할 때 활용하는 레이아웃/폼/데이터 페칭 패턴 레퍼런스.

## 대상 에이전트

`react-frontend-dev` — 이 스킬의 패턴을 라우트 구조와 컴포넌트 구현에 직접 적용한다.

## 라우트 그룹 + 중첩 레이아웃 패턴

```
src/app/
├── (marketing)/
│   ├── layout.tsx        # 마케팅 페이지 공통 레이아웃 (헤더/푸터)
│   └── page.tsx
├── (dashboard)/
│   ├── layout.tsx        # 인증 가드 + 사이드바
│   └── members/page.tsx
└── layout.tsx             # 루트 레이아웃 (폰트, 전역 Provider)
```

- 괄호로 감싼 라우트 그룹(`(marketing)`, `(dashboard)`)은 URL 경로에 나타나지 않으면서 레이아웃만 분리할 때 쓴다
- 인증 가드는 `(dashboard)/layout.tsx`(Server Component)에서 세션을 조회해 없으면 `redirect("/login")` — 미들웨어와 이중 방어

```tsx
// src/app/(dashboard)/layout.tsx
import { redirect } from "next/navigation";
import { auth } from "@/server/auth";

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const session = await auth();
  if (!session) redirect("/login");
  return (
    <div className="flex">
      <Sidebar role={session.user.role} />
      <main className="flex-1 p-6">{children}</main>
    </div>
  );
}
```

## Server/Client Component 경계 규칙

1. **기본은 Server Component** — `"use client"` 선언이 없는 모든 컴포넌트는 서버에서 렌더링된다
2. **`"use client"`는 최소 단위에만** — 버튼 클릭, `useState`, 브라우저 API가 필요한 가장 작은 리프 컴포넌트에만 붙인다
3. **데이터는 최대한 서버에서** — Server Component에서 직접 `await`로 데이터를 가져와 props로 내려준다. 클라이언트에서 `useEffect`로 다시 fetch하지 않는다
4. **Client Component에 Server 전용 값을 props로 넘길 때** 직렬화 불가능한 값(함수, 클래스 인스턴스, Prisma 모델 원본)은 넘기지 않는다 — 필요한 필드만 plain object로 매핑

```tsx
// 나쁜 예 — 전체 페이지가 클라이언트로
"use client";
export default function MembersPage() { /* fetch까지 client에서 */ }

// 좋은 예 — 서버에서 fetch, 상호작용만 client 컴포넌트로 분리
export default async function MembersPage() {
  const members = await getMembers(); // Server Component에서 직접 조회
  return <MemberTable members={members} />; // MemberTable 내부의 정렬 버튼 등만 "use client"
}
```

## 폼 + 서버 검증 에러 표시 (`useActionState`)

```tsx
"use client";
import { useActionState } from "react";
import { createMember } from "@/server/actions/member/create-member";

const initialState = { success: false, error: "" };

export function MemberForm() {
  const [state, formAction, pending] = useActionState(createMember, initialState);

  return (
    <form action={formAction} className="space-y-4">
      <input name="loginId" required className="border rounded px-3 py-2" />
      {!state.success && state.error && (
        <p className="text-red-600 text-sm">{state.error}</p>
      )}
      <button type="submit" disabled={pending} className="bg-primary text-white rounded px-4 py-2">
        {pending ? "저장 중..." : "저장"}
      </button>
    </form>
  );
}
```

- `useActionState`는 서버가 반환한 상태(성공/실패, 필드별 에러)를 그대로 화면에 반영한다 — 별도 전역 상태 관리 없이 폼 상태를 다룰 수 있다
- 클라이언트측 즉시 검증(타이핑 중 피드백)이 필요하면 `react-hook-form` + `zodResolver`를 같이 쓰되, 서버 검증(backend-dev 담당)은 별개로 반드시 유지한다

## 데이터 페칭 전략

| 상황 | 방법 |
|------|------|
| 페이지 최초 로드 데이터 | Server Component에서 직접 `await fetch(...)` 또는 Prisma 조회 |
| 캐싱이 필요 없는 실시간 데이터 | `fetch(url, { cache: "no-store" })` |
| 주기적 갱신이면 충분한 데이터 | `fetch(url, { next: { revalidate: 60 } })` (ISR) |
| mutation 후 캐시 갱신 | Server Action 끝에 `revalidatePath`/`revalidateTag` |
| 클라이언트에서 폴링/무한 스크롤 등 지속 상호작용 | TanStack Query (`useQuery`/`useInfiniteQuery`) — 이 경우만 클라이언트 페칭을 정당화 |

## shadcn/ui 컴포넌트 재사용 전략

- `npx shadcn@latest add {component}`로 필요한 원자 컴포넌트만 `src/components/ui/`에 추가한다(전체 설치 지양 — 번들/유지보수 부담)
- 프로젝트 전용 조합 컴포넌트(`MemberTable`, `LoginForm` 등)는 `src/components/{feature}/`에 두고, `ui/` 원자 컴포넌트를 조합해서 만든다
- 디자인 토큰(색상/폰트/radius)은 `tailwind.config.ts`의 `theme.extend`에서 한 곳에 정의 — 컴포넌트마다 임의의 색상값을 흩뿌리지 않는다

## loading.tsx / error.tsx 컨벤션

- 데이터 페칭이 있는 라우트에는 `loading.tsx`(스켈레톤 UI)를 같은 폴더에 둔다 — Suspense 경계가 자동 적용된다
- 라우트별 에러 경계는 `error.tsx`(반드시 `"use client"`)로 두고, 전역 처리와 별개로 라우트 단위 복구(재시도 버튼)를 제공한다

## 폴더 구조 (검증된 관례)

```
src/app/
├── (marketing)/
├── (auth)/
│   └── login/
│       └── page.tsx
├── (dashboard)/
│   ├── layout.tsx
│   ├── loading.tsx
│   ├── error.tsx
│   └── {feature}/
│       └── page.tsx
└── layout.tsx

src/components/
├── ui/                # shadcn/ui 원자 컴포넌트
└── {feature}/          # 기능별 조합 컴포넌트 (일부만 "use client")
```

## 접근성(a11y) 체크리스트

- [ ] 모든 `<img>`/`next/image`에 `alt`
- [ ] `<label htmlFor="...">` ↔ `<input id="...">` 연결
- [ ] 폼 제출은 `<button type="submit">` (div/span에 클릭 이벤트로 대체 금지)
- [ ] 색상만으로 상태를 전달하지 않기 (아이콘/텍스트 병행)
- [ ] 키보드만으로 전체 플로우 완주 가능한지 확인(특히 커스텀 드롭다운/모달)
