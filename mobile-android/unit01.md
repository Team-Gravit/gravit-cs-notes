## ViewModel과 상태 보존

**ViewModel**은 화면(Activity·Fragment)보다 오래 살아남아 UI 상태를 보관하는 **생명주기 인식(Lifecycle-aware)** 컴포넌트다. 화면 회전 같은 구성 변경(Configuration Change)에서 상태를 지키고, 프로세스 종료까지 견뎌야 하는 값은 **SavedStateHandle**로 보완하는 것이 Jetpack 아키텍처의 출발점이다.

<br>

### 1. ViewModel이 필요한 이유

- Activity는 화면 회전·언어 변경·다크 모드 전환 같은 **구성 변경**이 일어나면 **파괴 후 재생성**됨 (Activity 생명주기 자체는 common-aos 챕터 참고)
- Activity의 멤버 변수에 담아 둔 목록·입력값·네트워크 결과는 재생성과 함께 **모두 사라짐**
- 매번 다시 요청하면 네트워크 낭비와 깜빡임이 생기고, 진행 중이던 비동기 작업은 이미 죽은 Activity를 참조해 **메모리 누수**나 크래시로 이어짐
- ViewModel은 UI 컨트롤러와 **데이터 보관·비즈니스 로직을 분리**해 이 문제를 구조적으로 해결함

```kotlin
// 안티패턴: Activity 필드에 상태 보관 → 회전하면 목록이 사라지고 다시 로드함
class UserActivity : AppCompatActivity() {
    private var users: List<User> = emptyList()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch { users = repository.fetchUsers() } // 회전마다 재호출
    }
}
```

> 💡 "회전하면 왜 데이터가 사라지나요?"는 안드로이드 면접의 단골 질문이다. 답은 **Activity 인스턴스가 새로 만들어지기 때문**이며, 그래서 인스턴스와 무관한 보관소(ViewModel)가 필요하다는 흐름으로 설명하면 된다.

<br>

### 2. 생명주기 인식 — ViewModel의 수명

### 2-1. ViewModelStore와 소유자

- ViewModel은 `ViewModelStore`라는 맵에 보관되고, 이 저장소를 가진 객체를 **ViewModelStoreOwner**라고 부름 (Activity, Fragment, NavBackStackEntry 등)
- 구성 변경 시 시스템은 Activity를 새로 만들지만 `ViewModelStore`는 **새 인스턴스에 그대로 넘겨줌** → 같은 키로 요청하면 기존 ViewModel을 돌려받음
- 소유자가 **완전히 종료**(사용자가 뒤로 가기, `finish()`)될 때만 `onCleared()`가 호출되고 ViewModel이 폐기됨

```
Activity 생명주기   onCreate ─ onStart ─ onResume ─ [회전] ─ onDestroy │ onCreate ─ ... ─ finish() ─ onDestroy
                    ├──────────── 인스턴스 #1 ─────────────────────┤ ├──── 인스턴스 #2 ───────────────────┤
ViewModel 수명      ├─────────────────────────── 하나의 ViewModel 인스턴스 ─────────────────────── onCleared()
```

<br>

### 2-2. 생성과 정리

```kotlin
class UserViewModel(private val repository: UserRepository) : ViewModel() {

    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()

    init {
        // viewModelScope: onCleared() 시점에 자동 취소되는 코루틴 스코프
        viewModelScope.launch { _users.value = repository.fetchUsers() }
    }

    override fun onCleared() {
        // 리소스 해제 (리스너 등록 해제 등). viewModelScope는 자동 취소됨
    }
}

// Activity: 같은 소유자에서는 몇 번을 호출해도 같은 인스턴스를 돌려받음
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels { UserViewModelFactory(repo) }
}
```

- 생성자 인자가 있으면 `ViewModelProvider.Factory`가 필요하며, 실무에서는 Hilt의 `@HiltViewModel`로 이를 대체함 (unit07 참고)
- `viewModelScope`에서 시작한 코루틴은 ViewModel이 폐기될 때 함께 취소되므로 **죽은 화면을 갱신하려는 작업**이 남지 않음

> ⚠️ ViewModel은 Activity보다 오래 살기 때문에 **Activity·View·Context를 필드로 잡으면 안 된다**. 회전 뒤에도 옛 Activity를 붙들어 메모리 누수가 생긴다. 앱 전역 Context가 꼭 필요하면 `AndroidViewModel`이나 Hilt의 `@ApplicationContext`를 사용한다.

<br>

### 3. 화면 회전 대응의 실제 흐름

```
사용자 회전
   │
   ▼
① Activity #1 onPause → onStop → onDestroy      (isChangingConfigurations == true)
② ViewModelStore를 NonConfigurationInstance로 보관
③ Activity #2 onCreate → by viewModels()로 같은 ViewModel 획득
④ StateFlow의 현재 값을 즉시 수집 → 네트워크 재요청 없이 UI 복원
```

- Jetpack Compose에서는 `viewModel()` 또는 `hiltViewModel()`이 가장 가까운 `ViewModelStoreOwner`를 찾아 같은 규칙으로 인스턴스를 재사용함
- Fragment 단위로 ViewModel을 두면 Fragment가 백스택에서 제거될 때 폐기되고, `activityViewModels()`를 쓰면 Activity 수명에 맞춰 여러 Fragment가 **하나의 ViewModel을 공유**함

<br>

### 4. SavedStateHandle — 프로세스 종료를 견디는 상태

### 4-1. ViewModel만으로 부족한 경우

- 앱이 백그라운드에 있을 때 시스템이 메모리 확보를 위해 **프로세스를 종료**하면 ViewModel도 메모리와 함께 사라짐
- 사용자가 다시 돌아오면 시스템은 Activity를 복원하려 하지만, ViewModel은 **초기 상태로 새로 생성**됨
- 이때 필요한 것이 Activity의 `onSaveInstanceState()` 메커니즘이며, ViewModel에서 이를 쓸 수 있게 감싼 것이 **SavedStateHandle**임

<br>

### 4-2. 사용 방법

```kotlin
class SearchViewModel(
    private val savedStateHandle: SavedStateHandle,   // Factory 없이도 기본 제공됨
    private val repository: SearchRepository,
) : ViewModel() {

    // 키 기반으로 저장·복원되며, StateFlow로 노출해 UI가 바로 구독할 수 있음
    val query: StateFlow<String> = savedStateHandle.getStateFlow(KEY_QUERY, "")

    fun onQueryChange(newQuery: String) {
        savedStateHandle[KEY_QUERY] = newQuery   // 저장 즉시 Bundle 후보에 반영됨
    }

    companion object { private const val KEY_QUERY = "query" }
}
```

- `SavedStateHandle`은 `Bundle`에 담을 수 있는 타입(기본형·String·Parcelable·Serializable 등)만 저장 가능
- Navigation의 인자도 같은 핸들에 들어오므로 `savedStateHandle.get<Long>("userId")`처럼 **화면 인자를 읽는 용도**로도 자주 쓰임 (unit08 참고)

> 💡 개발자 옵션의 **"액티비티 유지 안 함"**을 켜면 프로세스 종료 후 복원 시나리오를 손쉽게 재현할 수 있다. 회전 테스트만으로는 SavedStateHandle 누락을 잡아내지 못한다.

<br>

### 4-3. 구성 변경 vs 프로세스 종료

| **상황**                    | **Activity** | **ViewModel**            | **SavedStateHandle**        | **비고**                          |
| --------------------------- | ------------ | ------------------------ | --------------------------- | --------------------------------- |
| **화면 회전(구성 변경)**    | 재생성       | **유지**                 | 유지                        | ViewModel만으로 충분              |
| **뒤로 가기·finish()**      | 종료         | **onCleared() 후 폐기**  | 폐기                        | 상태를 지킬 필요 없음             |
| **백그라운드 프로세스 종료** | 복원 재생성 | 새로 생성(초기화됨)      | **Bundle에서 복원**         | 소량의 핵심 상태만 저장           |
| **사용자가 앱을 강제 종료** | 종료         | 폐기                     | 폐기                        | 필요하면 영속 저장소(DB·DataStore) |

<br>

### 5. 흔한 함정

- **큰 데이터를 SavedStateHandle에 저장**: Bundle은 프로세스 간 전송 한도(수백 KB 수준, 버전에 따라 다를 수 있음)를 넘으면 `TransactionTooLargeException`이 발생함. 목록 전체가 아니라 **ID·검색어·스크롤 위치** 같은 복원 키만 저장하고 데이터는 다시 불러오는 것이 원칙임
- **ViewModel을 상태 저장소가 아닌 Context 보관소로 사용**: 위 2-2의 경고와 같이 누수의 원인이 됨
- **`init`에서 무조건 로딩**: 복원된 상태가 있는데도 재요청하면 SavedStateHandle의 의미가 없음. 저장된 값이 있으면 그것으로 먼저 UI를 그리고 필요할 때만 갱신함
- **Fragment 스코프와 Activity 스코프 혼동**: 두 Fragment가 데이터를 공유해야 하는데 각자 `viewModels()`를 쓰면 서로 다른 인스턴스를 받음

❗️**ViewModel은 UI 상태의 보관소이지 영속 저장소가 아니다**: 앱을 껐다 켜도 남아야 하는 데이터는 Room·DataStore 같은 영속 계층에 두고, ViewModel은 그 데이터를 UI에 맞게 가공해 노출하는 역할에 집중한다.

<br>

### 6. 상태를 어디에 둘 것인가 — 선택 기준

| **저장 위치**              | **생존 범위**                    | **적합한 데이터**                              | **예시**                        |
| -------------------------- | -------------------------------- | ---------------------------------------------- | ------------------------------- |
| **Composable·View 로컬**   | 재구성·뷰 재생성 전까지          | 순수 UI 일시 상태                              | 드롭다운 펼침 여부              |
| **ViewModel**              | 구성 변경 이후까지               | 화면 데이터·로딩 상태·비동기 작업 결과         | 사용자 목록, 로딩 플래그        |
| **SavedStateHandle**       | 프로세스 종료 이후까지           | 복원에 필요한 **소량의 키 값**                 | 검색어, 선택한 탭, 화면 인자    |
| **영속 저장소(Room 등)**   | 앱 재설치 전까지                 | 사용자가 만든 데이터·캐시                      | 작성 중인 글 초안, 오프라인 캐시 |

<br>

### 7. 정리

- 구성 변경 시 Activity는 재생성되지만 **ViewModelStore는 새 인스턴스로 넘겨져** ViewModel이 유지됨
- ViewModel은 소유자가 완전히 종료될 때 `onCleared()`로 정리되며, `viewModelScope`의 코루틴도 함께 취소됨
- **프로세스 종료**는 ViewModel로 막을 수 없으므로 복원에 필요한 소량의 키 값은 **SavedStateHandle**에 저장함
- ViewModel에 Activity·View·Context 참조를 두면 누수가 발생하고, Bundle에 큰 객체를 넣으면 `TransactionTooLargeException`이 발생함
- 데이터의 생존 범위에 따라 **Composable 로컬 → ViewModel → SavedStateHandle → 영속 저장소** 순으로 저장 위치를 고름
- ViewModel에 상태를 두고 UI가 안전하게 구독하는 방법은 **unit06(생명주기 인식 데이터 수집)**, ViewModel 생성과 주입은 **unit07(의존성 주입)**을 참고할 것
