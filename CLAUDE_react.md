# Next.js/React 하네스 안내

이 파일은 Next.js/React 프로젝트에서 작업할 때 참고하는 보조 지침이다. 전역 `CLAUDE.md`
규칙과 함께 적용된다. 프로젝트가 Next.js/React(`package.json`에 `next` 의존성이 있음)로
판단되면 이 파일을 먼저 읽고 아래 내용을 따른다.

## 이게 뭔가

여러 Next.js/React 프로젝트에서 재사용하는 개인용 하네스(에이전트 팀 + 스킬)다. 특정 프로젝트에
종속되지 않고 `~/.claude/agents/`, `~/.claude/skills/`에 상주하며 모든 프로젝트에서 자동으로
사용 가능하다. Java/Spring 하네스(`CLAUDE_spring.md`)와 동일한 구조(아키텍트
+ 백엔드 + 프론트엔드 + QA + DevOps 5역할, 오케스트레이터 스킬 1개 + 확장 스킬 2개)를 Next.js
App Router 스택으로 재구성했다. 신규 프로젝트를 팀 개발로 시작할 때 이 하네스를 먼저 셋업하고
(스캐폴딩 포함) 실제 기능 개발에 들어가는 것을 전제로 한다.

## 트리거 방법

"Next.js 프로젝트 시작해줘", "React 웹앱 만들어줘", "App Router 프로젝트", "Next.js 팀 개발" 등으로
요청하면 `react-webapp` 스킬이 발동한다. 명시적으로 스킬을 부를 수도 있다.

## 기본 기술 스택 가정

이 하네스의 모든 에이전트/스킬은 아래 스택을 기본값으로 가정하고 작성되었다. 프로젝트가 다른
스택(Vue, 별도 상태관리 라이브러리 등)을 쓴다면 해당 부분만 에이전트 파일을 프로젝트 로컬로
오버라이드하거나 사용자에게 확인 후 조정한다.

| 구분 | 기본값 |
|------|--------|
| 프레임워크 | Next.js App Router + TypeScript strict |
| 스타일링/UI | Tailwind CSS + shadcn/ui |
| DB/ORM | PostgreSQL + Prisma (풀스택 성향일 때) |
| 인증 | Auth.js(NextAuth v5) Credentials + bcrypt |
| 폼/검증 | react-hook-form + Zod |
| 테스트 | Vitest + React Testing Library + Playwright |
| 배포 | Vercel(기본) / Docker(자체호스팅) |

## 프로젝트 성향 — 매번 먼저 결정

Spring 하네스와 달리 이 하네스는 시작 시 **프로젝트 성향**을 먼저 정한다. 성향에 따라 투입
에이전트와 스캐폴딩 내용(Prisma/Auth.js 설치 여부)이 달라진다 — 자세한 기준표는
`~/.claude/agents/react-architect.md`의 "프로젝트 성향별 구성" 참고.

- **풀스택(자체 DB)**: 이 Next.js 프로젝트 안에서 DB/인증까지 전부 처리
- **BFF·프론트엔드 전용**: 이미 있는 백엔드 API(예: Spring)를 소비만 함
- **정적·마케팅 사이트**: 백엔드 로직 없음

## 에이전트 (`~/.claude/agents/`)

| 에이전트 | 역할 |
|---------|------|
| `react-architect` | 요구사항 분석, 프로젝트 성향 결정, App Router 아키텍처, DB 모델링(Prisma), API 설계(Server Action/Route Handler) — 3가지 설계안(Option A/B/C) 비교까지 산출 |
| `react-backend-dev` | Server Actions/Route Handlers 구현, Prisma+Auth.js 연동, 비즈니스 로직 (BFF 성향이면 외부 API 클라이언트로 역할 축소) |
| `react-frontend-dev` | App Router + Tailwind + shadcn/ui 화면, Server/Client Component 경계 설계, 폼 연동 |
| `react-qa-engineer` | 단위/컴포넌트 테스트 + Playwright 기반 L1 실행 검증 + Design-vs-구현 Match Rate 산출 |
| `react-devops-engineer` | Docker Compose(풀스택), GitHub Actions CI, 빌드 게이트, Vercel/자체호스팅 배포 |

## 스킬 (`~/.claude/skills/`)

| 스킬 | 역할 |
|------|------|
| `react-webapp` | 오케스트레이터 — 5개 에이전트를 Phase 1(준비+**실제 스캐폴딩**)→2(설계+병렬 구현)→3(통합)으로 조율. 신규 프로젝트는 이 스킬의 Phase 1이 `create-next-app` 등 실제 프로젝트 보일러플레이트를 만든다. 작업 규모별 모드(풀 파이프라인/백엔드만/프론트만/리팩토링/솔로) 선택, 5단계 사용자 승인 체크포인트, 경량 품질 게이트 포함 |
| `react-security-checklist` | react-backend-dev 확장 — Auth.js 인증/인가 패턴, OWASP 대응, Server Action/Route Handler 실전 보안 함정 |
| `nextjs-ui-patterns` | react-frontend-dev 확장 — App Router 레이아웃/폼/데이터 페칭 패턴, shadcn/ui 컴포넌트 재사용 전략 |

상세 워크플로우·체크포인트 게이트·품질 게이트 표는 `~/.claude/skills/react-webapp/SKILL.md`
본문 참고 — 이 파일에서 중복 설명하지 않는다.

## 신규 프로젝트 시작 순서 (보일러플레이트 먼저)

1. Agent Teams가 꺼져 있으면 켠다 (전역 `CLAUDE.md` "신규 프로젝트는 하네스 먼저 켜고 시작" 규칙)
2. `react-webapp` 스킬 트리거 → Phase 1에서 프로젝트 성향 확인(체크포인트 0) → 승인 시 실제
   `create-next-app` 스캐폴딩 실행
3. 스캐폴딩 완료 후 바로 이어서 요구사항 정리 → 설계 → 구현 단계로 진행(같은 세션에서 자연스럽게
   이어진다, 스캐폴딩만 하고 끝내지 않는다)
4. 팀 개발이 확실하면(사용자가 이미 "팀 개발일 것 같다"고 언급하는 등) Phase 2를 솔로 모드로
   축소하지 않는다 — 전역 규칙의 "혼자/팀 모드 자율 판단" 기준보다 사용자가 명시한 팀 개발 의도를
   우선한다

## bkit과 함께 있는 프로젝트라면

같은 feature에 bkit PDCA(`docs/01-plan/` 등)와 이 하네스(`_workspace/`)를 동시에 쓰지 않는다 —
둘 다 유사한 오케스트레이션이라 산출물이 겹친다.

어느 쪽을 쓸지는 프로젝트마다 한 번만 정하고 그 프로젝트의 (project-local) `CLAUDE.md`에
`## 오케스트레이션 체계`로 기록해둔다(전역 `CLAUDE.md`의 "신규 프로젝트는 오케스트레이션 체계를
선택해 프로젝트 CLAUDE.md에 고정한다" 규칙 참고). 새 Next.js/React 프로젝트를 시작할 때 이미
기록이 있으면 그대로 따르고, 없으면 시작 전에 사용자에게 bkit PDCA 방식인지 이 하네스 방식인지
물어서 기록한다.

## 이 파일을 갱신할 때

에이전트/스킬 파일 자체를 고치면(`~/.claude/agents/react-*.md`, `~/.claude/skills/react-*`,
`~/.claude/skills/nextjs-ui-patterns`), 이 파일의 표도 같이 최신화한다 — 실제 파일 내용과 이
개요 문서가 어긋나지 않게 유지한다.
