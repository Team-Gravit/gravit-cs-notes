## Null Safety 설계

Kotlin은 **널 가능성(Nullability)을 타입 시스템에 포함**시켜 `NullPointerException`을 컴파일 시점에 걸러낸다. 이 문서는 널 가능 타입의 동작 원리, 자바 코드가 넘어올 때 생기는 플랫폼 타입, 안전 호출 체이닝, 그리고 강제 단언(`!!`)을 피해야 하는 이유를 다룬다.

<br>

### 1. 널 가능 타입과 널 불가 타입

- Kotlin의 모든 타입은 기본적으로 **널 불가(non-null)**이며, 널을 담으려면 타입 뒤에 `?`를 붙여 **널 가능(nullable)** 타입으로 선언해야 함
- `String`과 `String?`은 서로 다른 타입이고, `String?`에는 `String`의 메서드를 직접 호출할 수 없음
- 컴파일러가 널 검사 여부를 추적하므로, 자바에서 런타임에 터지던 NPE의 대부분이 **컴파일 에러**로 바뀜

```kotlin
val a: String = "kotlin"
val b: String? = null

a.length        // OK
b.length        // 컴파일 에러: Only safe (?.) or non-null asserted (!!.) calls are allowed
```

```
      타입 계층 (널 가능성 기준)
      ┌────────────┐
      │  String?   │  ← String 또는 null
      └─────┬──────┘
            │ 상위 타입
      ┌─────┴──────┐
      │  String    │  ← null 불가, String? 자리에 대입 가능 (역방향은 불가)
      └────────────┘
```

> 💡 "Kotlin은 NPE가 없나요?"라는 면접 질문에는 "타입 시스템이 대부분을 막지만 `!!`, 플랫폼 타입, `lateinit` 미초기화, 자바 상호운용에서는 여전히 발생할 수 있다"고 답해야 정확하다.

<br>

### 2. 널 처리 연산자와 스마트 캐스트

### 2-1. 안전 호출 `?.`과 엘비스 `?:`

- **안전 호출(safe call) `?.`**: 수신 객체가 null이면 호출을 건너뛰고 **결과 전체가 null**이 됨
- **엘비스(Elvis) 연산자 `?:`**: 좌변이 null일 때 사용할 **기본값**을 지정함. 우변에 `return`·`throw`를 둘 수 있어 조기 종료 패턴에 유용함

```kotlin
data class Address(val city: String?)
data class User(val name: String, val address: Address?)

fun cityOf(user: User?): String {
    // 안전 호출 체이닝: 중간에 하나라도 null이면 전체가 null → 엘비스로 기본값
    return user?.address?.city ?: "UNKNOWN"
}

fun requireCity(user: User?): String {
    val city = user?.address?.city ?: throw IllegalArgumentException("도시 정보 없음")
    return city.uppercase()   // 이 지점에서 city는 String으로 스마트 캐스트됨
}
```

<br>

### 2-2. 스마트 캐스트(Smart Cast)

- `if (x != null)` 같은 검사를 통과하면 그 블록 안에서 컴파일러가 `x`를 **자동으로 널 불가 타입으로 취급**함
- 단, 값이 중간에 바뀔 수 없다고 컴파일러가 보장할 수 있을 때만 동작함 → `val` 지역 변수, 커스텀 getter가 없는 `val` 프로퍼티는 가능하지만, **`var` 프로퍼티나 다른 모듈의 `open` 프로퍼티는 불가**
- Kotlin 2.0의 K2 컴파일러부터는 `||` 조건, 인라인 람다 내부, 로컬 변수에 검사 결과를 담은 경우 등 **스마트 캐스트가 적용되는 범위가 확장**됨

```kotlin
class Repo(var cache: String?)

fun bad(repo: Repo) {
    if (repo.cache != null) {
        // repo.cache.length  → 컴파일 에러: 다른 스레드가 사이에 바꿀 수 있는 var 프로퍼티
    }
}

fun good(repo: Repo) {
    val cache = repo.cache ?: return   // 지역 val로 스냅샷을 잡으면 스마트 캐스트 가능
    println(cache.length)
}
```

> ⚠️ `var` 프로퍼티는 검사와 사용 사이에 다른 스레드가 값을 바꿀 수 있으므로 스마트 캐스트가 거부된다. 프로퍼티를 **지역 `val`에 복사한 뒤 검사**하는 습관이 가장 깔끔한 해결책이다.

<br>

### 3. 플랫폼 타입(Platform Type)

자바 코드는 널 가능성 정보가 없기 때문에, Kotlin은 자바에서 넘어온 값을 **플랫폼 타입 `T!`**으로 취급한다. 플랫폼 타입은 "널일 수도, 아닐 수도 있음"을 뜻하며 개발자가 `T`와 `T?` 중 어느 쪽으로 받을지 **직접 선택**해야 한다.

```kotlin
// Java: public String findName(long id) { ... }  // null을 반환할 수도 있음
val name1: String = javaRepo.findName(1)    // 컴파일은 통과하지만 null이면 즉시 NPE
val name2: String? = javaRepo.findName(1)   // 안전: 이후 ?. 로 다룬다
val name3 = javaRepo.findName(1)            // 타입 추론 결과는 String! (플랫폼 타입 그대로)
```

- `T!` 표기는 IDE와 컴파일러 메시지에만 나타나며 소스 코드에 직접 쓸 수 없음
- 자바 쪽에 `@Nullable` / `@NotNull` 어노테이션이 있으면 Kotlin은 이를 읽어 **일반 널 가능·널 불가 타입으로 변환**함. Kotlin 2.1부터는 JSpecify(`org.jspecify.annotations`) 어노테이션이 기본적으로 **strict 모드**로 처리되어 위반 시 컴파일 에러가 됨 (어노테이션 종류별 자세한 내용은 unit09 참고)

| **구분**            | **널 불가 `T`**   | **널 가능 `T?`**    | **플랫폼 타입 `T!`**              |
| ------------------- | ----------------- | ------------------- | --------------------------------- |
| **출처**            | Kotlin 선언       | Kotlin 선언         | **어노테이션 없는 자바 코드**     |
| **null 대입**       | 컴파일 에러       | 허용                | 허용 (검사 없음)                  |
| **직접 메서드 호출** | 가능              | 불가 (`?.` 필요)    | 가능 (null이면 **런타임 NPE**)    |
| **권장 처리**       | -                 | `?.`·`?:`로 처리    | 명시적 타입을 선언해 `T?`로 고정  |

> 💡 자바 라이브러리를 호출하는 경계에서는 **반환 타입을 명시적으로 `T?`로 선언**하는 것이 원칙이다. 플랫폼 타입을 그대로 흘려보내면 NPE가 호출 지점이 아니라 한참 뒤에서 터져 원인 추적이 어려워진다.

<br>

### 4. 강제 단언 `!!`을 피해야 하는 이유

`!!`는 "이 값은 절대 null이 아니다"라고 컴파일러에게 단언하는 연산자로, null이면 그 자리에서 `NullPointerException`을 던진다. 컴파일러의 보호를 **개발자가 스스로 해제**하는 것이므로 다음과 같은 문제가 있다.

- 널 가능성 검증을 런타임으로 미루므로 Kotlin이 제공하는 **컴파일 시점 안전성을 포기**하는 것과 같음
- `!!`가 여러 개 연결된 `a!!.b!!.c!!`는 어느 지점에서 터졌는지 스택 트레이스만으로 구분하기 어려움
- 리팩터링으로 상위 로직이 바뀌어 "절대 null이 아니다"라는 전제가 깨져도 컴파일러가 알려주지 않음

```kotlin
// 안티패턴: 검사와 사용이 분리되어 있고, 단언이 반복됨
fun sendMail(user: User?) {
    if (user != null && user.address != null) {
        mailer.send(user!!.address!!.city!!)   // 이미 검사했는데도 !! 남발
    }
}

// 개선: 안전 호출 + 엘비스로 조기 종료, 이후에는 널 불가 타입만 다룸
fun sendMail(user: User?) {
    val city = user?.address?.city ?: return
    mailer.send(city)
}
```

**`!!` 대신 쓸 수 있는 도구**

| **상황**                              | **대안**                          | **설명**                                             |
| ------------------------------------- | --------------------------------- | ---------------------------------------------------- |
| null이면 기본값을 쓰고 싶음           | `?:`                              | **가장 기본**적인 대체 수단                          |
| null이면 즉시 종료하거나 예외를 던짐  | `?: return` / `?: throw`          | 이후 코드에서 스마트 캐스트가 적용됨                 |
| null이 아닐 때만 블록을 실행          | `?.let { }`                       | 결과가 필요 없으면 `?.also { }`                      |
| 계약상 null이 될 수 없음을 명시       | `requireNotNull()` / `checkNotNull()` | **메시지를 붙여** 실패 원인을 남길 수 있음       |
| 생성 이후 반드시 초기화되는 프로퍼티  | `lateinit var`                    | 미초기화 접근 시 원인이 명확한 예외 발생             |
| 초기화 비용이 크거나 지연이 필요함    | `by lazy { }`                     | 최초 접근 시 한 번만 초기화 (unit07 참고)            |

> ⚠️ `lateinit`은 기본 타입(`Int`, `Boolean` 등)과 널 가능 타입에는 쓸 수 없고, 초기화 전에 접근하면 `UninitializedPropertyAccessException`이 발생한다. `::prop.isInitialized`로 확인할 수 있지만, 이 검사가 자주 필요하다면 애초에 널 가능 타입으로 설계하는 것이 맞다.

<br>

### 5. 컬렉션과 제네릭에서의 널 처리

- `List<String?>`(원소가 null일 수 있음)과 `List<String>?`(리스트 자체가 null일 수 있음)은 전혀 다른 타입임
- `filterNotNull()`은 `List<T?>`를 `List<T>`로 바꿔 주고, `mapNotNull { }`은 변환과 null 제거를 한 번에 처리함
- 제네릭 타입 파라미터 `T`는 상한이 지정되지 않으면 `Any?`가 기본이므로 **null을 허용**함. null을 막으려면 `T : Any`로 상한을 명시해야 함

```kotlin
fun <T : Any> firstOrThrow(list: List<T?>): T =
    list.filterNotNull().firstOrNull() ?: error("null이 아닌 원소가 없음")

val ids: List<Int?> = listOf(1, null, 3)
val valid: List<Int> = ids.filterNotNull()          // [1, 3]
val doubled = ids.mapNotNull { it?.times(2) }        // [2, 6]
```

<br>

### 6. 안전 캐스트 `as?`와 타입 검사

- `as`는 캐스트에 실패하면 `ClassCastException`을 던지지만, **안전 캐스트 `as?`**는 실패 시 null을 돌려줌
- `as?`와 `?:`를 조합하면 "타입이 맞을 때만 처리하고, 아니면 기본값" 패턴을 한 줄로 표현할 수 있음
- `is` 검사를 통과하면 스마트 캐스트가 적용되므로, 대부분의 경우 `as`보다 `is`가 안전함

```kotlin
fun describe(value: Any?): String {
    val text = value as? String ?: return "문자열이 아님"
    return "길이 ${text.length}"
}
```

<br>

### 7. 정리 — 면접·실무 체크포인트

| **질문**                                     | **핵심 답변**                                                                 |
| -------------------------------------------- | ----------------------------------------------------------------------------- |
| `String`과 `String?`의 차이는?               | 후자는 null 허용. **서로 다른 타입**이며 전자는 후자의 하위 타입              |
| 스마트 캐스트가 안 되는 경우는?              | `var` 프로퍼티, 커스텀 getter, 다른 모듈의 `open` 프로퍼티 등 **값 변경 가능성**이 있을 때 |
| 플랫폼 타입이란?                             | 자바에서 온 널 정보 없는 타입 `T!`. **명시적으로 `T?`로 받는 것**이 원칙     |
| `!!`를 쓰면 안 되는 이유는?                  | 컴파일 시점 검증을 런타임으로 미루고, 전제가 깨져도 **컴파일러가 감지 못 함** |
| `!!` 대신 무엇을 쓰는가?                     | `?:`·`?.let`·`requireNotNull`·`lateinit`·`by lazy` 등 상황별 대안              |
| `lateinit`과 `by lazy`의 차이는?             | 전자는 `var`·외부 초기화, 후자는 `val`·최초 접근 시 자기 초기화 (unit07 참고)  |

- Kotlin의 널 안전성은 **타입 시스템**의 기능이며, 자바 경계와 `!!`가 그 보호막의 구멍임
- 자바 상호운용 시 널 어노테이션 처리 방식은 **unit09(자바 상호운용)**을 참고할 것
