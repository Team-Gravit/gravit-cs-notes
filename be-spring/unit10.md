## 인증·인가 필터 체인

스프링 시큐리티는 컨트롤러가 아니라 **서블릿 필터 체인**에서 인증(Authentication, 누구인가)과 인가(Authorization, 무엇을 할 수 있는가)를 처리한다. 요청이 어떤 필터를 어떤 순서로 통과하고, 인증된 사용자 정보가 어디에 저장되며, 직접 만든 JWT 필터를 어디에 끼워야 하는지 알아야 "로그인이 됐는데 403이 난다", "필터가 두 번 실행된다" 같은 문제를 구조적으로 해결할 수 있다.

<br>

### 1. 전체 구조 — DelegatingFilterProxy와 FilterChainProxy

```
[톰캣 필터 체인]
   ... → DelegatingFilterProxy ("springSecurityFilterChain" 빈에 위임)
              │
              ▼
         FilterChainProxy ─── 요청 URL로 SecurityFilterChain 선택 (securityMatcher)
              │
              ├─ SecurityFilterChain #1 (/api/**)   : [JwtFilter, AuthorizationFilter ...]
              └─ SecurityFilterChain #2 (/admin/**) : [UsernamePasswordAuthenticationFilter ...]
                          │
                          ▼
                    DispatcherServlet → 컨트롤러 (unit03)
```

- **DelegatingFilterProxy**: 서블릿 컨테이너에 등록된 표준 필터. 스프링 빈을 서블릿 필터 세계로 이어 주는 다리 역할
- **FilterChainProxy**: 스프링 시큐리티의 진입점. 여러 `SecurityFilterChain` 중 요청과 매칭되는 **첫 번째 체인 하나**만 적용함
- **SecurityFilterChain**: 실제 보안 필터 목록. Spring Security 5.7에서 `WebSecurityConfigurerAdapter`가 deprecated되고 **6.0에서 제거**되어, 지금은 `SecurityFilterChain` **빈을 등록하는 방식**만 사용함

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain apiChain(HttpSecurity http, JwtAuthenticationFilter jwtFilter) throws Exception {
        http
            .securityMatcher("/api/**")                                         // 이 체인이 담당할 경로
            .csrf(AbstractHttpConfigurer::disable)                              // 토큰 기반 API는 CSRF 비활성화
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)   // JWT 필터 배치
            .exceptionHandling(e -> e
                .authenticationEntryPoint(new JsonAuthenticationEntryPoint())        // 401
                .accessDeniedHandler(new JsonAccessDeniedHandler()));                // 403
        return http.build();
    }
}
```

> 💡 Spring Security 6부터 설정은 **람다 DSL**(`.csrf(c -> ...)`)이 표준이다. 구버전 자료의 `.and()` 체이닝은 6.1에서 deprecated되었고 7.0에서 제거되었다. `authorizeRequests()`도 `authorizeHttpRequests()`로 대체되었다.

<br>

### 2. 주요 필터 순서

`SecurityFilterChain` 내부 필터는 정해진 순서로 실행된다. 대표 필터만 추리면 다음과 같다.

```
요청 ──▶ SecurityContextHolderFilter      : 저장소(세션 등)에서 SecurityContext 를 지연 로드
      ──▶ CsrfFilter                      : CSRF 토큰 검증 (활성화 시)
      ──▶ LogoutFilter                    : /logout 처리
      ──▶ [커스텀 JwtAuthenticationFilter] ← addFilterBefore(…, UsernamePasswordAuthenticationFilter.class)
      ──▶ UsernamePasswordAuthenticationFilter : 폼 로그인 (POST /login)
      ──▶ BasicAuthenticationFilter       : HTTP Basic
      ──▶ AnonymousAuthenticationFilter   : 인증 없으면 익명 Authentication 채움
      ──▶ ExceptionTranslationFilter      : 뒤에서 발생한 인증·인가 예외를 401/403 응답으로 변환
      ──▶ AuthorizationFilter             : authorizeHttpRequests 규칙 검사 (인가)
      ──▶ DispatcherServlet
```

- 인증 필터들은 자신이 담당하는 요청(예: `POST /login`)이 아니면 **그냥 통과**시킴. 인증이 안 된 요청을 최종적으로 막는 것은 **AuthorizationFilter**
- `ExceptionTranslationFilter`는 **자기보다 뒤에서** 던져진 `AuthenticationException`·`AccessDeniedException`만 잡음 → 그 앞에 있는 커스텀 필터에서 던진 예외는 `EntryPoint`로 가지 않고 서블릿 컨테이너로 전파됨
- `AuthorizationFilter`는 5.5에서 도입되어 6.0부터 기본값이며, 이전의 `FilterSecurityInterceptor`를 대체함

<br>

### 3. 인증 객체와 저장 위치

| **구성 요소**                | **역할**                                                                  |
| ---------------------------- | ------------------------------------------------------------------------- |
| **Authentication**           | 인증 정보 객체 — `principal`(사용자), `credentials`(비밀번호 등), `authorities`(권한), `authenticated` 플래그 |
| **SecurityContext**          | `Authentication`을 담는 컨테이너                                          |
| **SecurityContextHolder**    | `SecurityContext`를 **ThreadLocal**에 보관하는 정적 접근점 (기본 전략)     |
| **SecurityContextRepository**| 요청 사이에 컨텍스트를 **영속화**하는 저장소 (세션 등)                     |
| **AuthenticationManager**    | 인증 요청을 받아 적절한 `AuthenticationProvider`에 위임 (`ProviderManager`) |
| **AuthenticationProvider**   | 실제 검증 (`DaoAuthenticationProvider` = `UserDetailsService` + `PasswordEncoder`) |

```
[요청 스레드]
SecurityContextHolder (ThreadLocal)
   └─ SecurityContext
        └─ Authentication { principal=UserDetails, authorities=[ROLE_USER], authenticated=true }

요청 시작: SecurityContextRepository.loadContext() → Holder에 세팅
요청 종료: Holder 비움 (ThreadLocal 누수 방지)
```

| **저장 방식**                              | **SecurityContextRepository 구현**       | **상태**       | **적합한 환경**             |
| ------------------------------------------ | ---------------------------------------- | -------------- | --------------------------- |
| **HTTP 세션**                              | `HttpSessionSecurityContextRepository`   | Stateful       | 폼 로그인, 서버 렌더링 웹   |
| **요청 속성 (요청 동안만)**                | `RequestAttributeSecurityContextRepository` | Stateless   | **JWT·API 키 기반 API**     |

- `SessionCreationPolicy.STATELESS`로 설정하면 세션 저장소를 쓰지 않으므로 **매 요청마다 토큰을 검증해 Holder를 채워야** 함
- Spring Security 6부터 `SecurityContextHolderFilter`가 컨텍스트를 **읽기만** 하고 자동 저장하지 않으므로(`requireExplicitSave` 기본 `true`), 세션에 인증을 유지하려면 `securityContextRepository.saveContext()`를 **명시적으로 호출**해야 함. 무상태 JWT 방식은 저장이 필요 없어 영향이 없음

> ⚠️ `SecurityContextHolder`는 **ThreadLocal** 기반이다. `@Async`·`CompletableFuture`·별도 스레드 풀에서 실행되는 코드는 인증 정보를 볼 수 없다. 자식 스레드로 전파하려면 `DelegatingSecurityContextExecutor`(또는 `DelegatingSecurityContextAsyncTaskExecutor`)로 실행기를 감싼다 (unit11 참고).

<br>

### 4. JWT 필터 구현과 배치

```java
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtTokenProvider tokenProvider;
    private final HandlerExceptionResolver handlerExceptionResolver;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String token = resolveToken(request);                      // "Authorization: Bearer xxx"
        if (token == null) {
            chain.doFilter(request, response);                     // 토큰 없음 → 통과, 인가 단계에서 판단
            return;
        }
        try {
            Authentication auth = tokenProvider.authenticate(token);       // 검증 + UserDetails 조회
            SecurityContext context = SecurityContextHolder.createEmptyContext();
            context.setAuthentication(auth);
            SecurityContextHolder.setContext(context);             // 이 요청(스레드) 동안만 유효
            chain.doFilter(request, response);
        } catch (JwtException e) {
            handlerExceptionResolver.resolveException(request, response, null, e);   // @ControllerAdvice로 위임
        }
    }
}
```

**배치 원칙**

- `addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)`: 폼 로그인 필터 **앞**에 두어 인증 필터 위치를 차지하게 함. 인증 결과가 뒤의 `AuthorizationFilter`보다 먼저 세팅되면 되므로 정확한 기준 필터는 프로젝트마다 달라도 됨
- **토큰이 없으면 예외를 던지지 말고 통과**시킴: `permitAll` 경로도 이 필터를 지나가므로, 여기서 401을 내면 공개 API까지 막힘. 인증 여부 판단은 `AuthorizationFilter`에 맡김
- 토큰이 **잘못된** 경우(만료·위조)는 401을 명확히 알려야 하므로 예외로 처리하되, `ExceptionTranslationFilter`보다 앞이라 `EntryPoint`를 타지 않으므로 `HandlerExceptionResolver`로 위임하거나 직접 응답을 씀 (unit04·unit09 참고)

> ⚠️ 필터를 `@Component`로 만들면 스프링 부트가 **서블릿 컨테이너 필터로도 자동 등록**해, 시큐리티 체인 안과 밖에서 **두 번 실행**된다. 빈으로 두려면 `FilterRegistrationBean`으로 `setEnabled(false)`를 지정하거나, `@Component` 없이 설정 클래스에서 `new`로 생성해 `addFilterBefore`에만 넘긴다.

<br>

### 5. 인가 — URL 규칙과 메서드 보안

| **방식**                              | **위치**                    | **설정**                                     | **적합한 경우**                    |
| ------------------------------------- | --------------------------- | -------------------------------------------- | ---------------------------------- |
| **URL 기반** (`authorizeHttpRequests`) | `AuthorizationFilter`       | `requestMatchers(...).hasRole(...)`           | 경로 단위의 굵은 정책              |
| **메서드 기반** (`@PreAuthorize`)      | 서비스·컨트롤러 메서드 AOP | `@EnableMethodSecurity` (6.0부터 권장 방식)   | 도메인 객체 소유권 등 세밀한 규칙 |

```java
@PreAuthorize("hasRole('ADMIN') or #memberId == authentication.principal.id")
public MemberResponse find(Long memberId) { ... }
```

- `hasRole("ADMIN")`은 내부적으로 `ROLE_ADMIN` 권한 문자열과 비교함. `hasAuthority("ROLE_ADMIN")`과 동일
- `requestMatchers` 규칙은 **위에서 아래로 첫 매칭**이 적용되므로 구체적인 경로를 먼저 씀
- 메서드 보안은 AOP 프록시라 self-invocation에서 무시됨 (unit02 참고). `AccessDeniedException`은 `@ControllerAdvice`에서 403으로 변환 가능

<br>

### 6. 세션 방식 vs 토큰 방식 — 필터 체인 관점 비교

| **항목**               | **세션 (폼 로그인)**                             | **JWT (무상태)**                                     |
| ---------------------- | ------------------------------------------------ | ---------------------------------------------------- |
| **인증 필터**          | `UsernamePasswordAuthenticationFilter`           | 커스텀 `JwtAuthenticationFilter`                     |
| **컨텍스트 저장**      | 세션 (`HttpSessionSecurityContextRepository`)    | 요청마다 재구성, 저장 안 함                          |
| **CSRF**               | **필요** (쿠키 자동 전송)                        | 보통 비활성화 (헤더 토큰은 자동 전송 안 됨)          |
| **로그아웃·강제 만료** | 세션 무효화로 즉시                               | 토큰 자체는 만료 전까지 유효 → 블랙리스트·짧은 만료 + 리프레시 토큰 |
| **수평 확장**          | 세션 공유 저장소(Redis) 필요                     | 서버 간 상태 공유 불필요                             |

- 인증 방식 자체의 장단점은 `web-security/unit06`, 세션 공격은 `web-security/unit07` 참고

<br>

### 7. 면접·실무 체크포인트

- 요청은 `DelegatingFilterProxy → FilterChainProxy → SecurityFilterChain`을 거치며, 체인은 **URL 매칭으로 하나만** 선택된다
- 인증 정보는 `Authentication → SecurityContext → SecurityContextHolder(ThreadLocal)`에 저장되고, 요청 간 유지는 `SecurityContextRepository`(세션)가 담당한다
- 6.x부터 컨텍스트 저장은 **명시적**(`requireExplicitSave`)이며, 무상태 JWT는 매 요청 Holder를 채우기만 하면 된다
- 인증되지 않은 요청을 최종적으로 막는 것은 **AuthorizationFilter**이고, 401/403 변환은 **ExceptionTranslationFilter**가 담당한다
- JWT 필터는 `UsernamePasswordAuthenticationFilter` **앞**에 두고, 토큰이 없으면 통과·잘못됐으면 명시적 401을 응답한다
- 필터를 `@Component`로 두면 **이중 등록**되며, `SecurityContextHolder`는 **다른 스레드로 전파되지 않는다**
