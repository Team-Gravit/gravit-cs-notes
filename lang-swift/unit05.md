## 제네릭과 associatedtype

**제네릭(Generics)**은 타입을 매개변수화해 "어떤 타입이든 같은 로직으로 처리"하면서도 컴파일 타임 타입 안전성을 유지하는 기법이며, `associatedtype`은 프로토콜 쪽의 제네릭이다. 이 유닛에서는 제약 조건(`where` 절)으로 제네릭을 유용하게 만드는 방법, 그리고 Swift 5.7 이후 프로토콜을 다루는 두 가지 방식인 **`some`(불투명 타입)과 `any`(existential)**의 차이와 선택 기준을 정리한다.

<br>

### 1. 제네릭이 필요한 이유

```swift
// 안티패턴: 타입마다 같은 함수를 복제하거나, Any로 받아 타입 안전성을 버림
func swapInts(_ a: inout Int, _ b: inout Int) { let t = a; a = b; b = t }
func swapAny(_ a: inout Any, _ b: inout Any) { let t = a; a = b; b = t }   // 호출부에서 캐스팅 지옥

// 개선: 타입 매개변수 T — 호출 시점의 구체 타입으로 특수화되며 안전성 유지
func swapValues<T>(_ a: inout T, _ b: inout T) { let t = a; a = b; b = t }

var x = 1, y = 2
swapValues(&x, &y)             // T = Int 로 추론
```

- `<T>`의 `T`를 **타입 매개변수(type parameter)**라 하며, 호출부의 인자로부터 추론됨
- 컴파일러는 실제 사용된 타입별로 코드를 생성하는 **특수화(specialization)**를 수행할 수 있어 `Any`·existential보다 빠름
- `Array<Element>`, `Dictionary<Key, Value>`, `Optional<Wrapped>` 등 표준 라이브러리 컬렉션은 모두 제네릭 타입임

<br>

### 2. 제약 조건(Constraints)

아무 제약이 없는 `T`로는 비교·해싱·연산을 할 수 없다. 제약을 걸어 "T가 이런 능력을 가진다"고 알려 주어야 그 능력을 쓸 수 있다.

```swift
// 타입 매개변수 제약: T는 Comparable이어야 <를 쓸 수 있다
func largest<T: Comparable>(_ items: [T]) -> T? {
    items.max()
}

// where 절: 더 복잡한 관계 표현 — 두 컬렉션의 요소 타입이 같고 Equatable일 것
func haveSameElements<A: Collection, B: Collection>(_ a: A, _ b: B) -> Bool
    where A.Element == B.Element, A.Element: Equatable {
    a.count == b.count && zip(a, b).allSatisfy { $0 == $1 }
}

// 조건부 확장: 요소가 숫자일 때만 sum 제공
extension Array where Element: Numeric {
    func sum() -> Element { reduce(0, +) }
}
```

| **제약 형태**                     | **문법 예시**                                  | **의미**                                          |
| --------------------------------- | ---------------------------------------------- | ------------------------------------------------- |
| **프로토콜 준수**                 | `<T: Hashable>`                                | T는 Hashable을 채택해야 함                        |
| **클래스 상속**                   | `<T: UIView>`                                  | T는 UIView 또는 그 서브클래스                     |
| **연관 타입 동일성**              | `where A.Element == B.Element`                 | 두 연관 타입이 같은 구체 타입                     |
| **연관 타입 준수**                | `where S.Element: Comparable`                  | 연관 타입이 프로토콜을 채택                       |
| **구체 타입 고정**                | `where Element == String`                      | 연관 타입을 특정 타입으로 제한 (조건부 확장에서)  |

> 💡 제약은 "제한"이 아니라 "**능력 부여**"로 이해하는 편이 정확하다. 제약이 없으면 `T`로 할 수 있는 일은 저장·전달뿐이고, `Comparable` 제약을 거는 순간 `<`, `max()`, `sorted()` 등이 열린다. 필요한 최소 제약만 거는 것이 재사용성을 높인다.

<br>

### 3. associatedtype — 프로토콜의 제네릭

프로토콜은 `<T>` 문법을 쓰지 않는다. 대신 **`associatedtype`**으로 "채택 타입이 나중에 정할 타입 자리"를 선언한다. 표준 라이브러리의 `IteratorProtocol.Element`, `Collection.Index`가 대표적이다.

```swift
protocol Container {
    associatedtype Item: Equatable          // 연관 타입에도 제약 가능
    var count: Int { get }
    mutating func append(_ item: Item)
    subscript(i: Int) -> Item { get }
}

struct IntStack: Container {
    // typealias Item = Int  ← append의 매개변수 타입에서 자동 추론되므로 생략 가능
    private var items: [Int] = []
    var count: Int { items.count }
    mutating func append(_ item: Int) { items.append(item) }
    subscript(i: Int) -> Int { items[i] }
}
```

- 채택 타입은 `typealias`로 명시하거나, 요구사항 구현의 시그니처로부터 **추론**시킬 수 있음
- `Self` 요구사항(예: `Equatable`의 `static func ==(lhs: Self, rhs: Self)`)도 연관 타입과 비슷하게 "구체 타입이 정해져야 의미 있는" 요구사항임
- 연관 타입이 있는 프로토콜을 **값의 타입**으로 쓰는 것은 Swift 5.7 전까지 금지되었고, 이를 우회하는 타입 소거 기법은 unit04 참고

<br>

### 3-1. 기본 연관 타입(Primary Associated Type)

Swift 5.7(SE-0346)부터 프로토콜 이름 뒤에 `<...>`로 **기본 연관 타입**을 선언하면, 제네릭 타입처럼 `Container<Int>`로 특정할 수 있다.

```swift
protocol Container<Item> {                // Item을 기본 연관 타입으로 지정
    associatedtype Item: Equatable
    mutating func append(_ item: Item)
}

func fill(_ c: inout some Container<Int>) { c.append(1) }   // where 절 없이 Item == Int 제약
let anyContainers: [any Container<String>] = []             // existential에도 적용 가능
```

> ⚠️ `Container<Int>`는 제네릭 타입 인스턴스화가 아니라 **제약의 축약 표현**이다. `where C.Item == Int`를 짧게 쓴 것이므로, 반드시 `some` 또는 `any` 뒤에서 사용해야 한다. 프로토콜은 여전히 타입 매개변수를 갖지 않는다.

<br>

### 4. some과 any

### 4-1. 동작 원리

| **항목**             | **`some P` (불투명 타입, Opaque Type)**                       | **`any P` (existential 타입)**                                  |
| -------------------- | ------------------------------------------------------------- | --------------------------------------------------------------- |
| **구체 타입**        | **하나로 고정** — 컴파일러는 알고, 호출자에게는 숨김           | 실행 중 **여러 타입**이 올 수 있음                              |
| **정체성 유지**      | 유지됨 — `Self`·연관 타입 관계를 계속 추적                    | 사라짐 — 값을 컨테이너에 박싱                                   |
| **디스패치**         | 정적, 특수화·인라이닝 가능                                    | 위트니스 테이블을 통한 간접 호출                                |
| **성능**             | **빠름**                                                      | 박싱·간접 호출 비용                                             |
| **이질적 컬렉션**    | 불가 — `[some P]`의 요소는 모두 같은 타입                     | **가능** — `[any P]`에 서로 다른 타입 혼합                      |
| **도입 버전**        | 반환 위치 5.1 (SE-0244), 매개변수 위치 5.7 (SE-0341)          | 5.6 (SE-0335), 모든 프로토콜에 사용 가능 5.7 (SE-0309)          |

```swift
// some: 반환 타입을 숨기되 "항상 같은 하나의 타입"임을 컴파일러가 보장
func makeSequence() -> some Sequence<Int> { [1, 2, 3] }     // 실제로는 [Int]

// 매개변수 위치의 some은 제네릭의 축약 표현
func printAll(_ items: some Collection<String>) { items.forEach { print($0) } }
// 위와 동일: func printAll<C: Collection<String>>(_ items: C)

// any: 서로 다른 구체 타입을 한 배열에 담을 때
let shapes: [any Shape] = [Circle(), Square()]
```

```
some Shape                             any Shape
┌────────────────────┐                 ┌───────────────────────────┐
│ 실제 타입: Circle  │ ← 컴파일러만 앎  │ [값 버퍼|메타데이터|PWT]   │ ← 런타임 박스
│ 호출자: "어떤 Shape"│                 │  Circle 또는 Square 또는 …│
└────────────────────┘                 └───────────────────────────┘
정적 디스패치, 타입 정체성 보존         동적 디스패치, 타입 정체성 소거
```

> 💡 SwiftUI의 `var body: some View`가 대표 사례다. 구체 타입(`VStack<TupleView<...>>`)이 수십 줄이라 적을 수 없지만 "항상 같은 하나의 타입"이므로 `some`으로 숨긴다. 만약 `if` 분기로 서로 다른 뷰를 반환하면 `some`의 "하나의 타입" 조건이 깨져 컴파일 오류가 나고, `@ViewBuilder`나 `AnyView`(타입 소거)가 필요해진다.

<br>

### 4-2. 선택 기준과 함정

- **기본값은 `some`(또는 명시적 제네릭)**: 성능이 좋고 타입 관계가 보존됨. 한 함수 안에서 타입이 하나로 정해지는 대부분의 경우
- **`any`가 필요한 경우**: 이질적 타입을 한 컬렉션에 담거나, 프로퍼티에 런타임에 다른 구현을 교체해 저장할 때(의존성 주입의 `let repository: any UserRepository`)
- Swift 5.7의 **암묵적 existential 열기(SE-0352)** 덕분에 `any P` 값을 `some P` 매개변수 함수에 그대로 넘길 수 있음. "`any`로 보관하다가 처리 함수에서는 `some`으로 받는" 조합이 실무 관용구임
- `any` 키워드는 5.6에서 도입되었고 원래 Swift 6에서 필수화할 예정이었으나 **보류**되어, Swift 6 언어 모드에서도 생략이 허용됨. `-enable-upcoming-feature ExistentialAny`로 강제할 수 있으며, 명시하는 습관이 권장됨(버전에 따라 경고 여부는 다를 수 있음)

> ⚠️ `any P` 값끼리는 `==`로 비교할 수 없다. `Equatable`은 `Self` 요구사항이라 서로 다른 구체 타입이 섞일 수 있는 existential에서는 성립하지 않기 때문이다. 비교가 필요하면 제네릭(`some`)으로 받거나 `AnyHashable` 같은 타입 소거 래퍼를 쓴다.

<br>

### 5. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                             |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **제네릭이 Any보다 나은 이유는?**                     | 컴파일 타임 타입 검사 유지 + 특수화로 성능 우위, 캐스팅 불필요                            |
| **associatedtype이 무엇인가?**                        | 프로토콜의 타입 자리표시자. 채택 타입이 typealias나 추론으로 확정                          |
| **some과 any의 차이는?**                              | `some`은 하나의 숨겨진 구체 타입(정적), `any`는 여러 타입을 담는 박스(동적)               |
| **`some View`를 쓰는 이유는?**                        | 거대한 구체 타입을 숨기면서 "단일 타입" 보장으로 성능·타입 관계 유지                      |
| **`[any Equatable]` 요소를 비교할 수 있나?**          | 없다. `Self` 요구사항은 existential에서 성립하지 않음                                     |
| **where 절은 언제 쓰나?**                             | 연관 타입 간 동일성·준수 등 `<T: P>`로 표현 못 하는 관계                                  |
| **기본 연관 타입(Primary Associated Type)이란?**      | `protocol P<Item>` 선언으로 `some P<Int>`·`any P<Int>` 축약 제약을 가능하게 하는 5.7 기능 |

- 제네릭은 **제약을 통해 능력을 부여**하고, 컴파일러는 특수화로 비용을 없앰
- 프로토콜의 제네릭은 `associatedtype`이며, 5.7의 기본 연관 타입으로 사용성이 크게 개선됨
- **`some`이 기본, `any`는 이질성이 필요할 때** — 이 판단은 unit04의 디스패치·existential 비용과 직결됨
