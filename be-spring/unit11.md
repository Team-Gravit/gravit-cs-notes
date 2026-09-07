## 스레드와 커넥션 자원 관리

스프링 부트 애플리케이션의 처리량은 결국 **톰캣 스레드 풀**과 **DB 커넥션 풀**이라는 두 유한 자원을 얼마나 효율적으로 쓰느냐에 달려 있다. 트랜잭션·시큐리티 컨텍스트가 **스레드에 묶여 있다**는 사실을 모르면 `@Async`에서 트랜잭션이 사라지고, 트랜잭션 안의 외부 호출이 커넥션 풀을 고갈시켜 서비스 전체가 멈추는 장애로 이어진다.

<br>

### 1. 요청 하나가 점유하는 자원

```
[클라이언트] ──▶ 톰캣 스레드 풀 (기본 최대 200)
                    │  스레드 1개 할당 ── 요청 처리 전 과정 동안 점유
                    ▼
              필터 → DispatcherServlet → 컨트롤러 → 서비스(@Transactional)
                                                        │  커넥션 1개 획득 (HikariCP 기본 최대 10)
                                                        ▼
                                                     [ Database ]
              ◀── 트랜잭션 종료: 커넥션 반환 ── 응답 완료: 스레드 반환
```

| **자원**            | **스프링 부트 기본값**                                  | **초과 시 동작**                                          |
| ------------------- | ------------------------------------------------------- | --------------------------------------------------------- |
| **톰캣 워커 스레드**| `server.tomcat.threads.max=200`, `min-spare=10`          | `accept-count`(기본 100) 큐에서 대기 → 초과 시 연결 거부   |
| **HikariCP 커넥션** | `maximum-pool-size=10`, `connection-timeout=30000`(ms)   | 30초 대기 후 `SQLTransientConnectionException`             |
| **@Async 실행기**   | `applicationTaskExecutor`: core 8, 큐 무제한             | 큐에 무한 적재 → 지연·메모리 증가 (버전에 따라 다를 수 있음) |

- 요청 처리 중 스레드는 **블로킹 I/O(DB·외부 API) 동안에도 점유**되므로, 느린 I/O가 많을수록 200개 스레드가 빨리 소진됨
- 커넥션은 **트랜잭션 시작 시 획득, 종료 시 반환**이 원칙. 트랜잭션이 길면 커넥션도 오래 묶임 (unit05 참고)

> 💡 "서버가 느려지는데 CPU는 놀고 있다"는 증상은 대개 **스레드가 I/O 대기로 묶여 있거나 커넥션 획득을 기다리는 상태**다. 스레드 덤프에서 `HikariPool.getConnection` 대기가 보이면 커넥션 고갈, 외부 소켓 읽기 대기가 보이면 타임아웃 미설정을 의심한다.

<br>

### 2. 커넥션 풀 고갈 시나리오

**시나리오 ① 트랜잭션 안의 외부 호출**

```java
// 안티패턴: 결제 API 호출(수 초) 동안 커넥션을 붙잡음 → 동시 요청 10개면 풀 고갈
@Transactional
public void order(OrderRequest req) {
    Order order = orderRepository.save(Order.from(req));
    PaymentResult result = paymentClient.pay(order);      // 외부 HTTP 호출, 타임아웃 없으면 무한 대기
    order.confirm(result);
}

// 개선: 트랜잭션을 DB 작업으로만 좁히고 외부 호출은 밖에서 (unit05의 TransactionTemplate 예시 참고)
public void order(OrderRequest req) {
    Long orderId = txTemplate.execute(s -> orderRepository.save(Order.from(req)).getId());
    PaymentResult result = paymentClient.pay(orderId);
    txTemplate.executeWithoutResult(s -> orderRepository.findById(orderId).orElseThrow().confirm(result));
}
```

**시나리오 ② REQUIRES_NEW 중첩 — 풀 데드락**

```
풀 크기 10, 동시 요청 10개
각 요청: 외부 트랜잭션(커넥션 1개 보유) → 내부 REQUIRES_NEW(커넥션 1개 더 요청)
   → 10개 요청이 각각 1개씩 잡고 2번째를 기다림 → 반환되는 커넥션이 없음 → 30초 후 전부 타임아웃
```

- HikariCP 문서의 데드락 회피 기준: **풀 크기 ≥ Tn × (Cm − 1) + 1** (Tn = 최대 동시 스레드 수, Cm = 스레드 하나가 동시에 잡는 최대 커넥션 수)
- 근본 해결은 한 요청이 커넥션을 두 개 잡는 구조를 피하는 것 (REQUIRES_NEW 최소화, 이벤트로 분리)

**시나리오 ③ OSIV로 인한 점유 연장**

- `spring.jpa.open-in-view=true`(기본)에서는 트랜잭션이 끝난 뒤 컨트롤러·직렬화 단계에서 **지연 로딩이 발생하면 커넥션을 다시 획득**하고, 요청이 끝날 때까지 유지될 수 있음 (unit06 참고)
- API 서버는 OSIV를 끄고 조회·변환을 트랜잭션 안에서 끝내는 편이 커넥션 효율에 유리함

> ⚠️ 외부 HTTP 호출·메시지 발행에는 **반드시 연결·읽기 타임아웃**을 설정한다. 타임아웃이 없는 외부 호출 하나가 트랜잭션 안에 있으면, 상대 시스템 장애가 커넥션 풀 고갈 → 톰캣 스레드 고갈로 전파되어 **무관한 API까지 전부 멈춘다**.

<br>

### 3. 트랜잭션과 스레드 경계

스프링 트랜잭션은 커넥션·`EntityManager`를 **`ThreadLocal`(TransactionSynchronizationManager)** 에 바인딩한다. 따라서 트랜잭션은 **스레드 하나의 범위**를 넘지 못한다.

```
[톰캣 스레드 A]  @Transactional 시작 → ThreadLocal(A) = {커넥션 #3, EntityManager}
        │
        ├─ executor.submit(() -> repository.save(x))   ──▶ [풀 스레드 B]  ThreadLocal(B) = 비어 있음
        │                                                       → 트랜잭션 없음 → 자동 커밋 또는 예외
        ▼
     커밋 (스레드 B의 작업과 무관하게 진행)
```

- 부모 스레드가 롤백해도 자식 스레드가 저장한 데이터는 **이미 별도로 커밋**되었거나 트랜잭션 없이 실행됨 → 원자성이 깨짐
- 자식 스레드에서 지연 로딩을 시도하면 `EntityManager`가 없으므로 `LazyInitializationException`
- 같은 이유로 `SecurityContextHolder`(unit10), MDC 로그 컨텍스트, `RequestContextHolder`도 새 스레드에서 비어 있음

```java
// 안티패턴: 트랜잭션 안에서 병렬 저장 → 자식 스레드는 트랜잭션 밖
@Transactional
public void importAll(List<Row> rows) {
    rows.parallelStream().forEach(row -> repository.save(Row.toEntity(row)));   // ForkJoinPool 스레드
}

// 개선 ①: 단일 스레드에서 배치 INSERT (unit08)  ② 병렬이 꼭 필요하면 작업마다 독립 트랜잭션으로 설계하고
//         실패 보상(재시도·삭제)을 명시적으로 처리
```

<br>

### 4. @Async — 비동기 처리의 동작과 함정

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "mailExecutor")
    public ThreadPoolTaskExecutor mailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);                                     // 무제한 큐 대신 상한 지정
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setThreadNamePrefix("mail-");
        executor.setTaskDecorator(runnable -> {                             // 컨텍스트 전파
            Map<String, String> mdc = MDC.getCopyOfContextMap();
            return () -> {
                if (mdc != null) MDC.setContextMap(mdc);
                try { runnable.run(); } finally { MDC.clear(); }
            };
        });
        executor.initialize();
        return executor;
    }
}

@Service
public class MailService {
    @Async("mailExecutor")
    public CompletableFuture<Void> sendWelcome(Long memberId) {             // void보다 Future 반환이 예외 추적에 유리
        // ...
        return CompletableFuture.completedFuture(null);
    }
}
```

| **함정**                                   | **원인**                                                        | **대응**                                                     |
| ------------------------------------------ | --------------------------------------------------------------- | ------------------------------------------------------------ |
| **동기로 실행됨**                          | self-invocation, `private` 메서드, 빈이 아닌 객체 (unit02)       | 별도 빈으로 분리                                             |
| **트랜잭션이 안 걸림 / 부모와 따로 커밋**  | 다른 스레드 → ThreadLocal 없음                                  | 비동기 메서드 자체에 `@Transactional`을 두고 **독립 트랜잭션**으로 설계 |
| **인증·MDC 정보가 사라짐**                 | ThreadLocal 미전파                                              | `TaskDecorator`, `DelegatingSecurityContextAsyncTaskExecutor` |
| **`void` 메서드의 예외가 사라짐**          | 호출자에게 전파되지 않음                                        | `AsyncUncaughtExceptionHandler` 등록 또는 `CompletableFuture` 반환 |
| **큐가 무한히 쌓임**                       | 기본 실행기의 `queueCapacity`가 무제한                            | 큐 상한 + 거부 정책 설정, 전용 실행기 분리                    |
| **커밋 전에 비동기 작업이 먼저 실행됨**    | `@Async` 호출이 트랜잭션 커밋보다 먼저 시작                      | `@TransactionalEventListener(phase = AFTER_COMMIT)` + `@Async` |

- `ThreadPoolTaskExecutor`는 **core → 큐 → max** 순서로 확장하므로, 큐가 무제한이면 max 설정은 사실상 무의미함
- 스프링 부트는 `@EnableAsync` 시 `applicationTaskExecutor` 빈을 기본 실행기로 사용하며, 이름을 지정하지 않은 `@Async`가 여기로 감. 용도별 실행기를 분리해야 한 작업의 지연이 다른 작업을 막지 않음

> 💡 "@Async와 @Transactional을 같이 쓰면?"이라는 질문의 답은 **"다른 스레드이므로 호출자 트랜잭션에 참여하지 않는다. 비동기 메서드 안에서 새 트랜잭션이 시작되며, 커밋 이후에 실행되게 하려면 `@TransactionalEventListener(AFTER_COMMIT)`를 조합한다"**이다.

<br>

### 5. 자원 설정 기준

| **설정**                                     | **기준**                                                                       |
| -------------------------------------------- | ------------------------------------------------------------------------------ |
| **HikariCP maximum-pool-size**               | 무조건 키우지 않음. DB 코어 수 기반(HikariCP 권장 공식: `코어 수 × 2 + 디스크 수`)에서 시작해 부하 테스트로 조정 |
| **connection-timeout**                       | 기본 30초는 장애 전파 시간이 김. 수 초 수준으로 낮춰 **빠르게 실패**하고 알림      |
| **leak-detection-threshold**                 | 개발·스테이징에서 켜서(예: 5000ms) 반환되지 않는 커넥션을 로그로 잡아냄           |
| **max-lifetime**                             | DB의 `wait_timeout`보다 **짧게** 설정해 끊긴 커넥션 사용을 방지                  |
| **톰캣 threads.max**                         | CPU 코어·I/O 비율·커넥션 풀 크기와 함께 결정. 스레드만 늘리면 커넥션 대기만 증가   |
| **@Transactional(timeout)**                  | 긴 트랜잭션에 상한을 두어 커넥션 점유 시간을 제한                                 |
| **외부 호출 타임아웃**                       | 연결·읽기 타임아웃 필수, 재시도는 멱등한 요청에만                                 |

- 풀 크기를 200으로 키워 "고갈"을 해결하려는 시도는 DB 쪽 컨텍스트 스위칭만 늘려 전체 처리량을 오히려 떨어뜨림. 고갈의 원인은 대부분 **커넥션 점유 시간**이므로 트랜잭션 범위와 외부 호출 위치를 먼저 본다

<br>

### 6. 가상 스레드(Virtual Thread)와의 관계

- **Spring Boot 3.2 + JDK 21**부터 `spring.threads.virtual.enabled=true`로 톰캣 요청 처리와 `@Async` 기본 실행기를 가상 스레드로 전환할 수 있음
- 가상 스레드는 블로킹 I/O 시 캐리어 스레드를 반납하므로 "스레드 200개 한계"는 사실상 사라지지만, **DB 커넥션 풀 한계는 그대로**임 → 요청이 커넥션 대기로 몰리는 병목이 더 두드러질 수 있음
- JDK 21에서는 `synchronized` 블록 안의 블로킹이 캐리어 스레드를 고정(pinning)하는 문제가 있어 일부 JDBC 드라이버·라이브러리에서 이점이 줄었고, JDK 24(JEP 491)에서 개선됨 (버전에 따라 다름)
- ThreadLocal 기반 컨텍스트(트랜잭션·시큐리티)는 가상 스레드에서도 동일하게 동작하지만, 수백만 개의 가상 스레드가 각자 ThreadLocal을 가지면 메모리 부담이 됨

<br>

### 7. 면접·실무 체크포인트

- 요청 하나는 **톰캣 스레드 1개**를 처리 내내, **DB 커넥션 1개**를 트랜잭션 동안 점유한다. 두 풀의 크기와 점유 시간이 처리량을 결정한다
- 커넥션 풀 고갈의 3대 원인: **트랜잭션 안의 외부 호출**, **REQUIRES_NEW 중첩(풀 데드락)**, **OSIV로 인한 점유 연장**
- 트랜잭션·시큐리티·MDC는 **ThreadLocal**에 묶여 있어 `@Async`·`parallelStream`·별도 실행기로 넘어가면 **사라진다**
- `@Async`는 프록시 기반이라 self-invocation에서 무시되며, 기본 실행기는 큐가 무제한이므로 **전용 실행기 + 큐 상한 + 거부 정책**을 지정한다
- 커밋 이후에 비동기 작업을 실행하려면 `@TransactionalEventListener(AFTER_COMMIT)`를 조합한다
- 풀 크기를 키우기 전에 **점유 시간**(트랜잭션 범위, 타임아웃)을 먼저 줄인다
- 트랜잭션 전파·readOnly는 unit05, 영속성 컨텍스트 범위는 unit06, 시큐리티 컨텍스트는 unit10 참고
