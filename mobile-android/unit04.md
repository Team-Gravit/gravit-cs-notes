## Compose 안정성과 최적화

재구성(Recomposition)이 불필요하게 넓게 일어나면 스크롤 버벅임과 프레임 드랍으로 이어진다. Compose 컴파일러가 컴포저블을 **건너뛸 수 있는지(Skippable)** 판단하는 근거인 **안정성(Stability)**, 파생 상태를 효율적으로 만드는 **derivedStateOf**, 리스트 항목의 정체성을 알려주는 **LazyList key**는 Compose 성능 최적화의 세 축이다.

<br>

### 1. 건너뛰기가 성능의 출발점

- unit02에서 본 것처럼, 재구성 대상 스코프 안에 있어도 **인자가 바뀌지 않은 컴포저블은 실행을 건너뜀**
- 그런데 컴파일러가 "인자가 안 바뀌었다"고 **확신할 수 없는 타입**이 인자로 오면 건너뛸 수 없어 매번 재실행됨
- 즉 성능 최적화의 첫 단계는 **컴파일러가 인자의 동일성을 신뢰할 수 있게 만드는 것**이며, 이것이 안정성 개념임

```
부모 재구성 발생
   │
   ├─ 자식 A(인자: String, Int)          → 안정적, 값 동일 → 건너뜀 ✔
   ├─ 자식 B(인자: List<User>)           → 불안정 → 값이 같아도 재실행 ✘ (기본 모드)
   └─ 자식 C(인자: data class(var ...))  → 불안정 → 재실행 ✘
```

<br>

### 2. 안정성(Stability)

### 2-1. 안정적 타입의 조건

컴파일러는 다음 조건을 만족하는 타입을 **안정적(Stable)**으로 추론한다.

- 두 인스턴스의 `equals` 결과가 **영원히 변하지 않음** (같았다면 계속 같음)
- 공개 프로퍼티가 바뀌면 **컴포지션에 통지됨** (`MutableState`처럼)
- 모든 공개 프로퍼티의 타입 역시 안정적임

| **타입**                                    | **판정**       | **이유**                                                    |
| ------------------------------------------- | -------------- | ----------------------------------------------------------- |
| **기본형·String·함수 타입**                 | **안정적**     | 불변이거나 값 비교가 안전함                                 |
| **`val`만 가진 data class (안정적 필드만)** | **안정적**     | 불변 + 구조적 `equals`                                      |
| **`var`를 가진 클래스**                     | 불안정         | 통지 없이 값이 바뀔 수 있음                                 |
| **`List`·`Map`·`Set` (코틀린 표준 인터페이스)** | 불안정      | 구현체가 `MutableList`일 수 있어 불변을 보장 못함           |
| **`MutableState<T>`**                       | **안정적**     | 변경 시 Compose에 통지됨                                    |
| **다른 모듈의 클래스**                      | 불안정(기본)   | 컴파일러가 소스를 분석할 수 없음 (설정으로 변경 가능)       |

> ⚠️ 가장 자주 걸리는 함정은 **`List<T>`**다. `listOf()`로 만든 읽기 전용 목록이어도 타입 자체가 `List` 인터페이스이므로 컴파일러는 불안정으로 취급한다. 항목 데이터 클래스가 완벽히 불변이어도 목록을 감싸는 순간 부모 재구성 때마다 자식이 재실행될 수 있다.

<br>

### 2-2. 안정성 확보 방법

```kotlin
// 안티패턴: List 인자와 var 프로퍼티 → 두 컴포저블 모두 건너뛰기 불가
data class User(var name: String, val tags: List<String>)

@Composable
fun UserList(users: List<User>) { /* ... */ }

// 개선 1: 불변 필드 + 불변 컬렉션(kotlinx.collections.immutable)으로 추론 가능하게 만듦
data class User(val name: String, val tags: ImmutableList<String>)

@Composable
fun UserList(users: ImmutableList<User>) { /* ... */ }

// 개선 2: 외부 모듈 타입 등 추론이 불가능할 때 개발자가 계약을 보증함
@Immutable                         // "생성 후 절대 바뀌지 않는다"를 약속
data class UiState(val items: List<Item>, val loading: Boolean)

@Stable                            // "바뀌면 반드시 Compose에 통지한다"를 약속
class ScrollController { var offset by mutableStateOf(0) }
```

- **`@Immutable`**: 모든 공개 값이 생성 후 불변임을 선언. 컴파일러는 검증하지 않으므로 약속을 어기면 화면이 갱신되지 않는 버그가 생김
- **`@Stable`**: 값은 바뀔 수 있지만 변경이 Snapshot 시스템에 통지된다는 약속. `MutableState`를 내부에 쓰는 상태 홀더에 적합
- **안정성 설정 파일**(Stability Configuration File): 외부 라이브러리 클래스(예: `java.time.LocalDate`)를 안정적으로 취급하도록 컴파일러 옵션으로 지정할 수 있음

<br>

### 2-3. Strong Skipping 모드

- 불안정한 인자를 가진 컴포저블도 **참조 동일성(`===`)**으로 비교해 건너뛸 수 있게 하는 컴파일러 모드
- Kotlin **2.0.20**의 Compose 컴파일러부터 **기본 활성화**되었으며, 그 이전 버전에서는 명시적으로 켜야 함 (프로젝트 버전에 따라 다를 수 있음)
- 불안정한 인자로 넘어오는 람다도 자동으로 `remember`되어 매번 새 인스턴스가 생기는 문제가 줄어듦
- 그러나 **참조가 바뀌면 여전히 재실행**되므로, 매번 새 `List`를 만들어 넘기는 코드(`items.filter { }` 등)는 여전히 문제임 → 안정성 설계는 여전히 중요함

> 💡 면접에서 "Strong Skipping이 있으면 @Immutable이 필요 없나요?"라고 물으면, **비교 기준이 `equals`에서 참조 동일성으로 완화된 것일 뿐**이며 새 인스턴스가 계속 생성되는 구조에서는 여전히 건너뛰지 못한다고 답하면 된다.

<br>

### 3. derivedStateOf — 파생 상태의 갱신 빈도 줄이기

- **자주 바뀌는 상태**에서 **드물게 바뀌는 값**을 계산할 때 사용함
- 계산 결과가 이전과 같으면 이 값을 읽은 컴포저블을 **무효화하지 않음** → 재구성 횟수가 결과 변경 횟수로 줄어듦
- 결과가 입력만큼 자주 바뀐다면(예: 단순 포매팅) 이득이 없고 오히려 오버헤드만 생김. 그런 경우 일반 `remember(key)`로 충분함

```kotlin
// 안티패턴: 스크롤 위치(매 프레임 변경)를 직접 읽음 → 버튼이 매 프레임 재구성됨
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    val visible = listState.firstVisibleItemIndex > 0
    if (visible) FloatingActionButton(onClick = { /* 위로 */ }) { Icon(Icons.Default.KeyboardArrowUp, null) }
}

// 개선: 결과(Boolean)가 바뀔 때만 재구성됨
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    val visible by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }
    if (visible) FloatingActionButton(onClick = { /* 위로 */ }) { Icon(Icons.Default.KeyboardArrowUp, null) }
}
```

| **상황**                                             | **적합한 도구**              | **이유**                                             |
| ---------------------------------------------------- | ---------------------------- | ---------------------------------------------------- |
| **입력은 매 프레임, 결과는 가끔 변함**               | `derivedStateOf`             | 결과가 같으면 무효화를 막아 재구성을 줄임            |
| **입력 변경 = 결과 변경** (포매팅, 단순 매핑)        | `remember(key) { }`          | 매번 결과가 바뀌므로 파생 상태의 이점이 없음         |
| **입력이 컴포저블 인자(상태 아님)**                  | `remember(arg) { }`          | 인자 변경은 Snapshot 추적이 아니므로 키로 갱신함     |

> ⚠️ `derivedStateOf` 블록 안에서 읽는 값이 **컴포저블 인자**라면 `remember(arg) { derivedStateOf { } }`처럼 키를 넘겨야 한다. 인자는 Snapshot 상태가 아니어서 자동으로 추적되지 않기 때문이다.

<br>

### 4. LazyList key와 contentType

### 4-1. key — 항목의 정체성 알려주기

- `LazyColumn`은 기본적으로 **인덱스**를 항목의 식별자로 사용함
- 맨 앞에 항목을 삽입하거나 순서를 바꾸면 모든 인덱스가 밀려 **모든 항목이 새 항목으로 인식**됨 → 상태(`remember`) 손실, 전체 재구성, 애니메이션 불가
- `key`로 안정적인 고유 ID를 주면 이동한 항목의 컴포지션과 상태를 그대로 재사용함

```kotlin
// 안티패턴: 인덱스 기반 → 맨 앞 삽입 시 전체가 다시 그려지고 항목 내부 상태가 어긋남
LazyColumn {
    items(messages) { message -> MessageRow(message) }
}

// 개선: 고유 ID를 key로, 유형별 contentType으로 재사용 풀을 분리
LazyColumn {
    items(
        items = messages,
        key = { it.id },
        contentType = { if (it.isMine) "mine" else "others" },
    ) { message ->
        MessageRow(message, modifier = Modifier.animateItem())   // key가 있어야 동작함
    }
}
```

<br>

### 4-2. 함정과 규칙

- key는 **Bundle에 저장 가능한 타입**이어야 하고(스크롤 위치 복원에 쓰임), 목록 안에서 **유일**해야 함. 중복되면 런타임 예외가 발생함
- 인덱스나 무작위 값을 key로 쓰면 안 쓴 것과 같거나 더 나쁨
- `contentType`은 서로 다른 레이아웃을 가진 항목(헤더·광고·일반 행)이 섞여 있을 때 같은 유형끼리만 컴포지션을 재사용하도록 힌트를 줌 (RecyclerView의 viewType과 같은 역할, unit09 참고)

<br>

### 5. 그 밖의 최적화 습관

- **상태 읽기 지연**: 매 프레임 바뀌는 값은 `Modifier.offset { }`, `graphicsLayer { }`, `drawBehind { }` 람다 안에서 읽어 레이아웃·그리기 단계만 재실행 (unit02 참고)
- **람다 안정화**: 메서드 참조(`viewModel::onClick`)나 `remember`된 람다를 넘겨 매번 새 람다 인스턴스가 생기지 않게 함 (Strong Skipping에서는 자동 처리되는 경우가 많음)
- **컬렉션 변환은 ViewModel에서**: 컴포저블 안에서 `filter`·`map`으로 새 목록을 만들면 매 재구성마다 새 참조가 생김. UI 상태로 미리 가공해 내려보냄
- **역방향 쓰기(Backwards Write) 금지**: 컴포지션 중에 이미 읽은 상태에 쓰면 무한 재구성 루프가 생김

<br>

### 6. 측정 도구

| **도구**                       | **확인 내용**                                                | **사용 시점**                        |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------ |
| **Compose 컴파일러 리포트**    | 각 함수의 Skippable·Restartable 여부, 불안정한 매개변수 목록 | 안정성 문제 진단                     |
| **Layout Inspector**           | 컴포저블별 재구성·건너뛰기 횟수                              | 어떤 컴포저블이 과도하게 실행되는지  |
| **Macrobenchmark + Baseline Profile** | 앱 시작·스크롤 프레임 시간, 사전 컴파일 효과         | 릴리스 빌드 성능 측정                |

❗️**측정 없이 최적화하지 않는다**: 재구성 횟수가 많아도 프레임 예산(약 16ms) 안에 끝나면 문제가 아니다. 반드시 **릴리스 빌드(R8 적용)**에서 측정해야 하며, 디버그 빌드의 느림은 실제 성능과 무관한 경우가 많다.

<br>

### 7. 정리

- 컴포저블을 건너뛰려면 인자가 **안정적**이어야 하며, `var`·`List` 인터페이스·외부 모듈 클래스는 기본적으로 불안정으로 추론됨
- 불변 필드 + 불변 컬렉션으로 추론을 돕고, 필요하면 **`@Immutable`·`@Stable`**로 계약을 보증하되 약속을 어기면 갱신 누락 버그가 생김
- **Strong Skipping**(Kotlin 2.0.20+ 기본)은 참조 동일성으로 비교를 완화하지만, 새 인스턴스를 계속 만드는 코드는 여전히 재실행됨
- **`derivedStateOf`**는 입력은 자주 바뀌고 결과는 드물게 바뀔 때만 이득이 있음
- **LazyList `key`**는 항목 정체성을 보존해 상태 손실·전체 재구성·애니메이션 불가를 막고, `contentType`은 재사용 풀을 분리함
- 최적화는 컴파일러 리포트와 Layout Inspector로 **측정한 뒤** 릴리스 빌드 기준으로 판단함
