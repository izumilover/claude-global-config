---
name: react-security-checklist
description: "Next.js/React API 보안 체크리스트. OWASP 기반 취약점 점검, Auth.js 인증/인가 패턴, Server Action/Route Handler 보안 함정, 입력 검증, 보안 헤더, 세션 설정을 제공하는 react-backend-dev 확장 스킬. 'Auth.js 설정', 'Server Action 보안', 'Next.js 보안', 'OWASP', '인증 구현', '보안 헤더' 등 Next.js/React 백엔드 보안 설계·구현 시 사용한다. 단, 침투 테스트 수행이나 WAF 구성은 이 스킬의 범위가 아니다."
---

# React Security Checklist — Next.js/React API 보안 체크리스트

react-backend-dev 에이전트가 Auth.js 기반 인증/인가와 Server Action/Route Handler를 구현할 때 활용하는 실전 체크리스트. 이론이 아니라 실제로 겪는 함정 위주로 정리했다.

## 대상 에이전트

`react-backend-dev` — 이 스킬의 체크리스트와 설정 패턴을 Auth.js 설정, Server Action, Route Handler 구현에 직접 적용한다.

## OWASP API Security Top 10 대응 (Next.js 기준)

| 순위 | 취약점 | Next.js/React에서의 방어 |
|------|--------|-------------------|
| A1 | BOLA(객체 수준 인가 결함) | Service 레이어에서 리소스 소유자 검증(`resource.ownerId === session.user.id`) |
| A2 | 인증 결함 | `bcrypt`(cost 12+), 로그인 실패 N회 잠금(DB 필드로 관리) |
| A3 | 객체 속성 수준 인가 | Prisma 모델을 그대로 응답하지 말고 `select`로 필드 제한 또는 DTO 매핑 |
| A4 | 무제한 리소스 소비 | 목록 조회는 `take`/`cursor` 페이지네이션 강제, 필요 시 Rate Limiting(Upstash 등) |
| A5 | 기능 수준 인가 결함 | 모든 Server Action/Route Handler 내부에서 세션 role 체크 — 미들웨어만 믿지 않는다 |
| A6 | SSRF | 외부 URL 입력 화이트리스트, 내부 IP 대역 차단 |
| A7 | 보안 설정 오류 | prod 빌드에서 디버그 로그/소스맵 노출 비활성, `AUTH_SECRET` 필수 설정 확인 |
| A8 | 비즈니스 흐름 결함 | 상태 전이는 Service 함수로만 — Action에서 Prisma를 직접 호출해 필드를 임의로 세팅하지 않는다 |
| A9 | 취약 자산 관리 | 사용하지 않는 Route Handler/구버전 API 정리 |
| A10 | 안전하지 않은 API 소비 | 외부 API 응답도 Zod로 검증, 타임아웃 명시 |

## 인증(Authentication) 표준 패턴 — Auth.js Credentials + bcrypt

```typescript
// src/server/auth/auth.config.ts
import Credentials from "next-auth/providers/credentials";
import bcrypt from "bcryptjs";
import { prisma } from "@/server/db/prisma";

export const authConfig = {
  providers: [
    Credentials({
      credentials: { loginId: {}, password: {} },
      async authorize(credentials) {
        const user = await prisma.user.findUnique({ where: { loginId: credentials.loginId as string } });
        if (!user) return null;
        const valid = await bcrypt.compare(credentials.password as string, user.passwordHash);
        if (!valid) return null;
        return { id: user.id, loginId: user.loginId, role: user.role };
      },
    }),
  ],
  callbacks: {
    // Provider가 반환한 필드는 자동으로 세션에 실리지 않는다 — 명시적으로 매핑해야 한다
    async jwt({ token, user }) {
      if (user) token.role = user.role;
      return token;
    },
    async session({ session, token }) {
      if (session.user) session.user.role = token.role as string;
      return session;
    },
  },
  session: { strategy: "jwt" },
};
```

## 실전 함정 (검증된 이슈)

### 함정 1 — Server Action을 "URL을 모르면 안전한 엔드포인트"로 착각

**증상**: 폼 UI에서는 권한 있는 사용자만 버튼을 보게 했는데, 실제로는 다른 사용자가 액션을 호출해 데이터를 변경할 수 있음.

**원인**: Server Action은 빌드 시 자동으로 POST 엔드포인트가 생성된다. 클라이언트 번들에 액션의 참조 ID가 포함되므로, 브라우저 개발자 도구로 직접 호출이 가능하다. UI에서 숨겼다고 서버가 막아주지 않는다.

**해결**: 모든 Server Action의 **첫 줄에서** 세션을 조회하고 권한을 검증한다. 미들웨어의 라우트 보호와 Action 내부의 권한 체크는 서로 다른 계층이며, 둘 다 필요하다.

```typescript
"use server";
export async function deleteMember(id: string) {
  const session = await auth();
  if (!session || session.user.role !== "ADMIN") {
    return { success: false, error: "권한이 없습니다." };
  }
  // ...
}
```

### 함정 2 — Route Handler에는 Server Action 같은 자동 Origin 검사가 없다

**증상**: 외부 사이트에서 자체 `<form>`으로 우리 Route Handler에 상태 변경 요청을 보낼 수 있음(CSRF류).

**원인**: Next.js Server Action은 기본적으로 Origin 헤더를 검사하지만(동일 출처 요청만 허용), 직접 만든 Route Handler(`app/api/.../route.ts`)에는 이런 보호가 자동으로 붙지 않는다.

**해결**: 상태를 변경하는 Route Handler는 가능하면 Server Action으로 대체한다. 불가피하게 Route Handler를 써야 하면(webhook 등 외부 호출이 목적인 경우 제외) `Origin`/`Referer` 헤더를 직접 검증하거나 별도 CSRF 토큰을 발급/검증한다.

### 함정 3 — Middleware(Edge Runtime)에서 무거운 인증 로직을 시도

**증상**: `middleware.ts`에서 Prisma로 DB 세션을 조회하려다 빌드 에러 또는 런타임 에러.

**원인**: `middleware.ts`는 기본적으로 Edge Runtime에서 실행되어 Node.js 전용 API(Prisma 네이티브 드라이버, `bcryptjs` 등)를 쓸 수 없다.

**해결**: Middleware는 JWT 세션 쿠키 존재 여부 같은 가벼운 체크와 리다이렉트만 담당한다. DB 조회가 필요한 세부 인가는 Server Component/Server Action(Node.js 런타임)에서 수행한다.

### 함정 4 — `NEXT_PUBLIC_` 접두사 실수로 시크릿이 클라이언트 번들에 노출

**증상**: `AUTH_SECRET`이나 DB 접속 정보가 브라우저 개발자 도구 네트워크 탭/소스에서 그대로 보임.

**원인**: `NEXT_PUBLIC_`으로 시작하는 환경변수는 빌드 시 클라이언트 번들에 인라인된다. 시크릿에 실수로 이 접두사를 붙이면 누구나 볼 수 있다.

**해결**: 시크릿은 절대 `NEXT_PUBLIC_` 접두사를 쓰지 않는다. 배포 전 `grep NEXT_PUBLIC_ .env*`로 목록을 눈으로 재확인한다.

## 비밀번호/세션 정책

| 항목 | 권장 설정 |
|------|----------|
| 해시 | bcrypt cost 12+ |
| 세션 전략 | JWT(무상태, 기본) 또는 DB 세션(즉시 무효화가 필요하면) |
| 세션 만료 | `session.maxAge` — 요구사항에 맞게 조정(기본 30일은 과도한 경우가 많음) |
| 세션 쿠키 | Auth.js 기본값이 `httpOnly`/`sameSite: lax` — prod는 `secure` 자동 적용 확인 |
| 로그인 실패 잠금 | 5회 → 계정 잠금, 관리자 수동 해제 |

## HTTP 보안 헤더 (`next.config.js`)

```javascript
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: "/:path*",
        headers: [
          { key: "X-Frame-Options", value: "DENY" },
          { key: "X-Content-Type-Options", value: "nosniff" },
          { key: "Referrer-Policy", value: "no-referrer" },
          { key: "Content-Security-Policy", value: "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'" },
        ],
      },
    ];
  },
};
```

`style-src 'unsafe-inline'`은 Tailwind JIT 인라인 스타일 때문에 흔히 필요하지만, 가능하면 nonce 기반으로 좁히는 것을 검토한다.

## 입력 검증 체크리스트

| 항목 | 방법 |
|------|------|
| 타입/형식 검증 | Zod 스키마(`z.string().email()` 등) + Server Action/Route Handler 진입점에서 `parse` |
| SQL Injection | Prisma 쿼리 빌더 사용(파라미터 바인딩 기본 제공), `$queryRaw`는 태그드 템플릿만 |
| XSS | JSX 기본 이스케이프 유지, `dangerouslySetInnerHTML`은 DOMPurify 등으로 sanitize 후에만 |
| 파일 업로드 | 크기 제한 + 확장자 + MIME 타입 검증, 가능하면 매직넘버까지 확인 |

## 감사 로그(AuditLog) 설계 원칙

- 액터는 문자열(`actorLoginId`)로 보존 — 계정 삭제 후에도 추적 가능하게
- 최소 기록 항목: `action`, `actor`, `target`, `clientIp`, `userAgent`, `success`, `createdAt`
- 인증 이벤트(로그인 성공/실패/로그아웃/계정잠금)는 Auth.js `events` 콜백(`signIn`, `signOut` 등)에서 기록 — 컴포넌트/Action에 흩어놓지 않는다
