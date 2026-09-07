## data·sealed·object

`data class`·`sealed class`·`object`는 Kotlin이 **"값"과 "상태"를 코드로 모델링**하기 위해 제공하는 세 가지 클래스 종류다. 이 문서는 data class가 값 비교를 어떻게 자동 생성하는지, sealed 계층으로 제한된 상태 집합을 표현하고 `when`의 완전성 검사를 얻는 방법, 그리고 object의 세 가지 용법을 다룬다.

<br>

### 1. data class — 값을 담는 클래스

- `data` 변경자를 붙이면 컴파일러가 **주 생성자의 프로퍼티를 기준으로** `equals()`·`hashCode()`·`toString()`·`copy()`·`componentN()`을 자동 생성함
- 자바에서 롬복(Lombok)이나 `record`로 하던 일을 언어 차원에서 지원하는 것이며, 목적은 **동일성(identity)이 아니라 값(value)으로 비교되는 객체**를 만드는 것
- 제약: 주 생성자에 파라미터가 최소 1개 있어야 하고 모두 `val`/`var`여야 하며, `abstract`·`open`·`sealed`·`inner`가 될 수 없음

```kotlin
data class Point(val x: Int, val y: Int)

val a = Point(1, 2)
val b = Point(1, 2)

a == b                  // true  — equals: 값 비교
a === b                 // false — 참조 비교 (서로 다른 인스턴스)
a.hashCode() == b.hashCode()   // true — HashSet·HashMap 키로 안전
println(a)              // Point(x=1, y=2)
val (x, y) = a          // 구조 분해: component1(), component2()
val moved = a.copy(y = 10)     // Point(x=1, y=10) — 원본은 그대로
```

<br>

### 2. data class의 함정

### 2-1. 본문 프로퍼티는 비교에서 제외된다

```kotlin
data class User(val id: Long) {
    var nickname: String = ""      // 주 생성자 밖 → equals·toString·copy 모두 제외
}

val u1 = User(1).apply { nickname = "a" }
val u2 = User(1).apply { nickname = "b" }
u1 == u2                            // true — nickname은 무시됨
u1.copy().nickname                  // ""   — copy는 본문 프로퍼티를 복사하지 않음
```

- 값 비교에 포함할 프로퍼티는 **반드시 주 생성자**에 두어야 함
- 반대로, ID처럼 일부 필드만으로 동일성을 정의하고 싶다면 `equals`를 직접 오버라이드하거나 data class를 쓰지 않는 것이 맞음

<br>

### 2-2. 배열과 가변 프로퍼티

- `Array`는 `equals`가 참조 비교이므로 `data class Buf(val bytes: ByteArray)`는 내용이 같아도 `==`가 false임. **`List`를 쓰거나** `equals`/`hashCode`를 `contentEquals`로 직접 구현해야 함
- `var` 프로퍼티를 가진 data class를 `HashSet`·`HashMap` 키로 쓰면, 값을 바꾼 순간 `hashCode`가 달라져 **컬렉션에서 찾을 수 없게 됨** → data class는 `val`만 쓰는 것이 원칙 (불변성은 unit08 참고)
- `copy()`는 **얕은 복사**임. 프로퍼티가 가변 컬렉션이면 원본과 복사본이 같은 컬렉션을 공유함

> ⚠️ JPA 엔티티를 data class로 만드는 것은 대표적인 안티패턴이다. 지연 로딩 프록시·양방향 연관관계에서 `hashCode`·`toString`이 무한 재귀나 예상치 못한 쿼리를 유발하고, 가변 필드 때문에 컬렉션 키로서의 계약도 깨진다. 엔티티는 일반 클래스로, DTO는 data class로 구분한다.

<br>

### 3. sealed class와 sealed interface — 제한된 계층

**sealed(봉인된)** 클래스·인터페이스는 **하위 타입의 집합이 컴파일 시점에 고정**된 계층이다. 하위 타입은 반드시 **같은 모듈·같은 패키지** 안에 선언해야 하며(Kotlin 1.5부터 파일은 달라도 됨), 외부 모듈에서는 확장할 수 없다.

| **구분**            | **enum class**                         | **sealed class / interface**                              |
| ------------------- | -------------------------------------- | --------------------------------------------------------- |
| **인스턴스**        | 각 상수가 **단일 인스턴스**            | 하위 타입마다 **여러 인스턴스** 생성 가능                 |
| **상태(데이터)**    | 모든 상수가 **같은 프로퍼티 집합**     | 하위 타입마다 **서로 다른 프로퍼티** 보유 가능            |
| **상속**            | 불가 (인터페이스 구현만)               | sealed interface는 **다중 구현** 가능                     |
| **when 완전성**     | 지원                                   | 지원                                                      |
| **적합한 상황**     | 요일·색상처럼 값만 다른 상수 집합      | `Loading / Success(data) / Error(cause)`처럼 **형태가 다른 상태** |

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val cause: Throwable) : UiState<Nothing>
}
```

```
UiState<T>  (sealed — 이 세 가지 외의 하위 타입은 존재할 수 없음)
 ├── Loading            : 데이터 없음 → data object (싱글턴)
 ├── Success(data: T)   : 결과 보유  → data class
 └── Error(cause)       : 실패 원인  → data class
```

> 💡 sealed 계층은 함수형 언어의 **대수적 데이터 타입(ADT, Algebraic Data Type)**의 "합 타입(sum type)"에 해당한다. "상태는 이 중 하나"라는 사실을 타입으로 표현하면, 불가능한 조합(`isLoading = true`인데 `data != null`)이 애초에 생길 수 없다.

<br>

### 4. when의 완전성(exhaustiveness) 검사

- `when`의 대상이 sealed 타입·enum·`Boolean`이면 컴파일러가 **모든 경우가 처리되었는지 검사**함
- 표현식(값을 반환하는) `when`은 처음부터 완전성이 강제되었고, **Kotlin 1.7부터는 문장(statement) `when`도 sealed·enum·Boolean 대상이면 누락 시 컴파일 에러**가 됨
- `else` 분기를 두면 검사가 꺼지므로, 나중에 하위 타입이 추가되어도 컴파일러가 알려주지 않음

```kotlin
// 안티패턴: else 때문에 새 상태(예: Empty)를 추가해도 컴파일러가 침묵함
fun render(state: UiState<List<Item>>) = when (state) {
    is UiState.Success -> showList(state.data)   // 스마트 캐스트로 data 접근
    else -> showSpinner()                         // Error가 스피너로 처리되는 버그가 숨음
}

// 개선: 모든 하위 타입을 명시 → Empty를 추가하는 순간 이 when이 컴파일 에러로 알려줌
fun render(state: UiState<List<Item>>) = when (state) {
    UiState.Loading    -> showSpinner()
    is UiState.Success -> showList(state.data)
    is UiState.Error   -> showError(state.cause)
}
```

- `is` 검사와 스마트 캐스트가 결합되어 각 분기에서 하위 타입의 프로퍼티에 바로 접근할 수 있음
- 하위 타입이 `object`(단일 인스턴스)면 `is` 없이 값 비교로 매칭할 수 있음

> ⚠️ sealed 타입을 **다른 모듈**에서 `when`으로 분기하면, 그 모듈은 하위 타입 추가를 컴파일 시점에 감지할 수 있지만 라이브러리 저자는 하위 타입 추가가 **사용자 코드를 깨뜨리는 변경**임을 알아야 한다. 공개 API의 sealed 계층 확장은 호환성 관점에서 신중해야 한다.

<br>

### 5. object의 세 가지 용법

| **형태**              | **문법**                          | **생성 시점**                       | **용도**                                       |
| --------------------- | --------------------------------- | ----------------------------------- | ---------------------------------------------- |
| **object 선언**       | `object Config { ... }`           | **최초 접근 시** (JVM 클래스 초기화, 스레드 안전) | 싱글턴, 상태 없는 유틸리티, sealed 하위 상태 |
| **companion object**  | `class A { companion object { } }` | 바깥 클래스 초기화 시               | 팩토리 메서드, 상수, 자바의 `static` 대체     |
| **object 표현식**     | `object : Listener { ... }`       | 표현식 평가 시마다 새로 생성        | 익명 클래스 (자바의 익명 내부 클래스)          |

```kotlin
object AppConfig {
    const val TIMEOUT_MS = 3_000L
    fun load(): Config = TODO()
}

class Order private constructor(val id: Long) {
    companion object {
        fun of(id: Long): Order = Order(id)    // 팩토리: 생성자를 숨기고 이름 있는 생성
    }
}

val listener = object : ClickListener {
    override fun onClick() = println("클릭")   // 매번 새 인스턴스
}
```

- `object` 선언은 JVM의 **클래스 로딩 시 정적 초기화 블록에서 인스턴스를 만들므로** 별도의 동기화 없이 스레드 안전한 지연 초기화 싱글턴이 됨
- `companion object`는 하나만 가질 수 있고 이름을 생략하면 `Companion`으로 접근함. 자바에서 정적 메서드로 보이게 하려면 `@JvmStatic`이 필요함 (unit09 참고)
- **`data object`(Kotlin 1.9 정식)**: `toString()`이 클래스 이름을 반환하고, 직렬화·리플렉션으로 인스턴스가 복제되어도 `equals`가 타입 기준으로 true를 유지함. sealed 계층의 상태 없는 하위 타입에 적합함

<br>

### 6. 상태 모델링 실전 패턴

```kotlin
// 결과 타입: 성공·실패를 예외 대신 값으로 표현
sealed interface Result<out T> {
    data class Ok<T>(val value: T) : Result<T>
    data class Fail(val error: AppError) : Result<Nothing>
}

sealed interface AppError {
    data object Network : AppError
    data class NotFound(val id: Long) : AppError
    data class Validation(val messages: List<String>) : AppError
}

fun handle(r: Result<User>): String = when (r) {
    is Result.Ok   -> "환영합니다, ${r.value.name}"
    is Result.Fail -> when (r.error) {                    // 중첩 sealed도 완전성 검사
        AppError.Network       -> "네트워크를 확인하세요"
        is AppError.NotFound   -> "사용자 ${r.error.id} 없음"
        is AppError.Validation -> r.error.messages.joinToString()
    }
}
```

- **불가능한 상태를 표현 불가능하게** 만드는 것이 목표: `Boolean` 플래그 여러 개 대신 sealed 하위 타입 하나로 상태를 나타냄
- 값을 담는 하위 타입은 `data class`, 값이 없는 하위 타입은 `data object`로 통일하면 `toString`·`equals`가 일관됨 (StateFlow의 중복 제거와 결합되는 이유는 unit04 참고)
- 제네릭 sealed 타입에서 `Nothing`을 타입 인자로 쓰면 `Result<Nothing>`이 모든 `Result<T>`의 하위 타입이 되어(`out` 공변) 실패 타입을 하나로 재사용할 수 있음

<br>

### 7. 정리 — 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                    |
| ------------------------------------------------- | -------------------------------------------------------------------------------- |
| data class가 자동 생성하는 것은?                  | `equals`·`hashCode`·`toString`·`copy`·`componentN` — **주 생성자 프로퍼티 기준**   |
| 본문에 선언한 프로퍼티는 왜 비교에서 빠지는가?    | 자동 생성 대상이 주 생성자에 한정되기 때문. 비교 대상은 주 생성자에 둔다           |
| `copy()`는 깊은 복사인가?                         | **얕은 복사**. 내부 가변 객체는 공유됨                                            |
| sealed와 enum의 차이는?                           | enum은 단일 인스턴스·동일 프로퍼티, sealed는 **하위 타입별로 다른 데이터**와 여러 인스턴스 |
| `when`에서 `else`를 피하는 이유는?                | 완전성 검사가 꺼져 **하위 타입 추가를 컴파일러가 잡지 못함**                       |
| `object`는 언제 초기화되는가?                     | **최초 접근 시** JVM 클래스 초기화로, 스레드 안전                                 |
| `data object`가 필요한 이유는?                    | `toString`이 이름을 반환하고, 직렬화로 복제돼도 `equals`가 유지됨                  |

- 값은 **data class**, 제한된 상태 집합은 **sealed**, 단일 인스턴스는 **object**로 표현하는 것이 Kotlin식 모델링임
- data class의 불변성 설계와 읽기 전용 컬렉션의 함정은 **unit08(가시성과 불변성)**을 참고할 것
