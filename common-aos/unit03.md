## 구성 변경과 상태 보존

화면 회전, 다크 모드 전환, 언어 변경 같은 **구성 변경(Configuration Change)**이 일어나면 시스템은 액티비티를 **파괴하고 다시 생성**한다. 이때 사용자가 입력한 값·스크롤 위치·네트워크 응답이 사라지지 않도록 상태를 어디에, 어떤 범위로 보존할지 결정하는 것이 이 유닛의 핵심이며, `savedInstanceState`·`ViewModel`·영속 저장소의 역할 분담을 정확히 이해해야 한다.

<br>

### 1. 구성 변경이란

**구성(Configuration)**은 화면 방향·크기·밀도, 로케일, 글꼴 크기, 다크 모드(uiMode), 키보드 상태 등 **리소스 선택에 영향을 주는 기기 환경**의 집합이다. 구성이 바뀌면 `res/values-land/`, `res/values-ko/`, `res/values-night/`처럼 **다른 리소스 세트가 적용**되어야 하므로, 시스템은 가장 확실한 방법인 **액티비티 재생성**을 택한다.

| **구성 변경 원인**       | **변경되는 리소스 한정자**   | **재생성 여부(기본)** |
| ------------------------ | ---------------------------- | --------------------- |
| **화면 회전**            | `orientation`, 크기 한정자   | 재생성                |
| **다크 모드 전환**       | `uiMode` (`-night`)          | 재생성                |
| **언어 변경**            | `locale` (`-ko`, `-en`)      | 재생성                |
| **글꼴 크기 변경**       | `fontScale`                  | 재생성                |
| **멀티윈도우·폴더블**    | `screenSize`, `smallestScreenSize` | 재생성          |
| **키보드 연결**          | `keyboardHidden`             | 재생성                |

```
화면 회전 발생
 onPause() → onStop() → onSaveInstanceState() → onDestroy()
                                │ Bundle
                                ▼
 onCreate(savedInstanceState) → onStart() → onRestoreInstanceState() → onResume()
```

> 💡 재생성은 "버그"가 아니라 **설계**다. 회전만 막으면 된다고 `android:screenOrientation="portrait"`를 걸어도 다크 모드·언어·폴더블 펼침에서는 똑같이 재생성되므로, 근본 해결은 **상태를 액티비티 밖에 두는 것**이다.

<br>

### 2. 무엇이 사라지고 무엇이 남는가

재생성되면 액티비티 인스턴스와 그 **멤버 변수는 모두 사라진다**. 반면 다음은 유지된다.

- **프로세스**: 같은 프로세스 안에서 재생성되므로 `Application` 객체와 정적 변수는 살아 있음
- **뷰 상태 일부**: `android:id`가 있는 뷰는 시스템이 `EditText` 텍스트, 스크롤 위치, 체크 상태 등을 자동으로 저장·복원함
- **ViewModel**: `ViewModelStore`에 보관되어 새 액티비티 인스턴스에 다시 연결됨

```kotlin
// 안티패턴: 액티비티 멤버 변수에 상태 저장 → 회전하면 0으로 초기화
class CounterActivity : AppCompatActivity() {
    private var count = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_counter)
        findViewById<Button>(R.id.plus).setOnClickListener {
            count++                                   // 회전 시 유실
            findViewById<TextView>(R.id.label).text = "$count"
        }
    }
}
```

<br>

### 3. 상태 보존 전략 3가지

### 3-1. savedInstanceState — 소량의 UI 상태

`onSaveInstanceState(outState: Bundle)`에 값을 넣으면 시스템이 보관했다가 `onCreate(savedInstanceState)`와 `onRestoreInstanceState()`로 돌려준다.

```kotlin
class CounterActivity : AppCompatActivity() {
    private var count = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_counter)
        count = savedInstanceState?.getInt(KEY_COUNT) ?: 0   // 복원
        render()
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putInt(KEY_COUNT, count)                    // 저장
    }

    companion object { private const val KEY_COUNT = "count" }
}
```

**범위와 한계**

- **구성 변경**과 **시스템에 의한 프로세스 종료** 양쪽에서 복원된다 — 프로세스가 죽어도 시스템이 Bundle을 대신 들고 있기 때문
- 사용자가 **뒤로 가기·finish()로 직접 종료**하거나 최근 앱에서 스와이프하면 **복원되지 않는다** (새 시작으로 취급)
- 데이터는 **직렬화되어 Binder IPC**로 시스템 프로세스에 전달되므로, 크기가 크면 `TransactionTooLargeException`이 발생한다. 실무 상한은 **수십 KB** 수준이며, 트랜잭션 버퍼 전체 한도가 약 1MB다

> ⚠️ Bundle에 이미지 비트맵이나 수천 개의 리스트 아이템을 넣는 것은 대표적인 실수다. Bundle에는 **ID·스크롤 위치·입력 중인 텍스트**처럼 재조회의 열쇠가 되는 소량 값만 넣고, 큰 데이터는 저장소에서 다시 읽는다.

<br>

### 3-2. ViewModel — 구성 변경을 넘어 살아남는 객체

`ViewModel`은 액티비티·프래그먼트의 `ViewModelStore`에 보관되어 **구성 변경 시 파괴되지 않고 같은 인스턴스가 재사용**된다. 네트워크 응답, 목록 데이터, 화면 로직 상태를 담기에 적합하다.

```kotlin
class CounterViewModel(private val handle: SavedStateHandle) : ViewModel() {
    // 구성 변경 + 프로세스 종료 모두 복원: SavedStateHandle
    val count: StateFlow<Int> = handle.getStateFlow(KEY_COUNT, 0)

    fun increment() { handle[KEY_COUNT] = count.value + 1 }

    companion object { private const val KEY_COUNT = "count" }
}

class CounterActivity : AppCompatActivity() {
    private val viewModel: CounterViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_counter)
        findViewById<Button>(R.id.plus).setOnClickListener { viewModel.increment() }
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.count.collect { findViewById<TextView>(R.id.label).text = "$it" }
            }
        }
    }
}
```

- `ViewModel`은 **메모리에만** 존재하므로 프로세스가 종료되면 사라진다. 그래서 `SavedStateHandle`이 `savedInstanceState` 메커니즘을 ViewModel 안으로 끌어와 **두 경우를 모두** 커버한다
- `onCleared()`는 액티비티가 **완전히 끝날 때(finish)**만 호출되며, 회전으로 재생성될 때는 호출되지 않는다

❗️**ViewModel에 Context·View를 넣지 말 것**: ViewModel은 액티비티보다 오래 살기 때문에 액티비티 Context나 뷰를 참조하면 파괴된 액티비티가 GC되지 못하는 **메모리 누수**가 된다 (unit05 참고). Context가 꼭 필요하면 `AndroidViewModel`의 Application Context를 쓴다.

<br>

### 3-3. 영속 저장소 — 앱을 껐다 켜도 남아야 하는 데이터

사용자 설정, 로그인 토큰, 작성 중인 글의 임시 저장본처럼 **앱 재시작 후에도 필요한 데이터**는 `DataStore`·`Room`·파일에 저장한다 (unit07 참고). 이 계층은 구성 변경과 무관하게 항상 유지된다.

| **전략**                 | **구성 변경** | **시스템의 프로세스 종료** | **사용자가 종료** | **적합한 데이터**                      | **크기**   |
| ------------------------ | ------------- | -------------------------- | ----------------- | -------------------------------------- | ---------- |
| **savedInstanceState**   | 유지          | 유지                       | 유실              | 스크롤 위치, 입력 중 텍스트, 선택 ID   | 수십 KB 이하 |
| **ViewModel**            | 유지          | **유실**                   | 유실              | 네트워크 응답, 목록, 화면 로직 상태    | 메모리 허용 범위 |
| **SavedStateHandle**     | 유지          | 유지                       | 유실              | ViewModel 상태 중 복원이 꼭 필요한 값  | 수십 KB 이하 |
| **영속 저장소**          | 유지          | 유지                       | **유지**          | 설정, 인증 토큰, 임시 저장 초안        | 제한 없음  |

<br>

### 4. 구성 변경을 직접 처리하는 경우

매니페스트에 `android:configChanges="orientation|screenSize|..."`를 선언하면 해당 변경 시 재생성 대신 `onConfigurationChanged()`만 호출된다.

- **장점**: 비디오 플레이어·카메라 프리뷰·WebView처럼 재생성 비용이 매우 큰 화면에서 끊김을 막을 수 있음
- **단점**: 리소스 한정자(가로 레이아웃, 야간 색상 등)가 **자동 적용되지 않으므로** 개발자가 직접 뷰를 갱신해야 하며, 선언하지 않은 구성 변경(언어 변경 등)에서는 여전히 재생성됨
- 상태 보존 문제를 회피하는 수단으로 남용하면 안 되며, 프로세스 종료에는 어차피 대비해야 한다

<br>

### 5. 프로세스 종료 상황 테스트하기

구성 변경은 회전으로 쉽게 재현되지만, 프로세스 종료 후 복원은 놓치기 쉽다. 개발 중에는 다음 방법으로 강제 재현한다.

```bash
adb shell am kill com.example.app   # 홈으로 보낸 뒤 프로세스만 종료 → 재진입 시 savedInstanceState로 복원돼야 함
```

- 개발자 옵션의 **"활동 유지 안함(Don't keep activities)"**을 켜면 화면을 벗어날 때마다 액티비티가 파괴되어 복원 로직을 검증할 수 있다
- 복원 후 `ViewModel`이 비어 있는데 `savedInstanceState`만 남아 있는 상황을 처리하지 않으면 **빈 화면이나 NullPointerException**이 발생한다

> 💡 면접에서 "회전 시 데이터 유지 방법"을 물으면 ViewModel 하나만 답하기보다, **savedInstanceState(소량·프로세스 종료 대비) + ViewModel(대량·구성 변경 대비) + SavedStateHandle(둘의 결합)**의 역할 분담으로 답하는 것이 좋다.

<br>

### 6. 정리 — 면접·실무 체크포인트

| **질문**                                             | **핵심 답변**                                                                          |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **화면 회전 시 액티비티는 어떻게 되는가?**           | 파괴 후 재생성. 리소스 한정자를 다시 적용하기 위한 설계                                |
| **savedInstanceState는 언제 복원되는가?**            | 구성 변경·시스템 프로세스 종료 시. **사용자가 직접 종료하면 복원 안 됨**              |
| **savedInstanceState의 크기 제한은?**                | Binder 트랜잭션 한도(약 1MB) — 실무에선 수십 KB 이하, 큰 데이터는 넣지 않음           |
| **ViewModel은 왜 회전에 살아남는가?**                | `ViewModelStore`에 보관되어 새 액티비티 인스턴스에 재연결되기 때문                     |
| **ViewModel도 못 지키는 상황은?**                    | 프로세스 종료. `SavedStateHandle`로 보완                                               |
| **configChanges를 쓰면 안 되는 이유는?**             | 리소스 자동 적용이 끊기고, 프로세스 종료는 어차피 못 막음 — 특수 화면에서만 제한 사용 |

- 상태는 **수명과 크기**에 따라 `savedInstanceState` → `ViewModel` → 영속 저장소로 계층화한다
- 프로세스가 언제 종료되는지, 그 우선순위는 **unit04**에서 이어진다
