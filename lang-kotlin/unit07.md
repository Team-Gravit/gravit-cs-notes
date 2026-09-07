## 위임과 프로퍼티

**위임(Delegation)**은 "상속 대신 합성"을 언어 문법(`by`)으로 지원하는 Kotlin의 기능으로, 클래스 수준에서는 인터페이스 구현을 다른 객체에 넘기고 프로퍼티 수준에서는 getter/setter 로직을 별도 객체에 맡긴다. 이 문서는 프로퍼티의 기본 구조, `by` 위임의 두 형태, `lazy`의 동작 원리, 그리고 커스텀 위임 프로퍼티를 만드는 방법을 다룬다.

<br>

### 1. Kotlin 프로퍼티의 구조

- Kotlin의 프로퍼티는 자바의 "필드 + getter/setter"를 하나의 개념으로 묶은 것으로, `val`은 getter만, `var`는 getter와 setter를 자동 생성함
- 접근자를 직접 정의할 수 있으며, 접근자 안에서 `field`는 **backing field(실제 저장 공간)**를 가리킴. 접근자가 `field`를 쓰지 않으면 backing field는 생성되지 않음
- setter의 가시성만 따로 제한(`private set`)해 **외부에서는 읽기만, 내부에서는 쓰기**를 허용하는 패턴이 흔함

```kotlin
class Account {
    var balance: Long = 0
        private set                       // 외부에서는 읽기 전용

    val isEmpty: Boolean                  // backing field 없음 — 매번 계산
        get() = balance == 0L

    var nickname: String = ""
        set(value) {
            require(value.isNotBlank()) { "닉네임은 비어 있을 수 없음" }
            field = value.trim()          // field = backing field
        }
}
```

> 💡 자바에서 보면 `balance`는 `getBalance()`만 public이고 `setBalance()`는 private이다. Kotlin 프로퍼티는 결국 **JVM 메서드 쌍**으로 컴파일된다는 사실을 기억하면 위임 프로퍼티의 동작도 쉽게 이해된다.

<br>

### 2. 클래스 위임 — 인터페이스 구현을 넘기기

`class A(b: B) : I by b`는 "인터페이스 `I`의 모든 멤버 호출을 `b`에게 전달하라"는 뜻이다. 데코레이터·래퍼를 만들 때 **위임 메서드를 일일이 작성할 필요가 없어** 보일러플레이트가 사라진다.

```kotlin
// 안티패턴: 상속으로 기능 추가 → 부모 구현 세부(add가 addAll을 호출하는지 등)에 종속됨
class CountingSet<T> : HashSet<T>() {
    var added = 0
    override fun add(element: T): Boolean { added++; return super.add(element) }
    override fun addAll(elements: Collection<T>): Boolean {
        added += elements.size                       // HashSet.addAll이 내부적으로 add를 호출하면 이중 계산
        return super.addAll(elements)
    }
}

// 개선: 합성 + 위임. 관심 있는 메서드만 오버라이드하고 나머지는 자동 전달
class CountingSet<T>(private val inner: MutableSet<T> = HashSet()) : MutableSet<T> by inner {
    var added = 0
    override fun add(element: T): Boolean { added++; return inner.add(element) }
    override fun addAll(elements: Collection<T>): Boolean {
        added += elements.size
        return inner.addAll(elements)                // inner의 addAll이 inner.add를 불러도 이쪽 add는 안 거침
    }
}
```

- 오버라이드한 멤버는 직접 구현이 쓰이고, 나머지는 컴파일러가 생성한 전달 메서드가 `inner`를 호출함
- 위임 객체는 래퍼의 존재를 모르므로, **위임 객체 내부에서 자기 메서드를 호출해도 래퍼의 오버라이드를 거치지 않음**. 상속의 "취약한 기반 클래스" 문제가 사라지는 이유이자, 자기 호출을 가로채고 싶다면 위임이 답이 아닌 이유임

<br>

### 3. 프로퍼티 위임 — 접근자 로직을 객체에 맡기기

`val x: T by delegate`는 "`x`를 읽을 때 `delegate.getValue()`를, 쓸 때 `delegate.setValue()`를 호출하라"는 뜻이다. 컴파일러는 위임 객체를 숨겨진 필드에 저장하고 접근자가 그 객체에 전달하도록 코드를 만든다.

```
소스:   class A { val x: Int by Delegate() }

컴파일: class A {
          private val x$delegate = Delegate()
          val x: Int
            get() = x$delegate.getValue(this, this::x)     // thisRef, KProperty
        }
```

- `getValue(thisRef, property)` / `setValue(thisRef, property, value)`는 **`operator` 함수**이며, 시그니처만 맞으면 어떤 클래스든 위임 객체가 될 수 있음
- `thisRef`는 프로퍼티를 가진 객체(최상위 프로퍼티면 null), `property`는 `KProperty<*>`로 프로퍼티 이름·타입 등 메타데이터를 제공함

<br>

### 4. 표준 라이브러리의 위임

| **위임**                       | **대상** | **동작**                                                  | **대표 용도**                          |
| ------------------------------ | -------- | --------------------------------------------------------- | -------------------------------------- |
| **lazy { }**                   | `val`    | **최초 접근 시 한 번만** 계산하고 캐시                    | 비용 큰 초기화 지연, 순환 의존 회피   |
| **Delegates.observable(init)** | `var`    | 값이 바뀔 때마다 **콜백**(old, new) 호출                   | 변경 감지, 로깅, UI 갱신              |
| **Delegates.vetoable(init)**   | `var`    | 콜백이 `false`를 반환하면 **변경을 거부**                  | 유효성 검증                            |
| **Delegates.notNull()**        | `var`    | 초기화 전 접근 시 예외. 기본 타입에도 사용 가능           | `lateinit`을 쓸 수 없는 `Int` 등       |
| **Map / MutableMap**           | 둘 다    | 프로퍼티 이름을 **키**로 맵에서 읽고 씀                    | JSON·설정 파싱 결과를 객체처럼 접근   |
| **다른 프로퍼티 (`::prop`)**   | 둘 다    | 다른 프로퍼티로 접근을 **전달**                            | 이름 변경 시 하위 호환(`@Deprecated`)  |

```kotlin
class Settings(map: Map<String, Any?>) {
    val theme: String by map                         // map["theme"]
    val fontSize: Int by map                         // map["fontSize"]
}

class Form {
    var email: String by Delegates.vetoable("") { _, _, new -> "@" in new }
    var name: String by Delegates.observable("") { prop, old, new ->
        println("${prop.name}: $old → $new")
    }
}
```

<br>

### 5. lazy의 동작 원리와 스레드 안전 모드

`lazy`는 `Lazy<T>` 객체를 반환하고, `Lazy<T>`에는 `getValue` 확장 함수가 있어 위임 대상이 된다. 값은 최초 `value` 접근 시 람다를 실행해 계산되고 이후에는 캐시된 값을 돌려준다.

```
첫 접근:  getValue() → 초기화됨? 아니오 → 람다 실행 → 결과 저장 → 반환
이후 접근: getValue() → 초기화됨? 예   → 저장된 값 반환 (람다는 다시 실행되지 않음)
```

| **모드 (`LazyThreadSafetyMode`)** | **동기화**                                           | **여러 스레드가 동시에 첫 접근하면**                 | **선택 기준**                           |
| --------------------------------- | ---------------------------------------------------- | ---------------------------------------------------- | --------------------------------------- |
| **SYNCHRONIZED** (기본값)         | 락으로 보호 (이중 검사)                              | 한 스레드만 계산, 나머지는 대기 후 같은 값 사용      | 초기화가 **정확히 한 번**이어야 할 때   |
| **PUBLICATION**                   | 락 없음, 결과 저장만 원자적                          | 여러 스레드가 각자 계산할 수 있으나 **첫 결과만 채택** | 초기화가 부수 효과 없고 락 비용을 피하고 싶을 때 |
| **NONE**                          | 없음                                                 | 정의되지 않은 동작 (여러 번 계산·불일치 가능)        | **단일 스레드**임이 확실할 때 (안드로이드 UI 등) |

```kotlin
class ReportService(private val repo: ReportRepository) {
    // 생성 시점에는 만들지 않고, 실제로 필요할 때 한 번만 생성 (기본 SYNCHRONIZED)
    private val template: Template by lazy { Template.compile(repo.loadTemplateSource()) }

    // 메인 스레드에서만 접근하는 것이 확실하면 락 비용 제거
    private val cache by lazy(LazyThreadSafetyMode.NONE) { HashMap<String, Report>() }
}
```

> ⚠️ `lazy` 람다가 예외를 던지면 값이 저장되지 않아 **다음 접근 때 람다가 다시 실행**된다. 초기화가 실패할 수 있는 외부 호출을 `lazy` 안에 두면, 실패가 매 접근마다 반복되며 원인 파악이 어려워질 수 있다. 또한 `lateinit`과 달리 `lazy`는 `val` 전용이고 초기화 로직을 프로퍼티가 스스로 가진다는 점이 선택 기준이다 (unit01 참고).

<br>

### 6. 커스텀 위임 프로퍼티 만들기

### 6-1. ReadOnlyProperty · ReadWriteProperty 인터페이스

표준 라이브러리는 `getValue`/`setValue` 시그니처를 강제해 주는 **함수형 인터페이스**를 제공한다. 이를 구현하면 시그니처 실수를 컴파일러가 잡아 준다.

```kotlin
// 값이 바뀔 때만 저장하고 변경 이력을 남기는 위임
class Audited<T>(private var value: T, private val log: MutableList<String>) : ReadWriteProperty<Any?, T> {
    override fun getValue(thisRef: Any?, property: KProperty<*>): T = value

    override fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        if (value != newValue) {
            log += "${property.name}: $value → $newValue"
            value = newValue
        }
    }
}

class Profile(log: MutableList<String>) {
    var email: String by Audited("", log)
    var age: Int by Audited(0, log)
}
```

- 람다 하나로 충분한 읽기 전용 위임은 `ReadOnlyProperty { thisRef, prop -> ... }`처럼 SAM 변환으로 바로 만들 수 있음
- `property.name`을 활용하면 SharedPreferences·환경 변수·DB 컬럼처럼 **이름 기반 저장소**를 프로퍼티로 매핑하는 위임을 쉽게 만들 수 있음

<br>

### 6-2. provideDelegate — 위임 생성 시점에 개입하기

- `operator fun provideDelegate(thisRef, property)`를 정의하면 프로퍼티가 **초기화될 때 한 번** 호출되어, 실제 위임 객체를 만들기 전에 프로퍼티 이름 검증이나 등록 같은 작업을 수행할 수 있음
- 예: 프로퍼티 이름이 설정 키 규칙에 맞는지 검사하고, 맞으면 위임 객체를 돌려주고 아니면 예외를 던짐

```kotlin
class Config(private val source: Map<String, String>) {
    val dbUrl: String by required()            // 키 "dbUrl"이 없으면 객체 생성 시점에 실패

    private fun required() = PropertyDelegateProvider { _: Config, prop ->
        val v = source[prop.name] ?: error("설정 키 없음: ${prop.name}")
        ReadOnlyProperty<Config, String> { _, _ -> v }
    }
}
```

> 💡 "위임 프로퍼티가 어떻게 동작하는가?"라는 질문에는 "컴파일러가 **`$delegate` 필드**를 만들고 접근자가 `getValue`/`setValue` operator 함수를 호출하도록 코드를 생성한다"고 답하면 된다. `lazy`도 특별한 문법이 아니라 이 규약을 따르는 `Lazy` 객체일 뿐이다.

<br>

### 7. 정리 — 면접·실무 체크포인트

| **질문**                                    | **핵심 답변**                                                                      |
| ------------------------------------------- | ---------------------------------------------------------------------------------- |
| 클래스 위임의 장점은?                       | 상속 없이 인터페이스 구현을 다른 객체에 전달. **전달 메서드를 컴파일러가 생성**하고 기반 클래스 세부에 종속되지 않음 |
| 위임 객체의 자기 호출이 래퍼를 거치는가?    | **거치지 않음**. 위임 객체는 래퍼를 모른다                                          |
| 프로퍼티 위임은 어떻게 컴파일되는가?        | `$delegate` 필드 + 접근자가 `getValue`/`setValue` operator 함수를 호출              |
| `lazy`의 기본 스레드 안전 모드는?           | **SYNCHRONIZED**. 단일 스레드면 `NONE`으로 락 비용 제거 가능                        |
| `lazy` 람다에서 예외가 나면?                | 값이 저장되지 않아 **다음 접근 때 재실행**됨                                        |
| `lateinit`과 `lazy`의 차이는?               | `lateinit`은 `var`·외부 초기화·기본 타입 불가, `lazy`는 `val`·자기 초기화           |
| 커스텀 위임을 만들려면?                     | `ReadOnlyProperty`/`ReadWriteProperty` 구현 또는 `getValue`/`setValue` operator 정의 |

- 위임은 **합성을 문법으로 지원**하는 장치이며, 프로퍼티 위임은 접근자 로직의 재사용 수단임
- `val`의 의미와 읽기 전용이 불변과 다른 이유는 **unit08(가시성과 불변성)**을 참고할 것
