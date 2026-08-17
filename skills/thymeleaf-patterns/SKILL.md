---
name: thymeleaf-patterns
description: "Thymeleaf + Tailwind CSS 서버사이드 렌더링 패턴 라이브러리. 레이아웃 프래그먼트, 폼 검증 연동, CSP 호환 정적 자산 빌드, 컴포넌트 재사용 전략을 제공하는 spring-frontend-dev 확장 스킬. '레이아웃 프래그먼트', 'Thymeleaf 패턴', 'Tailwind 빌드', 'CSP nonce', '폼 검증 화면' 등 Spring 서버사이드 프론트엔드 설계 시 사용한다. 단, React/Vue 등 SPA 구현이나 백엔드 로직은 이 스킬의 범위가 아니다."
---

# Thymeleaf Patterns — Spring 서버사이드 렌더링 패턴

spring-frontend-dev 에이전트가 Thymeleaf + Tailwind 기반 화면을 구현할 때 활용하는 레이아웃/폼/빌드 패턴 레퍼런스.

## 대상 에이전트

`spring-frontend-dev` — 이 스킬의 패턴을 템플릿 구조와 정적 자산 빌드에 직접 적용한다.

## 레이아웃 프래그먼트 패턴

### 공통 레이아웃 — `templates/layout/base.html`

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title th:text="${title} + ' | Admin'">Admin</title>
    <link rel="stylesheet" th:href="@{/dist/admin.css}">
</head>
<body>
    <header th:replace="~{layout/header :: header}"></header>
    <main>
        <div th:replace="${content}"></div>
    </main>
    <footer th:replace="~{layout/footer :: footer}"></footer>
</body>
</html>
```

### 페이지에서 레이아웃 사용

```html
<!-- templates/member/list.html -->
<div th:fragment="content">
    <h1>회원 목록</h1>
    <table>
        <tr th:each="m : ${members}">
            <td th:text="${m.loginId}"></td>
            <td th:text="${m.role}"></td>
        </tr>
    </table>
</div>
```

Controller에서는 레이아웃을 감싸는 뷰 이름을 반환하거나, `layout-dialect` 없이 순수 Thymeleaf만 쓸 경우 `th:replace="layout/base :: layout(~{::content})"` 패턴을 사용한다 (Thymeleaf 3.x 파라미터 프래그먼트).

## 폼 + 서버 검증 에러 표시

```html
<form th:action="@{/admin/members}" th:object="${memberForm}" method="post">
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"/>

    <label for="loginId">아이디</label>
    <input id="loginId" th:field="*{loginId}" th:errorclass="border-red-500"/>
    <p th:if="${#fields.hasErrors('loginId')}" th:errors="*{loginId}" class="text-red-600 text-sm"></p>

    <button type="submit">저장</button>
</form>
```

- `th:field`는 자동으로 `id`/`name`/`value`를 바인딩하고, 검증 실패 시 입력값을 유지한다
- `#fields.hasErrors('*')`로 전체 에러 여부, 필드별로는 `#fields.hasErrors('fieldName')`
- Spring Security를 쓰면 `_csrf` 토큰을 폼에 명시적으로 넣어야 한다 (Thymeleaf Spring Security 확장을 쓰면 자동 삽입되는 설정도 있으니 프로젝트 컨벤션 확인)

## CSP nonce 패턴 (인라인 스크립트 없이)

```html
<!-- 지양: 인라인 스크립트/이벤트 핸들러 -->
<button onclick="doSomething()">...</button>

<!-- 권장: 외부 JS 파일 + data 속성으로 값 전달 -->
<button id="save-btn" th:data-member-id="${member.id}">...</button>
<script th:src="@{/js/member-form.js}"></script>
```

CSP `script-src 'self'`를 쓰는 프로젝트에서는 인라인 스크립트/이벤트 핸들러(`onclick=`)를 전부 제거하고 외부 JS + `data-*` 속성으로 값을 전달한다. Alpine.js를 쓴다면 `@alpinejs/csp` 빌드를 사용해 `unsafe-eval` 없이 CSP를 통과시킨다.

## Tailwind 빌드 파이프라인 (CDN 미사용)

### `package.json` (모듈 루트)

```json
{
  "scripts": { "build": "tailwindcss -i ./frontend/input.css -o ./src/main/resources/static/dist/admin.css --minify" },
  "devDependencies": { "tailwindcss": "^3" }
}
```

### Gradle 연동 (해당 web 모듈의 build.gradle.kts)

```kotlin
val npmInstall = tasks.register<Exec>("npmInstall") {
    workingDir = projectDir
    commandLine("npm", "install", "--no-audit", "--no-fund")
}
val npmBuildFrontend = tasks.register<Exec>("npmBuildFrontend") {
    dependsOn(npmInstall)
    workingDir = projectDir
    commandLine("npm", "run", "build")
}
tasks.named("processResources") { dependsOn(npmBuildFrontend) }
```

이렇게 하면 `./gradlew build`만으로 Tailwind CSS까지 함께 빌드된다. 빌드 산출물(`static/dist/`)은 `.gitignore`에 추가한다.

## 폴더 구조 (검증된 관례)

```
src/main/resources/
├── templates/
│   ├── layout/          # base.html, header.html, footer.html
│   ├── auth/             # login.html (화이트라벨 대체 시)
│   ├── {feature}/        # list.html, form.html, detail.html
│   └── error/            # 404.html, 500.html
└── static/
    ├── dist/              # 빌드 산출물 (gitignore)
    └── js/                # 원본 JS (CSP 호환, 인라인 없이)

frontend/                  # 웹 모듈 루트, Tailwind 소스
├── input.css
└── tailwind.config.js
```

## 화면 개발 우선순위 (디자인/퍼블리싱 미확정 시)

1. Spring Security 기본 화이트라벨 로그인 페이지를 임시로 그대로 사용 — API/인증 로직 검증에는 지장 없음
2. 관리자 화면은 Tailwind 기본 유틸리티 클래스로 최소 기능 우선 구현 (표/폼/버튼)
3. 디자인 시안이 오면 레이아웃 프래그먼트만 교체 — 컨트롤러/데이터 바인딩 로직은 그대로 재사용

## 접근성(a11y) 체크리스트

- [ ] 모든 `<img>`에 `alt` (또는 `th:alt`)
- [ ] `<label for="...">` ↔ `<input id="...">` 연결
- [ ] 폼 제출 버튼은 `<button type="submit">` (div/span에 클릭 이벤트로 대체 금지)
- [ ] 색상만으로 상태를 전달하지 않기 (아이콘/텍스트 병행)
- [ ] 키보드만으로 전체 플로우 완주 가능한지 확인
