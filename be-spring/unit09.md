## 예외 처리와 응답 규약

API 서버에서 예외는 "어디서 잡아 어떤 형식으로 응답할 것인가"가 통일되지 않으면 클라이언트마다 다른 에러 파싱 로직을 갖게 되고, 내부 스택 트레이스가 그대로 노출되는 보안 문제로도 이어진다. 스프링 MVC는 **HandlerExceptionResolver**와 **@ControllerAdvice**로 예외를 한 곳에서 응답으로 변환하는 구조를 제공하므로, 이 위에 **예외 계층과 에러 응답 표준**을 설계하는 방법을 정리한다.

<br>

### 1. 스프링 MVC의 예외 처리 흐름

컨트롤러(또는 인터셉터·ArgumentResolver)에서 던져진 예외는 `DispatcherServlet.doDispatch()`가 잡아 **HandlerExceptionResolver 체인**에 넘긴다. 체인은 순서대로 시도하며, 먼저 처리에 성공한 리졸버의 결과를 응답으로 사용한다.

```
컨트롤러 예외 발생
      ↓
① ExceptionHandlerExceptionResolver   → @ExceptionHandler 메서드 탐색 (컨트롤러 내부 → @ControllerAdvice)
      ↓ (미처리)
② ResponseStatusExceptionResolver     → @ResponseStatus 애노테이션, ResponseStatusException 처리
      ↓ (미처리)
③ DefaultHandlerExceptionResolver     → 스프링 내부 예외를 표준 상태 코드로 (400, 405, 415 ...)
      ↓ (미처리)
서블릿 컨테이너로 전파 → /error 재요청 → BasicErrorController (스프링 부트 기본 에러 응답)
```

- `@ExceptionHandler`는 컨트롤러 안에 두면 그 컨트롤러에만, `@ControllerAdvice`에 두면 **전역**에 적용됨. 같은 예외를 둘 다 처리하면 **컨트롤러 내부가 우선**
- 여러 `@ExceptionHandler`가 매칭되면 예외 클래스 계층에서 **가장 가까운(구체적인) 타입**이 선택됨
- 필터에서 던진 예외는 DispatcherServlet 밖이므로 이 체인을 타지 않고 바로 `BasicErrorController`로 감 (unit04 참고)

> 💡 "@ControllerAdvice는 어떻게 동작하는가"를 물으면 "`ExceptionHandlerExceptionResolver`가 기동 시 `@ControllerAdvice` 빈의 `@ExceptionHandler` 메서드를 예외 타입별로 캐싱해 두고, 예외 발생 시 가장 구체적인 핸들러를 골라 호출한다"고 답한다. 이 리졸버가 체인의 **첫 번째**라는 점이 핵심이다.

<br>

### 2. @ControllerAdvice와 @RestControllerAdvice

```java
@RestControllerAdvice(basePackages = "com.app.api")     // 적용 범위 제한 가능 (annotations, assignableTypes)
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException e) {
        ErrorCode code = e.getErrorCode();
        return ResponseEntity.status(code.getStatus())
                .body(ErrorResponse.of(code));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)   // @Valid 실패
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        return ResponseEntity.badRequest()
                .body(ErrorResponse.of(ErrorCode.INVALID_INPUT, e.getBindingResult()));
    }

    @ExceptionHandler(Exception.class)                         // 최후 방어선
    public ResponseEntity<ErrorResponse> handleUnknown(Exception e) {
        log.error("Unhandled exception", e);                   // 내부 정보는 로그로만
        return ResponseEntity.internalServerError()
                .body(ErrorResponse.of(ErrorCode.INTERNAL_ERROR));
    }
}
```

- `@RestControllerAdvice` = `@ControllerAdvice + @ResponseBody`. 반환값이 `HttpMessageConverter`로 직렬화됨
- 여러 Advice가 있으면 `@Order`로 우선순위를 정함. 범위가 좁은 Advice를 앞에 둠
- `ResponseEntityExceptionHandler`를 상속하면 스프링 MVC 내부 예외(`HttpRequestMethodNotSupportedException`, `MethodArgumentNotValidException` 등) 처리 메서드가 이미 정의되어 있어 필요한 것만 오버라이드할 수 있음

> ⚠️ `Exception.class`를 잡는 최후 핸들러는 반드시 두되, 원인을 **로그로 남기고 응답에는 일반화된 메시지만** 내보낸다. `e.getMessage()`나 스택 트레이스를 그대로 응답에 담으면 내부 테이블명·경로가 노출되어 보안 취약점이 된다.

<br>

### 3. 예외 계층 설계

**비즈니스 예외 하나 + 에러 코드 열거형** 조합이 실무에서 가장 흔한 구조다. 예외 클래스를 무한히 늘리지 않으면서 상태 코드·메시지를 한 곳에서 관리할 수 있다.

```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {
    // 4xx: 클라이언트 원인
    INVALID_INPUT(HttpStatus.BAD_REQUEST, "C001", "입력값이 올바르지 않습니다"),
    UNAUTHORIZED(HttpStatus.UNAUTHORIZED, "C002", "인증이 필요합니다"),
    ORDER_NOT_FOUND(HttpStatus.NOT_FOUND, "O001", "주문을 찾을 수 없습니다"),
    INSUFFICIENT_STOCK(HttpStatus.CONFLICT, "O002", "재고가 부족합니다"),
    // 5xx: 서버 원인
    INTERNAL_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "S001", "일시적인 오류가 발생했습니다");

    private final HttpStatus status;
    private final String code;
    private final String message;
}

@Getter
public class BusinessException extends RuntimeException {     // 런타임 예외 → 트랜잭션 롤백 (unit05)
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
}

public class OrderNotFoundException extends BusinessException {   // 필요한 경우에만 세분화
    public OrderNotFoundException(Long id) {
        super(ErrorCode.ORDER_NOT_FOUND);
    }
}
```

```
RuntimeException
   └─ BusinessException (ErrorCode 보유)
        ├─ EntityNotFoundException 계열   → 404
        ├─ InvalidValueException 계열     → 400
        └─ 도메인별 예외 (InsufficientStockException ...)  → 409 등
```

**설계 원칙**

- 커스텀 예외는 **`RuntimeException` 기반**으로: 서비스 시그니처가 `throws`로 오염되지 않고, `@Transactional` 기본 롤백 규칙에 맞음
- **에러 코드는 상태 코드보다 세밀하게**: HTTP 404 하나로는 "주문 없음"과 "회원 없음"을 구분할 수 없으므로 애플리케이션 코드(`O001`)를 별도로 둠
- 계층 간 예외 변환: 리포지토리의 `DataAccessException`, 외부 API의 `HttpClientErrorException`을 서비스 경계에서 **비즈니스 예외로 감싸** 상위 계층이 인프라 기술을 몰라도 되게 함
- 흐름 제어에 예외를 쓰지 않음 — 존재 여부 확인은 `Optional`·`boolean` 반환으로

<br>

### 4. 에러 응답 표준화

```java
@Getter
@Builder
public class ErrorResponse {
    private final String code;                   // 애플리케이션 에러 코드 ("O001")
    private final String message;                // 사용자에게 보여줄 메시지
    private final LocalDateTime timestamp;
    private final List<FieldError> errors;       // 검증 실패 시 필드별 상세, 없으면 빈 배열

    public static ErrorResponse of(ErrorCode errorCode) {
        return of(errorCode, List.of());
    }

    public static ErrorResponse of(ErrorCode errorCode, BindingResult bindingResult) {
        List<FieldError> fieldErrors = bindingResult.getFieldErrors().stream()
                .map(fe -> new FieldError(fe.getField(), String.valueOf(fe.getRejectedValue()), fe.getDefaultMessage()))
                .toList();
        return of(errorCode, fieldErrors);
    }

    private static ErrorResponse of(ErrorCode errorCode, List<FieldError> errors) {
        return ErrorResponse.builder()
                .code(errorCode.getCode()).message(errorCode.getMessage())
                .timestamp(LocalDateTime.now()).errors(errors).build();
    }

    public record FieldError(String field, String value, String reason) {}
}
```

| **항목**            | **자체 표준 (ErrorResponse)**                    | **RFC 7807 Problem Details (Spring 6+)**                 |
| ------------------- | ------------------------------------------------ | -------------------------------------------------------- |
| **형식**            | 팀이 정의한 JSON 구조                             | `type`, `title`, `status`, `detail`, `instance` 표준 필드 |
| **Content-Type**    | `application/json`                               | **`application/problem+json`**                           |
| **스프링 지원**     | 직접 구현                                        | `ProblemDetail` 클래스, `ErrorResponseException`, `spring.mvc.problemdetails.enabled=true` |
| **장점**            | 기존 클라이언트와 호환, 자유로운 필드             | 업계 표준, 확장 필드(`properties`) 추가 가능              |
| **선택 기준**       | 사내 규약이 이미 있을 때                          | 신규 API, 외부 공개 API                                   |

```java
// Spring 6 ProblemDetail 사용 예시
@ExceptionHandler(BusinessException.class)
public ProblemDetail handleBusiness(BusinessException e) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(e.getErrorCode().getStatus(), e.getMessage());
    problem.setTitle(e.getErrorCode().name());
    problem.setProperty("code", e.getErrorCode().getCode());     // 확장 필드
    return problem;
}
```

**응답 규약 체크리스트**

- **HTTP 상태 코드는 의미대로**: 성공을 200으로 보내면서 본문에 `"success": false`를 담는 방식은 캐시·모니터링·클라이언트 처리 모두를 어렵게 함
- 성공·실패 응답의 **최상위 구조를 일관되게** 유지하고, 에러 코드 목록을 문서화해 클라이언트와 공유함
- 검증 실패(400)는 **필드 단위 상세**를 포함해 프런트엔드가 폼에 매핑할 수 있게 함

<br>

### 5. 검증 예외와 스프링 내부 예외

| **예외**                                   | **발생 상황**                              | **권장 상태 코드** |
| ------------------------------------------ | ------------------------------------------ | ------------------ |
| **MethodArgumentNotValidException**        | `@RequestBody` + `@Valid` 실패             | 400                |
| **BindException**                          | `@ModelAttribute` 바인딩·검증 실패          | 400                |
| **HandlerMethodValidationException**       | `@RequestParam`·`@PathVariable` 제약 검증 실패 (Spring 6.1+) | 400     |
| **HttpMessageNotReadableException**        | JSON 파싱 실패, 타입 불일치                | 400                |
| **MissingServletRequestParameterException**| 필수 파라미터 누락                          | 400                |
| **HttpRequestMethodNotSupportedException** | 지원하지 않는 HTTP 메서드                   | 405                |
| **NoResourceFoundException**               | 매핑 없는 경로 (Spring 6.1+)               | 404                |
| **AccessDeniedException**                  | 시큐리티 인가 실패 (`@PreAuthorize`)        | 403                |

- `@ResponseStatus`를 예외 클래스에 붙이면 별도 핸들러 없이 상태 코드가 결정되지만, 응답 본문 형식을 제어할 수 없어 표준 응답 구조와 어울리지 않음
- Spring 6.1부터 `@RequestParam`·`@PathVariable`에 붙은 제약 애노테이션(`@Min` 등) 검증은 컨트롤러 클래스에 `@Validated` 없이도 동작하며, 실패 시 `HandlerMethodValidationException`이 발생함 (버전에 따라 다름)

<br>

### 6. 흔한 안티패턴

```java
// 안티패턴 ①: 컨트롤러마다 try-catch → 중복, 누락, 형식 불일치
@GetMapping("/{id}")
public ResponseEntity<?> find(@PathVariable Long id) {
    try {
        return ResponseEntity.ok(orderService.find(id));
    } catch (OrderNotFoundException e) {
        return ResponseEntity.status(404).body(Map.of("error", e.getMessage()));
    } catch (Exception e) {
        return ResponseEntity.status(500).body(e.toString());      // 스택 정보 노출
    }
}

// 개선: 컨트롤러는 정상 흐름만, 예외는 @RestControllerAdvice가 표준 형식으로 변환
@GetMapping("/{id}")
public OrderResponse find(@PathVariable Long id) {
    return orderService.find(id);                                  // 없으면 OrderNotFoundException
}
```

- **안티패턴 ②**: 서비스에서 예외를 잡아 `null`을 반환 → 호출자가 NPE로 실패 원인을 잃음
- **안티패턴 ③**: 필터(JWT 검증 등)에서 던진 예외를 `@ControllerAdvice`가 잡을 거라 기대 → 필터는 체인 밖이므로 `HandlerExceptionResolver`에 직접 위임하거나 필터 안에서 응답을 써야 함 (unit04·unit10 참고)
- **안티패턴 ④**: 체크 예외를 던지면서 `@Transactional` 롤백을 기대 → 기본은 커밋됨 (unit05 참고)

> 💡 "예외 처리 전략을 어떻게 설계했는가"는 프로젝트 경험 질문으로 자주 나온다. **"런타임 기반 BusinessException + ErrorCode enum + @RestControllerAdvice + 표준 ErrorResponse"** 네 가지를 구조로 설명하고, 필터 예외와 최후 방어선 로깅까지 언급하면 충분하다.

<br>

### 7. 정리

- 예외는 `HandlerExceptionResolver` 체인이 처리하며, `@ExceptionHandler`(`ExceptionHandlerExceptionResolver`)가 **첫 번째**로 시도된다
- `@RestControllerAdvice`에 전역 핸들러를 두고, 컨트롤러는 정상 흐름만 작성한다
- 커스텀 예외는 **`RuntimeException` 기반 + ErrorCode enum**으로 설계해 상태 코드·메시지를 한 곳에서 관리한다
- 에러 응답은 **일관된 구조**(code·message·timestamp·errors)로 표준화하고, 신규 API라면 RFC 7807 `ProblemDetail`을 검토한다
- 최후 방어선(`Exception.class`)은 내부 정보를 로그로만 남기고 응답에는 일반화된 메시지만 담는다
- 필터 예외는 체인 밖이며, 트랜잭션 롤백은 예외 타입에 좌우된다 (unit04·unit05·unit10 참고)
