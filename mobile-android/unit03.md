## Compose 상태 관리

Compose에서 **상태(State)**는 시간에 따라 변하고 UI에 영향을 주는 모든 값이다. 상태를 **어디에 두고**(`remember`·`rememberSaveable`·ViewModel), **누가 소유하며**(상태 호이스팅), 어떤 경로로 흐르게 할지(단방향 데이터 흐름)를 결정하는 것이 Compose 설계의 핵심이다.

<br>

### 1. 상태와 MutableState

- Compose가 변경을 감지하려면 값이 **Snapshot 상태 객체**(`State<T>` / `MutableState<T>`)여야 함 (읽기 추적 원리는 unit02 참고)
- `mutableStateOf(초기값)`으로 생성하며 `.value`를 읽으면 추적되고, 쓰면 읽은 컴포저블이 무효화됨
- 목록·맵은 `mutableStateListOf()`, `mutableStateMapOf()`를 쓰면 원소 단위 변경도 추적됨. 일반 `MutableList`를 `mutableStateOf`에 넣고 `add()`만 하면 **참조가 같아서 변경이 감지되지 않음**

```kotlin
// 안티패턴: 일반 변수 → 값이 바뀌어도 재구성이 일어나지 않아 화면이 갱신되지 않음
@Composable
fun Counter() {
    var count = 0
    Button(onClick = { count++ }) { Text("$count") }   // 항상 0
}

// 개선: 상태 객체 + remember → 변경 감지, 재구성 사이에 값 유지
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }        // by 위임으로 .value 생략
    Button(onClick = { count++ }) { Text("$count") }
}
```

> 💡 `mutableStateOf`만 쓰고 `remember`를 빼면 **재구성마다 초기값으로 새 객체가 만들어져** 값이 항상 0으로 돌아간다. "상태 객체를 만드는 것"과 "재구성 사이에 유지하는 것"은 별개의 문제다.

<br>

### 2. remember와 rememberSaveable

### 2-1. remember — 컴포지션 안에서 기억하기

- `remember { }`는 계산 결과를 **컴포지션 트리의 해당 위치**에 저장하고, 재구성 시 다시 계산하지 않고 꺼내 씀
- 컴포저블이 트리에서 **제거되면 값도 함께 버려짐** (조건부로 사라지는 UI, LazyColumn 밖으로 스크롤된 항목 등)
- 키를 넘기면(`remember(key1) { }`) 키가 바뀔 때만 다시 계산함. 상태뿐 아니라 무거운 객체(포매터, 계산 결과) 캐시에도 사용함

<br>

### 2-2. rememberSaveable — 구성 변경과 프로세스 종료까지 견디기

- `remember`는 **Activity 재생성(화면 회전)** 시 컴포지션이 통째로 사라지므로 값이 초기화됨
- `rememberSaveable`은 값을 `Bundle`(SavedInstanceState)에 함께 저장해 회전·프로세스 종료 후 복원함
- Bundle에 담을 수 있는 타입은 자동 처리되고, 그 외 타입은 **Saver**를 직접 정의해야 함

```kotlin
data class Filter(val keyword: String, val onlyActive: Boolean)

// 사용자 정의 타입은 Saver로 Bundle 호환 형태로 변환
val FilterSaver = mapSaver(
    save = { mapOf("keyword" to it.keyword, "onlyActive" to it.onlyActive) },
    restore = { Filter(it["keyword"] as String, it["onlyActive"] as Boolean) },
)

@Composable
fun FilterBar() {
    var filter by rememberSaveable(stateSaver = FilterSaver) {
        mutableStateOf(Filter(keyword = "", onlyActive = false))
    }
    TextField(value = filter.keyword, onValueChange = { filter = filter.copy(keyword = it) })
}
```

<br>

### 2-3. 비교

| **항목**                | **remember**                        | **rememberSaveable**                          | **ViewModel + SavedStateHandle**        |
| ----------------------- | ----------------------------------- | --------------------------------------------- | --------------------------------------- |
| **재구성 사이 유지**    | **유지**                            | 유지                                          | 유지                                    |
| **화면 회전(구성 변경)** | 초기화                              | **유지**                                      | 유지                                    |
| **프로세스 종료 후 복원** | 초기화                             | **복원**(Bundle)                              | 복원(SavedStateHandle)                  |
| **컴포지션에서 제거 시** | 소멸                               | 소멸                                          | **유지**(소유자 수명)                   |
| **적합한 데이터**       | 애니메이션 값, 캐시된 객체          | 텍스트 입력, 펼침 여부, 선택된 탭             | 화면 데이터, 비즈니스 상태              |

> ⚠️ `rememberSaveable`은 Bundle에 저장되므로 **큰 목록이나 이미지 같은 무거운 데이터를 넣으면 안 된다**. 프로세스 종료 대비가 필요한 대용량 데이터는 ID만 저장하고 ViewModel·저장소에서 다시 불러온다 (unit01 참고).

<br>

### 3. 상태 호이스팅(State Hoisting)

### 3-1. 개념

- 상태를 컴포저블 내부에 두지 않고 **호출자(부모)로 끌어올리고**, 자식은 값(`value`)과 변경 요청 콜백(`onValueChange`)만 받는 패턴
- 상태를 가진 컴포저블은 **Stateful**, 상태를 외부에서 받는 컴포저블은 **Stateless**라고 부름
- 결과적으로 **상태는 아래로, 이벤트는 위로** 흐르는 **단방향 데이터 흐름(UDF, Unidirectional Data Flow)**이 만들어짐

```
            ┌────────────────────────────┐
            │  상태 소유자 (부모·ViewModel)  │
            └──────┬──────────────▲──────┘
      상태(value)  │              │  이벤트(onValueChange)
                   ▼              │
            ┌──────────────────────┐
            │  Stateless 자식 컴포저블  │
            └──────────────────────┘
```

<br>

### 3-2. 안티패턴과 개선

```kotlin
// 안티패턴: 검색어를 TextField 컴포저블이 소유 → 부모(목록)가 검색어를 알 수 없음
@Composable
fun SearchField() {
    var query by remember { mutableStateOf("") }
    TextField(value = query, onValueChange = { query = it })
}

// 개선: 상태를 부모로 끌어올리고 자식은 Stateless로 만듦
@Composable
fun SearchField(query: String, onQueryChange: (String) -> Unit) {
    TextField(value = query, onValueChange = onQueryChange)
}

@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    val results by viewModel.results.collectAsStateWithLifecycle()
    Column {
        SearchField(query = query, onQueryChange = viewModel::onQueryChange)
        ResultList(results)                       // 같은 상태를 형제가 함께 사용
    }
}
```

<br>

### 3-3. 어디까지 끌어올릴 것인가

- **원칙**: 상태를 읽는 **모든 컴포저블의 가장 가까운 공통 부모**까지만 올림. 무조건 ViewModel까지 올리면 UI 전용 상태가 ViewModel을 오염시킴
- 상태를 **변경하는 곳**이 소유자보다 위에 있다면 그 위치까지 올려야 함
- 형제 컴포저블이 같은 상태를 봐야 하면 최소한 그 부모까지 올림

> 💡 호이스팅의 장점은 **재사용성·테스트 용이성·단일 진실 원천**이다. Stateless 컴포저블은 미리보기(Preview)와 UI 테스트에서 상태를 직접 주입해 검증할 수 있다 (unit10 참고).

<br>

### 4. 상태 홀더 — 상태를 둘 세 가지 위치

| **위치**                          | **수명**                     | **담는 상태**                                   | **선택 기준**                                  |
| --------------------------------- | ---------------------------- | ----------------------------------------------- | ---------------------------------------------- |
| **컴포저블 내부**(`remember`)     | 컴포지션 위치                | 단순 UI 상태 1~2개                              | 로직이 거의 없고 한 곳에서만 쓰는 경우         |
| **일반 상태 홀더 클래스**         | 컴포지션 위치(`remember`)    | 여러 UI 상태와 관련 로직 묶음                   | UI 로직이 복잡해져 컴포저블이 비대해지는 경우  |
| **ViewModel**                     | 화면(소유자) 수명            | 화면 데이터, 비즈니스 로직, 저장소 접근         | 구성 변경 생존·비동기 작업·DI가 필요한 경우    |

```kotlin
// 일반 상태 홀더: UI 로직을 컴포저블 밖으로 분리하되 ViewModel까지는 올리지 않음
@Stable
class SearchBarState(initialQuery: String) {
    var query by mutableStateOf(initialQuery)
        private set
    var expanded by mutableStateOf(false)
        private set

    fun onQueryChange(value: String) { query = value; expanded = value.isNotBlank() }
    fun collapse() { expanded = false }
}

@Composable
fun rememberSearchBarState(initialQuery: String = ""): SearchBarState =
    rememberSaveable(saver = SearchBarStateSaver) { SearchBarState(initialQuery) }
```

- 상태 홀더는 `remember`로 만들고, 복원이 필요하면 `rememberSaveable` + `Saver`로 감싼 `rememberXxxState()` 팩토리를 제공하는 것이 Compose 라이브러리의 관례임 (`rememberScrollState`, `rememberLazyListState` 등)

<br>

### 5. 흔한 함정

- **ViewModel에 `mutableStateOf` 노출 시 `private set` 누락**: 외부에서 임의로 쓸 수 있어 단방향 흐름이 깨짐. `StateFlow`의 `asStateFlow()`처럼 읽기 전용 타입으로 노출함
- **`remember`에 키 누락**: 인자에 따라 달라져야 하는 계산에 `remember { }`만 쓰면 첫 인자 기준 결과가 계속 재사용됨 → `remember(arg) { }`
- **컴포저블 인자로 ViewModel 전체 전달**: 자식이 ViewModel에 결합되어 미리보기·테스트가 어려워짐. 필요한 값과 콜백만 넘김
- **`derivedStateOf` 없이 파생 값을 매번 계산**: 자주 바뀌는 상태로부터 드물게 바뀌는 값을 만들 때 불필요한 재구성이 발생함 (unit04 참고)

❗️**상태는 한 곳에서만 소유한다**: 같은 정보를 ViewModel과 `remember` 양쪽에 두면 어느 쪽이 진실인지 알 수 없어진다. 소유자를 하나로 정하고 나머지는 그 값을 받아 그리기만 한다.

<br>

### 6. 정리

- 변경을 감지하려면 **Snapshot 상태 객체**여야 하고, 재구성 사이에 유지하려면 **`remember`**가 필요함
- `remember`는 회전 시 초기화되고, **`rememberSaveable`**은 Bundle을 통해 회전·프로세스 종료를 견딤. 큰 데이터는 넣지 않음
- **상태 호이스팅**으로 자식을 Stateless로 만들면 상태는 아래로, 이벤트는 위로 흐르는 **단방향 데이터 흐름**이 됨
- 끌어올리는 높이는 **상태를 읽고 쓰는 모든 곳의 가장 가까운 공통 부모**까지
- 상태 위치는 **컴포저블 로컬 → 상태 홀더 클래스 → ViewModel** 순으로 복잡도와 수명 요구에 따라 선택함
- 재구성 원리는 **unit02**, 안정성과 `derivedStateOf`는 **unit04**, ViewModel의 상태 수집은 **unit06**을 참고할 것
