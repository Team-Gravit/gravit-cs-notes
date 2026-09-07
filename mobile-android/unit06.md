## 생명주기 인식 데이터 수집

ViewModel이 **StateFlow·SharedFlow**로 노출한 데이터를 UI가 수집할 때, 화면이 보이지 않는 동안에도 수집이 계속되면 배터리 낭비와 크래시가 발생한다. `repeatOnLifecycle`과 `collectAsStateWithLifecycle`은 **생명주기 상태에 맞춰 수집을 시작·중단**해 이 문제를 구조적으로 해결하는 Jetpack의 표준 도구다.

<br>

### 1. 왜 "생명주기 인식" 수집이 필요한가

- `lifecycleScope.launch { flow.collect { } }`로 수집을 시작하면 코루틴은 **Activity가 파괴될 때까지** 살아 있음
- 홈 버튼으로 앱을 백그라운드에 보내도(`onStop`) 수집은 계속됨 → 화면에 보이지도 않는 UI를 갱신하고, 상위 데이터 소스(위치·DB 관찰 등)도 계속 동작함
- Fragment의 경우 View가 파괴된 뒤(`onDestroyView`) 값이 도착하면 이미 사라진 View에 접근해 **크래시**가 날 수 있음
- 즉 수집 범위는 "컴포넌트가 살아 있는 동안"이 아니라 "**화면이 실제로 보이는 동안**"으로 좁혀야 함

```kotlin
// 안티패턴: 백그라운드에서도 계속 수집됨
class UserFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        lifecycleScope.launch {
            viewModel.users.collect { render(it) }   // onStop 이후에도 살아 있음
        }
    }
}
```

> 💡 "LiveData 대신 Flow를 쓰면 무엇이 달라지나요?"라는 질문의 핵심이 여기에 있다. LiveData는 생명주기를 **스스로 인식**해 STARTED 미만에서는 값을 전달하지 않지만, Flow는 그런 개념이 없으므로 **수집하는 쪽이 생명주기를 맞춰 줘야** 한다.

<br>

### 2. StateFlow와 SharedFlow

### 2-1. 두 Flow의 성격

| **항목**               | **StateFlow**                                    | **SharedFlow**                                        |
| ---------------------- | ------------------------------------------------ | ----------------------------------------------------- |
| **성격**               | **상태(State)** 보관 — 항상 현재 값이 있음       | **이벤트(Event)** 방송 — 값을 보관하지 않을 수 있음   |
| **초기값**             | 필수                                             | 없음                                                  |
| **새 구독자가 받는 값** | **최신 값 1개 즉시**                            | `replay` 개수만큼 (기본 0 → 아무것도 받지 못함)       |
| **중복 값**            | `equals`로 같은 값이면 **방출 생략**             | 모두 방출                                             |
| **버퍼·손실**          | 최신 값만 유지 (중간 값은 건너뜀)                | `extraBufferCapacity`·`onBufferOverflow`로 조절       |
| **적합한 용도**        | 화면 UI 상태, 로딩·에러 플래그, 목록             | 구독자 여러 명에게 같은 이벤트 방송                   |

- `StateFlow`는 `SharedFlow(replay = 1)`에 중복 제거와 초기값이 더해진 특수형으로 볼 수 있음
- 뜨거운(Hot) 스트림이므로 구독자가 없어도 값이 존재하며, 여러 구독자가 **같은 값을 공유**함

<br>

### 2-2. ViewModel에서 노출하기

```kotlin
class ProductViewModel(repository: ProductRepository) : ViewModel() {

    // 상태: 항상 현재 값이 있고, 새 구독자는 최신 값을 즉시 받음
    val uiState: StateFlow<ProductUiState> = repository.observeProducts()
        .map { ProductUiState.Success(it) as ProductUiState }
        .catch { emit(ProductUiState.Error(it.message ?: "오류")) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),   // 구독자가 사라진 뒤 5초 후 상위 수집 중단
            initialValue = ProductUiState.Loading,
        )

    // 이벤트: 구독자가 있을 때만 전달됨
    private val _events = MutableSharedFlow<ProductEvent>()
    val events: SharedFlow<ProductEvent> = _events.asSharedFlow()

    fun onAddToCart(id: Long) {
        viewModelScope.launch { _events.emit(ProductEvent.AddedToCart(id)) }
    }
}
```

- `stateIn`의 **`WhileSubscribed(5_000)`**: 구독자가 0명이 된 뒤 5초 동안 기다렸다가 상위 Flow 수집을 멈춤. 화면 회전(구독 해제 후 즉시 재구독)에는 상위 스트림을 유지하고, 진짜 백그라운드 진입에만 중단하는 관용구임
- `Lazily`는 첫 구독 후 영원히 유지, `Eagerly`는 즉시 시작해 영원히 유지 → 대부분의 UI 상태에는 `WhileSubscribed`가 적합함

> ⚠️ `MutableSharedFlow`의 기본 설정(`replay = 0`)에서 `emit`은 **구독자가 없으면 값을 버린다**. 화면이 백그라운드에 있을 때 발생한 이벤트는 사용자가 돌아와도 받지 못하므로, 일회성 이벤트를 SharedFlow로 처리할 때는 유실 가능성을 반드시 고려해야 한다 (5번 참고).

<br>

### 3. repeatOnLifecycle — View 시스템의 표준 수집

- `lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) { }`는 생명주기가 **STARTED 이상이 될 때마다 블록을 새로 시작**하고, **미만으로 떨어지면 블록을 취소**함
- 블록 안의 코루틴이 취소되므로 상위 Flow 수집도 함께 멈춤 → `WhileSubscribed`와 결합하면 데이터 소스까지 정리됨
- 그 자체가 `suspend` 함수이며, 생명주기가 DESTROYED가 될 때 반환됨

```
생명주기   onStart ─── onResume ─── onPause ─── onStop ──── onStart ─── onResume ─── onDestroy
            │                                     │           │                          │
repeatOn    ├── 블록 시작(수집) ───────────────── 취소 ──── ├── 블록 재시작(수집) ──── 취소·반환
Lifecycle   ▲                                                 ▲
(STARTED)   새 구독 → StateFlow 최신 값 즉시 수신              재구독 → 다시 최신 값 수신
```

```kotlin
class ProductFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {                 // Fragment는 viewLifecycleOwner!
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                launch { viewModel.uiState.collect { render(it) } }   // 여러 Flow는 각각 launch
                launch { viewModel.events.collect { handle(it) } }
            }
        }
    }
}
```

- 단일 Flow만 수집한다면 `flow.flowWithLifecycle(lifecycle, STARTED)` 연산자로 더 짧게 쓸 수 있음. 내부적으로 `repeatOnLifecycle`을 사용함
- Fragment에서 `this.lifecycleScope`를 쓰면 View가 파괴된 뒤에도 코루틴이 살아 있으므로 **반드시 `viewLifecycleOwner`**를 사용함

❗️**`launchWhenStarted`는 사용하지 않는다**: 이전에 쓰이던 `lifecycleScope.launchWhenStarted { }`는 STARTED 미만에서 코루틴을 **취소가 아니라 일시 중단**만 한다. 상위 Flow는 계속 값을 만들어 버퍼에 쌓이고 리소스도 해제되지 않는다. 현재는 지원 중단(deprecated) 상태이므로 `repeatOnLifecycle`로 대체한다.

<br>

### 4. Compose에서의 수집 — collectAsStateWithLifecycle

- `collectAsState()`는 컴포지션에 있는 동안 계속 수집하므로 위와 같은 백그라운드 수집 문제가 있음
- **`collectAsStateWithLifecycle()`**(lifecycle-runtime-compose 라이브러리)은 내부적으로 `repeatOnLifecycle(STARTED)`를 사용해 화면이 보일 때만 수집함. 안드로이드 앱에서는 이것이 **기본 선택**임

```kotlin
@Composable
fun ProductRoute(viewModel: ProductViewModel = hiltViewModel()) {
    // 안티패턴: val state by viewModel.uiState.collectAsState()   → 백그라운드에서도 수집
    val state by viewModel.uiState.collectAsStateWithLifecycle()   // 개선: STARTED 이상에서만 수집

    LaunchedEffect(Unit) {                                         // 이벤트 수집도 생명주기에 맞춤
        viewModel.events.flowWithLifecycle(lifecycle = LocalLifecycleOwner.current.lifecycle)
            .collect { event -> /* 스낵바·이동 처리 */ }
    }
    ProductScreen(state)
}
```

| **API**                              | **수집 범위**                       | **적합한 환경**                                  |
| ------------------------------------ | ----------------------------------- | ------------------------------------------------ |
| **collectAsState()**                 | 컴포지션에 있는 동안 항상           | 멀티플랫폼 공통 코드 등 안드로이드 생명주기가 없는 경우 |
| **collectAsStateWithLifecycle()**    | **STARTED 이상**(기본값, 변경 가능) | 안드로이드 앱의 기본 선택                        |
| **repeatOnLifecycle + collect**      | 지정한 상태 이상                    | View 시스템(Activity·Fragment)                   |

<br>

### 5. 일회성 이벤트 처리의 함정

- 스낵바 표시, 화면 이동, 토스트 같은 **한 번만 처리해야 하는 이벤트**를 어떻게 전달할지는 오래된 논쟁거리임
- **SharedFlow**: 백그라운드에서 발생한 이벤트는 유실될 수 있음 (구독자 없음). `replay`를 늘리면 재구독 시 **중복 처리**됨
- **Channel**: 구독자가 없어도 버퍼에 보관해 유실은 줄지만, 여러 구독자에게 방송할 수 없고 `receiveAsFlow()` 사용 시 취소 시점에 값이 사라질 수 있음
- **현재 권장 방식**: 이벤트를 **UI 상태의 일부로 모델링**하고(`errorMessage: String?`), UI가 처리한 뒤 ViewModel에 "소비했다"고 알려 상태를 비움. 유실도 중복도 없고 프로세스 종료에도 대응 가능

```kotlin
// 이벤트를 상태로 모델링: 상태에 남아 있으므로 유실되지 않고, 소비 후 비우므로 중복되지 않음
data class ProductUiState(val products: List<Product> = emptyList(), val userMessage: String? = null)

fun onMessageShown() { _uiState.update { it.copy(userMessage = null) } }
```

> 💡 면접에서 "SharedFlow로 이벤트를 보내면 어떤 문제가 생기나요?"라고 물으면 **구독자 부재 시 유실**과 **replay 시 중복**을 말하고, 대안으로 **상태 기반 모델링**을 제시하면 된다. 어떤 방식이든 트레이드오프를 설명할 수 있어야 한다.

<br>

### 6. 흔한 함정 정리

- **`lifecycleScope.launch { collect }`** 단독 사용 → 백그라운드 수집. `repeatOnLifecycle`로 감쌈
- **Fragment에서 `lifecycleScope` 사용** → View 파괴 후 접근 크래시. `viewLifecycleOwner.lifecycleScope`
- **`stateIn(Eagerly)` 남용** → 화면이 없어도 상위 스트림이 영원히 동작. `WhileSubscribed(5_000)` 사용
- **`repeatOnLifecycle` 블록 안에서 여러 Flow를 순차 `collect`** → 첫 `collect`가 끝나지 않아 두 번째는 시작되지 않음. 각각 `launch`로 감쌈
- **`repeatOnLifecycle`을 `onCreate` 외의 반복 콜백(`onStart` 등)에서 호출** → 호출마다 새 코루틴이 생겨 중복 수집. 한 번만 호출되는 지점(`onCreate`·`onViewCreated`)에서 시작함

<br>

### 7. 정리

- Flow는 생명주기를 모르므로 **수집하는 쪽이 STARTED 이상에서만 수집**하도록 범위를 좁혀야 함
- **StateFlow**는 상태(항상 현재 값, 중복 제거), **SharedFlow**는 이벤트 방송(구독자 없으면 유실 가능)에 사용함
- ViewModel에서는 `stateIn(viewModelScope, WhileSubscribed(5_000), 초기값)`으로 회전은 견디고 백그라운드에서는 멈추는 상태를 만듦
- View 시스템은 **`repeatOnLifecycle(STARTED)`**(Fragment는 `viewLifecycleOwner`), Compose는 **`collectAsStateWithLifecycle()`**이 표준임
- `launchWhenStarted`는 취소가 아닌 일시 중단이라 리소스가 새므로 사용하지 않음
- 일회성 이벤트는 SharedFlow·Channel의 유실·중복 트레이드오프를 이해하고, 가능하면 **UI 상태로 모델링해 소비 후 비우는** 방식을 택함
- ViewModel 자체의 수명은 **unit01**, Compose 부수효과 API는 **unit05**를 참고할 것
