## 코루틴 구조화된 동시성

**구조화된 동시성(Structured Concurrency)**은 코루틴을 반드시 어떤 스코프(부모) 안에서 시작하게 해, 부모의 생명주기가 자식의 생명주기를 포함하도록 강제하는 원칙이다. 이 문서는 Job 계층이 어떻게 만들어지고, 취소가 어떻게 전파되며, 어떤 Dispatcher를 선택해야 하는지를 다룬다.

<br>

### 1. 왜 구조화된 동시성인가

- 스레드나 `GlobalScope`처럼 **아무 데서나 시작되는 비동기 작업**은 누가 소유하고 언제 끝나는지 추적할 수 없어, 화면이 닫혀도 계속 돌거나(누수) 예외가 조용히 사라짐
- 구조화된 동시성에서는 모든 코루틴이 **부모 Job의 자식**으로 등록되므로 부모가 자식 완료를 기다리고, 부모 취소가 자식에게 전파되며, 자식 실패가 부모에게 보고됨
- 결과적으로 "이 함수가 반환되면 그 안에서 시작된 작업은 모두 끝나 있다"는 **호출 스택과 같은 직관**을 비동기 코드에서도 유지할 수 있음

```kotlin
// 안티패턴: 소유자가 없는 코루틴 → 화면이 사라져도 계속 실행되고 예외도 추적 불가
fun loadBad() {
    GlobalScope.launch { repository.fetch() }
}

// 개선: 생명주기를 가진 스코프에 소속시킴 (안드로이드라면 viewModelScope 등)
class UserViewModel(private val scope: CoroutineScope) {
    fun load() = scope.launch { repository.fetch() }
}
```

> 💡 `GlobalScope`는 `@DelicateCoroutinesApi`로 표시되어 있어 사용하면 경고가 뜬다. 애플리케이션 전체 수명과 같은 작업이 정말 필요하다면 직접 `CoroutineScope(SupervisorJob() + Dispatchers.Default)`를 만들어 소유자를 명확히 하는 편이 낫다.

<br>

### 2. Job 계층 구조

**스코프·컨텍스트·Job의 관계**

- **CoroutineContext**는 `Job`, `CoroutineDispatcher`, `CoroutineName`, `CoroutineExceptionHandler` 같은 원소를 담는 맵 형태의 자료구조로, `+` 연산자로 합칠 수 있음
- **CoroutineScope**는 컨텍스트를 들고 있는 껍데기이며, `launch`·`async`는 스코프의 컨텍스트를 **상속**한 뒤 **새 Job을 만들어 부모 Job의 자식으로 연결**함
- `launch`는 결과가 없는 `Job`을, `async`는 결과를 `await()`으로 받는 `Deferred<T>`(Job의 하위 타입)를 반환함

```
CoroutineScope(Job A + Dispatchers.Default)
 └── launch  → Job B  (부모: A)
      ├── launch → Job C (부모: B)
      └── async  → Deferred D (부모: B)

취소 전파:  A.cancel()  ⇒  B, C, D 모두 취소
완료 대기:  B는 C·D가 끝나야 완료 상태로 전이
실패 전파:  C에서 예외 ⇒ B 취소 ⇒ D 취소 ⇒ A까지 전파 (SupervisorJob이 아닐 때)
```

**Job의 생명주기**

```
New ─(start)─▶ Active ─(자식 완료 대기)─▶ Completing ─▶ Completed
                 │                            │
                 └─(cancel / 예외)─▶ Cancelling ─┴──▶ Cancelled
```

| **상태**       | **isActive** | **isCompleted** | **isCancelled** | **설명**                                      |
| -------------- | ------------ | --------------- | --------------- | --------------------------------------------- |
| **Active**     | true         | false           | false           | 본문 실행 중                                  |
| **Completing** | true         | false           | false           | 본문은 끝났지만 **자식을 기다리는 중**        |
| **Cancelling** | false        | false           | true            | 취소 요청됨, `finally` 등 정리 작업 진행 중   |
| **Completed**  | false        | true            | false           | 정상 종료                                     |
| **Cancelled**  | false        | true            | true            | 취소 또는 실패로 종료                         |

> ⚠️ `Completing` 상태가 있다는 것은 부모 코루틴의 본문이 끝나도 **자식이 남아 있으면 부모는 완료되지 않는다**는 뜻이다. `runBlocking` 안에서 `launch`한 작업을 `join()`하지 않아도 끝날 때까지 기다리는 이유가 바로 이것이다.

<br>

### 3. 스코프 빌더 — coroutineScope와 withContext

- `coroutineScope { }`는 **현재 코루틴 안에 자식 스코프를 만드는 suspend 함수**로, 블록 안의 모든 자식이 끝나야 반환되며 자식 중 하나가 실패하면 나머지를 취소하고 예외를 다시 던짐
- `withContext(ctx) { }`는 컨텍스트(주로 Dispatcher)만 바꿔 블록을 실행하고 결과를 반환함. 내부적으로 새 스코프를 만들지만 병렬 분기 목적이 아니라 **실행 환경 전환** 목적임
- 병렬 분해(parallel decomposition)는 `coroutineScope` 안에서 `async`를 여러 개 띄운 뒤 `awaitAll()`로 모으는 것이 정석임

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val profile = async { userApi.profile() }      // 부모: coroutineScope의 Job
    val orders  = async { orderApi.recent() }
    Dashboard(profile.await(), orders.await())     // 둘 중 하나가 실패하면 나머지도 취소됨
}

suspend fun readFile(path: String): String = withContext(Dispatchers.IO) {
    File(path).readText()                          // 블로킹 I/O는 IO 디스패처로 격리
}
```

<br>

### 4. 취소(Cancellation)

### 4-1. 취소는 협력적이다

- `job.cancel()`은 코루틴을 강제로 죽이지 않고 **취소 요청 플래그를 세울 뿐**이며, 코루틴이 다음 **중단 지점(suspension point)**에 도달할 때 `CancellationException`이 던져지면서 종료됨
- `delay()`, `yield()`, `withContext()`, 채널 송수신 등 `kotlinx.coroutines`의 모든 suspend 함수는 취소를 확인함
- **CPU 집약 루프처럼 중단 지점이 없는 코드는 취소되지 않음** → `isActive` 검사나 `ensureActive()`, `yield()`를 주기적으로 호출해야 함

```kotlin
// 안티패턴: 중단 지점이 없어 cancel()해도 끝까지 계산함
val job = scope.launch(Dispatchers.Default) {
    var i = 0
    while (i < 1_000_000_000) { i++ }
}

// 개선: 루프 안에서 취소 여부를 확인 (ensureActive는 취소 시 CancellationException을 던짐)
val job = scope.launch(Dispatchers.Default) {
    var i = 0
    while (i < 1_000_000_000) {
        ensureActive()
        i++
    }
}
```

<br>

### 4-2. 취소 전파 규칙과 정리 작업

- 부모 취소 → **모든 자식 취소** (아래 방향은 항상 전파됨)
- 자식이 취소(예외 없이 `cancel()`)되어도 → **부모는 취소되지 않음** (`CancellationException`은 정상 종료로 취급됨)
- 자식이 예외로 실패 → **부모와 형제 모두 취소** (일반 `Job`일 때. 예외 정책의 상세는 unit03 참고)
- 취소된 코루틴 안에서 다시 suspend 함수를 호출하면 즉시 `CancellationException`이 발생하므로, 정리 작업 중 suspend가 필요하면 `withContext(NonCancellable)`로 감싸야 함

```kotlin
scope.launch {
    try {
        repeat(100) { delay(100); println("작업 $it") }
    } finally {
        withContext(NonCancellable) {   // 취소 중에도 suspend 호출이 필요할 때
            connection.close()
        }
    }
}
```

> 💡 `withTimeout(ms) { }`는 시간 초과 시 `TimeoutCancellationException`(`CancellationException`의 하위 타입)을 던지고, `withTimeoutOrNull`은 null을 반환한다. 네트워크 호출에 시간 제한을 걸 때 `Thread.sleep` 기반 타이머 대신 이 함수를 쓰는 것이 구조화된 동시성과 맞는 방식이다.

<br>

### 5. Dispatcher 선택

**Dispatcher**는 코루틴이 **어떤 스레드(풀)에서 실행될지** 결정하는 컨텍스트 원소다. 작업의 성격에 맞는 디스패처를 고르지 않으면 스레드 풀이 고갈되거나 CPU가 놀게 된다.

| **Dispatcher**             | **스레드 풀**                                        | **용도**                                | **주의**                                                |
| -------------------------- | ---------------------------------------------------- | --------------------------------------- | ------------------------------------------------------- |
| **Dispatchers.Default**    | CPU 코어 수만큼의 스레드 (최소 2)                    | **CPU 집약** 작업: 정렬, 파싱, 계산     | 블로킹 I/O를 넣으면 코어 수만큼의 스레드가 막혀 전체가 멈춤 |
| **Dispatchers.IO**         | 필요 시 확장되는 풀 (기본 상한 64 또는 코어 수 중 큰 값) | **블로킹 I/O**: 파일, JDBC, 동기 HTTP | Default와 **스레드를 공유**하므로 IO↔Default 전환 비용이 낮음 |
| **Dispatchers.Main**       | UI 스레드 (Android·JavaFX 등 플랫폼 의존)            | UI 갱신                                 | 서버 환경에는 없음. 오래 걸리는 작업 금지                |
| **Dispatchers.Unconfined** | 호출한 스레드에서 시작, 재개 시 재개시킨 스레드      | 테스트·특수 상황                        | 실행 스레드가 예측 불가하므로 **일반 코드에서 비권장**  |
| **limitedParallelism(n)**  | 기존 디스패처 위에 병렬도 상한을 씌운 뷰             | DB 커넥션 수만큼 동시성 제한 등         | 새 스레드를 만드는 것이 아니라 **동시 실행 수만 제한**  |

```kotlin
// JDBC처럼 블로킹인 데이터 접근은 IO에, 커넥션 풀 크기에 맞춰 병렬도를 제한
val dbDispatcher = Dispatchers.IO.limitedParallelism(10)

suspend fun findUser(id: Long): User = withContext(dbDispatcher) {
    jdbcTemplate.queryForObject(...)   // 블로킹 호출이지만 다른 코루틴을 막지 않음
}
```

> ⚠️ `Dispatchers.IO`의 스레드 상한(기본 64)은 `kotlinx.coroutines.io.parallelism` 시스템 프로퍼티로 조정할 수 있으나 버전에 따라 세부 동작이 다를 수 있다. 상한을 무작정 올리기보다 `limitedParallelism`으로 **자원(커넥션·파일 핸들)의 실제 한계에 맞추는 것**이 올바른 접근이다.

<br>

### 6. 자주 하는 실수

- **스코프 없는 코루틴**: `GlobalScope.launch`나 매번 새로 만드는 `CoroutineScope(Job())`은 취소·예외 추적이 끊김. 반드시 생명주기 소유자와 묶는다
- **자식 Job을 직접 넘김**: `launch(Job())`처럼 새 Job을 컨텍스트로 넘기면 **부모-자식 관계가 끊겨** 구조화된 동시성이 깨짐. 계층을 분리하려면 `supervisorScope`나 `SupervisorJob`을 의도적으로 사용한다 (unit03 참고)
- **runBlocking 남용**: suspend 함수 내부나 서버 요청 처리 스레드에서 `runBlocking`을 호출하면 스레드를 점유해 이벤트 루프가 막힘. 진입점(`main`, 테스트)에서만 사용한다
- **취소 확인 누락**: CPU 루프에 `ensureActive()`가 없으면 취소 요청이 무시됨

<br>

### 7. 정리 — 면접·실무 체크포인트

| **질문**                                             | **핵심 답변**                                                                          |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------- |
| 구조화된 동시성이란?                                 | 모든 코루틴이 **부모 스코프에 소속**되어 완료 대기·취소 전파·실패 보고가 자동으로 이뤄지는 원칙 |
| `coroutineScope`와 `withContext`의 차이는?           | 전자는 **병렬 분해용 자식 스코프**, 후자는 **컨텍스트(디스패처) 전환용**                 |
| `cancel()`을 호출했는데 코루틴이 안 멈추는 이유는?   | 취소는 **협력적**이라 중단 지점이 없으면 감지 못 함 → `ensureActive()`·`yield()` 추가     |
| 취소 중 정리 작업에서 suspend 함수를 쓰려면?         | `withContext(NonCancellable)`로 감싼다                                                 |
| Default와 IO는 어떻게 고르는가?                      | **CPU 계산은 Default, 블로킹 I/O는 IO**. 자원 한계는 `limitedParallelism`으로 표현       |
| `launch(Job())`이 위험한 이유는?                     | 부모와의 연결이 끊겨 부모 취소가 전파되지 않고 예외가 새어 나감                          |

- 예외가 계층을 타고 어떻게 전파되고 `SupervisorJob`이 이를 어떻게 끊는지는 **unit03(코루틴 예외 처리)**을 참고할 것
