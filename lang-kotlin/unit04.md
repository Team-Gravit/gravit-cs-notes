## Flow

**Flow**는 코루틴 위에서 동작하는 **비동기 데이터 스트림**으로, 값을 하나씩 순차적으로 방출(emit)하고 수집(collect)하는 추상화다. 이 문서는 cold와 hot 스트림의 차이, `StateFlow`·`SharedFlow`의 설계, 그리고 생산자와 소비자의 속도 차이를 다루는 버퍼링 전략을 다룬다.

<br>

### 1. Flow의 기본 구조

- `flow { emit(x) }` 빌더로 생산자를 정의하고, `map`·`filter` 같은 **중간 연산자**를 연결한 뒤, `collect`·`toList`·`first` 같은 **종단 연산자**가 호출될 때 비로소 실행됨
- suspend 함수가 값 하나를 돌려준다면, Flow는 **여러 값을 시간에 걸쳐** 돌려주는 suspend 스트림임
- 기본적으로 **순차 실행**: 값 하나가 생산 → 모든 연산자 통과 → 소비된 뒤에야 다음 값이 생산됨

```kotlin
fun prices(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)          // suspend 가능
        emit(i * 100)       // 값 방출
    }
}

suspend fun main() {
    prices()
        .filter { it > 100 }
        .map { "₩$it" }
        .collect { println(it) }   // 종단 연산자: 이때 flow 블록이 실행됨
}
```

```
생산자 ──emit(100)──▶ filter(탈락)
생산자 ──emit(200)──▶ filter ──▶ map("₩200") ──▶ collect
생산자 ──emit(300)──▶ filter ──▶ map("₩300") ──▶ collect
(각 단계가 같은 코루틴에서 순차 실행됨)
```

> 💡 Flow의 `emit`은 suspend 함수이므로 소비자가 느리면 생산자가 **자동으로 대기**한다. 별도의 백프레셔(backpressure) 프로토콜 없이 suspend 자체가 역압 역할을 한다는 점이 RxJava와의 큰 차이다.

<br>

### 2. Cold 스트림과 Hot 스트림

| **구분**            | **Cold (일반 Flow)**                              | **Hot (StateFlow · SharedFlow)**                        |
| ------------------- | ------------------------------------------------- | ------------------------------------------------------- |
| **실행 시점**       | **collect할 때마다** 처음부터 새로 실행           | 수집자와 무관하게 **이미 활성** 상태                    |
| **수집자 간 공유**  | 수집자마다 **독립적인** 실행 (N번 collect = N번 실행) | 여러 수집자가 **같은 방출**을 공유                   |
| **완료**            | 블록이 끝나면 완료됨                              | 완료되지 않음 (`collect`가 영원히 중단됨)               |
| **비유**            | 유튜브 다시보기: 재생할 때마다 처음부터           | 라이브 방송: 늦게 들어오면 이전 내용은 못 봄 (replay 제외) |
| **용도**            | DB 조회, 네트워크 응답, 파일 읽기                 | UI 상태, 이벤트 버스, 센서 값                           |

```kotlin
val cold = flow { println("실행"); emit(1) }
cold.collect { }   // "실행" 출력
cold.collect { }   // "실행" 또 출력 — 수집자마다 새로 실행됨

val hot = MutableSharedFlow<Int>()
scope.launch { hot.collect { println("A: $it") } }
scope.launch { hot.collect { println("B: $it") } }
hot.emit(1)        // A와 B 모두 같은 값을 받음 (수집자가 없었다면 사라졌을 값)
```

> ⚠️ cold Flow를 여러 곳에서 collect하면 **네트워크 요청이 수집자 수만큼 반복**된다. 하나의 결과를 여러 구독자가 공유해야 한다면 `shareIn`·`stateIn`으로 hot으로 변환해야 한다.

<br>

### 3. StateFlow — 상태 보관용 hot 스트림

- `MutableStateFlow(initial)`로 만들며 **항상 현재 값을 하나 보유**함 (`value`로 동기 접근 가능)
- 새 수집자는 즉시 **현재 값**을 받고, 이후 변경을 계속 받음 (replay = 1과 유사)
- **conflated**: 소비자가 느리면 중간 값을 건너뛰고 **최신 값만** 전달함
- `equals` 기준으로 같은 값을 다시 설정하면 **방출하지 않음** (`distinctUntilChanged` 내장)

```kotlin
class CounterViewModel(private val scope: CoroutineScope) {
    private val _state = MutableStateFlow(CounterState(count = 0))
    val state: StateFlow<CounterState> = _state.asStateFlow()   // 외부에는 읽기 전용으로 노출

    fun increment() {
        _state.update { it.copy(count = it.count + 1) }   // update는 CAS 기반 원자적 갱신
    }
}
```

- `_state.value = _state.value.copy(...)`는 두 스레드가 동시에 갱신하면 한쪽이 유실될 수 있음. **`update { }`**는 CAS(compare-and-set)로 재시도하므로 안전함
- 상태 클래스는 `data class`로 만들어 `equals`가 값 기준으로 동작하게 해야 중복 방출 억제가 의도대로 작동함 (unit06 참고)

<br>

### 4. SharedFlow — 이벤트 브로드캐스트용 hot 스트림

`MutableSharedFlow`는 세 가지 생성 파라미터로 동작이 결정된다.

| **파라미터**            | **기본값** | **의미**                                                                  |
| ----------------------- | ---------- | ------------------------------------------------------------------------- |
| **replay**              | 0          | 새 수집자에게 **다시 보내 줄 최근 값의 개수**                              |
| **extraBufferCapacity** | 0          | replay 외에 추가로 보관할 버퍼 크기 (수집자가 느릴 때 emit이 대기하지 않게 함) |
| **onBufferOverflow**    | `SUSPEND`  | 버퍼가 찼을 때: `SUSPEND`(emit 대기) / `DROP_OLDEST` / `DROP_LATEST`       |

```kotlin
// 일회성 UI 이벤트(토스트, 화면 이동) 용도: 놓치지 않되 느린 수집자 때문에 emit이 막히지 않게
private val _events = MutableSharedFlow<UiEvent>(
    replay = 0,
    extraBufferCapacity = 64,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
val events: SharedFlow<UiEvent> = _events.asSharedFlow()

fun notify(e: UiEvent) {
    _events.tryEmit(e)   // 버퍼가 있으므로 suspend 없이 성공 (버퍼 0이면 수집자 없을 때 false)
}
```

- **수집자가 없을 때** `replay = 0`이면 emit된 값은 그냥 **사라짐** — 화면이 백그라운드에 있을 때 이벤트를 잃는 흔한 원인
- `tryEmit`은 버퍼 여유가 없으면 `false`를 돌려주며, 기본 설정(버퍼 0)에서는 수집자가 있어도 대부분 실패함

> 💡 `StateFlow`는 사실상 `SharedFlow(replay = 1, onBufferOverflow = DROP_OLDEST)`에 **중복 제거**를 더한 특수형이다. "상태"는 StateFlow, "이벤트"는 SharedFlow라는 구분이 기본 선택 기준이다.

<br>

### 5. cold를 hot으로 — shareIn과 stateIn

- `shareIn(scope, started, replay)` → `SharedFlow`, `stateIn(scope, started, initialValue)` → `StateFlow`
- 업스트림 cold Flow를 **주어진 scope에서 한 번만 실행**하고 여러 수집자에게 공유함
- `started` 파라미터가 업스트림을 **언제 시작하고 언제 멈출지** 결정함

| **SharingStarted**             | **시작 시점**       | **중지 시점**                                   | **적합한 상황**                          |
| ------------------------------ | ------------------- | ----------------------------------------------- | ---------------------------------------- |
| **Eagerly**                    | 즉시                | scope 취소 시                                   | 항상 최신이어야 하는 전역 상태           |
| **Lazily**                     | 첫 수집자 등장 시   | scope 취소 시                                   | 한 번 시작하면 계속 유지해도 되는 경우   |
| **WhileSubscribed(timeout)**   | 첫 수집자 등장 시   | 마지막 수집자가 떠난 뒤 **timeout 경과 시**     | UI 화면 상태 (화면 회전 등 짧은 이탈 허용) |

```kotlin
val user: StateFlow<User?> = repository.observeUser()        // cold Flow
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),      // 5초 내 재구독이면 업스트림 유지
        initialValue = null
    )
```

<br>

### 6. 버퍼링과 속도 차이 처리

기본 Flow는 생산과 소비가 **같은 코루틴에서 순차** 실행되므로, 생산 100ms + 소비 300ms면 값 하나에 400ms가 걸린다. 버퍼링 연산자는 생산자와 소비자를 **별도 코루틴으로 분리**해 이 병목을 해소한다.

| **연산자**           | **동작**                                                    | **값 유실** | **적합한 상황**                          |
| -------------------- | ----------------------------------------------------------- | ----------- | ---------------------------------------- |
| **buffer(n)**        | 생산자를 분리하고 n개까지 버퍼에 쌓음 (기본 64)             | 없음        | 모든 값을 처리해야 하지만 속도 차이가 있을 때 |
| **conflate()**       | 소비자가 바쁘면 **중간 값을 버리고 최신 값만** 전달          | 있음        | 진행률·좌표처럼 최신 값만 의미 있을 때   |
| **collectLatest { }** | 새 값이 오면 **진행 중인 소비 블록을 취소**하고 새로 시작   | 있음        | 검색어 입력 시 이전 검색 취소            |
| **flowOn(ctx)**      | **업스트림**의 실행 컨텍스트를 바꿈 (내부적으로 버퍼 생성)  | 없음        | 생산은 IO, 소비는 Main에서 할 때         |

```
buffer 없음  : 생산(100) → 소비(300) → 생산(100) → 소비(300)   총 800ms / 2개
buffer(64)   : 생산(100) 생산(100) ...                            생산자 코루틴
               └──────▶ 소비(300) → 소비(300)                     소비자 코루틴  총 ≈ 700ms / 2개
conflate     : 생산 1,2,3 → 소비 1 (2는 건너뜀) → 소비 3
collectLatest: 생산 1,2,3 → 소비 1 시작 → 2 도착, 1 취소 → 3 도착, 2 취소 → 소비 3 완료
```

```kotlin
searchQuery                       // StateFlow<String>
    .debounce(300)                // 입력이 300ms 멈췄을 때만 통과
    .distinctUntilChanged()
    .flatMapLatest { q -> repository.search(q) }   // 이전 검색 Flow는 취소
    .flowOn(Dispatchers.IO)       // 위쪽(검색·네트워크)만 IO에서 실행
    .collect { render(it) }       // 호출한 컨텍스트(Main)에서 실행
```

> ⚠️ `flowOn`은 **자기보다 위쪽(업스트림)** 연산자에만 영향을 준다. `collect` 블록의 컨텍스트를 바꾸려면 `flowOn`이 아니라 수집 자체를 `withContext`나 다른 스코프에서 해야 한다. 또한 `flow { }` 안에서 `withContext`로 컨텍스트를 바꿔 `emit`하는 것은 **컨텍스트 보존 규칙 위반**으로 예외가 발생하며, 이런 경우에는 `channelFlow`를 사용한다.

<br>

### 7. 예외와 완료 처리

- `catch { }`는 **업스트림에서 발생한 예외만** 잡음. `collect` 블록 안의 예외는 잡지 못하므로, 소비 로직은 `onEach { }`로 올리고 마지막에 `catch`를 두는 패턴이 흔함
- `retry(n) { e -> 조건 }`은 예외 발생 시 업스트림을 재실행함
- `onCompletion { cause -> }`는 정상 완료·예외·취소 모두에서 호출되며 `cause`로 구분함
- Flow 수집은 코루틴 취소를 그대로 따르므로 별도의 구독 해제 코드가 필요 없음 (unit03 참고)

```kotlin
repository.observeOrders()
    .onEach { render(it) }
    .retry(3) { it is IOException }
    .catch { e -> showError(e) }          // onEach·retry 위쪽의 예외만 도달
    .launchIn(viewModelScope)             // collect를 launch로 감싼 단축 표현
```

<br>

### 8. 정리 — 면접·실무 체크포인트

| **질문**                                     | **핵심 답변**                                                                   |
| -------------------------------------------- | ------------------------------------------------------------------------------- |
| cold와 hot의 차이는?                         | cold는 **collect마다 새로 실행**, hot은 수집자와 무관하게 활성 상태이며 방출을 공유 |
| StateFlow와 SharedFlow는 언제 구분하는가?    | **상태**(현재 값 보유, 중복 제거, conflated)는 StateFlow, **이벤트**는 SharedFlow  |
| SharedFlow에서 이벤트가 사라지는 이유는?     | `replay = 0`이고 **수집자가 없으면** 값이 보관되지 않음                          |
| `WhileSubscribed(5000)`의 의미는?            | 마지막 수집자 이탈 후 5초까지 업스트림을 유지 → 화면 회전 시 재요청 방지          |
| buffer·conflate·collectLatest 차이는?        | buffer는 **전부 처리**, conflate는 **최신만**, collectLatest는 **진행 중 작업 취소** |
| `flowOn`의 영향 범위는?                      | **업스트림만**. collect 블록의 컨텍스트는 바뀌지 않음                             |
| `catch`가 collect의 예외를 못 잡는 이유는?   | `catch`는 업스트림 전용 → 소비 로직을 `onEach`로 올려야 함                       |

- Flow는 suspend 기반 역압을 갖춘 cold 스트림이며, `stateIn`·`shareIn`으로 hot으로 승격함
- 코루틴 취소·예외 전파의 기본 규칙은 **unit02·unit03**을 참고할 것
