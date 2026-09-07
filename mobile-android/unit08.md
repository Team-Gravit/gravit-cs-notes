## Navigation과 화면 전환

**Navigation 컴포넌트**는 앱 내 화면 이동을 **그래프(Graph)와 백스택(Back Stack)**으로 모델링해, 뒤로 가기·인자 전달·딥링크(Deep Link)를 일관된 규칙으로 다루게 해 준다. 백스택을 어떻게 다루느냐가 "로그인 후 뒤로 가면 다시 로그인 화면이 나오는" 류의 버그를 좌우하므로, `popUpTo`·`launchSingleTop`·`saveState`의 동작을 정확히 알아야 한다.

<br>

### 1. 구성 요소

| **구성 요소**        | **역할**                                                              | **Compose에서의 형태**                       |
| -------------------- | --------------------------------------------------------------------- | -------------------------------------------- |
| **NavGraph**         | 앱의 모든 목적지(Destination)와 이동 경로의 집합                      | `NavHost { composable<Route> { } }`          |
| **NavHost**          | 현재 목적지를 화면에 표시하는 컨테이너                                | `NavHost(navController, startDestination)`   |
| **NavController**    | 이동 명령을 받고 **백스택을 관리**하는 중심 객체                      | `rememberNavController()`                    |
| **NavBackStackEntry** | 백스택의 한 항목. 인자·`SavedStateHandle`·**ViewModelStoreOwner**를 가짐 | `composable` 람다의 매개변수                 |

```
NavController 백스택 (아래가 바닥)
   ┌──────────────────────┐   ← 현재 화면 (화면에 표시됨)
   │ ProductDetail(id=42) │
   ├──────────────────────┤
   │ ProductList          │
   ├──────────────────────┤
   │ Home (startDestination) │
   └──────────────────────┘
   navigate() → 위에 쌓임 / 뒤로 가기·popBackStack() → 맨 위 제거
```

- 각 `NavBackStackEntry`는 자체 `ViewModelStore`를 가지므로, `hiltViewModel()`로 얻은 ViewModel은 **해당 화면이 백스택에서 제거될 때** 폐기됨 (unit01·unit07 참고)

<br>

### 2. 타입 안전 경로와 인자 전달

### 2-1. 경로 정의

Navigation **2.8.0** 이후로 문자열 경로(`"product/{id}"`) 대신 **`@Serializable` 클래스**로 목적지를 정의하는 타입 안전 방식이 권장된다 (이전 버전은 문자열 경로만 지원하므로 프로젝트 버전에 따라 다를 수 있음).

```kotlin
@Serializable object Home
@Serializable object ProductList
@Serializable data class ProductDetail(val id: Long, val fromSearch: Boolean = false)

@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(navController = navController, startDestination = Home) {
        composable<Home> { HomeRoute(onOpenList = { navController.navigate(ProductList) }) }
        composable<ProductList> {
            ProductListRoute(onClick = { id -> navController.navigate(ProductDetail(id)) })
        }
        composable<ProductDetail> { backStackEntry ->
            val route = backStackEntry.toRoute<ProductDetail>()     // 타입 안전하게 인자 복원
            ProductDetailRoute(productId = route.id)
        }
    }
}
```

<br>

### 2-2. ViewModel에서 인자 읽기

- 인자는 `SavedStateHandle`에도 들어오므로 ViewModel이 직접 읽는 것이 가장 깔끔함. UI가 인자를 받아 다시 넘길 필요가 없음

```kotlin
@HiltViewModel
class ProductDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    repository: ProductRepository,
) : ViewModel() {
    private val productId = savedStateHandle.toRoute<ProductDetail>().id   // 인자 복원
    val product = repository.observeProduct(productId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), null)
}
```

> ⚠️ 인자로는 **ID 같은 작은 값만** 전달한다. 객체 전체(Parcelable 목록 등)를 넘기면 저장 상태(Bundle) 크기 제한에 걸리고, 원본이 갱신돼도 목적지는 옛 복사본을 보게 된다. 목적지는 ID로 저장소에서 **다시 조회**하는 것이 원칙이다.

<br>

### 3. 백스택 관리

### 3-1. popUpTo와 inclusive

- `navigate` 시 `popUpTo<대상>`을 지정하면 **대상 목적지까지 백스택을 걷어낸 뒤** 새 목적지를 쌓음
- `inclusive = true`면 대상 자신까지 제거함

```kotlin
// 로그인 성공: 로그인 화면을 스택에서 제거해 뒤로 가기로 돌아갈 수 없게 함
navController.navigate(Home) {
    popUpTo<Login> { inclusive = true }
}
```

```
이동 전               navigate(Home) { popUpTo<Login>{inclusive=true} }   이동 후
┌─────────┐                                                              ┌─────────┐
│ Login   │  ← 제거                                                      │ Home    │
├─────────┤                                                              ├─────────┤
│ Splash  │  ← popUpTo 대상 아님, 그대로 유지                           │ Splash  │
└─────────┘                                                              └─────────┘
```

<br>

### 3-2. launchSingleTop

- 현재 맨 위 목적지와 **같은 목적지**로 이동할 때 새 항목을 쌓지 않고 기존 항목을 재사용함
- 버튼 연타로 같은 화면이 여러 장 쌓이는 문제, 하단 탭을 반복 터치할 때 스택이 늘어나는 문제를 막음

<br>

### 3-3. saveState와 restoreState — 하단 탭 전환

- 하단 탭(Bottom Navigation)처럼 탭 간 이동 시 각 탭의 **스크롤 위치·화면 상태를 보존**하려면 상태 저장·복원이 필요함
- `popUpTo(시작 목적지) { saveState = true }`로 걷어낸 탭의 상태를 저장하고, `restoreState = true`로 그 탭에 돌아올 때 복원함

```kotlin
// 하단 탭 이동의 관용구
fun NavHostController.navigateToTab(route: Any) = navigate(route) {
    popUpTo(graph.findStartDestination().id) { saveState = true }   // 다른 탭 스택은 저장
    launchSingleTop = true                                          // 중복 쌓기 방지
    restoreState = true                                             // 이전 상태 복원
}
```

| **옵션**                  | **하는 일**                                             | **대표 상황**                          |
| ------------------------- | ------------------------------------------------------- | -------------------------------------- |
| **popUpTo**               | 지정 목적지까지 스택 제거 후 이동                       | 로그인·온보딩 완료, 플로우 종료        |
| **inclusive**             | popUpTo 대상 자신도 제거                                | 로그인 화면 자체를 없앨 때             |
| **launchSingleTop**       | 맨 위와 같은 목적지면 재사용                            | 버튼 연타, 탭 반복 터치                |
| **saveState / restoreState** | 걷어낸 목적지 상태 저장 / 재진입 시 복원              | 하단 탭 전환                           |
| **popBackStack()**        | 맨 위 제거(뒤로 가기와 동일). 스택이 비면 `false` 반환  | 완료 후 이전 화면 복귀                 |

> 💡 "결제 완료 후 뒤로 가기를 누르면 결제 화면으로 돌아가지 않게 하려면?"이라는 질문은 **`popUpTo` + `inclusive`**를 묻는 것이다. 어느 목적지까지 걷어낼지를 그림으로 설명할 수 있어야 한다.

<br>

### 4. 딥링크(Deep Link)

- 외부(브라우저·알림·다른 앱)에서 **URI로 앱의 특정 화면을 직접 여는** 기능
- 목적지에 `navDeepLink`를 선언하면 NavController가 URI의 경로·쿼리를 인자로 매핑해 줌
- **명시적 딥링크**: 앱 내부에서 `PendingIntent`로 만드는 것(알림 클릭). 시작 목적지부터 합성된 백스택 위에 목적지를 쌓아 뒤로 가기가 자연스러움
- **암시적 딥링크**: 외부 URI로 열리는 것. `AndroidManifest.xml`의 `<intent-filter>`가 필요하며, 웹 도메인 소유를 검증한 **App Links**(Android App Links)를 쓰면 확인 대화상자 없이 바로 열림

```kotlin
composable<ProductDetail>(
    deepLinks = listOf(
        navDeepLink<ProductDetail>(basePath = "https://example.com/product"),   // https://example.com/product/42
    ),
) { backStackEntry -> ProductDetailRoute(productId = backStackEntry.toRoute<ProductDetail>().id) }
```

```xml
<!-- AndroidManifest.xml: 암시적 딥링크는 인텐트 필터가 있어야 시스템이 앱을 후보로 띄움 -->
<activity android:name=".MainActivity" android:exported="true">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="example.com" android:pathPrefix="/product" />
    </intent-filter>
</activity>
```

❗️**딥링크로 들어온 인자는 외부 입력이다**: URI의 값은 사용자가 조작할 수 있으므로 서버 검증 없이 권한이 필요한 화면(관리자 페이지, 타인의 주문 상세)을 그대로 열면 안 된다. 인증·인가 확인은 화면이 아니라 데이터 계층에서 수행한다.

<br>

### 5. 중첩 그래프와 화면 간 데이터 공유

- 관련 화면(회원가입 1·2·3단계)을 **중첩 그래프(`navigation<SignUpGraph>`)**로 묶으면 그래프 단위로 `popUpTo`할 수 있고, 외부에서는 그래프 하나로 진입함
- 그래프의 `NavBackStackEntry`를 소유자로 `hiltViewModel(parentEntry)`를 얻으면 **여러 단계가 하나의 ViewModel을 공유**함. 그래프를 벗어나면 함께 폐기됨

```kotlin
navigation<SignUpGraph>(startDestination = SignUpStep1) {
    composable<SignUpStep1> { entry ->
        val parentEntry = remember(entry) { navController.getBackStackEntry<SignUpGraph>() }
        val sharedViewModel: SignUpViewModel = hiltViewModel(parentEntry)   // 그래프 스코프 ViewModel
        SignUpStep1Route(sharedViewModel)
    }
    composable<SignUpStep2> { /* 같은 방식으로 sharedViewModel 획득 */ }
}
```

- 이전 화면으로 **결과를 돌려줄 때**는 `previousBackStackEntry?.savedStateHandle["key"] = 결과`를 사용하되, 복잡한 경우 공유 ViewModel이나 저장소를 통하는 편이 안전함

> ⚠️ `getBackStackEntry<SignUpGraph>()`는 해당 그래프가 **현재 백스택에 없으면 `IllegalArgumentException`**을 던진다. 그래프 밖의 화면에서 그래프 스코프 ViewModel을 얻으려 하면 안 되며, 그래프 안에서도 `remember(entry)`로 감싸 재구성마다 재조회하지 않도록 한다.

<br>

### 6. 흔한 함정

- **NavController를 컴포저블 깊숙이 전달**: 하위 컴포저블이 내비게이션에 결합되어 미리보기·테스트가 어려움. 화면 컴포저블은 `onNavigateToDetail: (Long) -> Unit` 같은 **콜백만** 받고, NavHost에서 실제 이동을 처리함
- **`navigate`를 컴포저블 본문에서 직접 호출**: 재구성마다 이동이 반복됨. 이동은 콜백이나 `LaunchedEffect` 안에서 수행 (unit05 참고)
- **큰 객체를 인자로 전달**: Bundle 제한·데이터 불일치. ID만 전달하고 재조회
- **하단 탭에 `saveState`·`restoreState` 누락**: 탭을 오갈 때마다 스크롤 위치가 초기화되고 스택이 무한히 쌓임
- **`popBackStack()` 반환값 무시**: 딥링크로 직접 진입해 스택이 비어 있으면 아무 일도 일어나지 않음. 이 경우 시작 목적지로 이동하거나 Activity를 종료하는 대안이 필요함

<br>

### 7. 정리

- Navigation은 **그래프 + 백스택** 모델이며, `NavController`가 이동 명령과 스택을 관리하고 각 `NavBackStackEntry`가 인자·`SavedStateHandle`·ViewModel 소유자를 제공함
- 인자는 **`@Serializable` 타입 안전 경로**(2.8.0+)로 정의하고 `toRoute()`로 복원하며, ID 같은 작은 값만 전달함
- **`popUpTo`+`inclusive`**로 돌아가면 안 되는 화면을 제거하고, **`launchSingleTop`**으로 중복 쌓기를 막고, **`saveState`/`restoreState`**로 탭 상태를 보존함
- 딥링크는 `navDeepLink` + 매니페스트 인텐트 필터로 구성하며, App Links로 도메인을 검증하면 대화상자 없이 열림. 딥링크 인자는 외부 입력으로 취급함
- 중첩 그래프는 플로우 단위 관리와 **그래프 스코프 ViewModel 공유**를 가능하게 함
- 화면 컴포저블은 NavController 대신 **콜백**만 받아 내비게이션과 결합을 끊음 (테스트 관점은 unit10 참고)
