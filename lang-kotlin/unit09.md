## 자바 상호운용

Kotlin은 JVM 바이트코드로 컴파일되어 **자바 코드와 양방향으로 호출**할 수 있지만, 두 언어의 개념(프로퍼티·널 가능성·최상위 함수·기본 인자·람다)이 1:1로 대응하지 않아 경계에서 변환 규칙을 알아야 한다. 이 문서는 자바를 Kotlin에서 호출할 때의 규칙, Kotlin을 자바에서 호출할 때 필요한 `@Jvm*` 어노테이션, SAM 변환, 그리고 널 어노테이션 처리를 다룬다.

<br>

### 1. 자바를 Kotlin에서 호출하기

- **getter/setter → 프로퍼티**: `getName()`/`setName()` 쌍은 `name` 프로퍼티로 접근함. `boolean isActive()`는 `isActive` 그대로 사용
- **`void` → `Unit`**, 정적 멤버는 `ClassName.member`로 접근
- **검사 예외(checked exception)**: Kotlin에는 검사 예외가 없으므로 자바 메서드가 `throws IOException`을 선언해도 `try/catch`를 강제하지 않음
- **배열**: 자바 `int[]`는 `IntArray`, `Integer[]`는 `Array<Int>`. 가변 인자에는 스프레드 연산자 `*`를 사용
- **널 가능성**: 어노테이션이 없으면 **플랫폼 타입** `T!`으로 들어옴 (unit01 참고)

```kotlin
// Java: public class Person { String getName(); void setName(String n); boolean isAdult(); }
val p = Person()
p.name = "kim"                 // setName("kim")
println(p.isAdult)             // isAdult()

val nums = intArrayOf(1, 2, 3)
Arrays.sort(nums)              // int[] 그대로 전달
String.format("%d-%d", *arrayOf(1, 2))   // varargs에 배열을 펼쳐 전달
```

<br>

### 2. Kotlin을 자바에서 호출하기

Kotlin 선언은 자바 관점에서 아래처럼 보인다. 대부분은 그대로 쓸 수 있지만, **자바에는 없는 개념**은 조금 어색한 형태로 노출된다.

| **Kotlin 선언**                  | **자바에서 보이는 형태**                              | **개선 어노테이션**       |
| -------------------------------- | ----------------------------------------------------- | ------------------------- |
| 최상위 함수 (`Utils.kt`)         | `UtilsKt.fn()` — **파일명 + Kt** 클래스의 static      | `@file:JvmName("Utils")`  |
| `val x` / `var x`                | `getX()` / `setX()`                                   | `@JvmField` (필드 직접 노출) |
| `const val`                      | `public static final` 필드                            | 불필요                    |
| `object Foo { fun bar() }`       | `Foo.INSTANCE.bar()`                                  | `@JvmStatic`              |
| `companion object { fun bar() }` | `Foo.Companion.bar()`                                 | `@JvmStatic`              |
| 기본 인자가 있는 함수            | **전체 인자**를 넘기는 오버로드 하나만                | `@JvmOverloads`           |
| 예외를 던지는 함수               | `throws` 선언 없음 → 자바에서 catch 시 컴파일 에러     | `@Throws(IOException::class)` |
| 제네릭 시그니처 충돌 (타입 소거) | 같은 JVM 시그니처로 컴파일 에러                       | `@JvmName("fnForInts")`   |
| `suspend fun`                    | 마지막 파라미터로 `Continuation` 추가된 메서드        | 래퍼 필요 (아래 참고)      |

```kotlin
@file:JvmName("StringUtils")           // 자바: StringUtils.capitalizeFirst("a")
package util

fun capitalizeFirst(s: String): String = s.replaceFirstChar { it.uppercase() }

class Money(val amount: Long) {
    companion object {
        @JvmStatic fun zero() = Money(0)         // 자바: Money.zero()
        @JvmField val KRW = "KRW"                // 자바: Money.KRW (getter 없이 필드)
    }

    @JvmOverloads
    fun format(locale: String = "ko", showSymbol: Boolean = true): String = TODO()
    // 자바: format(), format("ko"), format("ko", true) 세 개의 오버로드 생성

    @Throws(IOException::class)
    fun save() { /* ... */ }                     // 자바: try { m.save(); } catch (IOException e)
}
```

> 💡 `@JvmStatic`을 붙이지 않은 companion 멤버도 자바에서 호출은 되지만 `Money.Companion.zero()`처럼 어색하다. **자바 코드에서 쓰일 공개 API**라면 `@JvmStatic`·`@JvmOverloads`·`@file:JvmName`을 기본으로 고려한다.

<br>

### 3. SAM 변환 — 람다와 함수형 인터페이스

### 3-1. 자바 인터페이스로의 SAM 변환

**SAM(Single Abstract Method)** 인터페이스는 추상 메서드가 하나뿐인 인터페이스다. Kotlin은 자바 SAM 인터페이스를 받는 자리에 **람다를 직접 넘길 수 있게** 자동 변환한다.

```kotlin
// Java: executor.execute(Runnable r), button.setOnClickListener(OnClickListener l)
executor.execute { println("실행") }                 // Runnable로 자동 변환
button.setOnClickListener { view -> handle(view) }

val task = Runnable { println("SAM 생성자로 명시적 변환") }   // 변수에 담을 때
```

- 자바 메서드의 파라미터뿐 아니라 Kotlin 함수의 파라미터가 자바 SAM 타입일 때도 변환됨 (Kotlin 1.4부터)
- SAM 생성자 `Runnable { }`는 변환 대상 타입을 명시할 때 사용함

<br>

### 3-2. Kotlin 인터페이스는 fun interface가 필요하다

- Kotlin에서 선언한 일반 인터페이스는 추상 메서드가 하나여도 **SAM 변환이 적용되지 않음** — Kotlin에는 함수 타입 `(A) -> B`가 있으므로 굳이 인터페이스로 만들 이유가 없다는 설계 판단 때문
- 람다로 받고 싶은 Kotlin 인터페이스에는 **`fun interface`**(Kotlin 1.4+)를 선언함

```kotlin
// 안티패턴: 일반 인터페이스 → 람다를 넘길 수 없어 object 표현식을 써야 함
interface Validator { fun validate(s: String): Boolean }
val v1 = object : Validator { override fun validate(s: String) = s.isNotBlank() }

// 개선: fun interface → 람다로 생성 가능, 자바에서도 함수형 인터페이스로 사용 가능
fun interface Validator { fun validate(s: String): Boolean }
val v2 = Validator { it.isNotBlank() }
```

**Kotlin 함수 타입을 자바에서 호출할 때**

- `(Int) -> String`은 자바에서 `Function1<Integer, String>`으로 보이며, `Unit`을 반환하는 람다는 자바 쪽에서 **`return Unit.INSTANCE;`**를 명시해야 함
- 자바 호출자가 많다면 함수 타입 대신 `fun interface`나 자바 표준 `java.util.function.*`를 파라미터로 받는 것이 자바 쪽 사용성이 좋음

| **인터페이스 종류**             | **Kotlin에서 람다 전달** | **자바에서 람다 전달** | **비고**                                  |
| ------------------------------- | ------------------------ | ---------------------- | ----------------------------------------- |
| **자바 SAM 인터페이스**         | 가능 (자동 변환)         | 가능                   | `Runnable`, `Comparator`, `Function` 등    |
| **Kotlin `fun interface`**      | 가능                     | 가능                   | 양쪽에서 가장 매끄러움                    |
| **Kotlin 일반 인터페이스**      | **불가** (object 표현식) | 가능 (자바는 SAM 허용) | Kotlin 쪽 사용성이 나쁨                   |
| **Kotlin 함수 타입 `(A) -> B`** | 가능                     | 가능하지만 `Unit.INSTANCE` 필요 | 자바에서는 `FunctionN` 제네릭으로 보임 |

<br>

### 4. 널 어노테이션 처리

자바 코드에 널 어노테이션이 있으면 Kotlin은 플랫폼 타입 대신 **일반 널 가능·널 불가 타입**으로 읽어 컴파일 시점 검사를 적용한다.

| **어노테이션 계열**                    | **패키지**                          | **Kotlin의 처리**                                                  |
| -------------------------------------- | ----------------------------------- | ------------------------------------------------------------------ |
| **JetBrains**                          | `org.jetbrains.annotations`         | 기본 인식. Kotlin 컴파일러가 **자기 바이트코드에도 이 어노테이션을 기록**함 |
| **JSpecify**                           | `org.jspecify.annotations`          | `@NullMarked`로 범위 전체를 널 불가 기본으로 지정. **Kotlin 2.1부터 strict 모드가 기본**이라 위반 시 컴파일 에러 |
| **JSR-305**                            | `javax.annotation`                  | 기본은 경고. `-Xjsr305=strict` 옵션으로 에러 승격 (Spring 5~6가 사용) |
| **Android**                            | `androidx.annotation`               | 기본 인식                                                          |
| **기타** (Eclipse, FindBugs, RxJava 등) | 각자                                | 대부분 인식. 목록은 버전에 따라 다를 수 있음                        |

```java
// Java (JSpecify)
@NullMarked
public class UserRepository {
    public User find(long id) { ... }          // 반환값 널 불가로 해석
    public @Nullable User findOrNull(long id) { ... }
}
```

```kotlin
val u1: User = repo.find(1)          // OK — 플랫폼 타입이 아니라 User로 읽힘
val u2: User = repo.findOrNull(1)    // 컴파일 에러 — User?를 User에 대입 (2.1+ strict)
val u3 = repo.findOrNull(1)?.name    // 안전 호출 필요
```

- Kotlin이 컴파일한 클래스에는 파라미터·반환 타입에 `@NotNull`/`@Nullable`(JetBrains)이 자동으로 붙으므로, 자바 쪽 IDE와 정적 분석 도구가 이를 활용할 수 있음
- Kotlin 함수의 널 불가 파라미터에 자바가 null을 넘기면, 함수 진입 시 생성된 **`checkNotNullParameter` 검사**가 즉시 `NullPointerException`을 던져 문제를 경계에서 드러냄

> ⚠️ Spring Framework 7과 Spring Boot 4처럼 최근 프레임워크는 JSR-305에서 **JSpecify로 이전**하는 추세다. 사용하는 프레임워크·라이브러리가 어떤 계열의 어노테이션을 쓰는지, 그리고 Kotlin 버전이 그것을 strict로 처리하는지는 **버전에 따라 다를 수 있으므로** 빌드 로그의 경고를 확인해야 한다.

<br>

### 5. 제네릭 가변성과 와일드카드

- Kotlin의 `List<out T>` 같은 **선언 지점 공변성**은 자바 시그니처에서 `List<? extends T>` 와일드카드로 변환됨 (파라미터 위치일 때). 반환 타입은 기본적으로 와일드카드 없이 방출됨
- 자바 호출자가 `List<? extends Item>`을 다루기 번거로우면 `@JvmSuppressWildcards`로 와일드카드를 제거하고, 반대로 필요하면 `@JvmWildcard`로 강제할 수 있음

```kotlin
fun render(items: List<Item>)                         // 자바: render(List<? extends Item>)
fun render(items: List<@JvmSuppressWildcards Item>)   // 자바: render(List<Item>)
```

<br>

### 6. suspend 함수를 자바에 노출하기

- `suspend fun load(): User`는 자바에서 `Object load(Continuation<? super User> c)`로 보이며, 자바가 `Continuation`을 직접 구현하는 것은 비현실적임
- 자바 호출자를 위해 **`CompletableFuture`나 블로킹 래퍼**를 별도로 제공하는 것이 일반적임

```kotlin
class UserService(private val scope: CoroutineScope) {
    suspend fun load(id: Long): User = TODO()

    // 자바용: kotlinx-coroutines-jdk8의 future 빌더
    fun loadAsync(id: Long): CompletableFuture<User> = scope.future { load(id) }

    // 자바용(블로킹): 호출 스레드를 점유하므로 요청 처리 스레드에서는 지양
    fun loadBlocking(id: Long): User = runBlocking { load(id) }
}
```

> 💡 자바 팀과 협업하는 라이브러리라면 "suspend 함수는 Kotlin 전용 API, 자바에는 `CompletableFuture` 래퍼"처럼 **언어별 진입점을 분리**하는 것이 관례다. 자바 호출자에게 `Continuation`을 노출하는 것은 사실상 사용 불가능한 API를 제공하는 것과 같다.

<br>

### 7. 정리 — 면접·실무 체크포인트

| **질문**                                                | **핵심 답변**                                                                   |
| ------------------------------------------------------- | ------------------------------------------------------------------------------- |
| 자바 getter/setter는 Kotlin에서 어떻게 보이는가?        | **프로퍼티**로 접근 (`getX`/`setX` → `x`)                                       |
| Kotlin 최상위 함수는 자바에서 어떻게 호출하는가?        | `파일명Kt.함수()`. `@file:JvmName`으로 클래스 이름 지정                          |
| `@JvmStatic`이 필요한 이유는?                           | companion/object 멤버가 기본적으로 `Companion`/`INSTANCE` 경유라 **진짜 static이 아니기 때문** |
| `@JvmOverloads`는 무엇을 하는가?                        | 기본 인자마다 **자바용 오버로드**를 생성                                          |
| SAM 변환이란?                                           | 추상 메서드 하나인 인터페이스 자리에 **람다를 넘기면 자동 변환**. Kotlin 인터페이스는 `fun interface` 필요 |
| Kotlin 람다를 자바에서 구현하면 반환값은?               | `Unit` 반환이면 **`Unit.INSTANCE`**를 명시                                       |
| 자바 널 어노테이션은 어떻게 처리되는가?                 | 인식되면 일반 타입으로, 없으면 **플랫폼 타입**. JSpecify는 2.1부터 strict 기본   |
| 검사 예외는 어떻게 되는가?                              | Kotlin은 강제하지 않음. 자바가 catch하려면 `@Throws` 필요                        |

- 경계에서 필요한 것은 **널 정보(어노테이션)**, **호출 편의(`@Jvm*`)**, **람다 호환(`fun interface`)** 세 가지로 요약됨
- 플랫폼 타입의 위험과 처리는 **unit01(Null Safety 설계)**, `internal`의 이름 맹글링은 **unit08(가시성과 불변성)**을 참고할 것
