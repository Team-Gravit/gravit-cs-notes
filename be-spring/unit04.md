## 요청 전후 처리 계층

인증·로깅·인코딩·공통 파라미터 주입처럼 **모든 요청에 반복되는 처리**는 컨트롤러가 아니라 그 앞뒤의 계층에 두어야 한다. 스프링 MVC에는 **필터(Filter)·인터셉터(Interceptor)·ArgumentResolver**라는 세 계층이 있고, 각각 실행 위치·접근 가능한 정보·예외 처리 범위가 다르므로 "무엇을 어디에 둘 것인가"를 판단하는 기준을 알아야 한다.

<br>

### 1. 세 계층의 위치

```
[톰캣]
  │
  ▼
Filter 1 ──▶ Filter 2 ──▶ ... ──▶ ┌────────────────── DispatcherServlet ──────────────────┐
  ▲             ▲                 │  Interceptor.preHandle                                 │
  │             │                 │        ↓                                               │
  │             │                 │  ArgumentResolver (파라미터 조립)                       │
  │             │                 │        ↓                                               │
  │             │                 │  Controller 메서드 실행                                │
  │             │                 │        ↓                                               │
  │             │                 │  Interceptor.postHandle  → 뷰 렌더링 / 본문 직렬화     │
  │             │                 │        ↓                                               │
  │             │                 │  Interceptor.afterCompletion                           │
  └─────────────┴─────────────────┴────────────────────────────────────────────────────────┘
       (응답은 역순으로 필터를 되돌아 나감)
```

- **필터**는 서블릿 스펙(`jakarta.servlet.Filter`)이라 **DispatcherServlet 바깥**에서 동작하고, 스프링 MVC를 몰라도 됨
- **인터셉터**는 스프링 MVC 스펙(`HandlerInterceptor`)이라 **DispatcherServlet 안**에서 동작하며, 어떤 컨트롤러 메서드가 실행될지(`HandlerMethod`)를 알 수 있음
- **ArgumentResolver**는 컨트롤러 메서드의 **파라미터 하나를 만들어 주는** 역할로, 흐름을 막거나 통과시키는 계층이 아님

<br>

### 2. 필터(Filter)

```java
@Slf4j
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        ContentCachingRequestWrapper wrapped = new ContentCachingRequestWrapper(request);  // 본문을 여러 번 읽도록 감쌈
        long start = System.currentTimeMillis();
        try {
            chain.doFilter(wrapped, response);              // 다음 필터 → DispatcherServlet
        } finally {
            log.info("{} {} {}ms", request.getMethod(), request.getRequestURI(),
                     System.currentTimeMillis() - start);
        }
    }
}

@Configuration
public class FilterConfig {
    @Bean
    public FilterRegistrationBean<RequestLoggingFilter> loggingFilter() {
        FilterRegistrationBean<RequestLoggingFilter> bean = new FilterRegistrationBean<>(new RequestLoggingFilter());
        bean.addUrlPatterns("/api/*");
        bean.setOrder(1);                                   // 낮을수록 먼저 실행
        return bean;
    }
}
```

- `chain.doFilter()`를 호출하지 않으면 요청이 거기서 끝남 → 인증 실패 시 직접 응답을 써서 차단 가능
- `HttpServletRequest/Response`를 **래퍼로 교체**할 수 있는 유일한 계층 → 본문 재사용, 응답 압축, 인코딩 처리에 적합
- `OncePerRequestFilter`를 상속하면 `forward`·`error` 디스패치로 같은 요청이 다시 들어와도 **한 번만** 실행됨
- 스프링 시큐리티의 `FilterChainProxy`도 서블릿 필터 하나로 등록되어 동작함 (unit10 참고)

> ⚠️ 필터 클래스에 `@Component`를 붙이면 스프링 부트가 **모든 URL에 자동 등록**한다. 여기에 `FilterRegistrationBean`으로 한 번 더 등록하거나 시큐리티 체인에도 `addFilterBefore`로 넣으면 **필터가 두 번 실행**된다. 경로·순서를 제어하려면 `@Component` 없이 `FilterRegistrationBean`으로만 등록한다.

<br>

### 3. 인터셉터(Interceptor)

```java
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        if (!(handler instanceof HandlerMethod handlerMethod)) {
            return true;                                            // 정적 리소스 등은 통과
        }
        if (handlerMethod.hasMethodAnnotation(PublicApi.class)) {   // 컨트롤러 메서드의 애노테이션 확인 가능
            return true;
        }
        if (request.getSession(false) == null) {
            throw new UnauthorizedException("로그인이 필요합니다");   // @ControllerAdvice가 처리
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        MDC.clear();                                                // 예외 여부와 무관하게 항상 실행
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/auth/**", "/api/health");
    }
}
```

| **메서드**            | **호출 시점**                       | **예외 발생 시**                              | **주요 용도**                          |
| --------------------- | ----------------------------------- | --------------------------------------------- | -------------------------------------- |
| **preHandle**         | 컨트롤러 실행 전                     | `false` 반환 또는 예외로 중단                 | 인증·인가 확인, 요청 컨텍스트 설정     |
| **postHandle**        | 컨트롤러 실행 후, 뷰 렌더링 전       | 컨트롤러 예외 시 **호출되지 않음**            | 모델 공통 데이터 추가 (뷰 기반)        |
| **afterCompletion**   | 응답 완료 후                         | **항상 호출** (예외 객체 전달)                | 자원 정리, MDC·ThreadLocal 해제         |

- 핸들러 객체를 받으므로 **"어느 컨트롤러 메서드인가"에 따라 분기**하는 처리(애노테이션 기반 권한 체크)에 적합함
- `@RestController`는 `postHandle` 시점에 이미 응답 본문이 쓰여 있으므로 본문을 수정할 수 없음 (unit03 참고)
- 인터셉터는 스프링 빈이므로 다른 빈(서비스·리포지토리)을 주입받아 쓰기 편함

<br>

### 4. ArgumentResolver — 커스텀 파라미터 주입

컨트롤러마다 세션·헤더에서 사용자를 꺼내는 코드가 반복된다면, `HandlerMethodArgumentResolver`로 **파라미터 조립 자체를 공통화**한다.

```java
// 안티패턴: 컨트롤러마다 반복되는 사용자 조회
@GetMapping("/me")
public MemberResponse me(HttpSession session) {
    Long memberId = (Long) session.getAttribute("memberId");
    if (memberId == null) throw new UnauthorizedException();
    return memberService.find(memberId);
}

// 개선: @LoginMember 애노테이션 + ArgumentResolver
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface LoginMember {}

public class LoginMemberArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(LoginMember.class)
                && parameter.getParameterType().equals(Long.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mav,
                                  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        Long memberId = (Long) request.getSession().getAttribute("memberId");
        if (memberId == null) throw new UnauthorizedException();
        return memberId;
    }
}

@GetMapping("/me")
public MemberResponse me(@LoginMember Long memberId) {      // 컨트롤러는 비즈니스에만 집중
    return memberService.find(memberId);
}
```

- `WebMvcConfigurer.addArgumentResolvers()`로 등록하며, `supportsParameter()`가 `true`인 첫 리졸버가 사용됨
- 인터셉터가 `preHandle`에서 검증한 값을 `request.setAttribute()`로 넘기고, 리졸버가 꺼내 쓰는 조합이 흔함
- 스프링 시큐리티를 쓰면 `@AuthenticationPrincipal`이 같은 원리로 동작하는 리졸버임

> 💡 "인터셉터에서 인증하고, ArgumentResolver로 사용자 객체를 주입한다"는 조합은 실무 코드에서 매우 흔하다. 두 계층의 역할이 **"차단"과 "주입"**으로 분리되어 있다는 점을 설명하면 좋다.

<br>

### 5. 비교와 선택 기준

| **항목**                 | **필터**                              | **인터셉터**                              | **ArgumentResolver**                 |
| ------------------------ | ------------------------------------- | ----------------------------------------- | ------------------------------------ |
| **스펙**                 | 서블릿 (`jakarta.servlet`)             | 스프링 MVC                                | 스프링 MVC                           |
| **실행 위치**            | DispatcherServlet **바깥**             | DispatcherServlet **안**                  | HandlerAdapter 안 (파라미터 조립)    |
| **핸들러 정보 접근**     | 불가                                   | **가능** (`HandlerMethod`)                | 가능 (`MethodParameter`)             |
| **Request/Response 교체**| **가능** (래퍼)                        | 불가 (객체 참조만)                        | 불가                                 |
| **예외 처리**            | `@ControllerAdvice` **적용 안 됨**     | `@ControllerAdvice` 적용됨                | `@ControllerAdvice` 적용됨           |
| **적합한 작업**          | 인코딩, 로깅, CORS, 시큐리티, 본문 캐싱 | 인증·인가 분기, 요청 컨텍스트, 성능 측정  | 로그인 사용자·헤더 값 등 **파라미터 주입** |

**선택 기준 요약**

- 스프링과 무관한 전역 처리(인코딩·압축·본문 래핑)이거나 **DispatcherServlet에 닿기 전에 잘라야 한다** → 필터
- 컨트롤러 메서드 정보(애노테이션·클래스)를 보고 판단해야 한다 → 인터셉터
- 컨트롤러 파라미터를 만들어 주는 것이 목적이다 → ArgumentResolver
- 서비스 계층 메서드 단위의 공통 처리(트랜잭션·캐시)는 이 세 계층이 아니라 **AOP**(unit02)

<br>

### 6. 예외 처리 경계와 흔한 함정

- 필터에서 던진 예외는 DispatcherServlet에 도달하기 전이므로 `@ControllerAdvice`가 잡지 못하고, 톰캣의 오류 처리 → `/error` → `BasicErrorController`로 흐름이 넘어감. 필터에서 JSON 에러 응답을 주려면 **필터 안에서 직접 `response`에 쓰거나**, 예외를 잡아 `HandlerExceptionResolver`에 위임해야 함
- 인터셉터 `preHandle`에서 던진 예외는 DispatcherServlet의 `doDispatch()` 안이므로 `@ControllerAdvice`가 정상적으로 처리함
- 인터셉터의 `postHandle`은 컨트롤러 예외 시 건너뛰므로 자원 해제는 반드시 `afterCompletion`에 둠
- 비동기 컨트롤러(`DeferredResult`, `Callable`)는 인터셉터가 두 번 진입하므로 `AsyncHandlerInterceptor.afterConcurrentHandlingStarted()`를 고려해야 함

```java
// 필터에서 예외를 @ControllerAdvice로 넘기는 패턴
@RequiredArgsConstructor
public class JwtExceptionFilter extends OncePerRequestFilter {
    private final HandlerExceptionResolver handlerExceptionResolver;   // @Qualifier("handlerExceptionResolver")

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        try {
            chain.doFilter(req, res);
        } catch (JwtException e) {
            handlerExceptionResolver.resolveException(req, res, null, e);   // @ExceptionHandler 재사용
        }
    }
}
```

> 💡 "필터와 인터셉터의 차이"에 **실행 위치·핸들러 접근·예외 처리 범위** 세 가지를 들고, "그래서 인증은 어디에 두느냐"에는 "스프링 시큐리티를 쓰면 필터, 직접 구현하면 인터셉터 + ArgumentResolver 조합"이라고 답하면 실무 감각까지 보여줄 수 있다.

<br>

### 7. 정리

- 필터 → DispatcherServlet → 인터셉터 → ArgumentResolver → 컨트롤러 순으로 실행되며, 응답은 **역순**으로 되돌아간다
- **필터**는 서블릿 스펙으로 Request/Response 교체가 가능하지만 스프링 예외 처리 밖에 있다
- **인터셉터**는 핸들러 정보를 알고 `@ControllerAdvice` 안에서 동작하며, `afterCompletion`은 항상 호출된다
- **ArgumentResolver**는 파라미터 주입 전용이며, 인터셉터의 "차단"과 조합해 쓴다
- 메서드 단위 공통 처리는 AOP(unit02), 스프링 시큐리티 필터 체인은 unit10, 예외 응답 표준화는 unit09 참고
