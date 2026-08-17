---
name: spring-security-checklist
description: "Spring Boot API 보안 체크리스트. OWASP 기반 취약점 점검, Spring Security 6 인증/인가 패턴, 입력 검증, 보안 헤더, CSRF/세션 설정을 제공하는 spring-backend-dev 확장 스킬. 'Spring Security 설정', 'API 보안', 'OWASP', '인증 구현', 'CSRF', '보안 헤더', '401 vs 403' 등 Java/Spring 백엔드 보안 설계·구현 시 사용한다. 단, 침투 테스트 수행이나 WAF 구성은 이 스킬의 범위가 아니다."
---

# Spring Security Checklist — Java/Spring Boot API 보안 체크리스트

spring-backend-dev 에이전트가 Spring Security 기반 인증/인가를 구현할 때 활용하는 실전 체크리스트. 이론이 아니라 실제로 겪는 함정 위주로 정리했다.

## 대상 에이전트

`spring-backend-dev` — 이 스킬의 체크리스트와 설정 패턴을 SecurityConfig 구현에 직접 적용한다.

## OWASP API Security Top 10 대응 (Spring 기준)

| 순위 | 취약점 | Spring에서의 방어 |
|------|--------|-------------------|
| A1 | BOLA(객체 수준 인가 결함) | Service 계층에서 리소스 소유자 검증(`resource.getOwnerId().equals(currentUserId)`) |
| A2 | 인증 결함 | `BCryptPasswordEncoder(12)`, 로그인 실패 N회 잠금(`Member.recordFailedLogin`) |
| A3 | 객체 속성 수준 인가 | Entity를 직접 응답하지 말고 DTO/Projection으로 필드 제한 |
| A4 | 무제한 리소스 소비 | `Pageable` 강제(무제한 조회 방지), 필요 시 Rate Limiting(Bucket4j 등) |
| A5 | 기능 수준 인가 결함 | `@PreAuthorize("hasRole('ADMIN')")` 또는 `authorizeHttpRequests`의 경로별 권한 매트릭스 |
| A6 | SSRF | 외부 URL 입력 화이트리스트, 내부 IP 대역 차단 |
| A7 | 보안 설정 오류 | prod 프로파일에서 시드러너/디버그 로그/whitelabel 에러 비활성 |
| A8 | 비즈니스 흐름 결함 | 상태 전이는 도메인 메서드로만 (`unlock()`, `recordFailedLogin()`) — 외부에서 status 직접 세팅 금지 |
| A9 | 취약 자산 관리 | 사용하지 않는 엔드포인트/구버전 API 정리 |
| A10 | 안전하지 않은 API 소비 | 외부 API 응답도 검증, 타임아웃 명시 |

## 인증(Authentication) 표준 패턴 — Form Login + BCrypt

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);
}

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/login", "/error").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .formLogin(form -> form
            .loginProcessingUrl("/login")
            .successHandler((req, res, auth) -> res.setStatus(HttpServletResponse.SC_OK))
            .failureHandler((req, res, ex) -> res.sendError(HttpServletResponse.SC_UNAUTHORIZED))
            .permitAll()
        )
        // 미인증 401은 아래 "실전 함정 1" 참고 — 전역 authenticationEntryPoint()는 쓰지 말 것
        .exceptionHandling(ex -> ex.defaultAuthenticationEntryPointFor(
            new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED),
            new AntPathRequestMatcher("/admin/**")))
        .sessionManagement(session -> session.maximumSessions(1))
        .headers(headers -> headers
            .contentSecurityPolicy(csp -> csp.policyDirectives(
                "default-src 'self'; script-src 'self'; style-src 'self'; frame-ancestors 'none'"))
            .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
            .referrerPolicy(r -> r.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER))
        );
    return http.build();
}
```

## 실전 함정 (직접 재현·검증한 이슈)

### 함정 1 — `exceptionHandling().authenticationEntryPoint()` 전역 설정이 `/login` 페이지를 없애버린다

**증상**: `GET /login`이 404. 미인증 API는 401로 잘 나오는데 로그인 폼 자체가 사라짐.

**원인**(Spring Security 6.3 소스로 확인): `DefaultLoginPageConfigurer.configure()`는 다음 조건일 때만 기본 로그인 페이지 필터를 등록한다.
```java
if (this.loginPageGeneratingFilter.isEnabled() && authenticationEntryPoint == null) {
    http.addFilter(this.loginPageGeneratingFilter);
    ...
}
```
`exceptionHandling(ex -> ex.authenticationEntryPoint(x))`(단수형, 전역)을 호출하면 `ExceptionHandlingConfigurer.getAuthenticationEntryPoint()`가 non-null을 반환해 이 조건이 거짓이 되고, 로그인 페이지 필터가 아예 등록되지 않는다.

**해결**: 전역 `authenticationEntryPoint()` 대신 `defaultAuthenticationEntryPointFor(entryPoint, requestMatcher)`로 **보호 대상 경로에만** 401 엔트리포인트를 스코프한다. `/login`처럼 전역 매처에 안 걸리는 경로는 기본 로그인 페이지 필터가 정상 등록된다.

**진단 팁**: `@EnableWebSecurity(debug = true)`로 켜면 매 요청마다 실제 등록된 필터 체인 목록이 로그로 찍힌다 — `DefaultLoginPageGeneratingFilter`가 리스트에 있는지 바로 확인 가능. 진단 후에는 반드시 `debug = true`를 제거한다(운영에 절대 남기지 말 것).

### 함정 2 — curl로 POST 로그인 테스트하면 403이 뜬다

**증상**: 오패스워드 테스트인데 401이 아니라 403이 뜬다.

**원인**: CSRF 보호가 기본 켜져 있어서, `_csrf` 토큰 없이 POST하면 인증 로직에 도달하기도 전에 CsrfFilter가 막는다. 401(인증 실패)과 403(CSRF 실패)을 혼동하기 쉽다.

**해결**: 먼저 `GET /login`으로 세션 쿠키 + `_csrf` hidden input 값을 파싱하고, 그 토큰을 POST body에 같이 넣는다. 로그인 실패 후 CSRF 토큰이 회전할 수 있으므로, 재시도 시 로그인 페이지를 다시 GET해서 새 토큰을 받는다.

### 함정 3 — 멀티모듈에서 `@EntityScan`/`@EnableJpaRepositories` 누락

**증상**: `NoSuchBeanDefinitionException` 또는 Repository 빈을 못 찾음.

**원인**: `@SpringBootApplication`의 기본 컴포넌트 스캔은 애플리케이션 클래스와 같은 패키지 이하만 본다. domain 모듈이 다른 최상위 패키지(`{base}.domain`)에 있으면 못 찾는다.

**해결**: `@EntityScan(basePackages = "{base}.domain")` + `@EnableJpaRepositories(basePackages = "{base}.domain")`을 애플리케이션 클래스에 명시.

## 비밀번호/세션 정책

| 항목 | 권장 설정 |
|------|----------|
| 해시 | BCrypt strength 12+ |
| 세션 타임아웃 | 30분 (요구사항에 맞게 조정) |
| 세션 쿠키 | `HttpOnly`, `SameSite=Strict`, prod에서 `Secure` |
| 로그인 실패 잠금 | 5회 → 계정 잠금, ADMIN 수동 해제 |
| 동시 세션 | `maximumSessions(1)` (요구사항에 따라) |

## HTTP 보안 헤더 (Spring Security `headers()` DSL)

| 헤더 | DSL | 목적 |
|------|-----|------|
| `Content-Security-Policy` | `.contentSecurityPolicy(csp -> ...)` | XSS 방지 |
| `X-Frame-Options` | `.frameOptions(...::deny)` | 클릭재킹 방지 |
| `X-Content-Type-Options` | 기본 활성(별도 설정 불필요) | MIME 스니핑 방지 |
| `Referrer-Policy` | `.referrerPolicy(r -> ...)` | 리퍼러 정보 제한 |
| `Strict-Transport-Security` | prod 프로파일에서 활성 | HTTPS 강제 |

## 입력 검증 체크리스트

| 항목 | 방법 |
|------|------|
| 타입/형식 검증 | Bean Validation(`@NotNull`, `@Email`, `@Size`) + `@Valid` |
| SQL Injection | JPA 파라미터 바인딩 사용, 네이티브 쿼리는 `:param` 바인딩만 |
| LIKE 와일드카드 이스케이프 | 사용자 입력에 `%`/`_` 포함 시 이스케이프 처리 |
| XSS | 출력 이스케이프 기본 유지(Thymeleaf `th:text`), 필요 시 jsoup sanitizer |
| 파일 업로드 | 크기 제한 + 확장자 + MIME + 매직넘버(Tika) 4중 검증 |

## 감사 로그(AuditLog) 설계 원칙

- 액터는 문자열(`actorLoginId`)로 보존 — 계정 삭제 후에도 추적 가능하게
- 최소 기록 항목: `action`, `actor`, `target`, `clientIp`, `userAgent`, `success`, `createdAt`
- 인증 이벤트(LOGIN_SUCCESS/FAILURE/LOGOUT/ACCOUNT_LOCKED)는 `AuthenticationSuccessEventListener`/`AuthenticationFailureBadCredentialsEvent` 리스너로 자동 기록 — 컨트롤러에 흩어놓지 않는다
