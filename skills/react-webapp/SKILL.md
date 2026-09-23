---
name: react-webapp
description: "Next.js/React 웹앱의 요구사항 분석, 초기 스캐폴딩, 설계, 백엔드, 프론트엔드, 테스트, 배포를 에이전트 팀이 협업하여 개발하는 풀 개발 파이프라인. 신규 프로젝트는 Phase 1에서 실제 create-next-app 스캐폴딩까지 수행한다(프로젝트 보일러플레이트). 'Next.js 프로젝트 시작해줘', 'React 웹앱 만들어줘', 'Next.js 팀 개발', 'App Router 프로젝트', 'React 프론트엔드 개발' 등 Next.js/React 웹 애플리케이션 개발 전반에 이 스킬을 사용한다. 기존 Next.js/React 프로젝트가 있는 경우에도 기능 추가나 리팩토링을 지원한다. 단, Node 백엔드만 있고 프론트가 없는 순수 API 서버, Vue/Svelte 등 non-React 프론트엔드, 모바일 앱, 게임 개발은 이 스킬의 범위가 아니다."
---

# React Webapp — Next.js/React 웹앱 개발 파이프라인

Next.js/React 웹앱의 요구사항→스캐폴딩→설계→백엔드→프론트엔드→테스트→배포를 에이전트 팀이 협업하여 개발한다.

## 실행 모드

**에이전트 팀** — 최대 5명이 SendMessage로 직접 통신하며 교차 검증한다. Agent Teams(`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`)가 꺼져 있으면 먼저 켤 것을 사용자에게 안내한다.

## 상태 추적 & 체크포인트 (경량, 외부 도구 무의존)

무거운 상태관리 시스템(별도 MCP 서버, 커스텀 훅, 자동화 레벨 다이얼 등) 없이 **파일 + git만으로** 충분한 수준의 추적성을 확보한다.

- **상태 파일** — `_workspace/.status.json`:
  ```json
  { "feature": "member-dashboard", "orientation": "fullstack", "collaboration": "team", "branch": "feature/member-dashboard", "phase": "do", "matchRate": null, "iterationCount": 0, "updatedAt": "2026-01-01T00:00:00Z" }
  ```
  각 Phase 전환 시 오케스트레이터가 직접 갱신한다. `phase`는 `plan → design → do → check → report` 중 하나. `orientation`은 `fullstack`/`bff-frontend`/`static` 중 하나. `collaboration`은 `solo`/`team` 중 하나, `branch`는 팀 모드에서만 채운다(솔로는 `null`).
- **체크포인트 = phase 경계마다 git commit** — 별도 스냅샷 포맷을 만들지 않는다. `git commit -m "[{feature}] plan → design"`처럼 phase 전환 시점마다 커밋하면, 되돌리기는 `git revert`/`git reset`으로 git이 이미 가장 잘 하는 일을 그대로 쓰는 것이다. 커밋 전에는 반드시 `git status`/`git diff`로 실제 변경분을 확인한다. 팀 모드에서는 현재 브랜치가 아니라 Phase 1에서 만든 feature 브랜치에 커밋한다 — 자세한 건 "협업 방식: 솔로 vs 팀" 참고.
- **Task 도구 활용** — 여러 단계짜리 작업은 TodoWrite/Task로 진행 상황을 표시한다. 별도 Task 파일 포맷을 새로 만들지 않는다.

## 체크포인트 게이트 (사용자 승인 지점)

아래 5개 지점에서 AskUserQuestion으로 명시적 승인을 받는다 — 조용히 다음 단계로 넘어가지 않는다.

| # | 시점 | 확인 내용 |
|---|------|----------|
| 0 | 신규 프로젝트 스캐폴딩 직전 | "프로젝트 성향(풀스택/BFF·프론트전용/정적사이트)과 협업 방식(솔로/팀)이 맞나요? 이 구성으로 `create-next-app` 스캐폴딩을 시작해도 될까요?" |
| 1 | 요구사항 정리 직후 | "이해가 맞나요? 빠진 게 없나요?" |
| 2 | architect의 3가지 설계안 제시 후 | "Option A/B/C 중 어떤 걸 선택하시겠습니까?" (추천안 명시) |
| 3 | 구현 착수 직전 | "이 범위(파일 N개, 예상 규모)로 시작해도 되겠습니까?" |
| 4 | qa-engineer 리뷰 후 🔴/🟡 발견 시 | "지금 모두 수정 / Critical만 수정 / 그대로 진행" 중 선택 |

기존 프로젝트에 이어서 작업하는 경우 체크포인트 0은 생략한다(스캐폴딩이 필요 없으므로).

**팀 모드 주의**: 위 체크포인트들은 지금 세션을 운영하는 한 사람의 "구현 착수 허가"일 뿐, 팀
전체의 최종 합의가 아니다. 실제 팀 합의는 Phase 3에서 여는 PR 리뷰를 통해 받는다("협업 방식:
솔로 vs 팀" 참고).

## 품질 게이트 (경량 지표)

qa-engineer가 Phase 3 직전에 아래 표를 채운다. 자동 수집 도구 없이 **직접 코드/문서를 대조해서 산출**한다.

| 지표 | 산출 방법 | 목표 |
|------|----------|------|
| Match Rate | Design §API명세/§DB스키마의 항목 수 대비, 실제 Server Action/Route Handler/Prisma 모델에 구현된 항목 수 비율 | ≥ 90% |
| Critical Issue Count | 코드 리뷰에서 발견한 🔴(보안/기능 결함) 개수 | 0 |
| Convention Compliance | 코드 품질 기준 체크리스트(각 에이전트 파일 §코드 품질 기준) 통과 항목 비율 | ≥ 90% |
| L1 실행 검증 통과율 | react-qa-engineer.md의 L1 표준 시나리오 중 실제 Playwright로 통과한 비율 | 100% (핵심 시나리오는 생략 불가) |

Match Rate가 90% 미만이면 report로 넘어가지 않고 gap 목록을 backend-dev/frontend-dev에게 돌려보낸다(최대 2회 반복, 그 이상은 사용자에게 보고).

## 에이전트 구성

| 에이전트 | 파일 | 역할 |
|---------|------|------|
| react-architect | `~/.claude/agents/react-architect.md` | 요구사항, 프로젝트 성향 결정, App Router 아키텍처, DB(Prisma), API(Server Action/Route Handler) 설계 |
| react-backend-dev | `~/.claude/agents/react-backend-dev.md` | Server Actions/Route Handlers, Prisma, Auth.js, 비즈니스 로직 (BFF 성향이면 외부 API 클라이언트로 역할 축소) |
| react-frontend-dev | `~/.claude/agents/react-frontend-dev.md` | App Router + Tailwind + shadcn/ui 화면, Server/Client 경계 |
| react-qa-engineer | `~/.claude/agents/react-qa-engineer.md` | 단위/컴포넌트 테스트 + Playwright 기반 L1 실행 검증 |
| react-devops-engineer | `~/.claude/agents/react-devops-engineer.md` | Docker Compose(풀스택), CI, 빌드 게이트, Vercel/자체호스팅 배포 |

## 워크플로우

### Phase 1: 준비 + 스캐폴딩 (오케스트레이터 직접 수행)

1. **팀 규칙부터 확인한다** — 프로젝트 로컬 `CLAUDE.md`, `CONTRIBUTING.md`, `.editorconfig`,
   기존 커밋의 실제 컨벤션(주석 언어, 커밋 메시지 포맷 등)을 훑어본다. 있으면 이 하네스/전역
   `CLAUDE.md`의 개인 규칙보다 팀 규칙을 우선한다(전역 `CLAUDE.md` "팀 프로젝트에서는 팀 규칙이
   최우선" 규칙).
2. 사용자 입력에서 추출한다:
   - **앱 설명**: 만들려는 웹앱의 목적과 핵심 기능
   - **프로젝트 성향**: 풀스택(자체 DB) / BFF·프론트엔드 전용(외부 API 소비) / 정적·마케팅 사이트 — 모호하면 사용자에게 확인
   - **협업 방식**: 솔로 / 팀(여러 명이 같은 저장소에서 git으로 협업) — 사용자가 "팀 개발"을
     언급했거나 저장소에 이미 다른 작성자의 커밋이 있으면 팀으로 판단, 애매하면 확인
   - **규모** (선택): MVP/소규모/중규모/대규모
   - **기존 코드** (선택): 확장할 기존 Next.js/React 프로젝트
   - **배포 대상** (선택): Vercel(기본) / 자체호스팅(Docker)
3. **신규 프로젝트라면 스캐폴딩을 먼저 실행한다** — 이 저장소의 다른 하네스와 달리 react-webapp은 Phase 1에서 실제 프로젝트 뼈대를 만든다(체크포인트 0 승인 후):
   ```bash
   npx create-next-app@latest {project} --typescript --tailwind --app --eslint --src-dir --import-alias "@/*"
   cd {project}
   npx shadcn@latest init -d
   npm install zod react-hook-form @hookform/resolvers

   # 풀스택 성향일 때만:
   npm install prisma @prisma/client next-auth@beta bcryptjs
   npx prisma init --datasource-provider postgresql

   # 테스트 도구 (모든 성향 공통):
   npm install -D vitest @vitejs/plugin-react @testing-library/react @testing-library/jest-dom jsdom @playwright/test
   npx playwright install --with-deps chromium

   git init && git add -A && git commit -m "chore: react-webapp 스캐폴딩"
   ```
   정적·마케팅 사이트 성향이면 Prisma/Auth.js/bcryptjs 설치를 생략한다. BFF·프론트엔드 전용 성향이면 Prisma/bcryptjs는 생략하고 `next-auth`는 세션 프론트만 필요하면 유지한다.
4. **협업 방식이 팀이면, 스캐폴딩 직후 하네스를 프로젝트 저장소에 포함시키고 feature 브랜치를
   만든다** — 절차는 "협업 방식: 솔로 vs 팀" 참고. 솔로면 건너뛴다.
5. `_workspace/` 디렉토리를 프로젝트 루트에 생성한다
6. 입력(성향·협업 방식 포함)을 정리하여 `_workspace/00_input.md`에 저장한다
7. 기존 코드가 있으면 분석하고(라우트 구조, 컴포넌트 컨벤션) 해당 단계를 조정하며 스캐폴딩 단계는 건너뛴다
8. 요청 범위에 따라 **실행 모드를 결정**한다 (아래 "작업 규모별 모드" 참조)

### Phase 2: 팀 구성 및 실행

| 순서 | 작업 | 담당 | 의존 | 산출물 |
|------|------|------|------|--------|
| 1 | 아키텍처 설계 | react-architect | 없음 | `01_architecture.md`, `02_api_spec.md`, `03_db_schema.md`(풀스택만) |
| 2a | 백엔드/서버 레이어 개발 | react-backend-dev | 작업 1 | Server Actions/Route Handlers, Prisma 스키마, Auth.js 설정 |
| 2b | 프론트엔드 개발 | react-frontend-dev | 작업 1, (2a의 Action 시그니처) | App Router 라우트/컴포넌트, 폼 |
| 2c | 배포 설정 | react-devops-engineer | 작업 1 | `05_deploy_guide.md`, docker-compose(풀스택), CI 설정 |
| 3 | 테스트 & 실행검증 | react-qa-engineer | 작업 2a, 2b | `04_test_plan.md`, `06_review_report.md`, 테스트 코드 + L1 Playwright 검증 |

작업 2a(백엔드)와 2c(DevOps)는 설계에만 의존하므로 병렬 가능. 2b(프론트)는 2a가 Server Action 시그니처를 어느 정도 확정해야 효율적이므로, 무리한 병렬화보다 정확성 우선(2a 착수 후 바로 이어 붙는 편이 안전).

**팀원 간 소통 흐름:**
- architect 완료 → backend에게 API·DB·인증 전달, frontend에게 라우트 구조·권한 규칙 전달, devops에게 인프라 요구사항 전달, qa에게 기능 요구사항 전달
- backend ↔ frontend: Server Action 시그니처 연동 중 실시간 소통 (변경, 에러 형식 등)
- devops 완료 → 전체에게 환경변수, 배포 절차 공유
- qa는 모든 코드를 리뷰 + **반드시 실제로 앱을 기동해서 Playwright로 L1 검증**한다. 🔴 필수 수정 발견 시 해당 개발자에게 수정 요청 → 재작업 → 재검증 (최대 2회)

### Phase 3: 통합 및 최종 산출물

1. 모든 코드와 문서를 확인한다
2. 리뷰의 🔴 필수 수정이 모두 반영되었는지, L1 실행 검증이 실제로 통과했는지 확인한다 (코드만 보고 "완료"라고 하지 않는다)
3. 최종 요약을 사용자에게 보고한다:
   - 아키텍처 — `01_architecture.md` / API 명세 — `02_api_spec.md` / DB 스키마 — `03_db_schema.md`(있으면)
   - 테스트 계획+L1 결과 — `04_test_plan.md` / 배포 가이드 — `05_deploy_guide.md` / 리뷰 보고서 — `06_review_report.md`
   - 소스 코드 — `src/app`, `src/server`, `src/components`

## 협업 방식: 솔로 vs 팀

Phase 1에서 정한 협업 방식(솔로/팀)에 따라 하네스 공유와 git 워크플로우가 달라진다.

### 하네스 공유 (팀만 해당)

개인 전역(`~/.claude`)에만 있는 `react-*` 에이전트/스킬은 이 저장소(`claude-global-config`)를
symlink하지 않은 팀원에게는 보이지 않는다. 팀 프로젝트라면 스캐폴딩 직후 하네스 파일을 프로젝트
저장소 안으로 복사해 커밋한다:

```bash
mkdir -p .claude/agents .claude/skills
cp ~/.claude/agents/react-*.md .claude/agents/
cp -r ~/.claude/skills/react-webapp ~/.claude/skills/react-security-checklist ~/.claude/skills/nextjs-ui-patterns .claude/skills/
printf '{\n  "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }\n}\n' > .claude/settings.json
git add .claude && git commit -m "chore: react-webapp 하네스를 프로젝트에 포함"
```

이후 팀원은 저장소를 clone하기만 하면 동일한 에이전트/스킬을 쓸 수 있다(개인
`~/claude-global-config` 셋업이 없어도 됨). 프로젝트에 복사된 사본은 이후 개인 전역 버전과
별개로 관리된다 — 팀 프로젝트 안에서 에이전트/스킬을 고치면 그 저장소 안의 사본을 고친다
(개인 `claude-global-config`로 자동 동기화하지 않는다, 반영하고 싶으면 나중에 수동으로 판단한다).

### Git 워크플로우

| 협업 방식 | 커밋 방식 | 최종 단계 |
|----------|----------|----------|
| 솔로 | phase 경계마다 현재 브랜치에 직접 `git commit` | Phase 3에서 사용자에게 요약 보고 |
| 팀 | Phase 1에서 만든 feature 브랜치(`git checkout -b feature/{feature-name}`)에 phase 경계마다 커밋 | Phase 3에서 직접 머지하지 않고 `gh pr create`로 PR을 연다 |

팀 모드 PR 본문에는 아키텍처 결정(Option A/B/C 중 선택과 이유), Match Rate, L1 검증 결과를
요약해 넣어 팀이 리뷰할 수 있게 한다. 체크포인트 게이트의 승인은 "구현 착수 허가"이지 팀의
최종 합의가 아니라는 점을 다시 확인 — 실제 설계 합의는 이 PR 리뷰를 통해 팀에게 받는다.

### 동시 진행 주의

같은 feature를 여러 팀원이 동시에 이 하네스로 진행하지 않는다 — `_workspace/.status.json`과
git 커밋이 충돌한다. 여러 feature를 동시에 진행해야 하면 feature마다 별도 브랜치에서 순차적으로
진행하고, 같은 브랜치·같은 `_workspace/`를 두 feature가 동시에 쓰지 않는다.

## 작업 규모별 모드

| 사용자 요청 패턴 | 실행 모드 | 투입 에이전트 |
|----------------|----------|-------------|
| "Next.js 웹앱 만들어줘", "풀스택 개발" | **풀 파이프라인** | 5명 전원 |
| "API/Server Action만 만들어줘" | **백엔드 모드** | architect + backend + qa |
| "화면만 만들어줘" (API 있음, 또는 BFF 성향) | **프론트 모드** | architect + frontend + qa |
| "이 코드 리팩토링해줘" | **리팩토링 모드** | architect + 해당 개발자 + qa |
| "배포 설정만 해줘" | **DevOps 모드** | devops 단독 |
| 간단한 단일 feature(예: 컴포넌트 하나 + 스타일처럼 순차적으로만 의미 있는 작업) | **솔로 모드** | 팀 구성 없이 오케스트레이터가 직접 처리 — 병렬화 이득이 없으면 무리해서 팀을 쓰지 않는다 |

**프로젝트 성향에 따른 조정**: BFF·프론트엔드 전용 성향은 풀 파이프라인이어도 backend-dev의 산출물이 Prisma/DB 없이 API 클라이언트/타입 정의로 축소된다. 정적·마케팅 사이트 성향은 backend-dev를 투입하지 않는다(위 react-architect.md "프로젝트 성향별 구성" 표 참고).

**기존 코드 활용**: 사용자가 기존 Next.js/React 프로젝트를 제공하면, architect가 라우트 구조/컴포넌트 컨벤션을 분석해 확장 지점을 파악하고 필요한 에이전트만 투입한다.

## 데이터 전달 프로토콜

| 전략 | 방식 | 용도 |
|------|------|------|
| 파일 기반 | `_workspace/` + `src/` 소스 | 설계 문서 + 소스 코드 |
| 메시지 기반 | SendMessage | Server Action 연동 이슈, 코드 리뷰, 수정 요청 |
| 태스크 기반 | TaskCreate/TaskUpdate | 진행 상황 추적, 의존 관계 관리 |

## 에러 핸들링

| 에러 유형 | 전략 |
|----------|------|
| 요구사항 모호 | 가장 일반적인 인증 + CRUD 패턴 적용, 가정 사항 문서화 |
| 프로젝트 성향 미확정 | 임의로 단정하지 않고 사용자에게 확인 — 스캐폴딩(Prisma/Auth.js 설치 여부)이 바뀌는 결정이므로 |
| 기술 스택 미지정 | Next.js(App Router) + TypeScript strict + Tailwind + Prisma(풀스택 성향) 기본 적용 |
| 빌드 에러 | 에러 로그 분석 → 해당 개발자가 수정 → qa 재검증 |
| 에이전트 실패 | 1회 재시도 → 실패 시 해당 산출물 없이 진행, 리뷰에 명시 |
| 리뷰에서 🔴 발견 | 해당 개발자에 수정 요청 → 재작업 → 재검증 (최대 2회) |
| Server Action/미들웨어 인증 이상동작 | react-backend-dev.md "자주 겪는 함정" 섹션부터 확인 |

## 테스트 시나리오

### 정상 흐름 (풀스택)
**프롬프트**: "이메일 로그인 기반 대시보드를 만들어줘. 로그인, 멤버 목록(권한 2단계), 프로필 수정"
**기대 결과**:
- 아키텍처: App Router 라우트 그룹((auth)/(dashboard)), ERD(User, Session), API 명세
- 백엔드: Auth.js Credentials + bcrypt, Role 기반 권한, Prisma 마이그레이션
- 프론트: 로그인 화면, 멤버 목록/프로필 화면, `useActionState` 폼 연동
- 테스트: 미인증 redirect/오인증 에러/정상로그인/권한부족 L1 시나리오 통과
- 배포: docker-compose(Postgres) 또는 Vercel 배포 가이드, CI

### BFF·프론트엔드 전용 흐름
**프롬프트**: "이미 있는 Spring API를 소비하는 관리자 화면을 Next.js로 만들어줘"
**기대 결과**: architect가 외부 API 계약을 `02_api_spec.md`에 정리, backend-dev는 Prisma 없이 API 클라이언트 타입/래퍼만 구현, frontend가 화면 구현, qa가 Playwright로 화면-API 연동 L1 검증

### 기존 프로젝트 확장 흐름
**프롬프트**: "이 Next.js 프로젝트에 게시판 기능을 추가해줘" + 기존 코드
**기대 결과**: architect가 기존 라우트/컴포넌트 구조 분석 후 게시판 설계, backend가 CRUD Server Action 추가, frontend가 목록/상세/폼 화면 추가, qa가 L1 검증

### 에러 흐름
**프롬프트**: "간단한 웹앱 만들어줘"
**기대 결과**: 요구사항 모호 → architect가 기본 인증 스킬레톤 제안, 프로젝트 성향을 사용자에게 확인, MVP 규모 기본 스택 적용, 리뷰 보고서에 "요구사항 가정 적용" 명시

## 에이전트별 확장 스킬

| 스킬 | 대상 에이전트 | 역할 |
|------|-------------|------|
| `react-security-checklist` | react-backend-dev | Auth.js 인증/인가 패턴, OWASP 대응, Server Action/Route Handler 보안 함정 |
| `nextjs-ui-patterns` | react-frontend-dev | App Router 레이아웃/폼/데이터 페칭 패턴, shadcn/ui 컴포넌트 재사용 |
