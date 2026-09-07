## 액티비티와 프래그먼트 생명주기

액티비티와 프래그먼트는 시스템이 생성·정지·파괴를 결정하는 컴포넌트이므로, 개발자는 **생명주기 콜백(Lifecycle Callback)**에 맞춰 자원을 할당하고 해제해야 한다. 콜백 호출 순서, 특히 **화면 전환 시 두 화면의 콜백이 교차하는 순서**를 정확히 알아야 리소스 누수·중복 초기화·상태 유실 같은 버그를 막을 수 있다.

<br>

### 1. 생명주기가 필요한 이유

- 모바일 환경에서는 전화 수신, 홈 버튼, 화면 회전, 메모리 부족 등으로 앱이 **언제든 중단**될 수 있다
- 시스템은 액티비티를 직접 파괴·재생성하면서 그 사실을 콜백으로 알려주고, 개발자는 각 시점에 맞게 **자원 확보와 해제, 상태 저장**을 수행한다
- 콜백 시점을 잘못 고르면 화면이 보이지 않는데 GPS가 계속 돌거나(배터리 낭비), 복귀했는데 데이터가 갱신되지 않는(정합성) 문제가 생긴다

<br>

### 2. 액티비티 생명주기

### 2-1. 기본 콜백 7개

```
            onCreate() ──▶ onStart() ──▶ onResume()
               ▲              ▲              │
               │              │         [화면 활성: 사용자와 상호작용]
               │              │              │
               │        onRestart()      onPause()
               │              ▲              │
               │              │         [부분 가림 / 다른 화면 위로]
               │              │              │
               │              └──────── onStop()
               │                             │
               │                        [완전히 가려짐]
               │                             │
        (프로세스 종료 후 재생성)          onDestroy()
```

| **콜백**          | **호출 시점**                           | **적합한 작업**                                  |
| ----------------- | --------------------------------------- | ------------------------------------------------ |
| **onCreate()**    | 액티비티 최초 생성 (1회)                | 뷰 바인딩, ViewModel 연결, 저장 상태 복원        |
| **onStart()**     | 화면이 사용자에게 **보이기 시작**       | UI 갱신용 리스너 등록                            |
| **onResume()**    | 화면이 **포커스를 얻어 상호작용 가능**  | 카메라·센서·애니메이션 재개                      |
| **onPause()**     | 포커스를 잃음 (다이얼로그, 다른 화면)   | 카메라·센서 해제, 가벼운 저장 — **매우 짧게**    |
| **onStop()**      | 화면이 **완전히 보이지 않음**           | 무거운 자원 해제, DB 저장, 리스너 해제           |
| **onRestart()**   | onStop 이후 다시 보이기 직전            | onStart 직전에 필요한 재초기화                   |
| **onDestroy()**   | 액티비티 파괴 직전                      | 남은 자원 정리 (`isFinishing`으로 원인 구분)     |

- **가시 수명(Visible Lifetime)**: `onStart()` ~ `onStop()` 사이 — 화면이 보이는 구간
- **포그라운드 수명(Foreground Lifetime)**: `onResume()` ~ `onPause()` 사이 — 상호작용 가능한 구간

> ⚠️ `onPause()`가 끝나야 다음 액티비티의 `onResume()`이 호출된다. `onPause()`에서 DB 저장이나 네트워크 호출 같은 무거운 작업을 하면 **화면 전환이 눈에 띄게 느려진다**. 무거운 정리는 `onStop()`으로 미룬다.

<br>

### 2-2. onDestroy()는 호출을 보장하지 않는다

- 메모리 부족으로 **프로세스가 통째로 종료**되면 `onDestroy()`는 호출되지 않는다 (unit04 참고)
- 따라서 반드시 저장해야 할 데이터는 `onStop()` 이전에 처리하고, 상태 복원은 `onSaveInstanceState()`로 대비한다 (unit03 참고)
- `onSaveInstanceState()`는 API 28 이상에서 `onStop()` **이후**에, 그 이전 버전에서는 `onStop()` **이전**에 호출된다 (버전에 따라 순서가 다름)

<br>

### 3. 화면 전환 시 두 액티비티의 생명주기 교차

액티비티 A에서 B를 시작할 때, 두 액티비티의 콜백은 **번갈아** 호출된다. 이 순서는 면접에서 가장 자주 묻는 질문이며, 정답은 다음과 같다.

```
A → B 전환 (startActivity)
 A.onPause()
 B.onCreate()
 B.onStart()
 B.onResume()          ◀ 이 시점부터 B와 상호작용 가능
 A.onStop()            ◀ B가 화면을 완전히 덮은 뒤에야 A가 정지
 (A.onSaveInstanceState() — API 28+에서는 onStop 이후)

B에서 뒤로 가기 (finish)
 B.onPause()
 A.onRestart()
 A.onStart()
 A.onResume()          ◀ A 복귀 완료
 B.onStop()
 B.onDestroy()
```

핵심은 두 가지다.

- **A.onStop()은 B.onResume() 다음**에 호출된다 → A의 `onStop()`에서 해제한 자원을 B가 필요로 하면 문제없지만, A의 `onPause()`에서 해제하면 B가 화면에 뜨기 전에 사라진다
- B가 **투명하거나 다이얼로그 테마**라면 A는 `onPause()`까지만 가고 `onStop()`은 호출되지 않는다 (여전히 부분적으로 보이기 때문)

```kotlin
class PlayerActivity : AppCompatActivity() {
    private var player: ExoPlayer? = null

    // 안티패턴: onCreate에서 만들고 onDestroy에서만 해제
    // → 홈 버튼으로 나가도 재생기가 살아 있어 배터리·메모리 낭비

    // 개선: 가시 수명(onStart~onStop)에 맞춰 대칭적으로 할당·해제
    override fun onStart() {
        super.onStart()
        player = ExoPlayer.Builder(this).build()
    }

    override fun onStop() {
        super.onStop()
        player?.release()
        player = null
    }
}
```

> 💡 자원은 **대칭 콜백 쌍**에서 할당·해제한다. `onCreate ↔ onDestroy`, `onStart ↔ onStop`, `onResume ↔ onPause`. 짝이 어긋나면 화면이 여러 번 열리고 닫힐 때 중복 등록이나 누수가 생긴다 (unit05 참고).

<br>

### 4. 프래그먼트 생명주기

프래그먼트(Fragment)는 액티비티 안에서 **재사용 가능한 UI 조각**이며, 호스트 액티비티의 생명주기에 **종속**된다. 액티비티보다 콜백이 많은 이유는 **프래그먼트 자체의 수명과 뷰(View)의 수명이 분리**되어 있기 때문이다.

```
onAttach()          ─ 액티비티에 연결, context 사용 가능
onCreate()          ─ 프래그먼트 자체 초기화 (뷰 없음)
onCreateView()      ─ 레이아웃 인플레이트 → View 반환
onViewCreated()     ─ 뷰 바인딩·리스너·옵저버 등록 (viewLifecycleOwner 사용)
onViewStateRestored()
onStart() / onResume()
        ⋮
onPause() / onStop()
onSaveInstanceState()
onDestroyView()     ─ 뷰 파괴 (프래그먼트 인스턴스는 살아 있음!)
onDestroy()         ─ 프래그먼트 파괴
onDetach()          ─ 액티비티와 연결 해제
```

<br>

### 4-1. 뷰 수명과 프래그먼트 수명의 분리

백 스택에 들어간 프래그먼트는 `onDestroyView()`까지만 호출되고 `onDestroy()`는 호출되지 않는다. 뒤로 가기로 돌아오면 `onCreateView()`부터 **뷰만 다시 생성**된다. 이 특성 때문에 두 가지 함정이 생긴다.

```kotlin
class ListFragment : Fragment(R.layout.fragment_list) {
    private var _binding: FragmentListBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        _binding = FragmentListBinding.bind(view)

        // 안티패턴: this(프래그먼트)를 LifecycleOwner로 사용
        // → 뷰가 파괴돼도 옵저버가 남아 있어 복귀 시 옵저버가 두 개가 됨
        // viewModel.items.observe(this) { ... }

        // 개선: 뷰 수명에 맞는 viewLifecycleOwner 사용
        viewModel.items.observe(viewLifecycleOwner) { items ->
            binding.recyclerView.adapter = ItemAdapter(items)
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null   // 뷰 참조를 끊어 메모리 누수 방지
    }
}
```

- **옵저버 중복**: `this`로 관찰하면 프래그먼트가 살아 있는 동안 옵저버가 계속 누적된다 → `viewLifecycleOwner`로 뷰 수명에 맞춰 자동 해제
- **바인딩 누수**: 뷰 바인딩 객체를 `onDestroyView()`에서 null로 만들지 않으면 파괴된 뷰 트리 전체가 프래그먼트에 붙잡혀 남는다

<br>

### 4-2. 액티비티와 프래그먼트 콜백의 대응

| **액티비티**      | **프래그먼트**                                     | **비고**                                    |
| ----------------- | -------------------------------------------------- | ------------------------------------------- |
| **onCreate()**    | onAttach → onCreate → onCreateView → onViewCreated | 액티비티 onCreate 중 프래그먼트가 생성됨    |
| **onStart()**     | onStart()                                          | 호스트가 먼저 START 상태가 됨              |
| **onResume()**    | onResume()                                         | 호스트가 먼저 RESUMED가 됨                  |
| **onPause()**     | onPause()                                          | **프래그먼트가 먼저** 호출됨               |
| **onStop()**      | onStop()                                           | 프래그먼트가 먼저 호출됨                    |
| **onDestroy()**   | onDestroyView → onDestroy → onDetach               | 프래그먼트가 먼저 파괴됨                    |

- 올라갈 때(생성·시작)는 **액티비티가 먼저**, 내려갈 때(정지·파괴)는 **프래그먼트가 먼저** 상태를 바꾼다
- 프래그먼트 간 전환은 `FragmentTransaction`(replace/add + addToBackStack)으로 이루어지며, `replace`는 기존 프래그먼트의 `onDestroyView()`를 호출하지만 백 스택에 있으면 인스턴스는 유지된다

> 💡 Jetpack의 `Lifecycle`·`LifecycleOwner`·`LifecycleObserver`를 쓰면 콜백을 직접 오버라이드하지 않고도 컴포넌트가 생명주기를 스스로 관찰한다. `repeatOnLifecycle(Lifecycle.State.STARTED)`로 코루틴 수집 범위를 가시 수명에 맞추는 것이 현재 권장 패턴이다.

<br>

### 5. 정리 — 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                 |
| ----------------------------------------------------- | ----------------------------------------------------------------------------- |
| **A에서 B로 전환 시 콜백 순서는?**                    | A.onPause → B.onCreate → B.onStart → B.onResume → **A.onStop**               |
| **onPause와 onStop의 차이는?**                        | onPause는 **포커스 상실**(부분 가림), onStop은 **완전히 안 보임**            |
| **onDestroy에서 저장하면 안 되는 이유는?**            | 프로세스 강제 종료 시 **호출이 보장되지 않음**                                |
| **프래그먼트에 onDestroyView가 따로 있는 이유는?**    | 백 스택에서 **뷰만 파괴되고 인스턴스는 유지**되기 때문                        |
| **viewLifecycleOwner를 쓰는 이유는?**                 | 뷰 수명에 맞춰 옵저버가 해제되어 **중복 옵저버·누수** 방지                    |
| **투명 액티비티를 띄우면?**                           | 아래 액티비티는 onPause까지만 호출되고 onStop은 호출되지 않음                 |

- 자원은 **대칭 콜백 쌍**에서 할당·해제하고, `onPause()`는 최대한 가볍게 유지한다
- 구성 변경(화면 회전)으로 인한 재생성과 상태 보존은 **unit03**, 프로세스 종료 우선순위는 **unit04**에서 다룬다
