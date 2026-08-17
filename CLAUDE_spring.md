# Java/Spring Boot 하네스 안내

이 파일은 Java/Spring Boot 프로젝트에서 작업할 때 참고하는 보조 지침이다. 전역 `CLAUDE.md`
규칙과 함께 적용된다. 프로젝트가 Java/Spring Boot(Gradle/Maven)로 판단되면 이 파일을 먼저
읽고 아래 내용을 따른다.

## 이게 뭔가

여러 Spring Boot 프로젝트에서 재사용하는 개인용 하네스(에이전트 팀 + 스킬)다. 특정 프로젝트에
종속되지 않고 `~/.claude/agents/`, `~/.claude/skills/`에 상주하며 모든 프로젝트에서 자동으로
사용 가능하다. `revfactory/harness-100`의 fullstack-webapp 구조를 참고해 Java/Spring 스택으로
재구성했고, bkit의 상태추적·품질게이트·gap-detector 개념 중 값어치 있는 부분을 외부 인프라(MCP
서버·훅) 없이 파일+git 기반으로 이식했다.

## 트리거 방법

"Spring Boot 프로젝트 시작해줘", "Java 백엔드 만들어줘", "관리자 콘솔 만들어줘", "Spring 웹앱
개발" 등으로 요청하면 `spring-webapp` 스킬이 발동한다. 명시적으로 스킬을 부를 수도 있다.

## 에이전트 (`~/.claude/agents/`)

| 에이전트 | 역할 |
|---------|------|
| `spring-architect` | 요구사항 분석, 멀티모듈 Gradle 아키텍처, DB 모델링(JPA+Flyway), API 설계 — 3가지 설계안(Option A/B/C) 비교까지 산출 |
| `spring-backend-dev` | Controller/Service/Repository 레이어드 구현, Spring Security 인증/인가, 비즈니스 로직 |
| `spring-frontend-dev` | Thymeleaf + Tailwind 서버사이드 화면, 레이아웃 프래그먼트, 폼/CSP 처리 |
| `spring-qa-engineer` | 단위/통합 테스트 + curl 기반 L1 실행 검증 + Design-vs-구현 Match Rate 산출 |
| `spring-devops-engineer` | Docker Compose, GitHub Actions CI, SpotBugs/FindSecBugs/OWASP 빌드 게이트, systemd 배포 |

## 스킬 (`~/.claude/skills/`)

| 스킬 | 역할 |
|------|------|
| `spring-webapp` | 오케스트레이터 — 5개 에이전트를 Phase 1(준비)→2(설계+병렬 구현)→3(통합)으로 조율. 작업 규모별 모드(풀 파이프라인/백엔드만/프론트만/리팩토링/솔로) 선택, 4단계 사용자 승인 체크포인트, 경량 품질 게이트 포함 |
| `spring-security-checklist` | spring-backend-dev 확장 — OWASP 대응, Spring Security 6 실전 함정(전역 `authenticationEntryPoint`가 `/login` 페이지를 지우는 문제 등 재현된 이슈 포함) |
| `thymeleaf-patterns` | spring-frontend-dev 확장 — 레이아웃 프래그먼트, 폼 검증 화면, CSP nonce 패턴, Tailwind 빌드 파이프라인 |

상세 워크플로우·체크포인트 게이트·품질 게이트 표는 `~/.claude/skills/spring-webapp/SKILL.md`
본문 참고 — 이 파일에서 중복 설명하지 않는다.

## bkit과 함께 있는 프로젝트라면

같은 feature에 bkit PDCA(`docs/01-plan/` 등)와 이 하네스(`_workspace/`)를 동시에 쓰지 않는다 —
둘 다 유사한 오케스트레이션이라 산출물이 겹친다.

어느 쪽을 쓸지는 프로젝트마다 한 번만 정하고 그 프로젝트의 (project-local) `CLAUDE.md`에
`## 오케스트레이션 체계`로 기록해둔다(전역 `CLAUDE.md`의 "신규 프로젝트는 오케스트레이션 체계를
선택해 프로젝트 CLAUDE.md에 고정한다" 규칙 참고). 새 Spring 프로젝트를 시작할 때 이미 기록이
있으면 그대로 따르고, 없으면 시작 전에 사용자에게 bkit PDCA 방식인지 이 하네스 방식인지 물어서
기록한다.

## 이 파일을 갱신할 때

에이전트/스킬 파일 자체를 고치면(`~/.claude/agents/spring-*.md`, `~/.claude/skills/spring-*`),
이 파일의 표도 같이 최신화한다 — 실제 파일 내용과 이 개요 문서가 어긋나지 않게 유지한다.
