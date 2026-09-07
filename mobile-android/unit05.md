## Compose 부수효과 API

컴포저블 함수는 재구성(Recomposition)마다 다시 실행되고 실행 횟수·순서를 보장하지 않으므로, 네트워크 호출·리스너 등록·외부 객체 변경 같은 **부수효과(Side Effect)**를 본문에 직접 쓸 수 없다. Compose는 부수효과를 **컴포지션의 생명주기에 맞춰 안전하게 실행**하기 위한 API(`LaunchedEffect`·`DisposableEffect`·`SideEffect` 등)를 제공하며, 상황에 맞는 것을 고르는 기준을 아는 것이 핵심이다.

<br>

### 1. 부수효과란 무엇이고 왜 문제인가

- **부수효과**: 컴포저블 함수의 범위 밖에서 관찰 가능한 상태 변화 (API 호출, 로그 전송, 스낵바 표시, 콜백 등록, 전역 변수 변경 등)
- 컴포저블은 **멱등**해야 하고 언제든 다시 실행될 수 있으므로(unit02 참고), 본문에 부수효과가 있으면 재구성 횟수만큼 반복됨
- 게다가 컴포지션이 **취소·폐기**될 수 있어(화면 이탈, 조건부 UI 제거), 정리(clean-up) 없이 시작한 작업은 누수나 죽은 화면 갱신으로 이어짐
- 해결책은 부수효과를 **컴포지션에 진입·이탈하는 시점**과 **특정 키의 변경 시점**에 묶어 실행하는 것

```
컴포저블 수명
   진입(Enter) ───────── 재구성 ─── 재구성 ─── 재구성 ───────── 이탈(Leave)
      │                    │          │          │                  │
LaunchedEffect(key) 시작   │   key 변경 시 취소 후 재시작            │  코루틴 취소
DisposableEffect(key) 등록 │   key 변경 시 onDispose 후 재등록       │  onDispose 실행
SideEffect          매 성공적 재구성 후 실행 ──────────────────────┘
```

<br>

### 2. LaunchedEffect — 컴포지션에 묶인 코루틴

- 컴포지션에 **진입할 때** `key`를 기준으로 코루틴을 시작하고, 컴포저블이 **이탈하면 자동 취소**됨
- `key`가 바뀌면 실행 중인 코루틴을 취소하고 새 키로 다시 시작함
- `suspend` 함수 호출, 스낵바 표시, 특정 값 변경에 반응하는 일회성 작업에 적합함

```kotlin
// 안티패턴: 본문에서 스낵바 표시 → 재구성마다 중복 표시, suspend 함수라 컴파일조차 되지 않음
@Composable
fun ProfileScreen(state: ProfileUiState, snackbarHostState: SnackbarHostState) {
    if (state.errorMessage != null) {
        snackbarHostState.showSnackbar(state.errorMessage)   // 컴포저블 본문에서 suspend 호출 불가
    }
}

// 개선: 에러 메시지가 바뀔 때 한 번만 실행되고, 화면 이탈 시 취소됨
@Composable
fun ProfileScreen(state: ProfileUiState, snackbarHostState: SnackbarHostState, onErrorShown: () -> Unit) {
    LaunchedEffect(state.errorMessage) {
        val message = state.errorMessage ?: return@LaunchedEffect
        snackbarHostState.showSnackbar(message)
        onErrorShown()                                       // 소비 후 상태를 비워 재표시 방지
    }
}
```

- `LaunchedEffect(Unit)` 또는 `LaunchedEffect(true)`는 "진입 시 딱 한 번"을 뜻하지만, **컴포지션에서 나갔다 다시 들어오면 다시 실행**됨. 화면 이동 후 복귀 시 재호출되어도 괜찮은 작업인지 확인해야 함

> ⚠️ `LaunchedEffect`의 키에 **매번 새로 만들어지는 객체**(람다, 새 리스트)를 넣으면 재구성마다 취소·재시작이 반복된다. 키는 "이 값이 바뀌면 작업을 다시 해야 하는가"라는 질문으로 결정한다.

<br>

### 3. DisposableEffect — 정리가 필요한 효과

- 리스너·콜백·옵저버처럼 **등록과 해제가 짝을 이루는** 작업에 사용함
- 블록의 마지막에 반드시 `onDispose { }`를 반환해야 하며, 컴포저블 이탈 시와 `key` 변경 시 호출됨
- 대표 예: `LifecycleObserver` 등록, 시스템 브로드캐스트 수신, 센서 리스너, 뒤로 가기 콜백

```kotlin
@Composable
fun LifecycleLogger(lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current, onResume: () -> Unit) {
    val currentOnResume by rememberUpdatedState(onResume)   // 최신 람다를 항상 참조

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_RESUME) currentOnResume()
        }
        lifecycleOwner.lifecycle.addObserver(observer)

        onDispose {                                           // 이탈·키 변경 시 반드시 해제
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

- `onDispose`를 빠뜨리면 컴파일 오류가 나므로 정리 누락을 구조적으로 막아 줌
- `LaunchedEffect`와 달리 코루틴 스코프가 아니므로 내부에서 `suspend` 함수는 호출할 수 없음

<br>

### 4. SideEffect — 매 재구성 후 외부 객체 동기화

- 컴포지션이 **성공적으로 적용된 직후** 매번 실행됨. 취소·정리 개념이 없음
- Compose가 관리하지 않는 외부 객체(분석 SDK, 시스템 UI 컨트롤러 등)에 **현재 컴포지션 상태를 반영**할 때 사용함
- 재구성이 중단(취소)되면 실행되지 않으므로, 실제로 화면에 반영된 상태만 외부에 전달됨

```kotlin
@Composable
fun AnalyticsUser(user: User, analytics: FirebaseAnalytics) {
    // 재구성이 확정될 때마다 외부 SDK에 최신 사용자 정보를 동기화
    SideEffect { analytics.setUserProperty("userType", user.type.name) }
    ProfileContent(user)
}
```

> 💡 `SideEffect`는 "매번 실행"이 목적이므로 가볍고 멱등한 작업만 넣는다. 네트워크 요청처럼 무거운 작업이나 한 번만 해야 하는 작업을 넣으면 안 된다.

<br>

### 5. 보조 API

### 5-1. rememberCoroutineScope — 이벤트 핸들러에서 코루틴 시작

- `LaunchedEffect`는 컴포저블 본문에서만 호출 가능함. 버튼 클릭 같은 **콜백 안에서** 코루틴을 시작하려면 `rememberCoroutineScope()`로 얻은 스코프를 사용함
- 이 스코프는 컴포저블 이탈 시 취소되므로 죽은 화면을 갱신하는 문제가 없음

```kotlin
@Composable
fun ScrollToTopFab(listState: LazyListState) {
    val scope = rememberCoroutineScope()
    FloatingActionButton(onClick = {
        scope.launch { listState.animateScrollToItem(0) }   // 콜백에서 suspend 호출
    }) { Icon(Icons.Default.KeyboardArrowUp, contentDescription = "맨 위로") }
}
```

> ⚠️ `rememberCoroutineScope`로 시작한 작업은 컴포저블이 이탈하면 **함께 취소**된다. 화면이 사라져도 끝까지 완료되어야 하는 작업(저장·업로드·결제 요청)은 UI 스코프가 아니라 `viewModelScope`나 WorkManager에서 실행해야 한다.

<br>

### 5-2. rememberUpdatedState — 재시작 없이 최신 값 참조

- 오래 실행되는 효과 안에서 **람다나 값이 바뀌어도 효과를 재시작하고 싶지 않을 때** 사용함
- 키에 넣으면 재시작되고, 넣지 않으면 오래된 값을 캡처하는 딜레마를 해결함 (3번 예시의 `currentOnResume` 참고)

```kotlin
@Composable
fun SplashScreen(onTimeout: () -> Unit) {
    val currentOnTimeout by rememberUpdatedState(onTimeout)
    LaunchedEffect(Unit) {          // onTimeout이 바뀌어도 타이머는 재시작되지 않음
        delay(3_000)
        currentOnTimeout()          // 호출 시점의 최신 람다 실행
    }
}
```

<br>

### 5-3. produceState와 snapshotFlow

- **`produceState`**: 비 Compose 데이터 소스(콜백·Flow)를 `State<T>`로 변환. 내부적으로 `LaunchedEffect`와 `remember`를 조합한 것
- **`snapshotFlow`**: 반대로 Compose 상태를 `Flow`로 변환. `LazyListState` 스크롤 위치 변화를 Flow 연산자(`distinctUntilChanged`, `debounce`)로 다룰 때 유용함

<br>

### 6. 선택 기준 비교

| **API**                     | **실행 시점**                          | **정리(취소) 시점**            | **suspend 가능** | **대표 용도**                              |
| --------------------------- | -------------------------------------- | ------------------------------ | ---------------- | ------------------------------------------ |
| **LaunchedEffect(key)**     | 진입 시, key 변경 시                   | 이탈 시, key 변경 시 코루틴 취소 | **가능**         | 일회성 suspend 작업, 스낵바, 이벤트 반응   |
| **DisposableEffect(key)**   | 진입 시, key 변경 시                   | 이탈 시, key 변경 시 `onDispose` | 불가             | 리스너·옵저버 등록/해제                    |
| **SideEffect**              | **매 성공적 재구성 후**                | 없음                           | 불가             | 외부 객체에 현재 상태 반영                 |
| **rememberCoroutineScope**  | 콜백에서 직접 `launch`                 | 이탈 시 스코프 취소            | **가능**         | 클릭 등 사용자 이벤트로 시작하는 작업      |
| **rememberUpdatedState**    | 값이 바뀔 때 갱신                      | -                              | -                | 효과 재시작 없이 최신 값 참조              |

```
어떤 API를 쓸까?
   │
   ├─ 사용자 이벤트(클릭)에서 시작?  ── 예 → rememberCoroutineScope
   │
   ├─ 등록/해제 짝이 있는 작업?      ── 예 → DisposableEffect
   │
   ├─ suspend 함수·코루틴 필요?      ── 예 → LaunchedEffect(적절한 key)
   │
   └─ 매 재구성 후 외부에 동기화?    ── 예 → SideEffect
```

<br>

### 7. 흔한 함정

- **비즈니스 로직을 `LaunchedEffect`에 넣기**: 데이터 로딩은 ViewModel의 `viewModelScope`에서 시작하고 UI는 결과 상태만 수집하는 것이 원칙임. 화면 재진입마다 재요청되는 문제를 피할 수 있음 (unit06 참고)
- **일회성 이벤트를 상태 없이 처리**: 스낵바·내비게이션 같은 이벤트를 `LaunchedEffect(Unit)` 안에서 Flow로 수집하면 화면 재구성·복귀 시 유실되거나 중복될 수 있음. 이벤트를 **UI 상태로 모델링하고 소비 후 비우는** 방식이 권장됨
- **`DisposableEffect`의 key 누락**: `lifecycleOwner`처럼 바뀔 수 있는 객체를 키로 넣지 않으면 옛 객체에 등록된 채 남음
- **`LaunchedEffect` 안에서 `remember`되지 않은 외부 값 캡처**: 최신 값이 필요하면 `rememberUpdatedState`로 감싸야 함

❗️**효과는 UI의 부속물이지 로직의 본체가 아니다**: 부수효과 API는 "컴포지션 생명주기에 맞춰 무언가를 시작·정리하는 장치"일 뿐이다. 무엇을 할지는 ViewModel·저장소 계층이 결정하고, 컴포저블은 그 결과를 그리는 데 집중해야 테스트와 재사용이 쉬워진다.

<br>

### 8. 정리

- 컴포저블 본문의 부수효과는 재구성마다 반복되고 정리가 불가능하므로 **컴포지션 생명주기에 묶인 API**로 실행함
- **`LaunchedEffect`**: 키 기준으로 코루틴을 시작·취소. suspend 작업과 이벤트 반응에 사용
- **`DisposableEffect`**: 등록과 해제가 짝을 이루는 작업. `onDispose`로 정리를 강제함
- **`SideEffect`**: 매 성공적 재구성 후 외부 객체에 상태를 동기화하는 가벼운 작업
- **`rememberCoroutineScope`**는 콜백에서, **`rememberUpdatedState`**는 재시작 없이 최신 값을 참조할 때 사용
- 데이터 로딩은 ViewModel에서, 컴포저블은 결과를 그리는 데 집중함. Flow 수집과 이벤트 처리의 상세는 **unit06** 참고
