## AOP와 프록시 동작

**AOP(Aspect-Oriented Programming, 관점 지향 프로그래밍)**는 트랜잭션·로깅·보안처럼 여러 클래스에 흩어지는 공통 관심사(cross-cutting concern)를 핵심 로직에서 분리해 한 곳에서 관리하는 기법이다. 스프링 AOP는 **프록시(Proxy) 객체**로 구현되므로, 프록시가 어떻게 만들어지고 어디서 호출을 가로채는지 알아야 `@Transactional`·`@Cacheable`·`@Async`가 "동작하지 않는" 상황을 설명할 수 있다.

<br>

### 1. AOP 핵심 용어

| **용어**             | **의미**                                                              | **스프링에서의 예시**                      |
| -------------------- | --------------------------------------------------------------------- | ------------------------------------------ |
| **Aspect**           | 공통 관심사를 모듈화한 단위 (Advice + Pointcut)                        | `@Aspect` 클래스                           |
| **Join Point**       | Advice를 적용할 수 있는 지점                                          | 스프링 AOP는 **메서드 실행**만 지원        |
| **Pointcut**         | Join Point 중 실제로 적용할 대상을 고르는 표현식                       | `execution(* com.app..*Service.*(..))`     |
| **Advice**           | 실제로 수행되는 부가 기능 코드                                        | `@Before`, `@Around`, `@AfterReturning` 등 |
| **Target**           | Advice가 적용되는 원본 객체                                           | `OrderService` 인스턴스                    |
| **Weaving**          | Aspect를 대상 코드에 결합하는 과정                                    | 스프링 AOP는 **런타임 프록시 방식**        |

```java
@Aspect
@Component
public class ExecutionTimeAspect {

    @Around("execution(* com.app.order.service..*(..))")
    public Object measure(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();                       // 원본 메서드 호출
        } finally {
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            log.info("{} took {} ms", pjp.getSignature().toShortString(), elapsed);
        }
    }
}
```

> 💡 AspectJ는 컴파일·로드 시점에 바이트코드를 직접 수정(weaving)하므로 필드 접근·생성자 호출까지 가로챌 수 있다. 스프링 AOP는 AspectJ의 **포인트컷 표현식 문법만 빌려 쓰고**, 실제 적용은 프록시로 한다. "스프링 AOP와 AspectJ 차이"를 물으면 이 지점을 답한다.

<br>

### 2. 프록시 패턴 — AOP가 동작하는 원리

프록시는 원본 객체와 **같은 타입으로 보이는 대리 객체**다. 클라이언트는 프록시를 원본으로 알고 호출하고, 프록시는 부가 기능을 수행한 뒤 원본에 위임한다.

```
클라이언트 ──호출──▶ [프록시: OrderService$$SpringCGLIB]
                        │  ① Advice 전처리 (트랜잭션 시작 등)
                        │  ② target.createOrder() 위임
                        │  ③ Advice 후처리 (커밋 / 롤백)
                        ▼
                   [원본: OrderService]
```

- 컨테이너는 빈 초기화 마지막 단계(`postProcessAfterInitialization`, unit01 참고)에서 포인트컷에 매칭되는 빈을 프록시로 감싸 **프록시를 빈으로 등록**함
- 따라서 다른 빈에 주입되는 것은 항상 프록시이며, `@Transactional`·`@Cacheable`·`@Async`·`@PreAuthorize`·`@Retryable`은 모두 이 구조 위에서 동작함

<br>

### 3. JDK Dynamic Proxy vs CGLIB

| **항목**               | **JDK Dynamic Proxy**                              | **CGLIB**                                            |
| ---------------------- | -------------------------------------------------- | ---------------------------------------------------- |
| **생성 방식**          | `java.lang.reflect.Proxy` — **인터페이스 구현체** 생성 | 바이트코드 조작으로 **대상 클래스의 서브클래스** 생성 |
| **전제 조건**          | 대상이 **인터페이스를 구현**해야 함                 | 클래스·메서드가 `final`이 아니어야 함                |
| **주입 가능 타입**     | 인터페이스 타입으로만 주입 가능                     | 구체 클래스 타입으로도 주입 가능                     |
| **호출 가로채기**      | `InvocationHandler.invoke()`                        | `MethodInterceptor.intercept()`                      |
| **private 메서드**     | 인터페이스에 없으므로 불가                          | 오버라이드 불가하므로 **불가**                       |
| **스프링 부트 기본값** | -                                                  | **Boot 2.0+ 기본** (`spring.aop.proxy-target-class=true`) |

```java
// JDK Dynamic Proxy의 핵심 구조 (개념 예시)
OrderService proxy = (OrderService) Proxy.newProxyInstance(
        loader,
        new Class[]{OrderService.class},            // 인터페이스 배열
        (p, method, args) -> {
            System.out.println("before " + method.getName());
            Object result = method.invoke(target, args);   // 원본 호출
            System.out.println("after");
            return result;
        });
```

- 스프링 부트는 인터페이스가 있어도 CGLIB를 쓴다. 인터페이스 타입·구체 클래스 타입 어느 쪽으로 주입하든 동작해야 하고, 두 방식의 동작을 통일하기 위해서임
- CGLIB 프록시는 부모(원본) 생성자를 호출하지 않고 **Objenesis**로 인스턴스를 만들므로, 기본 생성자가 없어도 되지만 프록시 객체의 **필드는 초기화되지 않은 상태(null)**임

> ⚠️ 프록시가 필드를 갖지 않는다는 점 때문에, 주입받은 빈의 **public 필드를 직접 읽으면** null이 나온다. 원본 필드 값은 반드시 메서드를 통해 접근해야 한다. 같은 이유로 `final` 클래스·`final` 메서드에는 CGLIB 프록시(AOP·`@Transactional`)를 적용할 수 없다.

<br>

### 4. self-invocation — 프록시가 우회되는 지점

프록시는 **외부에서 들어오는 호출**만 가로챈다. 원본 객체 안에서 `this.other()`로 자기 메서드를 부르면, `this`는 프록시가 아니라 원본이므로 Advice가 적용되지 않는다.

```
외부 → 프록시.placeOrder()  ─▶ [Advice 적용] ─▶ 원본.placeOrder()
                                                     │
                                                     └─ this.saveHistory()  ─▶ 원본.saveHistory()  ✗ Advice 없음
```

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        // ... 주문 저장
        saveHistory(order);          // 내부 호출 → @Transactional(REQUIRES_NEW) 무시됨
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveHistory(Order order) { /* ... */ }
}
```

**해결 방법과 선택 기준**

| **방법**                                   | **설명**                                                        | **평가**                                     |
| ------------------------------------------ | --------------------------------------------------------------- | -------------------------------------------- |
| **별도 빈으로 분리**                       | `saveHistory`를 `OrderHistoryService`로 옮겨 프록시를 거치게 함 | **가장 권장** — 책임 분리로 설계도 개선됨    |
| **자기 자신 주입**                         | `ObjectProvider<OrderService>` 또는 `@Lazy`로 프록시를 주입받아 `self.saveHistory()` 호출 | 동작하지만 순환 구조가 어색함 (unit01 참고)  |
| **`AopContext.currentProxy()`**            | `@EnableAspectJAutoProxy(exposeProxy = true)` 후 현재 프록시 조회 | 코드가 AOP 인프라에 의존하게 됨              |
| **AspectJ 위빙**                           | 컴파일·로드 시점에 바이트코드를 직접 수정                        | 프록시 한계는 없지만 빌드·설정 복잡도 증가   |

```java
// 개선: 별도 빈으로 분리 → 외부 호출이 되어 프록시를 거침
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderHistoryService historyService;

    public void placeOrder(Order order) {
        // ... 주문 저장
        historyService.saveHistory(order);   // 프록시 경유 → REQUIRES_NEW 정상 적용
    }
}
```

> 💡 self-invocation은 `@Transactional`뿐 아니라 `@Cacheable`, `@Async`, `@Retryable`, `@PreAuthorize` 등 **프록시 기반 애노테이션 전부**에 해당한다. "캐시가 안 먹어요", "비동기가 동기로 도는데요" 문제의 상당수가 이 원인이다.

<br>

### 5. Advice 종류와 실행 순서

| **Advice**          | **실행 시점**                          | **특징**                                         |
| ------------------- | -------------------------------------- | ------------------------------------------------ |
| **@Before**         | 대상 메서드 실행 전                    | 인자 조회 가능, 실행 자체는 막지 못함(예외로만)  |
| **@AfterReturning** | 정상 반환 후                           | 반환값 조회 가능 (수정은 불가)                   |
| **@AfterThrowing**  | 예외 발생 후                           | 예외 로깅·변환에 사용                            |
| **@After**          | 정상·예외 무관하게 종료 후             | `finally`에 해당                                 |
| **@Around**         | 실행 전후 전체를 감쌈                  | **가장 강력** — 실행 여부·인자·반환값 모두 제어  |

- 여러 Aspect가 한 메서드에 적용되면 `@Order` 값이 **낮을수록 바깥쪽**에서 실행됨 (먼저 시작, 나중에 끝남)
- 트랜잭션 Advice의 순서는 `Ordered.LOWEST_PRECEDENCE`가 기본이므로, 커스텀 Aspect는 대개 트랜잭션 **바깥**에서 실행됨. 트랜잭션 안쪽에서 실행돼야 하면 `@EnableTransactionManagement(order = ...)`로 조정함

```
@Order(1) LoggingAspect  ─┐
   @Order(2) AuthAspect  ─┼─┐
      TransactionInterceptor ─┼─┐
             원본 메서드       │ │ │
      ◀────────────────────────┘ │ │
   ◀────────────────────────────┘ │
◀──────────────────────────────────┘
```

<br>

### 6. 프록시 기반 AOP의 한계 정리

- **메서드 실행 Join Point만 지원**: 필드 접근·생성자·정적 메서드는 가로챌 수 없음
- **self-invocation 무시**: 같은 객체 내부 호출은 프록시를 거치지 않음
- **가시성 제약**: JDK 프록시는 인터페이스 메서드(public)만, CGLIB는 `private`·`final`·`static` 메서드 불가. `@Transactional`의 경우 Spring 6.0부터 CGLIB 프록시에서 `protected`·패키지 접근 메서드도 지원됨
- **빈에만 적용**: `new`로 직접 만든 객체에는 Aspect가 적용되지 않음
- **생성자 안에서는 프록시가 없음**: 프록시는 초기화 이후에 만들어지므로 생성자·`@PostConstruct`에서 자기 메서드를 호출해도 Advice가 적용되지 않음

<br>

### 7. 면접·실무 체크포인트

- 스프링 AOP는 **런타임 프록시** 방식이며, AspectJ의 포인트컷 문법만 빌려 쓴다
- JDK Dynamic Proxy는 **인터페이스 기반**, CGLIB는 **서브클래스 기반**이며 스프링 부트는 **CGLIB가 기본값**이다
- 프록시는 **외부 호출만** 가로채므로 `this.method()` 내부 호출(self-invocation)에서는 `@Transactional`·`@Cacheable`·`@Async`가 무시된다 → **별도 빈으로 분리**가 정답
- CGLIB 프록시는 `final` 클래스·메서드, `private` 메서드에 적용할 수 없고, 프록시 객체의 필드는 초기화되지 않는다
- 여러 Aspect의 순서는 `@Order`로 제어하고, 트랜잭션 Advice는 기본적으로 가장 안쪽에서 실행된다
- 트랜잭션 프록시의 구체적 동작(전파·롤백 규칙)은 **unit05** 참고
