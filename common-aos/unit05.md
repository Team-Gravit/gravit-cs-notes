## 메모리 누수 패턴

**메모리 누수(Memory Leak)**는 더 이상 필요 없는 객체가 어딘가에서 계속 참조되어 **가비지 컬렉터(GC)가 회수하지 못하는 상태**다. 안드로이드에서는 액티비티·프래그먼트가 생명주기에 따라 수시로 파괴되는데, 파괴된 화면이 뷰 트리·비트맵과 함께 메모리에 남으면 곧 OOM(OutOfMemoryError)과 잦은 GC로 인한 버벅임으로 이어진다. 누수는 대부분 **몇 가지 반복되는 패턴**에서 나오므로, 패턴과 진단 흐름을 익히면 대부분 예방할 수 있다.

<br>

### 1. 누수가 생기는 원리

JVM 계열 런타임(ART)의 GC는 **GC 루트(GC Root)**에서 도달 가능한 객체는 살아 있다고 판단한다. 정적 변수, 실행 중인 스레드의 스택, JNI 참조 등이 루트다. 파괴된 액티비티가 루트에서 도달 가능한 **참조 체인** 위에 놓이면 회수되지 않는다.

```
GC 루트                                 누수된 객체
┌──────────────┐    ┌─────────────┐    ┌─────────────────┐    ┌──────────┐
│ static 필드  │──▶│ Singleton   │──▶│ 파괴된 Activity │──▶│ 뷰 트리   │
│ (클래스 로더)│    │ (context 보관)│    │ (onDestroy 완료)│    │ Bitmap …  │
└──────────────┘    └─────────────┘    └─────────────────┘    └──────────┘
        살아 있는 참조 체인 하나만 있어도 액티비티 전체(수 MB)가 남는다
```

- 액티비티 하나가 누수되면 그 액티비티가 붙잡은 **뷰 계층·어댑터·비트맵·리스너**까지 전부 남는다
- 화면 회전마다 액티비티가 재생성되므로 (unit03 참고), 누수가 있으면 **회전할 때마다 액티비티 인스턴스가 하나씩 쌓인다**

> 💡 코틀린·자바에는 C처럼 해제를 잊는 누수는 없다. 안드로이드에서 누수는 항상 "**수명이 긴 객체가 수명이 짧은 객체를 참조**"할 때 생긴다. 이 한 문장으로 아래 모든 패턴을 설명할 수 있다.

<br>

### 2. 대표 누수 패턴

### 2-1. Context를 오래 사는 객체에 저장

`Activity`는 `Context`의 구현체이므로, 액티비티 Context를 싱글턴이나 정적 필드에 넣으면 액티비티 전체가 붙잡힌다.

```kotlin
// 안티패턴: 액티비티 Context를 싱글턴에 저장
object AnalyticsTracker {
    private var context: Context? = null
    fun init(context: Context) { this.context = context }   // Activity가 들어오면 누수
}

// 개선: Application Context만 보관 — 프로세스와 수명이 같아 누수가 아님
object AnalyticsTracker {
    private lateinit var appContext: Context
    fun init(context: Context) { appContext = context.applicationContext }
}
```

| **Context 종류**          | **수명**              | **적합한 용도**                                     | **주의**                                   |
| ------------------------- | --------------------- | --------------------------------------------------- | ------------------------------------------ |
| **Application Context**   | 프로세스 전체         | 싱글턴, 라이브러리 초기화, 리소스 접근, DB·저장소   | 테마 없음 → 다이얼로그·레이아웃 인플레이트 불가 |
| **Activity Context**      | 액티비티 생명주기     | 뷰 생성, 다이얼로그, 화면 전환, 테마가 필요한 작업  | **오래 사는 객체에 저장 금지**             |
| **Service Context**       | 서비스 생명주기       | 서비스 내부 작업                                    | 서비스가 끝나면 무효                       |

<br>

### 2-2. 리스너·콜백·옵저버 미해제

등록은 했는데 해제하지 않으면, 등록을 받은 쪽(시스템 서비스, 이벤트 버스, 싱글턴)이 콜백을 통해 액티비티를 붙잡는다. 콜백이 익명 클래스나 람다라면 **외부 클래스(액티비티)에 대한 암묵적 참조**가 포함된다.

```kotlin
class MapActivity : AppCompatActivity() {
    private val locationManager by lazy { getSystemService(LocationManager::class.java) }
    private val listener = LocationListener { location -> updateMarker(location) }  // this 캡처

    // 안티패턴: onStart에서 등록만 하고 해제 없음 → 시스템 서비스가 액티비티를 계속 참조
    override fun onStart() {
        super.onStart()
        locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1000L, 1f, listener)
    }

    // 개선: 대칭 콜백에서 반드시 해제
    override fun onStop() {
        super.onStop()
        locationManager.removeUpdates(listener)
    }
}
```

**같은 계열의 흔한 사례**

- `SharedPreferences.registerOnSharedPreferenceChangeListener()` 후 `unregister` 누락
- `BroadcastReceiver`를 코드로 `registerReceiver()`한 뒤 `unregisterReceiver()` 누락
- RxJava `Disposable`을 `dispose()`하지 않음, `LiveData.observeForever()` 후 `removeObserver()` 누락
- 프래그먼트에서 `viewLifecycleOwner` 대신 `this`로 옵저버 등록 (unit02 참고)

<br>

### 2-3. 내부 클래스·핸들러·스레드

코틀린의 `inner class`, 자바의 비정적 내부 클래스와 익명 클래스는 **외부 인스턴스 참조를 자동으로 가진다**. 이 객체가 액티비티보다 오래 살면 누수가 된다.

```kotlin
class SplashActivity : AppCompatActivity() {
    // 안티패턴: 메인 루퍼 Handler에 지연 메시지 → 메시지 큐가 Runnable(→ this)을 붙잡음
    private val handler = Handler(Looper.getMainLooper())
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handler.postDelayed({ startActivity(Intent(this, MainActivity::class.java)) }, 60_000L)
    }

    // 개선: 파괴 시 큐에 남은 콜백 제거 (또는 lifecycleScope + delay 사용)
    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacksAndMessages(null)
    }
}
```

- **Handler 지연 메시지**: `MessageQueue`는 메인 루퍼(GC 루트)에 매달려 있어 메시지 실행 전까지 Runnable과 그 외부 참조가 살아 있음
- **스레드·`AsyncTask`·코루틴**: 실행 중인 스레드는 GC 루트다. 액티비티가 끝나도 네트워크 응답을 기다리는 스레드가 액티비티를 참조하면 응답이 올 때까지 누수. `lifecycleScope`·`viewModelScope`처럼 **생명주기에 묶인 스코프**를 쓰면 자동 취소된다
- **ViewModel이 View·Activity 참조**: ViewModel은 구성 변경 후에도 살아남으므로 (unit03 참고) 이전 액티비티를 붙잡는다

<br>

### 2-4. 뷰 참조·정적 뷰·바인딩

- 정적 필드에 `View`나 `Drawable`을 저장하면, 뷰는 자신의 Context(액티비티)를 참조하므로 액티비티가 누수된다
- 프래그먼트의 뷰 바인딩을 `onDestroyView()`에서 null 처리하지 않으면 파괴된 뷰 트리가 프래그먼트 인스턴스에 남는다 (unit02 참고)
- `RecyclerView.Adapter`가 액티비티를 직접 참조하면서 어댑터가 ViewModel이나 싱글턴에 보관되는 경우도 같은 문제다

> ⚠️ `WeakReference`로 감싸면 누수는 막을 수 있지만, 참조가 언제 사라질지 예측하기 어렵고 코드가 복잡해진다. 근본 해법은 **수명이 맞는 스코프에 객체를 두는 것**이며, `WeakReference`는 콜백을 받는 쪽을 직접 고칠 수 없는 경우의 차선책이다.

<br>

### 3. 누수 진단 흐름

```
[1] 의심 징후 관찰
    잦은 GC 로그, 회전 반복 시 메모리 우상향, OOM 크래시 리포트
          │
          ▼
[2] LeakCanary (디버그 빌드 자동 감시)
    파괴된 Activity·Fragment·View·ViewModel을 WeakReference로 추적
    → GC 후에도 남아 있으면 힙 덤프 → 참조 체인(Leak Trace) 자동 출력
          │
          ▼
[3] Android Studio Memory Profiler
    힙 덤프(.hprof) 캡처 → 클래스별 인스턴스 수·Retained Size 확인
    → 파괴됐어야 할 Activity 인스턴스가 2개 이상이면 누수 확정
          │
          ▼
[4] 참조 체인 역추적
    "GC 루트 → … → 누수 객체" 경로에서 끊어야 할 참조(등록 해제·null 처리) 결정
          │
          ▼
[5] 수정 후 재검증
    회전·화면 진입/이탈 반복 → 강제 GC → 인스턴스 수가 1로 유지되는지 확인
```

**Memory Profiler에서 보는 핵심 지표**

| **지표**                | **의미**                                                  | **누수 판단 기준**                                |
| ----------------------- | --------------------------------------------------------- | ------------------------------------------------- |
| **Allocations**         | 해당 클래스의 인스턴스 개수                               | 액티비티 클래스가 화면 수보다 많으면 의심         |
| **Shallow Size**        | 객체 자체가 차지하는 크기                                 | 작아도 Retained가 크면 문제                       |
| **Retained Size**       | 이 객체가 사라지면 함께 회수될 메모리 총량                | 파괴된 액티비티의 Retained가 수 MB면 뷰 트리 누수 |
| **Dominator / 참조 경로** | 객체를 살아 있게 만드는 참조 체인                       | 체인의 첫 앱 코드 지점이 수정 대상                |

```bash
adb shell dumpsys meminfo com.example.app   # Activities·Views 개수를 빠르게 확인 (회전 후 Activities가 증가하면 의심)
```

> 💡 LeakCanary는 `debugImplementation`으로만 추가해 릴리스 빌드에 포함되지 않게 한다. 누수 트레이스에서 `━━━` 밑줄로 표시된 부분이 "**여기서 참조가 끊겨야 한다**"고 알려주는 지점이며, 대부분 앱 코드의 static 필드·등록 해제 누락·내부 클래스 셋 중 하나다.

<br>

### 4. 예방 체크리스트

- 싱글턴·정적 필드·ViewModel에는 **Application Context만** 넣는다
- 등록(`register`·`add`·`observe`·`subscribe`)에는 반드시 **대칭 콜백에서 해제**를 짝지어 둔다
- 비동기 작업은 `lifecycleScope`·`viewModelScope`처럼 **자동 취소되는 스코프**에서 실행한다
- 프래그먼트는 `viewLifecycleOwner`를 쓰고 바인딩은 `onDestroyView()`에서 null 처리한다
- 내부 클래스가 액티비티보다 오래 살아야 한다면 정적(중첩) 클래스 + 필요한 값만 전달한다
- 디버그 빌드에 **LeakCanary**를 상시 적용하고, 화면 회전 10회 테스트를 QA 루틴에 넣는다

<br>

### 5. 정리 — 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                          |
| ------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **안드로이드에서 메모리 누수가 생기는 원리는?**   | GC 루트에서 도달 가능한 참조 체인 위에 **파괴된 액티비티**가 놓임                       |
| **Activity Context와 Application Context 차이는?** | 수명. 오래 사는 객체에는 **Application Context**, UI 작업에는 Activity Context        |
| **대표 누수 패턴 3가지는?**                       | Context 정적 보관, **리스너·콜백 미해제**, 내부 클래스·Handler·스레드의 암묵적 참조    |
| **누수를 어떻게 찾는가?**                         | LeakCanary로 자동 감지 → Memory Profiler 힙 덤프에서 **Retained Size·참조 경로** 분석 |
| **ViewModel에 Activity를 넣으면?**                | ViewModel이 구성 변경을 넘어 살아 이전 액티비티를 붙잡음 → 누수                        |

- 누수의 본질은 "**긴 수명 객체 → 짧은 수명 객체** 참조"이며, 해법은 수명이 맞는 스코프에 두거나 대칭 해제하는 것이다
- 생명주기 콜백 쌍은 **unit02**, ViewModel의 수명은 **unit03**, 백그라운드 스레드 관리 대안은 **unit06**을 참고할 것
