## 프로토콜 지향 프로그래밍

**프로토콜 지향 프로그래밍(POP, Protocol-Oriented Programming)**은 클래스 상속 대신 **프로토콜 + 확장(extension)**으로 동작을 조합하는 Swift의 핵심 설계 방식이다. 이 유닛에서는 확장 기본 구현이 재사용을 어떻게 대체하는지, 그 뒤에 숨은 **메서드 디스패치** 규칙이 만드는 함정, 그리고 프로토콜을 타입으로 쓸 때 필요한 **타입 소거(Type Erasure)**를 정리한다.

<br>

### 1. 상속의 한계와 프로토콜의 역할

- 클래스 상속은 **단일 상속**이라 여러 능력을 조합하기 어렵고, 값 타입(`struct`·`enum`)은 아예 상속할 수 없음
- 부모 클래스가 커질수록 자식이 쓰지도 않는 상태와 메서드를 물려받는 **거대 기반 클래스** 문제가 생김
- 프로토콜은 "무엇을 할 수 있는가"라는 **요구사항(requirement)**만 정의하고, 값 타입·참조 타입·열거형 어디에나 채택할 수 있으며 **여러 개를 동시에** 채택할 수 있음

```
클래스 상속 (is-a 계층)          프로토콜 조합 (can-do 능력)
      Animal                        Flyable   Swimmable   Walkable
     /      \                          \        |         /
   Bird     Fish                        Duck: Flyable, Swimmable, Walkable
    |                                   Penguin: Swimmable, Walkable
   Duck (수영은? 다중 상속 불가)         → 필요한 능력만 골라 조합
```

> 💡 WWDC 2015 "Protocol-Oriented Programming in Swift"에서 Apple이 제시한 원칙은 "**클래스가 아니라 프로토콜로 시작하라**"다. 표준 라이브러리 자체가 `Equatable`·`Hashable`·`Collection` 같은 프로토콜 계층 위에 구축되어 있다.

<br>

### 2. 확장 기본 구현(Default Implementation)

프로토콜 확장(`extension Protocol`)에 메서드 본문을 작성하면, 채택 타입이 직접 구현하지 않아도 그 구현을 **기본값으로** 사용한다. 상속 없이도 코드를 공유할 수 있는 POP의 핵심 장치다.

```swift
protocol Describable {
    var name: String { get }
    func describe() -> String        // 요구사항
}

extension Describable {
    func describe() -> String {      // 기본 구현
        "이름: \(name)"
    }
}

struct Book: Describable {
    let name: String                 // describe()는 기본 구현 사용
}

struct Movie: Describable {
    let name: String
    func describe() -> String {      // 기본 구현을 덮어씀
        "영화 <\(name)>"
    }
}
```

- 조건부 확장(`extension Collection where Element: Numeric`)으로 **특정 조건을 만족하는 타입에만** 기능을 추가할 수 있음
- 기본 구현은 **다른 프로토콜의 요구사항을 조합**해 만들 수 있음. 예를 들어 `Comparable`의 `<`만 구현하면 `>`, `<=`, `>=`는 표준 라이브러리 확장이 자동 제공함

> ⚠️ 기본 구현이 "덮어쓰기" 되는지는 그 메서드가 **프로토콜 요구사항으로 선언되어 있느냐**에 달려 있다. 확장에만 있고 프로토콜 본문에 없는 메서드는 동적 디스패치 대상이 아니므로, 채택 타입이 같은 이름을 구현해도 상황에 따라 호출되지 않는다. 다음 절에서 자세히 다룬다.

<br>

### 3. 메서드 디스패치

### 3-1. 세 가지 디스패치 방식

**메서드 디스패치(method dispatch)**는 호출할 함수의 실제 주소를 언제, 어떻게 결정하는가의 문제다.

| **방식**                       | **결정 시점**            | **적용 대상**                                                        | **성능**                       |
| ------------------------------ | ------------------------ | -------------------------------------------------------------------- | ------------------------------ |
| **정적(Static/Direct)**        | **컴파일 시점**          | 값 타입 메서드, `final`·`private` 메서드, **프로토콜 확장 전용 메서드** | 가장 빠름, 인라이닝 가능       |
| **테이블(Table)**              | 런타임 (테이블 조회)     | 클래스의 재정의 가능 메서드(vtable), **프로토콜 요구사항**(witness table) | 포인터 한 번 따라감            |
| **메시지(Message)**            | 런타임 (셀렉터 탐색)     | `@objc dynamic` 메서드 (Objective-C 런타임)                          | 가장 느림, 메서드 스위즐링 가능 |

- 프로토콜 요구사항은 채택 타입마다 생성되는 **프로토콜 위트니스 테이블(Protocol Witness Table)**을 통해 호출되므로, 프로토콜 타입 변수로 호출해도 실제 타입의 구현이 실행됨
- 프로토콜 **확장에만 있는 메서드**는 위트니스 테이블에 항목이 없으므로, 컴파일러는 변수의 **정적 타입**만 보고 호출 대상을 고정함

<br>

### 3-2. 확장 메서드 디스패치 함정

```swift
protocol Greeter {
    func hello() -> String            // ① 요구사항
}
extension Greeter {
    func hello() -> String { "기본 hello" }
    func bye()   -> String { "기본 bye" }     // ② 확장 전용 (요구사항 아님)
}
struct Korean: Greeter {
    func hello() -> String { "안녕하세요" }
    func bye()   -> String { "안녕히 가세요" }
}

let k = Korean()
let g: Greeter = Korean()
k.hello()   // "안녕하세요"   — 정적 타입 Korean
g.hello()   // "안녕하세요"   — 요구사항 → 위트니스 테이블로 동적 디스패치
k.bye()     // "안녕히 가세요" — 정적 타입 Korean
g.bye()     // "기본 bye"     — 요구사항이 아니므로 정적 타입 Greeter의 확장 구현 호출!
```

```
g.hello() 호출 흐름                    g.bye() 호출 흐름
  g: Greeter (existential)               g: Greeter (existential)
  → 위트니스 테이블 조회                  → 프로토콜 본문에 bye 없음
  → Korean.hello 실행                     → 컴파일 시점에 extension Greeter.bye로 고정
```

> ⚠️ 이 함정을 피하는 규칙은 하나다. **채택 타입이 덮어쓸 여지가 있는 메서드는 반드시 프로토콜 본문에 요구사항으로 선언**하고, 확장에는 기본 구현만 둔다. 확장 전용 메서드는 "모든 채택 타입에 동일하게 동작하는 유틸리티"로만 사용해야 한다.

- 클래스도 마찬가지로, `final`이나 `private`을 붙이면 vtable을 거치지 않는 정적 디스패치가 되어 성능이 향상되고 의도(재정의 금지)도 드러남. **Whole Module Optimization**이 켜져 있으면 컴파일러가 모듈 내 재정의 없는 클래스를 자동으로 `final` 취급함

<br>

### 4. 프로토콜을 타입으로 쓸 때 — existential과 타입 소거

**existential 컨테이너**

`let g: Greeter`처럼 프로토콜을 **값의 타입**으로 쓰면, 컴파일러는 실제 타입을 모르므로 값을 **existential 컨테이너**에 담는다. 이 컨테이너는 값 버퍼(3워드) + 타입 메타데이터 + 위트니스 테이블 포인터로 구성되며, 값이 버퍼보다 크면 힙에 박싱된다(unit01 참고).

- 장점: 서로 다른 타입을 **하나의 배열**(`[Greeter]`)에 섞어 담고 균일하게 호출할 수 있음
- 비용: 박싱·간접 호출 오버헤드, 그리고 컴파일러가 특수화(specialization) 최적화를 하지 못함
- Swift 5.6부터 existential 타입은 `any Greeter`로 명시할 수 있음. 원래 Swift 6에서 필수화할 계획이었으나 **보류되어** Swift 6 언어 모드에서도 생략이 허용되며, `ExistentialAny` 업커밍 기능 플래그로 강제할 수 있음(unit05 참고)

<br>

### 4-1. 타입 소거(Type Erasure)

`associatedtype`이나 `Self`를 사용하는 프로토콜은 구체 타입이 정해져야 의미가 있으므로, Swift 5.7 이전에는 아예 값의 타입으로 쓸 수 없었다(`Protocol 'X' can only be used as a generic constraint`). 이때 구체 타입 정보를 지우고 **하나의 구체 래퍼 타입**으로 감싸는 기법이 타입 소거다. 표준 라이브러리의 `AnyHashable`, `AnySequence`, Combine의 `AnyPublisher`가 대표 사례다.

```swift
protocol Storage {
    associatedtype Item
    func save(_ item: Item)
}

struct MemoryStorage: Storage { func save(_ item: String) { print("메모리:", item) } }
struct DiskStorage: Storage   { func save(_ item: String) { print("디스크:", item) } }

// 타입 소거 래퍼: 구체 타입을 클로저 뒤로 숨긴다
struct AnyStorage<Item>: Storage {
    private let _save: (Item) -> Void
    init<S: Storage>(_ base: S) where S.Item == Item {
        _save = base.save            // 구체 타입 S는 여기서 사라짐
    }
    func save(_ item: Item) { _save(item) }
}

let stores: [AnyStorage<String>] = [AnyStorage(MemoryStorage()), AnyStorage(DiskStorage())]
stores.forEach { $0.save("data") }   // 서로 다른 구현을 같은 배열에 담아 호출
```

> 💡 Swift 5.7부터는 **기본 연관 타입(primary associated type)**을 선언하면 `any Storage<String>`처럼 래퍼 없이 existential을 쓸 수 있어, 수동 타입 소거의 필요성이 크게 줄었다. 그래도 API 안정성을 위해 구체 타입을 숨기거나(`AnyPublisher`), 5.7 이전 버전을 지원해야 할 때는 여전히 유효한 기법이다.

<br>

### 5. 상속 vs 프로토콜 선택 기준

| **기준**                         | **클래스 상속**                          | **프로토콜 + 확장**                                    |
| -------------------------------- | ---------------------------------------- | ------------------------------------------------------ |
| **적용 타입**                    | 클래스만                                 | **struct·enum·class 모두**                             |
| **다중 조합**                    | 불가 (단일 상속)                         | **여러 프로토콜 동시 채택**                            |
| **상태(저장 프로퍼티) 공유**     | **가능**                                 | 불가 — 확장은 계산 프로퍼티만 추가 가능                |
| **동작 재정의**                  | `override` (vtable)                      | 요구사항만 재정의 가능 (witness table)                 |
| **테스트 대역(Mock) 만들기**     | 서브클래스 필요, `final`이면 불가        | **프로토콜 채택만으로 가능**                           |
| **적합한 경우**                  | UIKit 서브클래싱, 공통 상태가 많은 계층  | 의존성 추상화, 값 타입 확장, 능력 조합                 |

- 저장 프로퍼티를 공유해야 하면 프로토콜 확장으로는 불가능하므로, **컴포지션**(프로토콜 채택 타입이 공통 구조체를 프로퍼티로 보유)으로 해결하는 것이 일반적임
- 의존성 주입과 단위 테스트를 고려하면, 외부 서비스(네트워크·저장소)는 **프로토콜로 추상화**하는 것이 기본 선택임

<br>

### 6. 면접·실무 체크포인트

| **질문**                                                       | **핵심 답변**                                                                                      |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **POP가 상속보다 나은 점은?**                                  | 값 타입 적용, 다중 조합, 기본 구현으로 코드 공유, 테스트 대역 용이                                 |
| **프로토콜 확장 메서드가 재정의되지 않는 경우는?**             | 요구사항이 아닌 확장 전용 메서드를 프로토콜 타입 변수로 호출할 때 — 정적 디스패치                  |
| **Swift의 디스패치 방식 3가지는?**                             | 정적(값 타입·final), 테이블(vtable·witness table), 메시지(`@objc dynamic`)                         |
| **existential 타입의 비용은?**                                 | 컨테이너 박싱, 간접 호출, 특수화 불가                                                              |
| **타입 소거는 언제 필요한가?**                                 | 연관 타입 프로토콜을 균일한 타입으로 다뤄야 할 때. 5.7+에서는 `any P<T>`로 대체 가능한 경우가 많음 |
| **프로토콜 확장으로 저장 프로퍼티를 추가할 수 있나?**          | 없다. 계산 프로퍼티만 가능하며, 상태는 컴포지션으로 해결                                           |

- POP의 핵심은 "**요구사항은 프로토콜 본문에, 공통 구현은 확장에**"이며, 이 구분이 디스패치 동작을 결정함
- 프로토콜을 **제약(generic constraint)**으로 쓰면 정적 디스패치·특수화의 이점을, **타입(existential)**으로 쓰면 이질적 값을 담는 유연성을 얻음 — 이 선택은 **unit05(제네릭과 associatedtype)**에서 `some`과 `any`로 이어짐
