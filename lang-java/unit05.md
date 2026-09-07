## 예외 체계와 설계

Java의 예외는 `Throwable`을 뿌리로 하는 **클래스 계층**이며, 컴파일러가 처리를 강제하는 **Checked 예외**와 강제하지 않는 **Unchecked 예외**로 나뉜다. 어떤 상황에 어떤 종류의 예외를 던질지, 예외를 만드는 데 얼마의 비용이 드는지, 애플리케이션의 예외 계층을 어떻게 설계할지는 코드의 가독성과 성능, 장애 대응 속도를 좌우한다.

<br>

### 1. 예외 클래스 계층

```
 Throwable
 ├── Error                         ← JVM 수준의 심각한 문제. 잡아서 복구하지 않음
 │    ├── OutOfMemoryError
 │    ├── StackOverflowError
 │    └── NoClassDefFoundError
 └── Exception                     ← 애플리케이션이 처리할 수 있는 문제
      ├── IOException              ┐
      ├── SQLException             │ Checked: 컴파일러가 처리(catch 또는 throws)를 강제
      ├── InterruptedException     ┘
      └── RuntimeException         ┐
           ├── NullPointerException           │
           ├── IllegalArgumentException       │ Unchecked: 처리 강제 안 함
           ├── IllegalStateException          │
           ├── IndexOutOfBoundsException      │
           └── UnsupportedOperationException  ┘
```

- **`Error`**: 메모리 부족·스택 오버플로처럼 프로그램이 복구할 수 없는 상황. `catch (Throwable t)`로 삼키면 안 됨 (메모리 구조와 오류는 **unit01** 참고)
- **Checked 예외**: `Exception`의 하위이면서 `RuntimeException`의 하위가 아닌 것. 호출자가 반드시 `try-catch`로 잡거나 `throws`로 선언해야 컴파일됨
- **Unchecked 예외**: `RuntimeException`과 그 하위 클래스. 선언·처리 여부가 자유로움

<br>

### 2. Checked vs Unchecked

| **항목**             | **Checked 예외**                                            | **Unchecked 예외**                                      |
| -------------------- | ----------------------------------------------------------- | ------------------------------------------------------- |
| **상속**             | `Exception` (단, `RuntimeException` 제외)                     | **`RuntimeException`** 하위                              |
| **컴파일러 검사**     | **처리 강제** (catch 또는 throws 필수)                        | 강제 없음                                                |
| **설계 의도**         | 호출자가 **합리적으로 복구할 수 있는** 상황                     | 프로그래밍 오류·전제 조건 위반 (복구보다 수정이 답)          |
| **전파 비용**         | 모든 중간 계층의 시그니처에 `throws`가 번짐                      | 시그니처 오염 없이 상위로 자연스럽게 전파                    |
| **람다·스트림 호환**   | 함수형 인터페이스가 Checked 예외를 선언하지 않아 **매우 불편**    | 문제없음                                                 |
| **대표 예시**         | `IOException`, `SQLException`, `InterruptedException`        | `NullPointerException`, `IllegalArgumentException`      |

**설계 관점의 선택 기준**

- 호출자가 예외를 받고 **실질적으로 다른 행동을 할 수 있으면** Checked (예: 파일이 없으면 기본 설정으로 진행)
- 호출자가 할 수 있는 일이 로그 남기고 실패 응답을 주는 것뿐이면 **Unchecked**
- 현대 프레임워크(Spring, Hibernate 등)는 대부분 Unchecked 계층을 채택하며, `SQLException` 같은 Checked 예외를 `DataAccessException`으로 **번역(translation)**해 던짐

> 💡 Spring의 `@Transactional`은 기본적으로 **Unchecked 예외(`RuntimeException`, `Error`)에서만 롤백**하고 Checked 예외는 커밋한다. "Checked 예외를 던졌는데 왜 롤백이 안 되나요?"는 실무에서 실제로 자주 겪는 문제이며, `rollbackFor` 속성으로 조정할 수 있다.

```java
// 안티패턴: Checked 예외가 시그니처를 타고 모든 계층으로 번짐
public User findUser(Long id) throws SQLException { ... }
public UserDto getUser(Long id) throws SQLException { ... }   // 서비스 계층까지 JDBC에 종속

// 개선: 하위 기술 예외를 도메인 Unchecked 예외로 번역하고 원인(cause)을 보존
public User findUser(Long id) {
    try {
        return jdbc.query(...);
    } catch (SQLException e) {
        throw new DataAccessException("사용자 조회 실패: id=" + id, e);   // 원인 체이닝
    }
}
```

<br>

### 3. 예외 처리 비용

### 3-1. 비용이 발생하는 지점

예외는 "던질 때"보다 **"만들 때"** 비싸다. `Throwable` 생성자는 `fillInStackTrace()`를 호출해 현재 스레드의 **전체 스택 프레임을 순회하며 스택 트레이스를 캡처**하는데, 호출 깊이가 깊을수록 이 비용이 커진다.

```
 new SomeException("msg")
   └─ Throwable() 생성자
        └─ fillInStackTrace()  ← 네이티브 호출로 스택 프레임 전부 순회 (수 μs ~ 수십 μs)
 throw e
   └─ 스택을 거슬러 올라가며 catch 블록 탐색 (예외 테이블 조회)
   └─ 각 프레임의 finally 실행, 프레임 해제
```

- 정상 경로에서는 `try` 블록 진입 자체에 비용이 거의 없음 (JVM은 예외 테이블로 처리하므로 "try가 느리다"는 오해)
- 단순 반환 대비 예외 생성·던지기는 **수백~수천 배** 느릴 수 있으므로 **정상적인 제어 흐름에 예외를 쓰면 안 됨**
- HotSpot은 `NullPointerException` 등 일부 내장 예외가 같은 지점에서 반복되면 스택 트레이스 없는 사전 할당 객체를 재사용하는 최적화(`OmitStackTraceInFastThrow`)를 하며, 이 때문에 로그에 **스택 트레이스가 사라진 예외**가 보이기도 함

<br>

### 3-2. 제어 흐름으로 예외를 쓰는 안티패턴

```java
// 안티패턴: 예외로 반복 종료 — 매 호출마다 스택 트레이스 캡처
try {
    int i = 0;
    while (true) {
        process(items[i++]);
    }
} catch (ArrayIndexOutOfBoundsException e) {
    // 배열 끝에 도달
}

// 개선: 조건 검사로 처리
for (Item item : items) {
    process(item);
}
```

```java
// 안티패턴: 존재 여부 확인에 예외 사용
public boolean exists(String id) {
    try { repository.get(id); return true; }
    catch (NotFoundException e) { return false; }
}

// 개선: Optional 또는 boolean 반환 API 사용
public boolean exists(String id) {
    return repository.find(id).isPresent();
}
```

**스택 트레이스 비용을 줄이는 방법** (정말 필요한 경우에만)

- 예외 클래스에서 `fillInStackTrace()`를 재정의해 `this`를 반환하거나, `Throwable`의 4개 인자 생성자에서 `writableStackTrace=false`로 생성함
- 단, 스택 트레이스가 없으면 **장애 원인 추적이 불가능**해지므로 흐름 제어용이 아닌 진짜 오류에는 절대 적용하지 않음

> ⚠️ "예외가 느리니 쓰지 말라"가 아니라 "**예외적인 상황에만 쓰라**"는 것이 핵심이다. 파일 없음, 네트워크 단절 같은 진짜 예외는 초당 수천 번 발생하지 않으므로 비용이 문제되지 않는다. 문제는 `Integer.parseInt`로 숫자 여부를 판별하는 식의 **반복문 안의 예외**다.

<br>

### 4. 커스텀 예외 계층 설계

### 4-1. 계층 구조

애플리케이션 예외는 **하나의 루트 Unchecked 예외**를 두고, 그 아래를 도메인·원인별로 나누는 것이 일반적이다. 루트가 있으면 전역 예외 처리기에서 "우리 예외"와 "예상 못 한 예외"를 한 번에 구분할 수 있다.

```
 RuntimeException
 └── BusinessException (루트, 에러 코드·HTTP 상태 보유)
      ├── EntityNotFoundException      → 404
      │    └── UserNotFoundException
      ├── InvalidRequestException      → 400
      │    └── DuplicateEmailException
      └── AccessDeniedException        → 403
```

```java
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode, String detail) {
        super(errorCode.message() + " - " + detail);
        this.errorCode = errorCode;
    }

    public BusinessException(ErrorCode errorCode, String detail, Throwable cause) {
        super(errorCode.message() + " - " + detail, cause);   // 원인 보존
        this.errorCode = errorCode;
    }

    public ErrorCode errorCode() { return errorCode; }
}

public class UserNotFoundException extends BusinessException {
    public UserNotFoundException(Long id) {
        super(ErrorCode.USER_NOT_FOUND, "id=" + id);
    }
}

public enum ErrorCode {
    USER_NOT_FOUND(404, "사용자를 찾을 수 없습니다"),
    DUPLICATE_EMAIL(400, "이미 사용 중인 이메일입니다");

    private final int status;
    private final String message;
    ErrorCode(int status, String message) { this.status = status; this.message = message; }
    public int status() { return status; }
    public String message() { return message; }
}
```

<br>

### 4-2. 설계 원칙

| **원칙**                          | **설명**                                                                          |
| --------------------------------- | --------------------------------------------------------------------------------- |
| **추상화 수준에 맞는 예외**         | 서비스 계층이 `SQLException`을 노출하지 않도록 **하위 예외를 상위 개념으로 번역**       |
| **원인 체이닝 유지**               | 번역할 때 반드시 `cause`를 넘겨 원래 스택 트레이스를 보존 (`initCause` 또는 생성자)      |
| **메시지에 진단 정보 포함**         | "실패했습니다"가 아니라 **어떤 값으로 무엇을 하다** 실패했는지 (`id=42`, `email=...`)  |
| **예외 삼키기 금지**               | 빈 `catch` 블록, `catch (Exception e) { log.error(...) }` 후 정상 진행은 장애를 숨김    |
| **표준 예외 재사용**               | 인자 오류는 `IllegalArgumentException`, 상태 오류는 `IllegalStateException`을 우선 고려 |
| **한 곳에서 응답으로 변환**         | 컨트롤러마다 `try-catch` 대신 전역 처리기(`@RestControllerAdvice` 등)로 일괄 변환      |

❗️**`finally`에서 `return` 금지**: `finally` 블록의 `return`은 `try`에서 던진 예외를 **조용히 무시**하고 값을 반환한다. 컴파일러 경고가 있어도 실수하기 쉬운 패턴이다.

<br>

### 5. 자원 해제 — try-with-resources

Java 7부터 `AutoCloseable`을 구현한 자원은 **try-with-resources**로 선언하면 블록이 끝날 때 자동으로 `close()`가 호출된다. 선언 순서의 **역순**으로 닫히며, `close()`에서 발생한 예외는 원래 예외의 **억제된 예외(suppressed)**로 첨부되어 유실되지 않는다.

```java
// 안티패턴: 수동 해제 — close() 예외가 원래 예외를 덮어쓰고, null 검사도 번거로움
Connection conn = null;
try {
    conn = dataSource.getConnection();
    ...
} finally {
    if (conn != null) conn.close();   // 여기서 예외가 나면 try의 예외가 사라짐
}

// 개선: try-with-resources (Java 9+는 이미 선언된 effectively final 변수도 사용 가능)
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    ...
} catch (SQLException e) {
    for (Throwable s : e.getSuppressed()) log.warn("close 중 예외", s);
    throw new DataAccessException("조회 실패", e);
}
```

- **멀티 캐치**(`catch (IOException | SQLException e)`)로 같은 처리를 하는 예외를 묶을 수 있으며, 이때 `e`는 암묵적으로 `final`임
- `InterruptedException`을 잡았다면 `Thread.currentThread().interrupt()`로 **인터럽트 상태를 복원**해야 함 (**unit09** 참고)

> 💡 예외를 잡을 때는 **구체적인 타입부터** 나열해야 한다. 상위 타입을 먼저 쓰면 아래 `catch`는 도달 불가능해져 컴파일 오류가 난다. 또 `catch (Exception e)`로 넓게 잡는 것은 전역 처리기 같은 "마지막 방어선"에서만 허용된다.

<br>

### 6. 면접·실무 체크포인트

| **질문**                                        | **핵심 답변**                                                                                       |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Checked와 Unchecked의 차이는?**                | `RuntimeException` 상속 여부. Checked는 컴파일러가 처리를 강제하고, Unchecked는 강제하지 않음            |
| **어떤 기준으로 선택하는가?**                     | 호출자가 **복구할 수 있으면** Checked, 프로그래밍 오류·복구 불가면 Unchecked. 실무는 대부분 Unchecked     |
| **`Error`와 `Exception`의 차이는?**               | `Error`는 JVM 수준 문제로 복구 대상이 아님. `catch (Throwable)`로 삼키지 말 것                          |
| **예외가 비싼 이유는?**                            | 생성 시 `fillInStackTrace()`가 전체 스택을 캡처. 정상 제어 흐름에 예외를 쓰면 안 되는 이유                |
| **예외 번역과 체이닝은?**                          | 하위 계층 예외를 상위 개념 예외로 바꾸되 `cause`를 넘겨 원래 스택 트레이스를 보존                          |
| **try-with-resources의 장점은?**                  | 자동 `close()`, 역순 해제, `close()` 예외가 **억제 예외**로 보존되어 원래 예외가 유실되지 않음             |
| **Spring `@Transactional` 롤백 기준은?**           | 기본은 Unchecked 예외만 롤백. Checked는 `rollbackFor` 지정 필요                                        |

- 람다에서 Checked 예외가 불편한 이유는 **unit07**, 스레드 인터럽트 처리는 **unit09**에서 이어서 다룸
