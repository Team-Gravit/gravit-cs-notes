## 동시성 기본

여러 스레드가 **같은 힙 데이터를 동시에 읽고 쓸 때** 발생하는 문제와 이를 막는 언어 차원의 기본 도구인 `synchronized`와 `volatile`을 다룬다. 두 키워드의 차이는 단순히 "락이 있냐 없냐"가 아니라 **원자성·가시성·순서**라는 세 가지 보장 중 무엇을 제공하는지에 있으며, 이를 정확히 구분하는 것이 Java 동시성 이해의 출발점이다.

<br>

### 1. 스레드와 공유 상태

Java에서 각 스레드는 **자기만의 스택**을 갖지만 **힙은 공유**한다(unit01). 따라서 지역 변수는 안전하고, 힙에 있는 객체의 필드·static 변수·컬렉션이 동시성 문제의 대상이 된다.

```java
public class Counter {
    private int count = 0;                 // 힙에 있는 공유 상태

    public void increment() { count++; }   // 읽기 → +1 → 쓰기, 세 단계로 실행됨
    public int get() { return count; }
}

// 스레드 2개가 각각 10,000번 increment()를 호출하면 결과는 20,000보다 작을 수 있음
```

```
 count++ 는 원자적이지 않다
 스레드 A: count 읽기(0) ──▶ 0+1 계산 ──────────────▶ count 쓰기(1)
 스레드 B:        count 읽기(0) ──▶ 0+1 계산 ──▶ count 쓰기(1)
 결과: 두 번 증가했지만 count == 1  → 경쟁 상태(Race Condition), 갱신 유실
```

- 이런 문제가 생기는 코드 구간을 **임계 영역(Critical Section)**이라 하며, 한 번에 한 스레드만 들어가도록 **상호 배제(Mutual Exclusion)**가 필요함

<br>

### 2. 동시성의 세 가지 보장

| **보장**              | **의미**                                                        | **깨질 때 증상**                                           | **제공하는 도구**                          |
| --------------------- | --------------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------ |
| **원자성(Atomicity)**  | 연산이 **중간에 끼어들 수 없는 한 덩어리**로 실행됨                 | `count++` 갱신 유실, `check-then-act` 오류                  | `synchronized`, `Lock`, `Atomic*`(unit09)  |
| **가시성(Visibility)** | 한 스레드가 쓴 값을 **다른 스레드가 볼 수 있음**                    | 플래그를 바꿨는데 다른 스레드의 루프가 안 끝남                | `volatile`, `synchronized`, `Lock`         |
| **순서(Ordering)**     | 코드에 쓴 순서대로 다른 스레드에게 **보이는 것**                    | 객체 참조는 보이는데 필드는 초기화 전 값                      | `volatile`, `synchronized`, `final`        |

`synchronized`는 셋을 모두 제공하고, `volatile`은 **가시성과 순서**만 제공하며 원자성은 제공하지 않는다. 이 한 문장이 두 키워드 차이의 핵심이다.

<br>

### 3. 메모리 가시성과 Java 메모리 모델

### 3-1. 가시성 문제가 생기는 이유

CPU 코어마다 **캐시와 레지스터**가 있고, JIT 컴파일러는 성능을 위해 메모리 읽기를 레지스터에 **호이스팅**하거나 명령을 **재배치**한다. 따라서 한 스레드가 변수에 쓴 값이 다른 코어의 스레드에게 언제 보일지 아무 보장이 없다.

```
 코어 1 (스레드 A)            메인 메모리            코어 2 (스레드 B)
 ┌────────────┐              ┌──────────┐          ┌────────────┐
 │ 캐시: flag=1│ ──언젠가──▶  │ flag = ? │ ◀──?──── │ 캐시: flag=0│  ← 계속 0을 볼 수 있음
 └────────────┘              └──────────┘          └────────────┘
```

```java
// 안티패턴: 종료 플래그가 다른 스레드에게 영원히 안 보일 수 있음
public class Worker implements Runnable {
    private boolean running = true;             // volatile 없음

    public void run() {
        while (running) { /* 작업 */ }          // JIT가 running을 레지스터에 올려 두면 무한 루프
    }
    public void stop() { running = false; }
}

// 개선: volatile로 가시성 보장
private volatile boolean running = true;
```

> ⚠️ 위 코드는 로컬에서 몇 번 돌려 보면 "잘 되는 것처럼" 보인다. 가시성 버그는 JIT 최적화가 적용된 뒤, 특정 하드웨어에서, 간헐적으로 나타나므로 테스트로 잡기 어렵다. "지금까지 문제없었다"는 동시성 코드가 올바르다는 근거가 될 수 없다.

<br>

### 3-2. happens-before 규칙

**Java 메모리 모델(JMM)**은 "어떤 쓰기가 어떤 읽기에 보이는가"를 **happens-before** 관계로 정의한다. A가 B보다 happens-before이면 A의 결과는 B에게 반드시 보인다.

| **규칙**                  | **내용**                                                                 |
| ------------------------- | ------------------------------------------------------------------------ |
| **프로그램 순서**           | 한 스레드 안에서는 코드 순서대로 (단일 스레드 관점의 결과만 보장, 실제 재배치는 가능) |
| **모니터 락**              | `synchronized` 블록의 **해제(unlock)**는 이후 같은 락의 **획득(lock)**보다 먼저   |
| **volatile**              | volatile 변수에 대한 **쓰기**는 이후 같은 변수의 **읽기**보다 먼저                |
| **스레드 시작·종료**        | `Thread.start()`는 그 스레드의 모든 동작보다 먼저, 스레드의 모든 동작은 `join()` 반환보다 먼저 |
| **전이성**                 | A → B, B → C 이면 A → C                                                   |

- volatile의 중요한 부가 효과: volatile 쓰기 **이전의 모든 쓰기**(일반 변수 포함)가 volatile 읽기 **이후**에 보임. 즉, volatile 변수 하나가 다른 데이터의 "발행(publish)" 신호 역할을 할 수 있음
- `synchronized`가 락을 풀 때 캐시를 메모리로 밀어내고, 획득할 때 캐시를 무효화한다고 이해하면 실무적으로 충분함 (실제 구현은 메모리 배리어 명령)

<br>

### 4. synchronized — 동작 원리와 사용 형태

모든 Java 객체는 **모니터(monitor, 내재 락)**를 하나씩 갖는다. `synchronized`는 이 모니터를 획득한 스레드만 블록을 실행하게 하고, 같은 스레드는 **재진입(reentrant)**이 가능하다.

```java
public class BankAccount {
    private long balance;
    private static int instances;

    public synchronized void deposit(long amount) {     // 락 대상: this
        balance += amount;
    }

    public void withdraw(long amount) {
        synchronized (this) {                            // 블록 단위: 임계 영역을 최소화
            if (balance < amount) throw new IllegalStateException("잔액 부족");
            balance -= amount;
        }
    }

    public static synchronized void countUp() {          // 락 대상: BankAccount.class
        instances++;
    }
}
```

| **형태**                          | **락 객체**              | **비고**                                               |
| --------------------------------- | ------------------------ | ------------------------------------------------------ |
| **인스턴스 메서드에 선언**          | `this`                   | 같은 객체의 모든 synchronized 메서드가 서로 배타적         |
| **static 메서드에 선언**            | `클래스.class`           | 인스턴스 락과 **별개** → 인스턴스·static 메서드는 동시 실행 가능 |
| **블록 `synchronized (obj)`**      | 지정한 객체              | 임계 영역 최소화, 락 분리에 유리. `private final Object lock` 권장 |

- 락 객체로 `String` 리터럴이나 `Integer` 같은 **공유될 수 있는 객체를 쓰면** 전혀 다른 코드와 같은 락을 잡아 데드락·성능 저하가 생김
- 원자성 + 가시성 + 순서를 모두 보장하지만, 경쟁이 심하면 스레드가 **블로킹(대기 상태 전환)**되어 컨텍스트 스위칭 비용이 발생함
- HotSpot은 경쟁이 없을 때 **경량 락(CAS 기반)**을 쓰고, 경쟁이 생기면 OS 뮤텍스를 쓰는 **중량 락**으로 전환하는 등 상황별 최적화를 함 (편향 락은 Java 15부터 기본 비활성화)

<br>

### 4-1. wait / notify

모니터를 가진 스레드는 `wait()`으로 락을 놓고 조건이 될 때까지 대기할 수 있고, 다른 스레드가 `notify()`/`notifyAll()`로 깨운다. **반드시 synchronized 블록 안에서** 호출해야 하며, 조건은 `while`로 재검사한다(가짜 깨어남 방지).

```java
public class BoundedBuffer<T> {
    private final Queue<T> queue = new ArrayDeque<>();
    private final int capacity;
    public BoundedBuffer(int capacity) { this.capacity = capacity; }

    public synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) wait();     // if가 아닌 while
        queue.add(item);
        notifyAll();
    }

    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) wait();
        T item = queue.poll();
        notifyAll();
        return item;
    }
}
```

> 💡 실무에서는 `wait/notify`를 직접 쓰기보다 `BlockingQueue`·`CountDownLatch` 같은 **고수준 API(unit09)**를 사용한다. 그래도 면접에서는 "`wait()`은 락을 놓고 대기하지만 `Thread.sleep()`은 락을 쥔 채 잔다", "왜 `while`로 조건을 검사하는가"를 설명할 수 있어야 한다.

<br>

### 5. volatile

`volatile` 변수는 **항상 메인 메모리에서 읽고 쓰며**, 해당 변수 주변의 명령 재배치를 금지한다. 락이 없으므로 블로킹이 발생하지 않지만, **복합 연산의 원자성은 보장하지 않는다**.

```java
// 안티패턴: volatile로 카운터를 보호하려는 시도 — count++는 여전히 세 단계
private volatile int count;
public void increment() { count++; }            // 갱신 유실 그대로 발생

// 개선 1: synchronized
public synchronized void increment() { count++; }

// 개선 2: 원자 클래스 (unit09)
private final AtomicInteger count = new AtomicInteger();
public void increment() { count.incrementAndGet(); }
```

**volatile이 적합한 경우**

- 한 스레드만 쓰고 여러 스레드가 읽는 **상태 플래그** (`running`, `shutdownRequested`)
- 값이 이전 값에 의존하지 않는 **단순 대입** (설정 값 교체 등)
- 64비트 `long`·`double` 필드의 **찢어진 읽기(word tearing)** 방지 — JMM은 일반 `long`/`double` 쓰기가 32비트 두 번으로 나뉘는 것을 허용하지만 volatile이면 원자적임

**DCL 싱글턴 — volatile이 필요한 전형적인 예**

```java
public class Config {
    private static volatile Config instance;      // volatile이 없으면 깨진 객체를 볼 수 있음

    public static Config getInstance() {
        if (instance == null) {                            // ① 락 없이 1차 검사
            synchronized (Config.class) {
                if (instance == null) {                    // ② 락 안에서 2차 검사
                    instance = new Config();               // 할당 → 생성자 실행 → 참조 대입 (재배치 가능)
                }
            }
        }
        return instance;
    }
}
```

- `new Config()`는 메모리 할당, 생성자 실행, 참조 대입의 세 단계이며 JIT가 순서를 바꿀 수 있음. volatile이 없으면 다른 스레드가 **생성자가 끝나기 전의 참조**를 보고 초기화되지 않은 필드를 읽을 수 있음
- 더 간단한 대안은 **static 홀더 클래스 관용구**(클래스 로딩의 지연·원자성 활용, unit01)나 **enum 싱글턴**임

<br>

### 6. synchronized vs volatile 비교와 선택

| **항목**                | **synchronized**                                  | **volatile**                                     |
| ----------------------- | ------------------------------------------------- | ------------------------------------------------ |
| **원자성**              | **보장** (블록 전체)                                | 단일 읽기/쓰기만. `++` 같은 복합 연산은 **미보장**   |
| **가시성**              | 보장                                              | 보장                                             |
| **순서 재배치 금지**      | 블록 경계 기준                                     | 변수 접근 기준                                    |
| **블로킹**              | 경쟁 시 스레드 대기 발생                            | **없음** (락 없음)                                 |
| **적용 대상**            | 메서드·블록                                        | 필드                                             |
| **비용**                | 락 획득·해제, 경쟁 시 컨텍스트 스위칭                 | 캐시 우회·배리어 비용 (락보다 훨씬 가벼움)           |
| **적합한 상황**          | 여러 변수의 일관성, 읽고-수정하고-쓰기               | 단일 플래그·설정 값 발행, 단일 쓰기 스레드            |

**선택 흐름**

```
 공유 상태를 여러 스레드가 수정하는가?
   ├─ 아니오 (한 스레드만 쓰고 나머지는 읽음, 값이 이전 값에 무관) ──▶ volatile
   └─ 예
        ├─ 단일 변수의 증가·CAS 정도 ──▶ Atomic* (unit09)
        └─ 여러 변수·불변식 유지 필요 ──▶ synchronized 또는 ReentrantLock (unit09)
 공유 상태 자체를 없앨 수 있는가? ──▶ 불변 객체(unit10)·스레드 한정(ThreadLocal)이 최선
```

> 💡 Java 21 가상 스레드 환경에서는 `synchronized` 블록 안에서 블로킹(I/O, `sleep`)하면 가상 스레드가 캐리어 스레드에 **고정(pinning)**되어 확장성이 떨어지므로 `ReentrantLock` 사용이 권장되었다. 이 제약은 이후 JDK에서 개선되었으므로 사용 중인 버전의 릴리스 노트를 확인해야 한다.

<br>

### 7. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                                   |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **synchronized와 volatile의 차이는?**              | synchronized는 **원자성·가시성·순서** 모두, volatile은 **가시성·순서**만. volatile은 락이 없어 블로킹 없음 |
| **volatile로 `count++`가 안전한가?**                | 아니다. 읽기-계산-쓰기 세 단계라 갱신 유실. `AtomicInteger`나 synchronized 필요                     |
| **가시성 문제는 왜 생기는가?**                       | CPU 캐시·레지스터·JIT 재배치 때문. JMM의 happens-before가 성립해야 다른 스레드의 쓰기가 보임          |
| **DCL에 volatile이 필요한 이유는?**                  | 객체 생성의 세 단계가 재배치되어 **생성자 완료 전 참조**가 노출될 수 있음                             |
| **static synchronized와 인스턴스 synchronized는 서로 배타적인가?** | 아니다. 락 객체가 `클래스.class`와 `this`로 다름                                     |
| **`wait()`과 `sleep()`의 차이는?**                  | `wait()`은 락을 놓고 대기(모니터 필요), `sleep()`은 락을 쥔 채 대기                                 |

- `ReentrantLock`·`Atomic*`·동시성 컬렉션·스레드 풀은 **unit09**, 공유 상태를 없애는 불변 설계는 **unit10**에서 이어짐
