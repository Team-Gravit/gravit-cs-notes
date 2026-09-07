## 앱 아키텍처

안드로이드 앱은 생명주기에 따라 화면이 수시로 파괴·재생성되고 (unit02·unit03 참고), 데이터는 네트워크·DB·메모리 여러 곳에 흩어져 있다. 이 복잡성을 다루기 위해 구글은 **UI·Domain·Data 세 레이어로 책임을 나누고**, 상태가 **단방향(Unidirectional Data Flow, UDF)**으로만 흐르게 하는 아키텍처를 권장한다. 이 유닛은 각 레이어의 역할과 경계, 단방향 흐름이 해결하는 문제, 그리고 흔히 저지르는 경계 위반을 정리한다.

<br>

### 1. 왜 레이어를 나누는가

- **관심사 분리**: 액티비티·프래그먼트는 시스템이 언제든 파괴하는 "**접착 코드**"일 뿐이므로, 비즈니스 로직과 데이터 접근을 여기 두면 재생성 때마다 유실되고 테스트도 불가능하다
- **테스트 가능성**: 안드로이드 프레임워크에 의존하지 않는 순수 코틀린 계층이 있어야 JVM 단위 테스트를 빠르게 돌릴 수 있다
- **교체 가능성**: 네트워크 라이브러리나 DB를 바꿔도 UI가 영향을 받지 않아야 한다
- **협업**: 레이어 경계가 명확하면 화면 작업과 데이터 작업을 동시에 진행할 수 있다

```
┌───────────────────────────── UI Layer ─────────────────────────────┐
│  UI 요소(Activity·Fragment·View)  ◀── UiState ──  State Holder(ViewModel) │
│                                   ── Event  ──▶                          │
└──────────────────────────────────────┬───────────────────────────────┘
                                       │ 도메인 모델 / UseCase 호출
┌───────────────────────────── Domain Layer (선택) ──────────────────┐
│  UseCase: 여러 Repository를 조합하는 재사용 가능한 비즈니스 규칙        │
└──────────────────────────────────────┬───────────────────────────────┘
                                       │ Repository 인터페이스
┌───────────────────────────── Data Layer ───────────────────────────┐
│  Repository ──▶ DataSource(Remote: Retrofit / Local: Room·DataStore)   │
└─────────────────────────────────────────────────────────────────────┘
        의존 방향은 항상 아래로만: UI → Domain → Data. 역방향 참조 금지
```

> 💡 "MVVM·MVI·클린 아키텍처 중 뭘 쓰나요?"라는 질문에 이름만 답하면 부족하다. 이름은 달라도 핵심은 같다 — **UI는 상태를 그리기만 하고, 상태는 한 곳(State Holder)에서 만들어지며, 데이터 접근은 Repository 뒤에 숨긴다**. 이 원칙을 설명한 뒤 팀 규모에 맞게 Domain 레이어를 넣고 뺀다고 답하는 것이 좋다.

<br>

### 2. 레이어별 책임

| **레이어**       | **구성 요소**                                  | **책임**                                                   | **알아야 하는 것**          | **몰라야 하는 것**                 |
| ---------------- | ---------------------------------------------- | ---------------------------------------------------------- | --------------------------- | ---------------------------------- |
| **UI**           | Activity·Fragment·View, **ViewModel**(상태 홀더) | 상태를 화면에 그리고 사용자 이벤트를 상태 홀더에 전달        | UiState, 이벤트             | HTTP 코드, SQL, DTO                |
| **Domain**       | **UseCase**, 도메인 모델                        | 여러 Repository를 조합하는 비즈니스 규칙, 재사용 로직        | Repository 인터페이스       | 안드로이드 프레임워크, 구현체       |
| **Data**         | **Repository**, DataSource, DTO·Entity          | 단일 진실 공급원(SSOT), 캐시 전략, 오류 변환, 스레드 전환    | Retrofit·Room·DataStore     | UI 상태, 화면 구조                 |

<br>

### 2-1. UI 레이어 — 상태 홀더로서의 ViewModel

- ViewModel은 화면이 필요로 하는 **모든 상태를 하나의 `UiState`로 노출**하고, UI는 이를 관찰해 그리기만 한다
- ViewModel은 구성 변경에 살아남으므로 (unit03 참고) 화면 상태의 자연스러운 보관처다
- ViewModel은 **안드로이드 UI 클래스(View·Activity Context)를 참조하지 않는다** — 누수 (unit05 참고)와 테스트 불가의 원인

<br>

### 2-2. Data 레이어 — Repository와 단일 진실 공급원

- **Repository**는 "어디서 왔든 이 데이터의 정답은 하나"라는 **단일 진실 공급원(Single Source of Truth, SSOT)**을 제공한다. 보통 로컬 DB(Room)를 SSOT로 두고 네트워크는 DB를 갱신하는 수단으로 취급한다
- **DTO(네트워크 응답)·Entity(DB 행)는 Data 레이어 안에서만** 쓰고, 밖으로는 **도메인 모델**로 변환해 내보낸다. 서버 필드명이 바뀌어도 UI 코드는 그대로다
- Repository는 `Dispatchers.IO` 전환과 예외 → 도메인 오류 변환 (unit08 참고)까지 책임진다. 호출자는 메인 스레드에서 안전하게 부를 수 있어야 한다(**main-safe**)

<br>

### 2-3. Domain 레이어 — 언제 넣는가

- 여러 Repository를 조합하거나(사용자 + 주문 + 배송), 여러 ViewModel이 **같은 로직을 재사용**할 때 UseCase로 뽑는다
- 단순히 Repository 메서드를 한 줄 호출만 하는 UseCase는 오버엔지니어링이다. **작은 앱은 Domain 레이어를 생략**해도 권장 아키텍처에 부합한다
- UseCase는 하나의 동작만 하는 **`operator fun invoke()`** 형태로 두고, 안드로이드 의존성을 갖지 않게 유지한다

<br>

### 3. 단방향 데이터 흐름(UDF)

**단방향 데이터 흐름**은 상태가 **위(상태 홀더)에서 아래(UI)로**만 흐르고, 이벤트는 **아래에서 위로**만 흐르는 규칙이다. UI는 상태를 직접 바꾸지 않고 "이런 일이 있었다"는 이벤트를 보낼 뿐이며, 상태를 바꾸는 유일한 곳은 상태 홀더다.

```
       ┌──────────────── 상태(State) ────────────────┐
       │                                             ▼
 ViewModel(State Holder)                           UI
  · 이벤트를 받아                                · 상태를 그림
  · Repository 호출                              · 사용자 입력 감지
  · 새 UiState 생성                              · 이벤트 전달
       ▲                                             │
       └──────────────── 이벤트(Event) ───────────────┘
```

**UDF가 해결하는 문제**

- **상태 불일치**: 여러 곳에서 뷰를 직접 바꾸면 "로딩 스피너가 남아 있는데 데이터도 보이는" 조합이 생긴다. 상태가 한 객체면 불가능한 조합을 타입으로 막을 수 있다
- **디버깅**: 상태 변화의 원인이 항상 "어떤 이벤트가 어떤 상태를 만들었나"로 추적된다
- **구성 변경 대응**: UI가 재생성돼도 최신 UiState를 다시 그리기만 하면 되므로 별도 복원 코드가 줄어든다

<br>

### 3-1. 안티패턴 → 개선

```kotlin
// 안티패턴: UI가 데이터 소스를 직접 호출하고, 결과에 따라 뷰를 조각조각 갱신
class ProfileFragment : Fragment(R.layout.fragment_profile) {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.progress.isVisible = true
        lifecycleScope.launch {
            val user = retrofit.create(UserApi::class.java).getUser(userId)   // UI가 HTTP를 앎
            binding.name.text = user.name                                        // 상태가 뷰에 흩어짐
            binding.progress.isVisible = false                                   // 실패 시 스피너가 남음
        }
    }
}
```

```kotlin
// 개선: UiState 하나 + 이벤트 → ViewModel이 상태를 만들고 UI는 그리기만 한다
data class ProfileUiState(
    val isLoading: Boolean = false,
    val user: User? = null,
    val errorMessage: String? = null
)

class ProfileViewModel(private val getUser: GetUserUseCase) : ViewModel() {
    private val _uiState = MutableStateFlow(ProfileUiState())
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    fun onLoad(userId: Long) {                       // 이벤트: 위로
        _uiState.update { it.copy(isLoading = true, errorMessage = null) }
        viewModelScope.launch {
            getUser(userId)                          // Domain → Data, main-safe
                .onSuccess { user -> _uiState.update { it.copy(isLoading = false, user = user) } }
                .onFailure { e -> _uiState.update { it.copy(isLoading = false, errorMessage = e.toUserMessage()) } }
        }
    }
}

class ProfileFragment : Fragment(R.layout.fragment_profile) {
    private val viewModel: ProfileViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect(::render)  // 상태: 아래로, 한 곳에서 그림
            }
        }
        viewModel.onLoad(args.userId)
    }

    private fun render(state: ProfileUiState) {
        binding.progress.isVisible = state.isLoading
        binding.name.text = state.user?.name.orEmpty()
        binding.error.isVisible = state.errorMessage != null
        binding.error.text = state.errorMessage
    }
}
```

> ⚠️ `StateFlow`는 **최신 상태 하나**를 유지하므로 "토스트 한 번 띄우기"·"화면 이동" 같은 **일회성 이벤트**를 넣으면 회전 후 다시 그려질 때 토스트가 재발생한다. 공식 권장은 일회성 이벤트도 상태로 모델링하고(`errorMessage`를 보여준 뒤 `onErrorShown()`으로 소비), 꼭 필요하면 `Channel`·`SharedFlow`로 분리하되 유실·중복 가능성을 이해하고 쓰는 것이다.

<br>

### 4. 흔한 경계 위반과 수정

| **위반**                                               | **문제**                                             | **수정**                                                      |
| ------------------------------------------------------ | ---------------------------------------------------- | ------------------------------------------------------------- |
| **ViewModel이 Retrofit·Room을 직접 호출**              | 캐시·오류 변환이 화면마다 중복, 테스트 시 실제 통신 | Repository 인터페이스를 주입받아 호출                        |
| **DTO를 UI까지 전달**                                  | 서버 필드 변경이 UI 코드까지 전파                    | Data 레이어에서 도메인 모델로 매핑                            |
| **ViewModel이 Context·View 보관**                      | 메모리 누수, JVM 테스트 불가                         | 문자열 리소스는 ID로 전달, Context 필요 시 Application만       |
| **Fragment가 다른 Fragment의 ViewModel 상태를 직접 수정** | 상태 변경 지점이 분산                             | 공유 ViewModel(`activityViewModels`)에 이벤트로 전달          |
| **Repository가 UI 상태(로딩 여부)를 반환**             | Data 레이어가 화면을 알게 됨                         | Repository는 데이터·오류만, 로딩 표시는 ViewModel이 결정      |

- 레이어 간 의존은 **인터페이스**로 두고 구현체는 **의존성 주입(Hilt 등)**으로 조립하면 테스트에서 가짜(Fake) 구현으로 교체하기 쉽다
- 모듈 분리를 한다면 `:data`가 `:domain`에 의존하고, `:domain`은 아무 안드로이드 모듈에도 의존하지 않는 방향이 정석이다

> 💡 아키텍처 규칙은 "코드 줄 수를 늘리는 의식"이 아니라 **변경 비용을 줄이는 투자**다. 화면 3개짜리 앱에 UseCase·모듈 분리를 다 넣으면 오히려 비용이 크다. 다만 **UiState 단일 객체 + Repository 경계** 두 가지는 앱 크기와 무관하게 처음부터 지키는 편이 좋다.

<br>

### 5. 정리 — 면접·실무 체크포인트

| **질문**                                                | **핵심 답변**                                                                                 |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **권장 아키텍처의 세 레이어와 역할은?**                 | UI(상태 그리기·이벤트), Domain(재사용 비즈니스 규칙, 선택), Data(**SSOT**·캐시·오류 변환)     |
| **단방향 데이터 흐름이란?**                             | 상태는 **위→아래**, 이벤트는 **아래→위**로만 흐르고, 상태 변경 지점은 상태 홀더 하나          |
| **Repository의 핵심 역할은?**                           | 여러 소스를 감춘 **단일 진실 공급원**, DTO→도메인 변환, main-safe 보장                        |
| **Domain 레이어는 언제 필요한가?**                      | 여러 Repository 조합·로직 재사용이 있을 때. 단순 위임만 하면 생략                             |
| **ViewModel이 Context를 가지면 안 되는 이유는?**        | 구성 변경을 넘어 살아 **누수**되고, 프레임워크 의존으로 JVM 테스트 불가                       |
| **일회성 이벤트를 StateFlow에 넣으면?**                 | 재구독·재생성 시 **중복 발생** → 상태로 모델링하고 소비 이벤트로 지운다                       |

- 레이어 경계의 본질은 "**누가 무엇을 알아도 되는가**"다. UI는 HTTP·SQL을 모르고, Data는 화면을 모른다
- 상태 보존은 **unit03**, 누수는 **unit05**, Data 레이어의 저장소·네트워크 구현은 **unit07·unit08**을 참고할 것
