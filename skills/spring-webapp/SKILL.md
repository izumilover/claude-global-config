---
name: spring-webapp
description: "Java/Spring Boot 웹앱의 요구사항 분석, 설계, 백엔드, 프론트엔드, 테스트, 배포를 에이전트 팀이 협업하여 개발하는 풀 개발 파이프라인. 'Spring Boot 프로젝트 시작해줘', 'Java 백엔드 만들어줘', '관리자 콘솔 만들어줘', 'Spring 웹앱 개발', 'REST API 개발(Spring)', 'Spring Security 인증 구현', 'Gradle 멀티모듈 프로젝트' 등 Java/Spring 웹 애플리케이션 개발 전반에 이 스킬을 사용한다. 기존 Spring 프로젝트가 있는 경우에도 기능 추가나 리팩토링을 지원한다. 단, Node/Python/Go 등 non-JVM 백엔드, 모바일 앱, 게임 개발은 이 스킬의 범위가 아니다."
---

# Spring Webapp — Java/Spring Boot 웹앱 개발 파이프라인

Java/Spring Boot 웹앱의 요구사항→설계→백엔드→프론트엔드→테스트→배포를 에이전트 팀이 협업하여 개발한다.

## 실행 모드

**에이전트 팀** — 5명이 SendMessage로 직접 통신하며 교차 검증한다. Agent Teams(`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`)가 꺼져 있으면 먼저 켤 것을 사용자에게 안내한다.

## 에이전트 구성

| 에이전트 | 파일 | 역할 |
|---------|------|------|
| spring-architect | `~/.claude/agents/spring-architect.md` | 요구사항, 멀티모듈 아키텍처, DB(JPA+Flyway), API 설계 |
| spring-backend-dev | `~/.claude/agents/spring-backend-dev.md` | Controller/Service/Repository, Spring Security, 비즈니스 로직 |
| spring-frontend-dev | `~/.claude/agents/spring-frontend-dev.md` | Thymeleaf + Tailwind 서버사이드 화면 |
| spring-qa-engineer | `~/.claude/agents/spring-qa-engineer.md` | 단위/통합 테스트 + curl 기반 L1 실행 검증 |
| spring-devops-engineer | `~/.claude/agents/spring-devops-engineer.md` | Docker Compose, CI, 빌드 게이트, systemd 배포 |

## 워크플로우

### Phase 1: 준비 (오케스트레이터 직접 수행)

1. 사용자 입력에서 추출한다:
   - **앱 설명**: 만들려는 웹앱의 목적과 핵심 기능
   - **규모** (선택): MVP/소규모/중규모/대규모
   - **기존 코드** (선택): 확장할 기존 Spring 프로젝트
   - **배포 대상** (선택): 로컬만 / Docker / systemd 운영서버
2. `_workspace/` 디렉토리를 프로젝트 루트에 생성한다
3. 입력을 정리하여 `_workspace/00_input.md`에 저장한다
4. 기존 코드가 있으면 분석하고(모듈 구조, 패키지 컨벤션) 해당 단계를 조정한다
5. 요청 범위에 따라 **실행 모드를 결정**한다 (아래 "작업 규모별 모드" 참조)

### Phase 2: 팀 구성 및 실행

| 순서 | 작업 | 담당 | 의존 | 산출물 |
|------|------|------|------|--------|
| 1 | 아키텍처 설계 | spring-architect | 없음 | `01_architecture.md`, `02_api_spec.md`, `03_db_schema.md` |
| 2a | 백엔드 개발 | spring-backend-dev | 작업 1 | 도메인/Flyway/Controller/Service, Security 설정 |
| 2b | 프론트엔드 개발 | spring-frontend-dev | 작업 1, (2a의 API 형태) | Thymeleaf 템플릿, 정적 자산 |
| 2c | 배포 설정 | spring-devops-engineer | 작업 1 | `05_deploy_guide.md`, docker-compose, CI 설정 |
| 3 | 테스트 & 실행검증 | spring-qa-engineer | 작업 2a, 2b | `04_test_plan.md`, `06_review_report.md`, 테스트 코드 + L1 curl 검증 |

작업 2a(백엔드)와 2c(DevOps)는 설계에만 의존하므로 병렬 가능. 2b(프론트)는 2a가 API 형태를 어느 정도 확정해야 효율적이므로, 순수 서버사이드 렌더링 프로젝트에서는 2a 착수 후 바로 이어 붙는 편이 안전하다(무리한 병렬화보다 정확성 우선).

**팀원 간 소통 흐름:**
- architect 완료 → backend에게 API·DB·인증 전달, frontend에게 화면 목록·권한 규칙 전달, devops에게 인프라 요구사항 전달, qa에게 기능 요구사항 전달
- backend ↔ frontend: API 연동 중 실시간 소통 (엔드포인트 변경, 에러 형식 등)
- devops 완료 → 전체에게 환경변수, 배포 절차 공유
- qa는 모든 코드를 리뷰 + **반드시 실제로 앱을 기동해서 curl로 L1 검증**한다. 🔴 필수 수정 발견 시 해당 개발자에게 수정 요청 → 재작업 → 재검증 (최대 2회)

### Phase 3: 통합 및 최종 산출물

1. 모든 코드와 문서를 확인한다
2. 리뷰의 🔴 필수 수정이 모두 반영되었는지, L1 실행 검증이 실제로 통과했는지 확인한다 (코드만 보고 "완료"라고 하지 않는다)
3. 최종 요약을 사용자에게 보고한다:
   - 아키텍처 — `01_architecture.md` / API 명세 — `02_api_spec.md` / DB 스키마 — `03_db_schema.md`
   - 테스트 계획+L1 결과 — `04_test_plan.md` / 배포 가이드 — `05_deploy_guide.md` / 리뷰 보고서 — `06_review_report.md`
   - 소스 코드 — 각 Gradle 모듈

## 작업 규모별 모드

| 사용자 요청 패턴 | 실행 모드 | 투입 에이전트 |
|----------------|----------|-------------|
| "Spring 웹앱 만들어줘", "풀스택 개발" | **풀 파이프라인** | 5명 전원 |
| "API만 만들어줘" | **백엔드 모드** | architect + backend + qa |
| "관리자 화면만 만들어줘" (API 있음) | **프론트 모드** | architect + frontend + qa |
| "이 코드 리팩토링해줘" | **리팩토링 모드** | architect + 해당 개발자 + qa |
| "배포 설정만 해줘" | **DevOps 모드** | devops 단독 |
| 간단한 단일 feature(예: 도메인 모델 + 보안 설정처럼 순차적으로만 의미 있는 작업) | **솔로 모드** | 팀 구성 없이 오케스트레이터가 직접 처리 — 병렬화 이득이 없으면 무리해서 팀을 쓰지 않는다 |

**기존 코드 활용**: 사용자가 기존 Spring 프로젝트를 제공하면, architect가 모듈 구조/패키지 컨벤션을 분석해 확장 지점을 파악하고 필요한 에이전트만 투입한다.

## 데이터 전달 프로토콜

| 전략 | 방식 | 용도 |
|------|------|------|
| 파일 기반 | `_workspace/` + 각 Gradle 모듈 소스 | 설계 문서 + 소스 코드 |
| 메시지 기반 | SendMessage | API 연동 이슈, 코드 리뷰, 수정 요청 |
| 태스크 기반 | TaskCreate/TaskUpdate | 진행 상황 추적, 의존 관계 관리 |

## 에러 핸들링

| 에러 유형 | 전략 |
|----------|------|
| 요구사항 모호 | 가장 일반적인 관리자 인증 + CRUD 패턴 적용, 가정 사항 문서화 |
| 기술 스택 미지정 | Spring Boot 3.x + Java 21 + Gradle Kotlin DSL 기본 적용 |
| 빌드 에러 | 에러 로그 분석 → 해당 개발자가 수정 → qa 재검증 |
| 에이전트 실패 | 1회 재시도 → 실패 시 해당 산출물 없이 진행, 리뷰에 명시 |
| 리뷰에서 🔴 발견 | 해당 개발자에 수정 요청 → 재작업 → 재검증 (최대 2회) |
| Spring Security 라우팅/401 이상동작 | spring-backend-dev.md "자주 겪는 함정" 섹션부터 확인 |

## 테스트 시나리오

### 정상 흐름
**프롬프트**: "관리자 인증 기반 관리자 콘솔을 만들어줘. 로그인, 계정 관리(권한 3단계), 감사 로그"
**기대 결과**:
- 아키텍처: Gradle 멀티모듈(common/domain/infra/admin-web), ERD(member, audit_log), API 명세
- 백엔드: Spring Security Form Login + BCrypt, Role 기반 권한, Flyway V1
- 프론트: 로그인 화면(또는 화이트라벨 임시 사용), 계정 관리 화면
- 테스트: 미인증401/오인증401/정상로그인/권한부족403 L1 시나리오 통과
- 배포: docker-compose(MySQL), systemd 서비스 파일, CI

### 기존 프로젝트 확장 흐름
**프롬프트**: "이 Spring 프로젝트에 게시판 기능을 추가해줘" + 기존 코드
**기대 결과**: architect가 기존 모듈 구조 분석 후 게시판 도메인 설계, backend가 CRUD API 추가, frontend가 목록/상세/폼 화면 추가, qa가 L1 검증

### 에러 흐름
**프롬프트**: "간단한 백엔드 만들어줘"
**기대 결과**: 요구사항 모호 → architect가 기본 관리자 인증 스킬레톤 제안, MVP 규모 기본 스택(단일 모듈 + H2) 적용, 리뷰 보고서에 "요구사항 가정 적용" 명시

## 에이전트별 확장 스킬

| 스킬 | 대상 에이전트 | 역할 |
|------|-------------|------|
| `spring-security-checklist` | spring-backend-dev | Spring Security 인증/인가 패턴, OWASP 대응, 보안 헤더, 실전 함정 |
| `thymeleaf-patterns` | spring-frontend-dev | Thymeleaf 레이아웃/폼/CSP 패턴, Tailwind 빌드 파이프라인 |
