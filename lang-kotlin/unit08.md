## 가시성과 불변성

**가시성(Visibility)**은 "누가 볼 수 있는가"를, **불변성(Immutability)**은 "누가 바꿀 수 있는가"를 통제하는 장치로, 둘을 합쳐 캡슐화를 이룬다. 이 문서는 Kotlin 가시성 변경자 네 가지가 자바와 어떻게 다른지, `val`이 왜 불변이 아니라 읽기 전용인지, 그리고 `List`가 불변 컬렉션이 아닌 이유와 진짜 불변을 얻는 방법을 다룬다.

<br>

### 1. 가시성 변경자 네 가지

Kotlin의 기본 가시성은 **`public`**이며, 자바의 package-private에 해당하는 것이 없는 대신 **모듈 단위의 `internal`**이 있다.

| **변경자**     | **클래스 멤버**                         | **최상위 선언 (파일 레벨)**          | **자바와의 차이**                                        |
| -------------- | --------------------------------------- | ------------------------------------ | -------------------------------------------------------- |
| **public**     | 어디서나 (기본값)                       | 어디서나 (기본값)                    | 자바는 기본이 package-private, Kotlin은 **public**       |
| **internal**   | **같은 모듈** 안에서만                  | 같은 모듈 안에서만                   | 자바에 없음. 바이트코드상 public이지만 **이름이 변형**됨 |
| **protected**  | 자신과 **하위 클래스**만                | 사용 불가                            | 자바와 달리 **같은 패키지에서는 접근 불가**              |
| **private**    | 클래스 안에서만                         | **같은 파일** 안에서만               | 최상위 private은 자바에 없는 개념                        |

- **모듈**이란 함께 컴파일되는 단위(Gradle 소스 세트, IntelliJ 모듈 등)이며, `internal`은 "라이브러리 내부 구현이니 외부 모듈은 쓰지 말라"는 의도를 표현함
- `internal` 멤버는 JVM에서 public이지만 함수 이름이 `name$moduleName` 형태로 **맹글링(mangling)**되어 자바에서의 우발적 호출을 막음. 즉 접근 제어라기보다 **의도 표시**에 가까움 (unit09 참고)
- `protected` 멤버는 Kotlin에서 확장 함수로도 접근할 수 없음 (확장 함수는 정적 메서드이므로, unit05 참고)

```kotlin
// UserService.kt
private fun normalize(email: String) = email.trim().lowercase()   // 이 파일 안에서만

internal class UserCache            // 같은 모듈만. 다른 모듈에서 import하면 컴파일 에러

open class Base {
    protected val secret = 42       // 하위 클래스만 (같은 패키지라도 외부 클래스는 불가)
    private val hidden = 0          // 이 클래스 안에서만
}
```

> 💡 라이브러리를 만들 때는 컴파일러 옵션 `-Xexplicit-api=strict`(명시적 API 모드)를 켜면 **가시성을 생략한 public 선언이 에러**가 되어, 실수로 내부 구현이 공개 API가 되는 것을 막을 수 있다.

<br>

### 2. val과 var — 읽기 전용과 가변

- `val`은 **재대입이 불가능한 참조**(자바의 `final`)이고, `var`는 재대입 가능한 참조임
- `val`은 "값이 절대 변하지 않는다"가 아니라 "**이 이름으로는 다시 대입할 수 없다**"는 뜻임. 참조가 가리키는 객체 내부는 얼마든지 바뀔 수 있음
- 커스텀 getter를 가진 `val`은 접근할 때마다 **다른 값을 돌려줄 수도 있음**

```kotlin
val list = mutableListOf(1, 2)
list.add(3)              // OK — 참조는 그대로, 객체 내부만 변경
// list = mutableListOf() // 컴파일 에러 — 재대입 불가

class Clock {
    val now: Long            // val이지만 호출할 때마다 값이 다름
        get() = System.currentTimeMillis()
}
```

```
val ref ──▶ [MutableList 객체: 1, 2, 3]
 │            └── 내부 변경 가능 (add, remove …)
 └── 재대입만 금지 (ref = 다른 객체  ✗)
```

| **구분**                  | **val**                     | **var**                   | **불변 객체**                     |
| ------------------------- | --------------------------- | ------------------------- | --------------------------------- |
| **재대입**                | 불가                        | 가능                      | 참조와 무관                       |
| **객체 내부 변경**        | **가능** (객체가 가변이면)  | 가능                      | **불가** (모든 필드가 val + 불변 타입) |
| **스레드 간 안전한 공유** | 객체가 불변일 때만          | 동기화 필요               | **안전**                          |

> ⚠️ "`val`을 썼으니 불변이다"는 가장 흔한 오해다. 진짜 불변은 **참조가 `val`이고, 타입이 불변이며, 그 안의 모든 프로퍼티도 재귀적으로 불변**일 때만 성립한다. `val items: MutableList<Item>`은 불변과 거리가 멀다.

<br>

### 3. 읽기 전용 컬렉션이 불변이 아닌 이유

### 3-1. 인터페이스만 다르고 객체는 같다

Kotlin의 `List`·`Set`·`Map`은 **변경 메서드가 없는 인터페이스**이고, `MutableList` 등은 이를 상속해 변경 메서드를 추가한 인터페이스다. 런타임 객체는 대부분 `java.util.ArrayList` 같은 **가변 자바 컬렉션 그대로**이며, Kotlin은 그 객체를 어느 인터페이스로 보느냐만 제한한다.

```
   List<E>  (읽기 전용 인터페이스: get, size, contains …)
      ▲
      │ 상속
 MutableList<E>  (add, remove, set … 추가)
      ▲
      │ 구현
 java.util.ArrayList  ← 실제 객체. List로 보든 MutableList로 보든 같은 객체
```

```kotlin
val mutable = mutableListOf(1, 2, 3)
val readOnly: List<Int> = mutable          // 같은 객체를 List 타입으로 참조

mutable.add(4)
println(readOnly)                           // [1, 2, 3, 4] — 읽기 전용 참조인데 값이 바뀜

(readOnly as MutableList<Int>).add(5)       // 다운캐스트가 성공함 — 실제 객체가 ArrayList이므로
```

- `listOf(1, 2, 3)`이 반환하는 객체는 `java.util.Arrays$ArrayList`로, `add`를 호출하면 `UnsupportedOperationException`이 발생함. 즉 "실패하는 가변 컬렉션"이지 진짜 불변 컬렉션은 아님
- 자바 코드는 `List<E>`를 `java.util.List`로 보기 때문에 아무 제약 없이 `add`를 호출할 수 있음 (unit09 참고)

<br>

### 3-2. 방어적 복사와 노출 패턴

```kotlin
// 안티패턴: 내부 가변 리스트를 List 타입으로 노출 → 외부에서 캐스트하거나 내부 변경이 새어 나감
class Cart {
    private val _items = mutableListOf<Item>()
    val items: List<Item> get() = _items        // 같은 객체를 반환
}

// 개선 1: 방어적 복사 — 스냅샷을 돌려줌 (호출마다 복사 비용)
class Cart {
    private val _items = mutableListOf<Item>()
    val items: List<Item> get() = _items.toList()
}

// 개선 2: 상태 자체를 불변으로 유지 — 변경 시 새 리스트로 교체 (StateFlow와 궁합이 좋음)
class Cart {
    var items: List<Item> = emptyList()
        private set
    fun add(item: Item) { items = items + item }   // + 는 새 리스트를 만든다
}
```

- `toList()`·`toMap()`은 항상 **새 컬렉션을 생성**하므로 원본과의 연결이 끊김
- `_items`/`items` 쌍은 "내부는 가변, 외부는 읽기 전용"을 표현하는 관용구이지만, 반환하는 객체가 같으면 **읽기 전용 참조도 내부 변경을 그대로 관찰**한다는 점을 이해하고 써야 함

> 💡 "`List`와 `MutableList`의 차이는?"에는 "인터페이스 차이일 뿐 **런타임 객체는 같을 수 있으며**, 진짜 불변이 필요하면 `kotlinx.collections.immutable`의 `PersistentList`나 방어적 복사를 써야 한다"까지 답해야 완전하다.

<br>

### 4. 진짜 불변 컬렉션 — kotlinx.collections.immutable

- JetBrains가 제공하는 별도 라이브러리로, `ImmutableList`(변경 메서드가 없고 **구현체도 불변**)와 `PersistentList`(변경 시 **새 컬렉션을 효율적으로 반환**)를 제공함
- `persistentListOf()`, `toPersistentList()`로 만들고, `add`·`remove`는 원본을 두고 **구조를 공유하는 새 컬렉션**을 돌려줌
- Jetpack Compose처럼 "이 컬렉션은 안정적(stable)이다"라고 프레임워크에 보장해야 하는 상황에서 널리 쓰임

```kotlin
val base = persistentListOf(1, 2, 3)
val next = base.add(4)          // 새 컬렉션 반환, base는 [1, 2, 3] 그대로

// 다운캐스트로도 변경 불가 — MutableList를 구현하지 않음
```

| **선택지**                  | **변경 가능성**                       | **비용**                        | **적합한 상황**                           |
| --------------------------- | ------------------------------------- | ------------------------------- | ----------------------------------------- |
| **List (읽기 전용 참조)**   | 다른 참조·자바·캐스트로 **변경 가능** | 없음                            | 모듈 내부에서 규약을 지킬 수 있을 때      |
| **toList() 방어적 복사**    | 복사본은 원본과 독립                  | **O(n) 복사**                   | 외부에 스냅샷을 넘길 때                   |
| **PersistentList**          | 구현체 자체가 불변                    | 변경 시 구조 공유로 **O(log n)** 수준 | 상태를 자주 갱신하는 UI 상태·이벤트 소싱 |
| **Collections.unmodifiableList** | 변경 시 예외 (원본 변경은 반영됨) | 래핑 비용만                     | 자바 API와의 경계                         |

<br>

### 5. 불변 객체 설계 원칙

- **모든 프로퍼티를 `val`로**, 타입도 불변(`String`, `Int`, `List`가 아닌 진짜 불변 컬렉션 또는 방어적 복사)으로 선언
- 변경이 필요하면 **`data class`의 `copy()`**로 일부만 바꾼 새 객체를 만듦 (unit06 참고)
- 생성자에서 외부 가변 컬렉션을 받으면 **복사해서 저장**해 호출자가 나중에 원본을 바꿔도 영향을 받지 않게 함
- `const val`은 컴파일 타임 상수(기본 타입·String, 최상위 또는 object 안)로 인라인되며, `val`은 런타임에 결정되는 읽기 전용 값임

```kotlin
class Order private constructor(
    val id: Long,
    val lines: List<OrderLine>            // 외부에서 준 리스트를 그대로 갖지 않음
) {
    companion object {
        const val MAX_LINES = 100
        fun create(id: Long, lines: List<OrderLine>): Order {
            require(lines.size <= MAX_LINES)
            return Order(id, lines.toList())   // 방어적 복사
        }
    }
    fun withLine(line: OrderLine) = Order(id, lines + line)   // 변경 대신 새 객체
}
```

**불변 객체의 이점**

- 스레드 간 공유 시 **동기화가 필요 없음** — 코루틴·Flow에서 상태를 안전하게 전달하는 기반 (unit04 참고)
- `equals`/`hashCode`가 시간이 지나도 변하지 않아 컬렉션 키·캐시 키로 안전함
- 상태 변화가 "새 객체 생성"으로 명시되어 변경 지점을 추적하기 쉬움

> ⚠️ 불변 객체는 변경마다 새 인스턴스를 만들므로, 수만 건을 반복 갱신하는 핫 루프에서는 할당 비용이 눈에 띌 수 있다. 이런 구간은 지역 범위에서만 `MutableList`로 작업한 뒤 `toList()`로 봉인해 외부로 내보내는 **"내부는 가변, 경계는 불변"** 전략이 현실적이다.

<br>

### 6. 정리 — 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                    |
| ----------------------------------------------------- | -------------------------------------------------------------------------------- |
| Kotlin의 기본 가시성은?                               | **public**. 자바의 package-private은 없고 모듈 단위 `internal`이 있음             |
| `internal`은 바이트코드에서 어떻게 보이는가?          | public이지만 **이름이 맹글링**되어 자바에서 우발적 호출을 막음                    |
| Kotlin의 `protected`와 자바의 차이는?                 | Kotlin은 **같은 패키지에서 접근 불가**, 하위 클래스만 가능                        |
| `val`은 불변인가?                                     | **재대입 불가일 뿐**. 객체 내부 변경과 커스텀 getter로 값이 바뀔 수 있음          |
| `List`는 불변 컬렉션인가?                             | **아니다**. 읽기 전용 인터페이스이며 런타임 객체는 가변일 수 있음                 |
| 읽기 전용 참조가 바뀌는 상황은?                       | 같은 객체를 가리키는 가변 참조, 자바 코드, `MutableList`로의 다운캐스트            |
| 진짜 불변이 필요하면?                                 | `toList()` 방어적 복사 또는 `kotlinx.collections.immutable`의 `PersistentList`   |

- 가시성은 **모듈 경계**(`internal`)와 **파일 경계**(최상위 `private`)로 자바보다 세밀하게 표현할 수 있음
- 불변성은 `val` 하나로 얻어지지 않으며, **타입·컬렉션·복사 전략**을 함께 설계해야 함
