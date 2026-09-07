## 요청 처리 흐름

스프링 MVC는 **DispatcherServlet**이라는 단일 서블릿이 모든 HTTP 요청을 받아 컨트롤러를 찾고(HandlerMapping), 호출하고(HandlerAdapter), 결과를 응답으로 변환하는 **프런트 컨트롤러(Front Controller) 패턴**으로 동작한다. 이 흐름을 단계별로 알아야 "`@RequestBody`는 어디서 변환되는가", "404는 누가 던지는가", "인터셉터는 어느 시점에 끼어드는가"에 답할 수 있다.

<br>

### 1. 서블릿 컨테이너와 DispatcherServlet

- **서블릿(Servlet)**은 자바 웹 표준의 요청 처리 단위이며, 톰캣 같은 **서블릿 컨테이너**가 소켓 연결·스레드 할당·`HttpServletRequest/Response` 생성을 담당함
- 스프링 이전에는 URL마다 서블릿을 하나씩 만들어 `web.xml`에 매핑했으나, 공통 처리(인코딩·예외·뷰 렌더링)가 서블릿마다 중복됨
- DispatcherServlet은 `HttpServlet`을 상속한 **하나의 서블릿**으로 모든 요청을 받은 뒤, 실제 처리는 컨트롤러 빈에 위임함 → 공통 처리는 한 곳, 비즈니스 처리는 컨트롤러로 분리
- 스프링 부트는 `DispatcherServletAutoConfiguration`이 DispatcherServlet을 빈으로 만들어 `/` 경로에 자동 등록함 (`spring.mvc.servlet.path`로 변경 가능)

```
[클라이언트] ──HTTP──▶ [톰캣: 스레드 할당, Request/Response 생성]
                              │
                              ▼
                       [Filter 체인]  (서블릿 스펙, unit04 참고)
                              │
                              ▼
                       [DispatcherServlet]  ── 스프링 MVC의 시작점
```

> 💡 "서블릿 컨테이너와 스프링 컨테이너의 관계"를 묻는 질문에는, 톰캣이 요청을 받아 **DispatcherServlet(서블릿)**에 넘기고, DispatcherServlet이 **ApplicationContext(스프링 컨테이너)**에서 컨트롤러 빈을 찾아 호출한다고 답한다. 두 컨테이너는 계층이 다르다.

<br>

### 2. 전체 요청 처리 흐름

```
① 요청 수신          DispatcherServlet.doDispatch()
        ↓
② 핸들러 조회        HandlerMapping ──▶ HandlerExecutionChain (핸들러 + 인터셉터 목록)
        ↓
③ 어댑터 조회        HandlerAdapter.supports(handler) 로 호출 가능한 어댑터 선택
        ↓
④ preHandle          인터셉터 전처리 (false 반환 시 여기서 종료)
        ↓
⑤ 핸들러 호출        HandlerAdapter.handle() ── ArgumentResolver로 파라미터 조립 → 컨트롤러 실행
        ↓                                     └─ ReturnValueHandler로 반환값 처리
⑥ postHandle         인터셉터 후처리 (ModelAndView 접근 가능)
        ↓
⑦ 응답 생성          @ResponseBody → HttpMessageConverter / 뷰 이름 → ViewResolver → View.render()
        ↓
⑧ afterCompletion    인터셉터 마무리 (예외 발생 여부와 무관하게 실행)

   ※ ⑤~⑦ 중 예외 발생 시  →  HandlerExceptionResolver 체인 (unit09 참고)
```

- `doDispatch()`가 이 순서를 코드로 구현한 메서드이며, 각 단계는 **인터페이스에 위임**되어 있어 구현체 교체가 가능함
- 핸들러 조회부터 응답 생성까지 모두 **요청을 받은 톰캣 스레드 하나**에서 실행됨 (스레드 자원 관점은 unit11 참고)

<br>

### 3. HandlerMapping — 어떤 컨트롤러가 처리할지 찾기

HandlerMapping은 요청(URL·HTTP 메서드·헤더 등)을 보고 **처리할 핸들러 객체**를 반환한다. 여러 구현체가 우선순위 순서대로 등록되어 있고, 먼저 매칭된 것을 사용한다.

| **구현체**                        | **매핑 기준**                                 | **대표 대상**                                  |
| --------------------------------- | --------------------------------------------- | ---------------------------------------------- |
| **RequestMappingHandlerMapping**  | `@RequestMapping` 계열 애노테이션              | `@Controller`·`@RestController`의 메서드 (**HandlerMethod**) |
| **BeanNameUrlHandlerMapping**     | 빈 이름이 URL 패턴(`/hello`)인 경우            | 레거시 `Controller` 인터페이스 구현체          |
| **SimpleUrlHandlerMapping**       | 명시적 URL → 핸들러 매핑                       | 정적 리소스(`ResourceHttpRequestHandler`) 등    |

- 반환값은 핸들러 하나가 아니라 **HandlerExecutionChain**으로, 매칭된 인터셉터 목록까지 함께 들어 있음
- `RequestMappingHandlerMapping`은 기동 시 모든 `@RequestMapping` 메서드를 스캔해 `RequestMappingInfo → HandlerMethod` 맵을 만들어 두고, 요청마다 이 맵에서 조회함
- 경로 매칭은 Spring 5.3에서 도입된 `PathPatternParser`가 **Boot 2.6부터 기본값**이며, 이전 `AntPathMatcher`보다 빠르고 `**`는 패턴 끝에만 허용됨

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    // RequestMappingInfo: GET + /api/orders/{id} + produces=application/json
    @GetMapping(value = "/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
    public OrderResponse find(@PathVariable Long id) {   // 이 메서드 자체가 HandlerMethod
        return orderService.find(id);
    }
}
```

> ⚠️ 매칭되는 핸들러가 없으면 스프링 부트는 기본적으로 `BasicErrorController`가 404 응답을 만든다. `@ControllerAdvice`로 404를 직접 잡으려면 `spring.mvc.throw-exception-if-no-handler-found=true`로 `NoHandlerFoundException`을 던지게 해야 했으나, **Spring 6.1·Boot 3.2부터는 정적 리소스 미존재 시 `NoResourceFoundException`이 던져져** 별도 설정 없이도 `@ExceptionHandler`로 처리할 수 있다. 버전에 따라 동작이 다르므로 확인이 필요하다.

<br>

### 4. HandlerAdapter — 찾은 핸들러를 어떻게 호출할지

핸들러의 형태가 제각각(애노테이션 메서드, `Controller` 인터페이스, `HttpRequestHandler`)이므로 DispatcherServlet이 직접 호출하지 않고, **어댑터 패턴**으로 호출 방법을 추상화한다. DispatcherServlet은 등록된 어댑터를 순회하며 `supports(handler)`가 `true`인 것을 골라 `handle()`을 호출한다.

| **구현체**                        | **지원 핸들러**                    | **비고**                                        |
| --------------------------------- | ---------------------------------- | ----------------------------------------------- |
| **RequestMappingHandlerAdapter**  | `HandlerMethod`                    | **애노테이션 컨트롤러 전용**, 가장 복잡·중요    |
| **HttpRequestHandlerAdapter**     | `HttpRequestHandler`               | 정적 리소스, 서블릿 스타일 핸들러               |
| **SimpleControllerHandlerAdapter**| 레거시 `Controller` 인터페이스     | `ModelAndView` 반환                             |

**RequestMappingHandlerAdapter 내부 동작**

```
handle(request, response, handlerMethod)
   ├─ ① HandlerMethodArgumentResolver 목록 순회
   │      @PathVariable → PathVariableMethodArgumentResolver
   │      @RequestParam → RequestParamMethodArgumentResolver
   │      @RequestBody  → RequestResponseBodyMethodProcessor (HttpMessageConverter로 역직렬화)
   │      @ModelAttribute, HttpServletRequest, Principal, 커스텀 리졸버 ...
   ├─ ② 조립된 인자로 컨트롤러 메서드 리플렉션 호출
   └─ ③ HandlerMethodReturnValueHandler 목록 순회
          @ResponseBody / ResponseEntity → HttpMessageConverter로 직렬화 후 응답 본문에 기록
          String(뷰 이름)               → ModelAndView 로 감싸 DispatcherServlet에 반환
```

- `@RequestBody` JSON 변환은 어댑터가 아니라 **HttpMessageConverter**(`MappingJackson2HttpMessageConverter`)가 담당하며, 요청의 `Content-Type`과 파라미터 타입을 보고 컨버터를 선택함
- 컨트롤러 메서드에 도달하기 전 `@Valid` 검증도 이 단계(ArgumentResolver 내부)에서 수행되어 실패 시 `MethodArgumentNotValidException`이 던져짐
- ArgumentResolver를 직접 구현해 로그인 사용자 같은 커스텀 파라미터를 주입하는 방법은 unit04 참고

<br>

### 5. 응답 처리 — @ResponseBody vs 뷰 렌더링

| **항목**            | **@ResponseBody / @RestController**            | **뷰 이름 반환 (@Controller)**                  |
| ------------------- | ---------------------------------------------- | ----------------------------------------------- |
| **반환값 처리**     | ReturnValueHandler가 **즉시 응답 본문에 기록**  | `ModelAndView`로 DispatcherServlet에 전달       |
| **변환 주체**       | **HttpMessageConverter** (JSON·XML·문자열)     | **ViewResolver** → `View.render()` (Thymeleaf 등) |
| **Content-Type**    | `Accept` 헤더와 컨버터 협상(Content Negotiation) | 템플릿 엔진이 결정 (`text/html`)                |
| **postHandle 시점** | 이미 응답이 쓰인 뒤라 **본문 수정 불가**        | 렌더링 전이라 모델 수정 가능                    |

```java
// ResponseEntity로 상태 코드·헤더·본문을 명시적으로 제어 (REST API에서 권장)
@PostMapping
public ResponseEntity<OrderResponse> create(@Valid @RequestBody OrderCreateRequest request) {
    OrderResponse created = orderService.create(request);
    return ResponseEntity
            .created(URI.create("/api/orders/" + created.id()))   // 201 + Location 헤더
            .body(created);
}
```

> 💡 `@RestController`는 `@Controller + @ResponseBody`의 조합일 뿐 별도의 처리 경로가 아니다. 두 경우 모두 같은 HandlerMapping·HandlerAdapter를 거치고, **반환값을 다루는 ReturnValueHandler만 달라진다**고 설명하면 정확하다.

<br>

### 6. 확장 지점 정리

스프링 MVC의 각 단계는 인터페이스이므로, 원하는 지점을 골라 확장할 수 있다. 어떤 문제를 어디서 해결해야 하는지가 핵심 선택 기준이다.

| **하고 싶은 일**                          | **확장 지점**                     | **등록 방법**                                   |
| ----------------------------------------- | --------------------------------- | ----------------------------------------------- |
| **요청/응답 본문 감싸기, 인코딩, 로깅**    | `Filter`                          | `FilterRegistrationBean`, `@Component`          |
| **컨트롤러 호출 전후 공통 처리(인증 체크)** | `HandlerInterceptor`              | `WebMvcConfigurer.addInterceptors()`            |
| **컨트롤러 파라미터에 커스텀 객체 주입**   | `HandlerMethodArgumentResolver`   | `WebMvcConfigurer.addArgumentResolvers()`       |
| **JSON 직렬화 방식 변경**                 | `HttpMessageConverter`            | `WebMvcConfigurer.configureMessageConverters()` |
| **예외를 응답으로 변환**                  | `HandlerExceptionResolver`        | `@ControllerAdvice` + `@ExceptionHandler`       |
| **CORS 정책**                             | `CorsConfiguration`               | `WebMvcConfigurer.addCorsMappings()`            |

<br>

### 7. 면접·실무 체크포인트

- DispatcherServlet은 **프런트 컨트롤러**로, 톰캣이 넘긴 요청을 받아 HandlerMapping → HandlerAdapter → 응답 변환 순으로 **위임**만 한다
- **HandlerMapping**은 "누가 처리할지"(핸들러 + 인터셉터 체인), **HandlerAdapter**는 "어떻게 호출할지"를 담당한다
- 애노테이션 컨트롤러는 `RequestMappingHandlerMapping`이 찾고 `RequestMappingHandlerAdapter`가 호출하며, 파라미터 조립은 **ArgumentResolver**, 반환값 처리는 **ReturnValueHandler**가 담당한다
- `@RequestBody`·`@ResponseBody`의 JSON 변환은 **HttpMessageConverter**가, 뷰 렌더링은 **ViewResolver**가 담당한다
- 요청 처리 전 과정은 **톰캣 스레드 하나**에서 동기적으로 진행되며, 예외는 `HandlerExceptionResolver`에서 응답으로 변환된다 (unit09 참고)
- 필터·인터셉터·ArgumentResolver 중 무엇을 고를지는 **unit04** 참고
