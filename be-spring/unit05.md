## 트랜잭션 추상화

스프링은 JDBC·JPA·JTA 등 기술마다 다른 트랜잭션 API를 **PlatformTransactionManager** 인터페이스로 추상화하고, `@Transactional` 애노테이션과 AOP 프록시로 서비스 코드에서 트랜잭션 코드를 완전히 걷어낸다. 편리한 만큼 **전파 속성·롤백 규칙·프록시 한계**를 모르면 "롤백이 안 된다", "트랜잭션이 안 걸린다" 같은 장애로 이어지므로 동작 원리를 정확히 알아야 한다.

<br>

### 1. 왜 추상화가 필요한가

```java
// 안티패턴: JDBC 트랜잭션 코드가 비즈니스 로직에 섞임
Connection con = dataSource.getConnection();
try {
    con.setAutoCommit(false);
    orderDao.save(con, order);
    stockDao.decrease(con, order.getItemId());
    con.commit();
} catch (Exception e) {
    con.rollback();
    throw e;
} finally {
    con.close();
}

// 개선: 선언적 트랜잭션 — 기술(JDBC/JPA)과 무관한 코드
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);
    stockService.decrease(order.getItemId());
}
```

- 같은 커넥션을 DAO 메서드마다 파라미터로 넘겨야 하는 문제를 스프링은 **트랜잭션 동기화(TransactionSynchronizationManager)**로 해결함 → 커넥션을 `ThreadLocal`에 보관하고 같은 스레드의 리포지토리가 꺼내 씀
- 기술별 구현체: `DataSourceTransactionManager`(JDBC·MyBatis), `JpaTransactionManager`(JPA, 내부적으로 JDBC 커넥션도 함께 동기화), `JtaTransactionManager`(분산 트랜잭션)
- 스프링 부트는 클래스패스에 JPA가 있으면 `JpaTransactionManager`를 자동 등록함

<br>

### 2. @Transactional의 동작 — 프록시와 TransactionInterceptor

```
호출자 ──▶ [프록시] ──▶ TransactionInterceptor.invoke()
                            │ ① TransactionManager.getTransaction(정의)   ← 전파 속성에 따라 기존 참여 / 신규 생성
                            │ ② 커넥션 획득, autoCommit=false, ThreadLocal에 바인딩
                            │ ③ 원본 메서드 실행
                            │ ④ 예외 없음 → commit
                            │    롤백 대상 예외 → rollback,  아니면 → commit 후 예외 재전파
                            ▼
                       [원본 서비스]
```

- `@Transactional`이 붙은 빈은 AOP 프록시로 감싸지고, 실제 판단은 `TransactionInterceptor`가 수행함 (프록시 생성 원리는 unit02 참고)
- 애노테이션 위치: 클래스에 붙이면 모든 public 메서드에 적용되고, 메서드에 붙이면 클래스 설정을 덮어씀. 인터페이스보다 **구체 클래스에 붙이는 것을 권장**함
- 메서드 가시성: JDK 프록시는 public만, CGLIB 프록시는 **Spring 6.0부터 protected·패키지 접근 메서드도** 지원함. `private`은 어떤 경우에도 적용되지 않음

> ⚠️ 트랜잭션도 프록시이므로 **self-invocation**(같은 클래스 내부에서 `this.method()` 호출)에서는 애노테이션이 무시된다. `public void a() { b(); }` 에서 `b()`에 `@Transactional(REQUIRES_NEW)`를 붙여도 새 트랜잭션은 생기지 않는다. 해결법은 unit02 참고.

<br>

### 3. 전파 속성(Propagation)

전파 속성은 **이미 트랜잭션이 진행 중인 상태에서 `@Transactional` 메서드를 호출했을 때** 어떻게 할지를 결정한다.

| **속성**            | **기존 트랜잭션 있음**                 | **기존 트랜잭션 없음**    | **용도**                                       |
| ------------------- | -------------------------------------- | ------------------------- | ---------------------------------------------- |
| **REQUIRED** (기본) | **참여** (같은 물리 트랜잭션)           | 새로 생성                 | 일반적인 서비스 메서드                          |
| **REQUIRES_NEW**    | 기존을 **잠시 보류**하고 새 트랜잭션 생성 | 새로 생성                 | 이력·로그 저장처럼 본 작업과 무관하게 커밋할 때 |
| **NESTED**          | 세이브포인트 생성 (부분 롤백 가능)      | 새로 생성                 | JDBC 세이브포인트 지원 시. **JPA는 미지원**    |
| **SUPPORTS**        | 참여                                    | 트랜잭션 없이 실행        | 읽기 전용 조회                                  |
| **NOT_SUPPORTED**   | 기존을 보류하고 트랜잭션 없이 실행       | 트랜잭션 없이 실행        | 트랜잭션 밖에서 실행해야 하는 작업              |
| **MANDATORY**       | 참여                                    | **예외** 발생             | 반드시 트랜잭션 안에서 호출돼야 할 때           |
| **NEVER**           | **예외** 발생                           | 트랜잭션 없이 실행        | 트랜잭션 안에서 호출되면 안 될 때               |

**물리 트랜잭션과 논리 트랜잭션**

```
outer() @Transactional(REQUIRED)  ───────── 물리 트랜잭션 1 (커넥션 A) ──────────┐
   ├─ inner1() REQUIRED       → 논리 트랜잭션 (같은 커넥션 A 참여)                │
   └─ inner2() REQUIRES_NEW   → 물리 트랜잭션 2 (커넥션 B, 독립 커밋/롤백)        │
                                                                                 ▼ commit
```

- REQUIRED로 참여한 내부 메서드에서 **런타임 예외가 발생하고 외부에서 catch해도**, 트랜잭션은 이미 `rollback-only`로 표시되어 커밋 시점에 `UnexpectedRollbackException`이 발생함. "내부에서 예외가 났지만 잡았으니 커밋되겠지"는 통하지 않음
- REQUIRES_NEW는 **커넥션을 하나 더** 점유하므로 풀 크기가 작으면 커넥션 고갈·데드락의 원인이 됨 (unit11 참고)

> 💡 "REQUIRES_NEW를 언제 쓰는가"에는 "본 트랜잭션이 롤백되어도 남아야 하는 실패 이력·감사 로그"를 예로 들고, 대신 **커넥션을 2개 잡는 비용**과 self-invocation 함정을 함께 언급하면 좋다.

<br>

### 4. 롤백 규칙

| **예외 종류**                              | **기본 동작**   | **변경 방법**                                |
| ------------------------------------------ | --------------- | -------------------------------------------- |
| **RuntimeException 및 하위** (`IllegalStateException` 등) | **롤백**        | `noRollbackFor = XxxException.class`         |
| **Error 및 하위** (`OutOfMemoryError` 등)   | **롤백**        | -                                            |
| **Checked Exception** (`IOException`, 커스텀 `Exception` 상속) | **커밋**        | `rollbackFor = Exception.class`              |

- 기본 규칙은 EJB 관례를 따른 것으로, 체크 예외는 "비즈니스적으로 예상된 상황이라 복구 가능"이라고 간주함
- 실무에서는 커스텀 예외를 `RuntimeException` 기반으로 설계해 별도 설정 없이 롤백되게 하는 것이 일반적임 (예외 계층 설계는 unit09 참고)
- 롤백 여부는 **예외가 프록시까지 전파됐는지**로 판단함. 메서드 안에서 `catch`로 삼키면 프록시는 정상 종료로 보고 커밋함

```java
// 안티패턴: 체크 예외 → 커밋됨, 예외를 삼킴 → 커밋됨
@Transactional
public void transfer(Long from, Long to, long amount) throws InsufficientBalanceException {
    try {
        accountRepository.withdraw(from, amount);
        accountRepository.deposit(to, amount);    // 여기서 RuntimeException 발생해도
    } catch (RuntimeException e) {
        log.error("transfer failed", e);          // 삼켜 버리면 withdraw만 커밋됨
    }
}

// 개선: 롤백 대상을 명시하거나, 런타임 예외로 감싸 프록시까지 전파
@Transactional(rollbackFor = InsufficientBalanceException.class)
public void transfer(Long from, Long to, long amount) throws InsufficientBalanceException {
    accountRepository.withdraw(from, amount);
    accountRepository.deposit(to, amount);
}
```

- 예외를 잡되 롤백은 하고 싶다면 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`를 호출하거나, 프로그래밍 방식(`TransactionTemplate`)을 사용함

<br>

### 5. readOnly와 기타 속성

`@Transactional(readOnly = true)`는 단순한 표시가 아니라 여러 계층에 **최적화 힌트**를 전달한다.

| **계층**                 | **readOnly = true의 효과**                                                  |
| ------------------------ | --------------------------------------------------------------------------- |
| **Hibernate**            | 플러시 모드를 `MANUAL`로 → 변경 감지용 스냅샷 비교·더티 체킹 생략, 커밋 시 플러시 안 함 |
| **JDBC 드라이버**        | `Connection.setReadOnly(true)` → DB에 따라 쓰기 방지·최적화 힌트             |
| **DB 라우팅**            | `LazyConnectionDataSourceProxy` + `AbstractRoutingDataSource`로 **읽기 복제본(Replica)** 으로 분기 |

- 읽기 전용 트랜잭션에서 엔티티를 수정해도 **DB에 반영되지 않음** (예외 없이 조용히 무시되므로 주의)
- 그 외 속성: `isolation`(격리 수준, 기본은 DB 기본값), `timeout`(초 단위, 초과 시 롤백), `transactionManager`(다중 DB 환경에서 매니저 지정)

> 💡 조회 메서드에 `readOnly = true`를 붙이는 이유를 물으면 "더티 체킹 생략으로 메모리·CPU 절약, 플러시 방지, 읽기 DB 분기 힌트"의 세 가지를 답한다. 격리 수준의 의미는 `database/unit17` 참고.

<br>

### 6. 프록시 기반 트랜잭션의 한계

- **self-invocation**: 내부 호출은 프록시를 거치지 않음 (unit02)
- **가시성**: `private`·`final` 메서드에 적용 불가, JDK 프록시는 인터페이스 메서드만
- **스레드 경계**: 트랜잭션 자원은 `ThreadLocal`에 묶이므로 `@Async`·`CompletableFuture`·새 스레드에서는 **호출자의 트랜잭션에 참여하지 않음** (unit11)
- **예외 삼킴**: `catch`로 삼킨 예외는 프록시가 모르므로 커밋됨
- **트랜잭션 범위 = 커넥션 점유 시간**: 트랜잭션 안에서 외부 API 호출·파일 I/O를 하면 그 시간만큼 커넥션을 붙잡음 → 외부 호출은 트랜잭션 **밖**에서 수행하고, 트랜잭션은 최소 범위로 유지

```java
// 프로그래밍 방식: 트랜잭션 범위를 코드로 정밀 제어할 때
@RequiredArgsConstructor
public class OrderFacade {
    private final TransactionTemplate txTemplate;
    private final PaymentClient paymentClient;

    public void placeOrder(OrderRequest req) {
        Long orderId = txTemplate.execute(status -> orderService.create(req));   // 트랜잭션 1: 짧게
        PaymentResult result = paymentClient.pay(orderId, req.amount());       // 외부 호출: 트랜잭션 밖
        txTemplate.executeWithoutResult(status -> orderService.confirm(orderId, result));  // 트랜잭션 2
    }
}
```

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                              |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **@Transactional은 어떻게 동작하는가?**               | AOP 프록시 → `TransactionInterceptor`가 트랜잭션 매니저로 시작·커밋·롤백, 커넥션은 **ThreadLocal 동기화** |
| **체크 예외를 던지면 롤백되는가?**                    | 기본은 **커밋**. `rollbackFor`로 지정하거나 런타임 예외 기반으로 설계                         |
| **REQUIRED와 REQUIRES_NEW 차이는?**                   | 참여 vs 새 물리 트랜잭션(커넥션 추가). 내부 롤백 표시 시 REQUIRED는 `UnexpectedRollbackException` |
| **readOnly = true는 무슨 효과가 있는가?**             | 더티 체킹·플러시 생략, JDBC readOnly 힌트, 읽기 복제본 라우팅                                |
| **트랜잭션이 안 걸리는 대표 원인은?**                 | self-invocation, `private` 메서드, 빈이 아닌 객체, 다른 스레드, 예외 삼킴                    |
| **트랜잭션 안에서 외부 API를 호출하면?**              | 커넥션 점유 시간이 길어져 풀 고갈 → 외부 호출은 트랜잭션 밖으로 분리                          |
