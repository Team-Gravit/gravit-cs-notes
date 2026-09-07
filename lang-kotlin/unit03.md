## 코루틴 예외 처리

코루틴의 예외는 일반 함수처럼 호출자에게 던져지는 것이 아니라 **Job 계층을 따라 부모로 전파**되며, 그 과정에서 형제 코루틴까지 취소시킨다. 이 문서는 예외 전파의 기본 규칙, `SupervisorJob`·`supervisorScope`로 전파를 끊는 방법, `CoroutineExceptionHandler`가 동작하는 조건, 그리고 `CancellationException`이 특별하게 취급되는 이유를 다룬다.

<br>

### 1. 예외 전파의 기본 규칙

- 자식 코루틴에서 잡히지 않은 예외가 발생하면 **부모를 취소**하고, 부모는 **나머지 자식(형제)을 모두 취소**한 뒤 자신의 부모로 예외를 올려 보냄
- 이 전파는 **루트 코루틴**(부모 Job이 없는 코루틴)에 도달할 때까지 계속되며, 루트에서 최종적으로 처리됨
- 즉 "하나가 실패하면 전부 실패"가 기본 정책이며, 이는 unit02의 구조화된 동시성이 의도한 동작임

```
scope(Job)
 ├── launch A ── 예외 발생!
 │        └─▶ 부모 Job 취소 요청
 ├── launch B ── (아무 잘못 없지만) 취소됨
 └── launch C ── 취소됨
scope의 Job은 Cancelled 상태 → 이후 launch는 즉시 취소됨 (스코프가 죽음)
```

> ⚠️ 일반 `Job`으로 만든 스코프는 자식 하나의 실패로 **스코프 자체가 취소**되어 더는 새 코루틴을 실행할 수 없다. 화면이나 서버 컴포넌트처럼 오래 사는 스코프가 이유 없이 "죽어 있는" 버그의 원인은 대부분 이것이다.

<br>

### 2. launch와 async의 예외 노출 방식

| **구분**                  | **launch**                                          | **async**                                                   |
| ------------------------- | --------------------------------------------------- | ----------------------------------------------------------- |
| **예외 노출 시점**        | 발생 즉시 부모로 전파                               | `await()` 호출 시 호출자에게 던짐                           |
| **부모로의 전파**         | 항상 전파                                           | **일반 Job의 자식이면 await 전에도 부모를 취소**함           |
| **try/catch 위치**        | 코루틴 본문 안                                      | `await()`를 감싸는 곳                                       |
| **루트 코루틴일 때**      | `CoroutineExceptionHandler` 또는 기본 핸들러로 전달  | await 전까지 예외가 **보관**됨 (핸들러로 가지 않음)          |

```kotlin
// 안티패턴: await를 try/catch로 감쌌지만, async가 일반 Job의 자식이라 부모가 먼저 취소됨
val scope = CoroutineScope(Job())
scope.launch {
    val d = async { throw IllegalStateException("실패") }
    try { d.await() } catch (e: IllegalStateException) { /* 잡히긴 하지만 */ }
    println("여기 도달 못 함")   // 부모 launch가 이미 취소됨
}

// 개선: 예외 격리가 목적이라면 supervisorScope 안에서 async
scope.launch {
    supervisorScope {
        val d = async { throw IllegalStateException("실패") }
        try { d.await() } catch (e: IllegalStateException) { println("격리된 실패") }
        println("정상 진행")
    }
}
```

> 💡 "async의 예외는 await에서 잡으면 된다"는 설명은 **절반만 맞다**. async가 일반 부모의 자식이면 예외는 즉시 부모로도 전파된다. await에서 안전하게 잡으려면 `supervisorScope` 또는 `SupervisorJob` 아래에 두어야 한다.

<br>

### 3. SupervisorJob과 supervisorScope

**SupervisorJob**은 자식의 실패가 **자신과 다른 자식에게 전파되지 않도록** 하는 Job이다. 부모 → 자식 방향의 취소는 그대로 동작하지만, 자식 → 부모 방향의 실패 보고만 차단한다.

```
SupervisorJob
 ├── launch A ── 예외 발생 → A만 취소, 핸들러로 예외 전달
 ├── launch B ── 계속 실행
 └── launch C ── 계속 실행
```

- `SupervisorJob()`: 스코프를 만들 때 컨텍스트에 넣음. `CoroutineScope(SupervisorJob() + Dispatchers.Default)`가 오래 사는 컴포넌트 스코프의 표준 형태임
- `supervisorScope { }`: `coroutineScope`의 감독 버전. 블록 안에서 시작된 **직접 자식**끼리 실패를 격리함

```kotlin
suspend fun loadHome(): HomeData = supervisorScope {
    val banner = async { bannerApi.load() }        // 실패해도 다른 async에 영향 없음
    val feed   = async { feedApi.load() }
    HomeData(
        banner = runCatching { banner.await() }.getOrNull(),
        feed   = feed.await()                        // 핵심 데이터는 실패 시 그대로 던짐
    )
}
```

**자주 하는 실수 — 자식 코루틴에 SupervisorJob을 넘기기**

```kotlin
// 안티패턴: launch의 인자로 SupervisorJob을 넘기면 부모와의 연결이 끊긴다
scope.launch(SupervisorJob()) {
    launch { throw RuntimeException() }   // 이 예외는 형제를 취소함 (감독 효과 없음)
}
```

- `launch(SupervisorJob())`의 감독 대상은 그 launch의 **직접 자식**이 아니라 launch 자체이며, 오히려 `scope`와 launch 사이의 부모-자식 관계가 끊어져 구조화된 동시성이 깨짐
- 감독이 필요한 지점에서는 **스코프 생성 시 SupervisorJob을 넣거나 `supervisorScope`를 사용**해야 함

<br>

### 4. CoroutineExceptionHandler

`CoroutineExceptionHandler`는 **잡히지 않은 예외가 최종적으로 도달하는 곳**이다. 일반 try/catch처럼 예외를 복구하는 용도가 아니라 로깅·리포팅·애플리케이션 종료 판단 같은 **마지막 처리**를 위한 장치다.

```kotlin
val handler = CoroutineExceptionHandler { _, e -> log.error("처리되지 않은 예외", e) }

val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default + handler)
scope.launch { throw IllegalStateException("A") }   // handler가 받음, 스코프는 살아 있음
scope.launch { println("B는 정상 실행") }
```

**핸들러가 동작하는 조건**

| **위치**                                        | **동작 여부** | **이유**                                                       |
| ----------------------------------------------- | ------------- | -------------------------------------------------------------- |
| **루트 코루틴**(스코프에서 직접 launch)         | 동작          | 더 이상 전파할 부모가 없어 핸들러가 최종 처리자가 됨            |
| **SupervisorJob·supervisorScope의 직접 자식**   | 동작          | 감독 Job이 예외를 부모로 올리지 않고 자식이 스스로 처리하게 함  |
| **일반 launch의 자식 코루틴**                   | **무시됨**    | 예외가 부모로 전파되므로 자식에 붙인 핸들러는 사용되지 않음     |
| **async 루트 코루틴**                           | 무시됨        | 예외는 `Deferred`에 보관되어 `await()`에서 던져짐               |
| **coroutineScope 내부**                         | 무시됨        | `coroutineScope`가 예외를 **다시 던지므로** 호출자가 잡아야 함  |

> ⚠️ 핸들러가 없는 루트 코루틴에서 예외가 나면 JVM은 `Thread.uncaughtExceptionHandler`로, Android는 **앱 크래시**로 이어진다. 오래 사는 스코프에는 `SupervisorJob`과 핸들러를 **함께** 두는 것이 기본 구성이다.

<br>

### 5. CancellationException의 특수성

### 5-1. 취소는 예외로 구현되지만 실패가 아니다

- 코루틴 취소는 중단 지점에서 `CancellationException`을 던지는 방식으로 구현됨 (unit02 참고)
- 코루틴 라이브러리는 `CancellationException`을 **정상 종료 신호**로 취급함 → 부모로 전파되지 않고, 형제를 취소하지 않으며, `CoroutineExceptionHandler`에도 전달되지 않음
- `TimeoutCancellationException`(`withTimeout`)도 이 하위 타입이므로 같은 규칙을 따름

```
일반 예외          : 자식 실패 → 부모 취소 → 형제 취소 → 핸들러 호출
CancellationException: 해당 코루틴만 종료 → 부모는 정상으로 간주 → 핸들러 호출 안 됨
```

<br>

### 5-2. 흔한 함정 — 취소 예외를 삼키는 catch

`catch (e: Exception)`이나 `runCatching { }`은 `CancellationException`까지 잡아 버린다. 취소 신호가 삼켜지면 코루틴은 취소된 줄 모르고 계속 실행되어 **취소가 무력화**된다.

```kotlin
// 안티패턴: 취소 신호까지 삼켜 루프가 멈추지 않음
scope.launch {
    while (true) {
        try {
            delay(1000)
            poll()
        } catch (e: Exception) {   // CancellationException도 여기 잡힘
            log.warn("폴링 실패", e)
        }
    }
}

// 개선: 취소 예외는 반드시 다시 던진다
scope.launch {
    while (true) {
        try {
            delay(1000)
            poll()
        } catch (e: CancellationException) {
            throw e                              // 취소는 그대로 통과
        } catch (e: Exception) {
            log.warn("폴링 실패", e)
        }
    }
}
```

- `runCatching`을 코루틴 안에서 쓸 때도 결과의 예외가 `CancellationException`이면 다시 던지는 확장 함수를 만들어 사용하는 것이 안전함
- 취소된 코루틴에서 정리 작업 중 suspend 함수가 필요하면 `withContext(NonCancellable)`을 사용함 (unit02 참고)

> 💡 "`catch (e: Exception)`을 코루틴 안에서 쓰면 안 되는 이유는?"은 코루틴 면접의 대표 질문이다. 답은 "**취소 신호를 삼켜 구조화된 동시성이 깨지기 때문**이며, `CancellationException`은 반드시 재전파해야 한다"이다.

<br>

### 6. 예외 처리 전략 선택 기준

| **상황**                                          | **권장 도구**                              | **이유**                                             |
| ------------------------------------------------- | ------------------------------------------ | ---------------------------------------------------- |
| 특정 suspend 호출 하나의 실패를 복구              | 본문 안 `try/catch`                        | 가장 좁은 범위에서 처리, 취소 예외는 재전파           |
| 여러 병렬 작업 중 하나라도 실패하면 전체 실패      | `coroutineScope` + `async`                 | **기본 전파 규칙**이 원하는 동작과 일치               |
| 병렬 작업을 서로 **독립적으로** 실패시키고 싶음    | `supervisorScope` + `async` + 개별 `try`   | 실패 격리, 부분 결과 수집 가능                        |
| 오래 사는 컴포넌트 스코프                          | `SupervisorJob` + `CoroutineExceptionHandler` | 하나의 실패로 스코프가 죽지 않고, 예외는 로깅됨     |
| 시간 제한                                          | `withTimeout` / `withTimeoutOrNull`        | 취소 메커니즘을 그대로 사용, 타임아웃도 취소로 처리   |
| 여러 자식이 동시에 실패                            | 첫 예외가 대표, 나머지는 `suppressed`에 첨부 | 로그에서 `e.suppressed`를 확인해야 원인을 놓치지 않음 |

<br>

### 7. 정리 — 면접·실무 체크포인트

- 코루틴 예외는 **자식 → 부모 → 형제 취소** 순으로 전파되며, 일반 `Job` 스코프는 자식 하나의 실패로 죽음
- `async`의 예외는 `await()`에서 던져지지만, **일반 부모 아래에서는 await 전에 부모가 먼저 취소**됨
- `SupervisorJob`·`supervisorScope`는 **자식 → 부모 방향의 실패 전파만** 차단하고 부모 → 자식 취소는 유지함
- `SupervisorJob`을 `launch`의 인자로 넘기는 것은 감독 효과가 없고 계층만 끊는 실수임
- `CoroutineExceptionHandler`는 **루트 코루틴 또는 감독 스코프의 직접 자식**에서만 동작하는 최종 처리자임
- `CancellationException`은 실패가 아닌 **정상 종료 신호**이므로 `catch (e: Exception)`으로 삼키면 안 되고 반드시 재전파함
- Job 계층과 취소 메커니즘의 기초는 **unit02(코루틴 구조화된 동시성)**, Flow의 `catch` 연산자는 **unit04(Flow)**를 참고할 것
