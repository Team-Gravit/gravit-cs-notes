## 확장 함수와 스코프 함수

**확장 함수(Extension Function)**는 클래스를 수정하지 않고 마치 멤버처럼 호출할 수 있는 함수를 바깥에서 추가하는 기능이고, **스코프 함수(Scope Function)**는 객체를 받아 그 컨텍스트 안에서 블록을 실행하는 표준 라이브러리의 확장 함수 다섯 개(`let`·`run`·`apply`·`also`·`with`)다. 이 문서는 확장 함수가 정적으로 디스패치되는 원리와 그로 인한 함정, 그리고 스코프 함수의 선택 기준을 다룬다.

<br>

### 1. 확장 함수의 동작 원리

- `fun String.lastChar(): Char = this[length - 1]`처럼 **수신 타입(receiver type)** 뒤에 점을 찍어 선언하며, 본문에서 `this`는 수신 객체를 가리킴
- 컴파일하면 **수신 객체를 첫 번째 인자로 받는 정적 메서드**가 되어 클래스 자체는 전혀 바뀌지 않음
- 따라서 확장 함수는 클래스의 `private`·`protected` 멤버에 **접근할 수 없음** — 캡슐화를 깨지 않는 이유이자 한계

```kotlin
// StringExt.kt
fun String.lastChar(): Char = this[length - 1]

"kotlin".lastChar()   // 'n'
```

```java
// 컴파일 결과를 자바 관점에서 보면 — 정적 메서드와 동일
public final class StringExtKt {
    public static final char lastChar(String $this) {
        return $this.charAt($this.length() - 1);
    }
}
// 자바에서 호출: StringExtKt.lastChar("kotlin")
```

- 널 가능 수신 타입도 가능함: `fun String?.orEmpty(): String = this ?: ""` — 본문에서 `this`가 null일 수 있으므로 검사가 필요함
- **확장 프로퍼티**도 같은 원리로 선언할 수 있으나, 실제 필드가 없으므로 backing field를 가질 수 없고 getter만 정의함

> 💡 확장 함수는 자바의 유틸리티 클래스(`StringUtils.lastChar(s)`)를 **호출 문법만 멤버처럼** 바꾼 것이다. 마법이 아니라 정적 메서드라는 점을 이해하면 이후의 디스패치 규칙이 모두 자연스럽게 설명된다.

<br>

### 2. 정적 디스패치와 그 함정

### 2-1. 확장 함수는 선언된 타입으로 결정된다

멤버 함수는 런타임 객체의 실제 타입으로 호출이 결정되지만(동적 디스패치), 확장 함수는 **컴파일 시점의 변수 선언 타입**으로 결정된다(정적 디스패치). 확장 함수는 정적 메서드이므로 오버라이드라는 개념 자체가 없다.

```kotlin
open class Shape
class Circle : Shape()

fun Shape.describe() = "도형"
fun Circle.describe() = "원"

val c: Circle = Circle()
val s: Shape = Circle()          // 실제 객체는 Circle

c.describe()                     // "원"   — 변수 타입이 Circle
s.describe()                     // "도형" — 변수 타입이 Shape (실제 타입은 무시됨)
```

```
멤버 함수      : s.method()   → 런타임 타입(Circle) 확인 → Circle.method 호출 (가상 호출)
확장 함수      : s.describe() → 컴파일 시 변수 타입(Shape)으로 → ShapeExt.describe(s) 고정
```

<br>

### 2-2. 멤버가 항상 확장을 이긴다

- 같은 이름·같은 시그니처의 **멤버 함수가 있으면 확장 함수는 호출되지 않음** (컴파일러가 경고를 띄움)
- 라이브러리 클래스에 확장 함수를 추가해 두었는데, 이후 라이브러리가 같은 이름의 멤버를 추가하면 **소스 수정 없이 동작이 바뀜**
- 예외: 시그니처가 다르면(파라미터 타입 등) 확장이 선택될 수 있음

```kotlin
class Printer {
    fun print() = println("멤버")
}
fun Printer.print() = println("확장")   // 경고: Extension is shadowed by a member

Printer().print()   // "멤버"
```

> ⚠️ 정적 디스패치 때문에 확장 함수로 **다형성을 흉내 내려 하면 반드시 실패**한다. 하위 타입마다 다른 동작이 필요하면 확장 함수가 아니라 멤버 함수(또는 인터페이스)로 정의하거나, 확장 함수 안에서 `when (this)`로 분기해야 한다 (sealed 계층은 unit06 참고).

<br>

### 3. 확장 함수의 가시성과 임포트

- 확장 함수는 정의된 **패키지에서 임포트**해야 사용할 수 있음. 같은 이름의 확장이 여러 패키지에 있으면 임포트로 선택함
- 클래스 안에 선언한 확장 함수(멤버 확장)는 그 클래스 안에서만 보이며, **디스패치 수신 객체**(클래스)와 **확장 수신 객체**(확장 대상)가 동시에 존재함. 이름이 겹치면 확장 수신 객체가 우선이고 `this@클래스명`으로 바깥을 지정함
- `private`·`internal` 등 가시성 변경자를 그대로 적용할 수 있어 모듈 내부 헬퍼로 제한할 수 있음 (unit08 참고)

<br>

### 4. 스코프 함수 다섯 가지

스코프 함수는 모두 "객체를 받아 람다를 실행한다"는 점은 같고, **람다 안에서 객체를 어떻게 참조하는가**(`this` vs `it`)와 **무엇을 반환하는가**(람다 결과 vs 객체 자신) 두 축에서만 다르다.

| **함수**   | **객체 참조** | **반환값**       | **확장 함수 여부** | **대표 용도**                                  |
| ---------- | ------------- | ---------------- | ------------------ | ---------------------------------------------- |
| **let**    | `it`          | **람다 결과**    | 확장               | 널 아닐 때만 실행, 결과를 다른 값으로 변환     |
| **run**    | `this`        | **람다 결과**    | 확장               | 객체 설정 + 결과 계산을 한 번에                |
| **with**   | `this`        | **람다 결과**    | 비확장 (인자로 받음) | 이미 있는 객체로 여러 메서드를 그룹 호출     |
| **apply**  | `this`        | **객체 자신**    | 확장               | 객체 **초기화·설정** (빌더 스타일)             |
| **also**   | `it`          | **객체 자신**    | 확장               | 로깅·검증 같은 **부수 효과**를 체인 중간에 삽입 |

```kotlin
// 표준 라이브러리 정의 (inline이므로 런타임 오버헤드 없음)
inline fun <T, R> T.let(block: (T) -> R): R = block(this)
inline fun <T, R> T.run(block: T.() -> R): R = block()
inline fun <T> T.apply(block: T.() -> Unit): T { block(); return this }
inline fun <T> T.also(block: (T) -> Unit): T { block(this); return this }
inline fun <T, R> with(receiver: T, block: T.() -> R): R = receiver.block()
```

```
                     반환값이 람다 결과          반환값이 객체 자신
                   ┌──────────────────┬──────────────────┐
  this로 참조      │   run  /  with   │      apply       │
                   ├──────────────────┼──────────────────┤
  it 으로 참조     │       let        │      also        │
                   └──────────────────┴──────────────────┘
```

<br>

### 5. 선택 기준과 관용적 사용법

**상황별 선택**

- **널 처리 + 변환**: `user?.let { it.name.uppercase() }` — null이면 건너뛰고, 결과가 필요할 때 (unit01 참고)
- **객체 생성 후 속성 설정**: `Intent().apply { action = ...; putExtra(...) }` — 설정 대상이 `this`이므로 이름을 반복하지 않고, 설정된 객체를 그대로 돌려받음
- **체인 중간에 부수 효과**: `.also { log.debug("결과: $it") }` — 객체를 건드리지 않고 통과시킴. `it`을 쓰므로 바깥 `this`와 혼동이 없음
- **여러 멤버 호출을 묶어 결과 계산**: `with(canvas) { drawLine(); drawText(); width * height }`
- **초기화 블록이 값을 계산**: `val config = loadFile().run { parse(this).validated() }`

```kotlin
// 안티패턴: 임시 변수와 반복되는 이름, 널 검사가 뒤섞임
fun buildRequest(url: String?): Request? {
    if (url == null) return null
    val builder = Request.Builder()
    builder.url(url)
    builder.header("Accept", "application/json")
    val request = builder.build()
    log.debug("request=$request")
    return request
}

// 개선: 각 단계의 의도가 스코프 함수 이름으로 드러남
fun buildRequest(url: String?): Request? =
    url?.let { u ->                                   // null이면 전체가 null
        Request.Builder().apply {                     // 설정: this로 접근, 자기 자신 반환
            url(u)
            header("Accept", "application/json")
        }.build()
    }?.also { log.debug("request=$it") }              // 부수 효과 후 그대로 반환
```

**`this`와 `it`의 선택**

- 람다가 **객체의 멤버를 여러 번 호출**한다면 `this` 계열(`run`·`apply`·`with`)이 간결함
- 람다 안에서 **바깥 클래스의 멤버도 함께 사용**하거나 객체를 **인자로 넘겨야** 한다면 `it` 계열(`let`·`also`)이 안전함. `this` 계열을 중첩하면 어느 `this`인지 헷갈리기 쉬움
- `it` 대신 의미 있는 이름(`user ->`)을 붙이면 가독성이 더 좋아짐

```kotlin
// 안티패턴: apply 중첩으로 this가 가리키는 대상이 불명확
outer.apply {
    inner.apply {
        name = "x"        // outer.name? inner.name? — 안쪽 inner가 우선하지만 읽는 사람은 헷갈림
    }
}

// 개선: 바깥 객체는 it 계열로 받아 이름을 붙임
outer.also { o ->
    o.inner.apply { name = "x" }
}
```

> ⚠️ 스코프 함수는 **가독성을 위해** 있는 도구다. 세 단계 이상 중첩되거나, 한 블록에서 `this`와 `it`이 섞이거나, 단순 대입 한 줄에 `let`을 쓰는 코드는 오히려 읽기 어려워진다. 코틀린 공식 문서도 "과도한 사용은 피하라"고 명시한다.

<br>

### 6. 자주 하는 실수

- **`let`으로 널 검사를 대체**: `x?.let { ... } ?: run { ... }`처럼 if-else를 흉내 내면, 람다가 null을 반환할 때 else 분기도 실행되는 함정이 있음. 분기가 필요하면 그냥 `if (x != null)`을 쓴다
- **`apply`의 반환값 무시**: `apply`는 객체를 돌려주므로 `val x = obj.apply { }`의 `x`는 계산 결과가 아니라 `obj` 자신임. 결과가 필요하면 `run`
- **`also`에서 객체 수정**: `also`는 부수 효과용이며 안에서 객체를 바꾸면 체인의 의미가 흐려짐. 수정은 `apply`
- **확장 함수를 다형성 대체로 사용**: 정적 디스패치 때문에 상위 타입 변수로 호출하면 상위 확장이 선택됨

<br>

### 7. 정리 — 면접·실무 체크포인트

| **질문**                                       | **핵심 답변**                                                                  |
| ---------------------------------------------- | ------------------------------------------------------------------------------ |
| 확장 함수는 어떻게 컴파일되는가?               | 수신 객체를 첫 인자로 받는 **정적 메서드**. 클래스는 바뀌지 않으며 private 접근 불가 |
| 확장 함수와 멤버 함수가 이름이 같으면?         | **멤버가 항상 우선**. 확장은 무시되고 컴파일러 경고                            |
| 상위 타입 변수로 하위 타입 확장을 호출하면?    | **선언 타입 기준(정적 디스패치)**이라 상위 타입의 확장이 호출됨                 |
| `let`과 `run`의 차이는?                        | 참조 방식이 `it` vs `this`. 둘 다 **람다 결과**를 반환                          |
| `apply`와 `also`의 차이는?                     | 둘 다 **객체 자신** 반환. `apply`는 `this`로 설정용, `also`는 `it`으로 부수 효과용 |
| `with`만 다른 점은?                            | 확장 함수가 아니라 **객체를 인자로** 받는 일반 함수                             |
| 스코프 함수를 쓰지 말아야 할 때는?             | 중첩이 깊어 `this`가 모호할 때, 단순 대입 한 줄일 때, if-else를 흉내 낼 때       |

- 스코프 함수는 모두 `inline`이라 성능 차이는 없고 **의도를 이름으로 드러내는 것**이 선택의 전부임
- 널 처리와 결합한 관용구는 **unit01(Null Safety 설계)**을 참고할 것
