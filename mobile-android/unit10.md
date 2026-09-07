## 테스트 가능한 구조

안드로이드 앱이 테스트하기 어려운 이유는 로직이 Activity·Context·네트워크·시간에 **직접 묶여 있기** 때문이다. ViewModel을 순수 Kotlin 단위 테스트로 검증하고, **의존성을 가짜(Fake)로 대체**하며, 느리고 불안정한 UI 테스트는 **꼭 필요한 범위로 제한**하는 것이 Jetpack 아키텍처가 지향하는 테스트 전략이다.

<br>

### 1. 안드로이드 테스트의 종류와 비용

| **종류**                     | **실행 환경**                     | **속도**   | **검증 대상**                                 | **권장 비중** |
| ---------------------------- | --------------------------------- | ---------- | --------------------------------------------- | ------------- |
| **로컬 단위 테스트**         | JVM (`src/test`)                  | **매우 빠름** | ViewModel, UseCase, Repository, 순수 로직  | 대부분        |
| **로컬 UI 테스트**           | JVM + Robolectric                 | 빠름       | Compose 컴포저블 렌더링·상호작용              | 일부          |
| **계측(Instrumented) 테스트** | 에뮬레이터·실기기 (`src/androidTest`) | 느림   | 실제 기기 동작, 통합, 화면 흐름               | 핵심 경로만   |

```
          ▲ 느리고 비쌈·불안정
         ╱ ╲        E2E · 화면 흐름 테스트 (소수)
        ╱   ╲
       ╱     ╲      통합 · Compose UI 테스트 (일부)
      ╱       ╲
     ╱         ╲    ViewModel · Repository 단위 테스트 (대부분)
    ╱───────────╲
     빠르고 저렴·안정적
```

> 💡 테스트 피라미드의 핵심은 **"논리는 아래에서, 화면은 위에서 최소한으로"**다. ViewModel에 로직이 모여 있으면 대부분의 시나리오를 JVM 단위 테스트로 수 초 안에 검증할 수 있다.

<br>

### 2. 테스트 가능한 구조의 조건

- **의존성 역전**: ViewModel은 `UserRepository` 인터페이스에 의존하고, 구현체(네트워크·DB)는 DI로 주입됨 → 테스트에서는 가짜 구현을 넣음 (unit07 참고)
- **안드로이드 프레임워크와 분리**: ViewModel·UseCase에 `Context`·`Activity`·`View`가 없어야 JVM에서 실행 가능함. 문자열 리소스는 ID로 넘기고 UI에서 해석함
- **디스패처 주입**: `Dispatchers.IO`를 하드코딩하면 테스트에서 제어할 수 없음. 생성자로 `CoroutineDispatcher`를 받음
- **시간·난수·시계 주입**: `System.currentTimeMillis()` 대신 `Clock`을 주입해 결정적(Deterministic) 테스트를 만듦
- **단방향 데이터 흐름**: 입력(이벤트 함수 호출) → 출력(`StateFlow` 값)이 명확하면 테스트가 "호출하고 상태를 확인"으로 단순해짐 (unit03·unit06 참고)

```kotlin
// 안티패턴: 프레임워크·구현체·디스패처에 직접 결합 → JVM에서 실행 불가, 대체 불가
class OrderViewModel(private val context: Context) : ViewModel() {
    private val api = RetrofitClient.orderApi                       // 싱글턴 구현체 직접 참조
    fun load() = viewModelScope.launch(Dispatchers.IO) { /* ... */ }
}

// 개선: 인터페이스·디스패처 주입, Context 제거
@HiltViewModel
class OrderViewModel @Inject constructor(
    private val repository: OrderRepository,                        // 인터페이스
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,    // Qualifier로 구분된 디스패처
) : ViewModel() {
    private val _uiState = MutableStateFlow<OrderUiState>(OrderUiState.Loading)
    val uiState: StateFlow<OrderUiState> = _uiState.asStateFlow()

    fun load(orderId: Long) = viewModelScope.launch(ioDispatcher) {
        _uiState.value = runCatching { repository.getOrder(orderId) }
            .fold({ OrderUiState.Success(it) }, { OrderUiState.Error(it.message ?: "오류") })
    }
}
```

<br>

### 3. ViewModel 단위 테스트

### 3-1. Dispatchers.Main 교체

- `viewModelScope`는 `Dispatchers.Main`을 사용하는데, JVM 테스트에는 안드로이드 메인 루퍼가 없어 **그대로 실행하면 예외**가 발생함
- `kotlinx-coroutines-test`의 `Dispatchers.setMain(테스트 디스패처)`로 교체하고 테스트 후 `resetMain()`함. 이를 JUnit Rule로 만들어 재사용하는 것이 관례임

```kotlin
class MainDispatcherRule(
    val dispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : TestWatcher() {
    override fun starting(description: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(description: Description) = Dispatchers.resetMain()
}
```

<br>

### 3-2. 테스트 작성

```kotlin
class OrderViewModelTest {
    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    private val fakeRepository = FakeOrderRepository()          // 가짜 구현 (4번 참고)
    private lateinit var viewModel: OrderViewModel

    @Before
    fun setUp() {
        viewModel = OrderViewModel(fakeRepository, mainDispatcherRule.dispatcher)
    }

    @Test
    fun `주문 조회 성공 시 Success 상태가 된다`() = runTest {
        fakeRepository.orders[1L] = Order(id = 1L, amount = 12_000)

        viewModel.load(orderId = 1L)

        assertEquals(OrderUiState.Success(Order(1L, 12_000)), viewModel.uiState.value)
    }

    @Test
    fun `저장소가 예외를 던지면 Error 상태가 된다`() = runTest {
        fakeRepository.shouldFail = true

        viewModel.load(orderId = 1L)

        assertTrue(viewModel.uiState.value is OrderUiState.Error)
    }
}
```

- `runTest`는 가상 시간을 사용해 `delay`를 실제로 기다리지 않고 건너뜀. 디바운스·타임아웃 로직도 수 ms 안에 검증 가능함
- `stateIn(WhileSubscribed)`으로 만든 StateFlow는 **구독자가 있어야 상위 수집이 시작**되므로, 테스트에서 `backgroundScope.launch { viewModel.uiState.collect() }`로 구독을 먼저 걸어야 값이 갱신됨. Flow의 여러 방출을 순서대로 검증할 때는 **Turbine** 라이브러리(`flow.test { awaitItem() }`)가 편리함

> ⚠️ `UnconfinedTestDispatcher`는 코루틴을 즉시 실행해 테스트가 간단해지지만, 실제 앱의 실행 순서와 다를 수 있다. 동시성 순서 자체를 검증하려면 `StandardTestDispatcher` + `advanceUntilIdle()`로 스케줄링을 명시적으로 제어한다.

<br>

### 4. 의존성 대체 — Fake와 Mock

### 4-1. 선택 기준

| **항목**            | **Fake (가짜 구현)**                                | **Mock (모의 객체, MockK·Mockito)**                     |
| ------------------- | --------------------------------------------------- | ------------------------------------------------------- |
| **형태**            | 인터페이스를 **실제로 동작하는 단순 구현**으로 작성 | 라이브러리로 호출별 반환값·행동을 지정                  |
| **검증 방식**       | 결과 상태 검증 (상태 기반)                          | 호출 여부·인자 검증 (행위 기반)                         |
| **재사용성**        | 여러 테스트에서 공유, 리팩터링에 강함               | 테스트마다 설정, 구현 세부에 결합되기 쉬움              |
| **적합한 대상**     | Repository·데이터 소스 등 **핵심 협력 객체**        | 분석 SDK, 알림 등 **부수효과 확인**이 목적인 경우       |

```kotlin
// Fake: 인메모리 맵으로 동작하는 가짜 저장소. 실패 시나리오도 플래그로 제어
class FakeOrderRepository : OrderRepository {
    val orders = mutableMapOf<Long, Order>()
    var shouldFail = false

    override suspend fun getOrder(id: Long): Order {
        if (shouldFail) throw IOException("네트워크 오류")
        return orders[id] ?: throw NoSuchElementException("주문 없음: $id")
    }
}
```

- 공식 가이드는 **Fake를 우선**하고 Mock은 보조적으로 쓰기를 권장함. Mock으로 `verify(repository).getOrder(1L)`처럼 내부 호출을 검증하면 리팩터링 때마다 테스트가 깨짐

<br>

### 4-2. Hilt 테스트에서 모듈 교체

- 계측 테스트에서 실제 Hilt 그래프를 쓰되 특정 모듈만 가짜로 바꾸려면 **`@TestInstallIn(components, replaces)`**를 사용함
- 테스트 클래스에는 `@HiltAndroidTest`와 `HiltAndroidRule`이 필요하고, 특정 테스트에서만 모듈을 빼려면 `@UninstallModules`를 씀

```kotlin
@Module
@TestInstallIn(components = [SingletonComponent::class], replaces = [RepositoryModule::class])
abstract class FakeRepositoryModule {
    @Binds @Singleton
    abstract fun bindOrderRepository(impl: FakeOrderRepository): OrderRepository
}
```

> 💡 `@TestInstallIn`은 모듈 단위 교체라 **모든 계측 테스트에 일괄 적용**된다. 특정 테스트 클래스에서만 다른 구현이 필요하면 `@UninstallModules(RepositoryModule::class)`와 테스트 클래스 내부에 정의한 `@Module`을 조합한다. 다만 클래스마다 컴포넌트가 새로 생성되어 빌드 시간이 늘어나므로 남용하지 않는다.

<br>

### 5. UI 테스트의 범위

### 5-1. Compose UI 테스트

- `createComposeRule()`로 **컴포저블만 단독 렌더링**해 검증함. Stateless 컴포저블은 상태를 직접 주입할 수 있어 ViewModel·내비게이션 없이 테스트 가능 (unit03의 호이스팅이 여기서 빛남)
- 노드는 텍스트·`contentDescription`·`testTag`로 찾고, `performClick()` 등으로 상호작용한 뒤 `assertIsDisplayed()` 등으로 검증함
- Robolectric을 붙이면 JVM에서도 실행되어 계측 테스트보다 훨씬 빠름 (버전·설정에 따라 지원 범위가 다를 수 있음)

```kotlin
class OrderScreenTest {
    @get:Rule val composeRule = createComposeRule()

    @Test
    fun 에러_상태면_재시도_버튼이_보이고_클릭_시_콜백이_호출된다() {
        var retried = false
        composeRule.setContent {
            OrderScreen(state = OrderUiState.Error("오류"), onRetry = { retried = true })
        }

        composeRule.onNodeWithText("다시 시도").assertIsDisplayed().performClick()

        assertTrue(retried)
    }
}
```

<br>

### 5-2. 어디까지 UI 테스트로 검증할 것인가

| **검증 내용**                                  | **적합한 테스트**                     | **이유**                                         |
| ---------------------------------------------- | ------------------------------------- | ------------------------------------------------ |
| **상태 → 화면 표시 매핑** (에러면 재시도 버튼) | Compose UI 테스트(Stateless 컴포저블) | 상태 주입만으로 빠르게 검증                      |
| **사용자 입력 → 이벤트 콜백**                  | Compose UI 테스트                     | 상호작용 API로 직접 확인                         |
| **로딩·에러·성공 로직, 데이터 가공**           | ViewModel 단위 테스트                 | UI 없이 검증 가능, 훨씬 빠름                     |
| **로그인 → 홈 → 결제 전체 흐름**               | 계측 E2E 테스트 (소수)                | 실제 통합 확인이 필요하지만 느리고 불안정        |
| **픽셀 단위 디자인 일치**                      | 스크린샷 테스트                       | 어설션으로 표현하기 어려운 시각적 회귀 감지      |

❗️**UI 테스트로 비즈니스 로직을 검증하지 않는다**: "할인 계산이 맞는지"를 화면의 텍스트로 확인하면 느리고 깨지기 쉽다. 계산은 ViewModel·UseCase 테스트에서, 화면은 "주어진 상태를 올바르게 보여 주는가"만 검증한다.

<br>

### 6. 흔한 함정

- **`Thread.sleep()`으로 비동기 대기**: 느리고 불안정함(Flaky). `runTest`의 가상 시간이나 Compose의 `waitUntil`을 사용함
- **테스트 간 공유 상태**: 싱글턴 Fake를 여러 테스트가 공유하면 순서에 따라 결과가 달라짐. `@Before`에서 매번 새로 만듦
- **private 함수 테스트 시도**: 리플렉션으로 억지로 테스트하기보다 공개 입력(이벤트)과 출력(상태)으로 검증하도록 설계를 바꿈
- **Mock 남용으로 구현에 결합**: 호출 순서·횟수까지 검증하면 리팩터링마다 테스트 수정. 결과 상태 중심으로 검증함
- **테스트에서 `Dispatchers.Main` 교체 누락**: `viewModelScope` 사용 시 "Module with the Main dispatcher had failed to initialize" 예외 발생

<br>

### 7. 정리

- 테스트 가능한 구조의 조건은 **인터페이스 의존(DI)·프레임워크 분리·디스패처와 시계 주입·단방향 데이터 흐름**임
- ViewModel 단위 테스트는 `Dispatchers.setMain` Rule + `runTest`로 JVM에서 실행하며, 입력 함수 호출 후 **StateFlow 값을 검증**함
- 의존성 대체는 **Fake 우선**, Mock은 부수효과 확인용으로 제한함. Hilt 계측 테스트에서는 `@TestInstallIn`으로 모듈을 교체함
- Compose UI 테스트는 **Stateless 컴포저블에 상태를 주입**해 표시·상호작용만 검증하고, 로직은 단위 테스트에 맡김
- E2E·스크린샷 테스트는 핵심 흐름과 시각 회귀에 한정해 소수만 유지함
- `Thread.sleep`, 공유 상태, Mock 남용은 테스트를 느리고 불안정하게 만드는 대표 원인임
- 관련 설계 원칙은 **unit03(상태 호이스팅)**, **unit06(StateFlow)**, **unit07(Hilt)**를 참고할 것
