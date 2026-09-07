## 고수준 동시성 API

`synchronized`·`volatile`(unit08)만으로 실무 동시성 코드를 작성하면 스레드 생성 비용, 락 경합, 복잡한 대기 로직을 모두 직접 감당해야 한다. `java.util.concurrent` 패키지는 **Executor 프레임워크(스레드 풀)**, **원자 클래스(CAS 기반 무락 연산)**, **동시성 컬렉션**, **동기화 도구**를 제공해 이런 문제를 검증된 방식으로 해결하며, 백엔드 서비스의 스레드 풀 설정은 곧 장애 대응 능력과 직결된다.

<br>

### 1. Executor 프레임워크

### 1-1. 왜 스레드를 직접 만들지 않는가

플랫폼 스레드는 생성마다 OS 스레드와 스택 메모리(기본 약 1MB, unit01)를 할당하므로 요청마다 `new Thread()`를 하면 **생성 비용과 무제한 스레드 수** 두 가지가 문제가 된다. **Executor**는 작업 제출과 실행을 분리해 스레드를 **재사용**하고 **개수를 제한**한다.

```java
// 안티패턴: 요청마다 스레드 생성 — 부하가 몰리면 스레드 수천 개 → OOM(unable to create native thread)
new Thread(() -> handle(request)).start();

// 개선: 크기가 제한된 풀에 작업 제출
ExecutorService pool = Executors.newFixedThreadPool(16);
Future<Result> future = pool.submit(() -> handle(request));   // Callable → Future
Result r = future.get(3, TimeUnit.SECONDS);                     // 타임아웃 있는 대기
```

| **인터페이스·클래스**          | **역할**                                                              |
| ------------------------------ | --------------------------------------------------------------------- |
| **`Executor`**                  | `execute(Runnable)` 하나. "실행 방식"의 추상화                           |
| **`ExecutorService`**           | `submit`·`invokeAll`·`shutdown` 등 생명주기와 결과(`Future`) 관리          |
| **`ThreadPoolExecutor`**        | 실제 스레드 풀 구현. 코어·최대 크기, 큐, 거부 정책을 직접 설정              |
| **`ScheduledExecutorService`**  | 지연·주기 실행 (`Timer`의 대체)                                          |
| **`ForkJoinPool`**              | 작업 훔치기(work-stealing) 기반. 병렬 스트림·`CompletableFuture` 기본 풀      |

<br>

### 1-2. ThreadPoolExecutor의 동작 순서

```java
ExecutorService pool = new ThreadPoolExecutor(
        8,                                       // corePoolSize
        32,                                      // maximumPoolSize
        60, TimeUnit.SECONDS,                    // 코어 초과 스레드의 유휴 유지 시간
        new ArrayBlockingQueue<>(200),           // 작업 큐 (유한)
        new ThreadPoolExecutor.CallerRunsPolicy() // 거부 정책
);
```

```
 작업 제출
   ├─ ① 실행 중인 스레드 < core       ──▶ 새 스레드 생성해 즉시 실행
   ├─ ② core 가득 참                  ──▶ 큐에 넣음 (큐에 여유가 있는 동안)
   ├─ ③ 큐도 가득 참 && 스레드 < max   ──▶ 스레드 추가 생성해 실행
   └─ ④ 큐 가득 && 스레드 == max       ──▶ 거부 정책(RejectedExecutionHandler) 실행
```

- **핵심 함정**: "큐가 먼저, 스레드 추가는 나중"이다. 큐가 무한(`LinkedBlockingQueue` 기본)이면 ③은 **영원히 일어나지 않아** `maximumPoolSize`가 무의미해짐
- 거부 정책: `AbortPolicy`(기본, 예외), `CallerRunsPolicy`(제출한 스레드가 직접 실행 → 자연스러운 배압), `DiscardPolicy`, `DiscardOldestPolicy`

> ⚠️ `Executors` 팩토리 메서드는 편하지만 운영 환경에서는 위험하다. `newFixedThreadPool`은 **무한 큐**라 작업이 쌓이면 메모리가 고갈되고, `newCachedThreadPool`은 **스레드 수 상한이 없어** 부하 시 스레드가 폭증한다. 운영 코드에서는 `ThreadPoolExecutor`를 직접 생성해 유한 큐와 거부 정책을 명시하는 것이 원칙이다.

**풀 크기 가이드** — CPU 바운드 작업은 코어 수 ± 1, I/O 바운드 작업은 `코어 수 × (1 + 대기 시간 / 계산 시간)`에서 출발해 측정으로 조정한다. 풀은 반드시 `shutdown()`으로 종료하고 `awaitTermination()`으로 완료를 기다린다.

<br>

### 1-3. CompletableFuture와 가상 스레드

`Future.get()`은 블로킹만 가능하지만 **`CompletableFuture`(Java 8)**는 완료 시 실행할 후속 작업을 체이닝하고 여러 비동기 결과를 조합할 수 있다.

```java
ExecutorService io = Executors.newFixedThreadPool(32);

CompletableFuture<Order> order = CompletableFuture.supplyAsync(() -> orderApi.get(id), io);
CompletableFuture<User> user   = CompletableFuture.supplyAsync(() -> userApi.get(id), io);

OrderView view = order.thenCombine(user, OrderView::of)     // 두 결과 조합
        .exceptionally(ex -> OrderView.fallback())          // 예외 처리
        .orTimeout(2, TimeUnit.SECONDS)                     // Java 9+
        .join();
```

- 실행기를 넘기지 않으면 **`ForkJoinPool.commonPool()`**에서 실행되므로, 블로킹 I/O 작업은 반드시 **전용 풀**을 지정함 (unit07의 병렬 스트림과 같은 이유)
- **가상 스레드(Java 21 정식)**: JVM이 관리하는 경량 스레드로, 블로킹 시 캐리어 스레드를 놓아 주어 수십만 개를 동시에 만들 수 있음. I/O 대기가 많은 작업은 풀링 대신 `Executors.newVirtualThreadPerTaskExecutor()`로 **작업당 스레드 하나**를 쓰는 모델이 권장됨

```java
// Java 21: 요청마다 가상 스레드 — 풀 크기 고민이 사라짐 (CPU 바운드에는 이점 없음)
try (ExecutorService vt = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> results = ids.stream()
            .map(id -> vt.submit(() -> httpClient.fetch(id)))
            .toList();
}   // Java 19+ ExecutorService는 AutoCloseable: close()가 shutdown + 대기
```

<br>

### 2. 원자 클래스와 CAS(Compare-And-Swap)

`AtomicInteger`·`AtomicLong`·`AtomicReference` 등은 락 대신 CPU의 **CAS 명령**으로 원자성을 보장한다. CAS는 "메모리 값이 내가 기대한 값이면 새 값으로 바꿔라"를 하드웨어 수준에서 원자적으로 수행한다.

```
 incrementAndGet() 의 내부 (개념)
   do {
       expected = value 읽기            // 예: 5
       next = expected + 1              // 6
   } while (!compareAndSet(expected, next));   // 그 사이 다른 스레드가 바꿨으면(값≠5) 실패 → 재시도
```

```java
private final AtomicLong requestCount = new AtomicLong();
public void onRequest() { requestCount.incrementAndGet(); }          // 락 없이 원자적 증가

private final AtomicReference<Config> config = new AtomicReference<>(initial);
public void reload(Config fresh) { config.set(fresh); }              // 참조 교체 자체는 원자적
public void update(UnaryOperator<Config> fn) { config.updateAndGet(fn); }  // CAS 루프로 안전한 갱신
```

| **항목**            | **락(synchronized·Lock)**                    | **CAS(Atomic*)**                                  |
| ------------------- | -------------------------------------------- | ------------------------------------------------- |
| **방식**            | 비관적: 먼저 잠그고 실행                       | 낙관적: 일단 시도하고 충돌 시 재시도                  |
| **블로킹**          | 대기 스레드는 블로킹(컨텍스트 스위칭)            | **없음** (스핀 재시도)                              |
| **경쟁이 낮을 때**   | 락 오버헤드가 상대적으로 큼                     | 매우 빠름                                          |
| **경쟁이 높을 때**   | 안정적 (대기 후 순차 처리)                      | 재시도 폭증으로 CPU 낭비 → `LongAdder` 고려          |
| **적용 범위**        | 여러 변수·복합 로직                            | **단일 변수**의 갱신                                 |

- **ABA 문제**: 값이 A → B → A로 바뀌면 CAS는 변화를 감지하지 못함. 참조 타입에서 문제가 되면 `AtomicStampedReference`로 버전을 함께 비교함
- **`LongAdder`**(Java 8): 셀을 여러 개 두고 스레드를 분산시켜 고경쟁 카운터에서 `AtomicLong`보다 훨씬 빠름. 합계는 `sum()`으로 읽음 (읽기 시점의 근사값)

<br>

### 2-1. 명시적 락 — ReentrantLock

`synchronized`로 부족할 때(타임아웃, 인터럽트 가능 대기, 공정성, 조건 변수 여러 개) `Lock` 인터페이스를 쓴다.

```java
private final ReentrantLock lock = new ReentrantLock();

public boolean transfer(Account to, long amount) throws InterruptedException {
    if (!lock.tryLock(500, TimeUnit.MILLISECONDS)) return false;   // 데드락 대신 포기
    try {
        ...
        return true;
    } finally {
        lock.unlock();                                              // 반드시 finally에서 해제
    }
}
```

- `ReadWriteLock`은 읽기끼리는 동시 허용, 쓰기는 배타 — 읽기가 압도적으로 많을 때 유리
- `synchronized`는 블록을 벗어나면 자동 해제되지만 `Lock`은 **`unlock()`을 빠뜨리면 영원히 잠김**. 단순한 상호 배제라면 `synchronized`가 더 안전함

> ⚠️ 데드락은 두 스레드가 서로 다른 순서로 락 두 개를 잡을 때 생긴다. 예방책은 **모든 코드가 같은 순서로 락을 획득**하도록 규칙을 정하는 것이고, 그것이 어려우면 `tryLock(timeout)`으로 대기에 상한을 둔다. `jstack`으로 스레드 덤프를 뜨면 JVM이 감지한 데드락과 각 스레드가 기다리는 락이 표시된다.

<br>

### 3. 동시성 컬렉션

| **컬렉션**                       | **대체 대상**                         | **내부 방식**                                                | **특징·주의**                                                  |
| -------------------------------- | ------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------- |
| **`ConcurrentHashMap`**          | `HashMap`, `Hashtable`, `synchronizedMap` | Java 8+ **버킷 단위 CAS + 충돌 시 버킷 헤드 `synchronized`**    | 읽기는 락 없음, `null` 키·값 불가, 이터레이터는 약한 일관성          |
| **`CopyOnWriteArrayList`**       | `ArrayList`                           | 쓰기마다 **배열 전체 복사**, 읽기는 스냅샷                      | 읽기 압도적·쓰기 드문 리스너 목록 등에 적합. 쓰기 많으면 최악          |
| **`ConcurrentLinkedQueue`**      | `LinkedList`(큐 용도)                  | CAS 기반 무락 큐                                              | 비블로킹, 크기 제한 없음                                          |
| **`BlockingQueue` 구현체**        | `wait/notify` 직접 구현                | `ArrayBlockingQueue`(유한), `LinkedBlockingQueue`, `PriorityBlockingQueue` | `put`/`take`가 가득·비면 대기 → **생산자-소비자** 패턴의 표준       |
| **`ConcurrentSkipListMap`**      | `TreeMap`                             | 스킵 리스트                                                   | 정렬된 동시성 맵                                                 |

```java
// 안티패턴: 검사 후 행동(check-then-act) — 두 연산 사이에 다른 스레드가 끼어듦
Map<String, Integer> counts = new ConcurrentHashMap<>();
if (!counts.containsKey(word)) counts.put(word, 1);
else counts.put(word, counts.get(word) + 1);           // 갱신 유실

// 개선: 원자적 복합 연산 사용
counts.merge(word, 1, Integer::sum);                   // 또는 compute / computeIfAbsent
```

- `Collections.synchronizedMap()`은 모든 메서드에 하나의 락을 거는 방식이라 `ConcurrentHashMap`보다 훨씬 느리며, 순회 시 별도 동기화가 필요함
- `ConcurrentHashMap`의 `size()`·`isEmpty()`는 순간 스냅샷일 뿐 정확한 동기화 값이 아님. 컬렉션 내부 구조는 **unit04** 참고

> 💡 `ConcurrentHashMap`이 `null`을 허용하지 않는 이유는 `get(key)`가 `null`을 반환했을 때 "키가 없음"과 "값이 `null`"을 구분하기 위해 `containsKey`를 다시 부르면 그 사이 상태가 바뀔 수 있기 때문이다. 동시성 컬렉션에서는 이런 **두 연산의 조합이 원자적이지 않다**는 점을 항상 의식해야 한다.

<br>

### 4. 동기화 도구(Synchronizer)

| **도구**                | **용도**                                                 | **예시**                                              |
| ----------------------- | -------------------------------------------------------- | ----------------------------------------------------- |
| **`CountDownLatch`**     | N개의 이벤트가 끝날 때까지 대기 (**일회용**)                 | 서비스 초기화 완료 대기, 테스트에서 스레드 동시 출발       |
| **`CyclicBarrier`**      | N개 스레드가 모두 지점에 도달하면 함께 진행 (**재사용 가능**)  | 단계별 병렬 계산                                        |
| **`Semaphore`**          | 동시에 자원을 쓸 수 있는 스레드 수 제한                      | DB 커넥션·외부 API 동시 호출 수 제한                     |
| **`Phaser`**             | 단계가 동적으로 바뀌는 배리어                               | 참여 스레드 수가 변하는 작업                             |

```java
Semaphore limiter = new Semaphore(10);                    // 외부 API 동시 호출 10개로 제한

public Response call(Request req) throws InterruptedException {
    limiter.acquire();
    try { return externalApi.send(req); }
    finally { limiter.release(); }
}
```

<br>

### 5. 인터럽트와 ThreadLocal

**인터럽트**는 스레드를 강제 종료하는 것이 아니라 "멈춰 달라"는 **협력적 신호**다. `sleep`·`wait`·`BlockingQueue.take` 등은 인터럽트되면 `InterruptedException`을 던지고 **인터럽트 상태를 지운다**.

```java
// 안티패턴: InterruptedException을 삼킴 — 풀이 종료(shutdownNow)해도 스레드가 안 멈춤
try { Thread.sleep(1000); } catch (InterruptedException e) { }

// 개선: 상태를 복원해 상위 코드·풀이 종료 의사를 알 수 있게 함
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return;
}
```

**ThreadLocal**은 스레드마다 독립된 값을 갖는 변수로, 요청 컨텍스트(인증 정보, 트랜잭션, 로깅 MDC)를 전달할 때 쓴다.

- 스레드 풀에서는 스레드가 재사용되므로 **`remove()`하지 않으면 다음 요청이 이전 값을 읽거나** 메모리 누수(unit02)가 생김. `try-finally`로 반드시 정리함
- 가상 스레드는 수가 매우 많아 `ThreadLocal`이 메모리 부담이 되므로, 부모→자식 전달과 불변성을 갖춘 **`ScopedValue`**가 대안으로 도입되었음 (Java 21 프리뷰, 정식화 버전은 JDK 문서 확인)

<br>

### 6. 면접·실무 체크포인트

| **질문**                                                  | **핵심 답변**                                                                                       |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **ThreadPoolExecutor에 작업이 들어오면 어떤 순서로 처리되는가?** | core 스레드 → **큐** → max까지 스레드 추가 → 거부 정책. 무한 큐면 max는 무의미                        |
| **`Executors.newFixedThreadPool`의 문제는?**                | 무한 `LinkedBlockingQueue`라 작업 적체 시 메모리 고갈. 유한 큐와 거부 정책을 직접 설정                  |
| **CAS와 락의 차이는?**                                       | CAS는 낙관적 무락 재시도, 락은 비관적 대기. 저경쟁은 CAS, 고경쟁 카운터는 `LongAdder`, 복합 로직은 락    |
| **`ConcurrentHashMap`은 어떻게 동시성을 보장하는가?**         | Java 8+ 버킷 단위 CAS와 필요 시 버킷별 `synchronized`. 읽기는 무락. 복합 연산은 `merge`·`compute` 사용   |
| **`CopyOnWriteArrayList`는 언제 쓰는가?**                    | 읽기가 압도적이고 쓰기가 드문 경우. 쓰기마다 배열 복사                                                  |
| **`CountDownLatch`와 `CyclicBarrier`의 차이는?**             | 래치는 일회용이며 이벤트 카운트 대기, 배리어는 스레드 집결점이며 재사용 가능                             |
| **`InterruptedException`은 어떻게 처리하는가?**               | 삼키지 말고 **`Thread.currentThread().interrupt()`**로 상태 복원 후 종료                               |
| **가상 스레드는 언제 유리한가?**                              | I/O 대기가 많은 작업. CPU 바운드에는 이점 없음. `synchronized` 안 블로킹은 버전별 고정 이슈 확인          |

- 공유 가변 상태를 아예 없애는 접근(불변 객체·record)은 **unit10**에서 이어짐
