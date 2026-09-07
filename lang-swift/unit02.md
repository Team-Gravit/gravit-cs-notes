## ARC와 순환 참조

Swift는 참조 타입의 메모리를 **ARC(Automatic Reference Counting, 자동 참조 계수)**로 관리하며, 가비지 컬렉터 없이 컴파일 시점에 삽입된 `retain`/`release`로 인스턴스의 수명을 결정한다. ARC는 빠르고 예측 가능하지만 **순환 참조(Retain Cycle)**를 스스로 끊지 못하므로, `weak`·`unowned`와 클로저 캡처 리스트로 개발자가 직접 소유 관계를 설계해야 한다.

<br>

### 1. ARC의 동작 원리

- 클래스 인스턴스가 힙에 생성되면 **참조 카운트(reference count)**가 1로 시작함
- 그 인스턴스를 가리키는 강한 참조(strong reference)가 생길 때마다 +1, 참조가 사라질 때마다 −1
- 카운트가 **0이 되는 순간 즉시** `deinit`이 호출되고 메모리가 해제됨
- 카운트 증감 코드는 **컴파일러가 컴파일 시점에 삽입**하므로 런타임에 별도의 수집 스레드가 돌지 않음

```swift
final class Session {
    let id: Int
    init(id: Int) { self.id = id; print("init \(id)") }
    deinit { print("deinit \(id)") }
}

var a: Session? = Session(id: 1)   // rc = 1
var b = a                          // rc = 2
a = nil                            // rc = 1 → 아직 살아 있음
b = nil                            // rc = 0 → "deinit 1" 출력
```

| **항목**          | **ARC (Swift)**                          | **추적 GC (Java·Kotlin/JVM 등)**            |
| ----------------- | ---------------------------------------- | ------------------------------------------- |
| **해제 시점**     | 카운트 0이 되는 **즉시** (결정적)        | GC 실행 시점 (비결정적)                     |
| **런타임 비용**   | 참조 대입마다 원자적 증감                | 주기적 힙 스캔, 일시 정지(STW) 가능         |
| **순환 참조**     | **회수 불가** — 개발자가 끊어야 함       | 도달 불가능하면 자동 회수                   |
| **메모리 사용량** | 낮고 안정적                              | 스캔 전까지 쓰레기가 남아 상대적으로 높음   |

> 💡 "ARC는 GC인가?"라는 질문에는 "참조 카운팅 기반 자동 메모리 관리이지만, 추적(tracing) GC처럼 런타임이 도달 가능성을 분석하지 않고 컴파일러가 삽입한 코드로 동작하므로 순환 참조를 스스로 해결하지 못한다"고 답하면 된다. 값 타입은 ARC 대상이 아니라는 점도 함께 언급하자(unit01 참고).

<br>

### 2. 순환 참조(Retain Cycle)

두 인스턴스가 **서로를 강한 참조로 붙잡으면** 외부에서 참조를 모두 끊어도 카운트가 0이 되지 않아 영원히 해제되지 않는다. 이것이 Swift에서 메모리 누수의 가장 흔한 원인이다.

```swift
final class Person {
    let name: String
    var apartment: Apartment?
    init(name: String) { self.name = name }
    deinit { print("Person 해제") }
}
final class Apartment {
    let unit: String
    var tenant: Person?          // ← 강한 참조: 순환 발생
    init(unit: String) { self.unit = unit }
    deinit { print("Apartment 해제") }
}

var p: Person? = Person(name: "kim")
var apt: Apartment? = Apartment(unit: "101")
p!.apartment = apt
apt!.tenant = p
p = nil; apt = nil               // 아무것도 출력되지 않음 → 누수
```

```
외부 변수 p, apt 를 nil 로 끊어도…

  ┌────────────┐  apartment (strong)  ┌──────────────┐
  │ Person     │ ───────────────────→ │ Apartment    │
  │ rc = 1     │ ←─────────────────── │ rc = 1       │
  └────────────┘   tenant (strong)    └──────────────┘
         서로가 서로를 붙잡아 카운트가 0이 되지 않음
```

<br>

### 3. 순환을 끊는 도구 — weak와 unowned

**세 가지 참조의 비교**

| **항목**            | **strong (기본)**       | **weak**                                   | **unowned**                                       |
| ------------------- | ----------------------- | ------------------------------------------ | ------------------------------------------------- |
| **참조 카운트**     | **+1**                  | 증가 없음                                  | 증가 없음                                         |
| **대상 해제 시**    | 해제되지 않게 막음      | 자동으로 **nil**이 됨                      | 댕글링 → 접근 시 **크래시**                       |
| **선언 타입**       | 아무 타입               | 반드시 **옵셔널 `var`**                    | 비옵셔널 가능 (`let`도 가능)                      |
| **적합한 상황**     | 소유 관계               | 상대가 **먼저 사라질 수 있을 때**          | 상대의 수명이 **나보다 같거나 길다고 확신할 때**  |
| **대표 예시**       | 부모 → 자식             | delegate, 자식 → 부모                      | 신용카드 → 고객 (카드는 고객 없이 존재 불가)      |

- 위 예시에서 `Apartment.tenant`를 `weak var tenant: Person?`으로 바꾸면 순환이 끊어져 두 `deinit`이 모두 호출됨
- `weak`는 런타임이 사이드 테이블로 추적하며 대상 해제 시 nil로 바꾸므로 약간의 비용이 있고, `unowned`는 그 비용이 없는 대신 안전장치도 없음

> ⚠️ `unowned`는 "이 참조를 쓸 때 상대가 반드시 살아 있다"는 **개발자의 약속**이다. 약속이 깨지면 `Fatal error: Attempted to read an unowned reference but the object was already deallocated`로 즉시 크래시한다. 확신이 없으면 `weak`를 쓰고 옵셔널 바인딩으로 처리하는 것이 안전하다(unit03 참고).

<br>

### 3-1. delegate 패턴과 weak

UIKit의 delegate가 관례적으로 `weak var delegate: SomeDelegate?`로 선언되는 이유가 바로 순환 참조 방지다. 뷰(자식)가 뷰 컨트롤러(부모)를 delegate로 붙잡고, 뷰 컨트롤러는 뷰를 강하게 소유하므로 delegate까지 강하게 잡으면 순환이 된다.

```swift
protocol DataLoaderDelegate: AnyObject {      // weak를 쓰려면 클래스 전용 프로토콜이어야 함
    func didLoad(_ data: [String])
}

final class DataLoader {
    weak var delegate: DataLoaderDelegate?     // 자식 → 부모는 weak
    func load() { delegate?.didLoad(["a", "b"]) }
}
```

> 💡 `weak`는 클래스 인스턴스에만 붙일 수 있으므로, delegate 프로토콜은 `AnyObject`를 상속해 **클래스 전용**으로 제한해야 한다. 이 제약을 빠뜨리면 `'weak' must not be applied to non-class-bound protocol` 컴파일 오류가 난다.

<br>

### 4. 클로저와 캡처 리스트

### 4-1. 클로저가 만드는 순환

클로저는 참조 타입이며, 본문에서 사용하는 외부 변수를 **캡처(capture)**해 강하게 붙잡는다. 클래스 인스턴스가 클로저를 프로퍼티로 저장하고, 그 클로저가 `self`를 캡처하면 `self → 클로저 → self`의 순환이 생긴다.

```swift
final class Downloader {
    var onComplete: (() -> Void)?
    var progress = 0

    func start() {
        onComplete = {
            self.progress = 100          // self 강한 캡처 → self → onComplete → self 순환
        }
    }
    deinit { print("Downloader 해제") }
}
```

```
  ┌───────────────┐   onComplete (strong)   ┌────────────┐
  │ Downloader    │ ──────────────────────→ │ 클로저     │
  │ (self)        │ ←────────────────────── │ 캡처 컨텍스트│
  └───────────────┘   self 캡처 (strong)    └────────────┘
```

<br>

### 4-2. 캡처 리스트로 해결

클로저 매개변수 앞의 대괄호 `[weak self]` / `[unowned self]`가 **캡처 리스트(capture list)**이며, 캡처 방식을 지정한다.

```swift
func start() {
    onComplete = { [weak self] in
        guard let self else { return }   // Swift 5.7+: 축약 바인딩으로 self 승격
        self.progress = 100
    }
}
```

- `[weak self]`로 캡처하면 클로저 안의 `self`는 옵셔널이 되므로 `guard let self`로 언래핑해 사용함. 클로저 실행 시점에 이미 해제됐다면 조용히 빠져나감
- `[unowned self]`는 언래핑이 필요 없지만 `self`가 먼저 해제되면 크래시함. 클로저 수명이 `self`보다 짧다고 **확실**할 때만 사용
- 캡처 리스트는 값을 **클로저 생성 시점에** 복사한다. `[x]`처럼 값 타입을 넣으면 그 순간의 값이 고정됨

> ⚠️ 모든 클로저에 `[weak self]`를 붙이는 것은 과잉이다. `DispatchQueue.main.async { ... }`, `UIView.animate`, `Task { ... }`처럼 **클로저를 저장하지 않고 즉시 실행 후 버리는 경우**에는 순환이 생기지 않는다. 다만 클로저가 오래 살아남아 `self`의 해제를 지연시키는 것이 문제라면(예: 화면을 닫았는데 네트워크 콜백까지 뷰 컨트롤러가 유지됨) 그때는 `weak`가 필요하다. "순환 여부"와 "수명 연장 여부"를 구분해 판단하자.

<br>

### 5. 누수 진단과 예방 습관

- **`deinit`에 로그**를 남겨 화면을 닫았을 때 실제로 해제되는지 확인하는 것이 가장 빠른 검증법
- Xcode **Memory Graph Debugger**로 살아 있는 인스턴스와 그를 붙잡는 참조 경로를 시각적으로 확인함
- Instruments의 **Leaks**·**Allocations** 템플릿으로 장시간 사용 시 누수를 추적함
- 설계 단계에서 **소유자(owner)를 하나만** 정하고, 나머지 방향은 `weak`로 두는 것이 원칙임. 부모 → 자식은 strong, 자식 → 부모는 weak
- Swift 5.9 이후 `Task`·`async` 코드에서도 클로저 캡처 규칙은 동일하게 적용됨. 장시간 실행되는 `Task`에서 `self`를 강하게 잡으면 취소 전까지 해제가 미뤄짐(unit06 참고)

<br>

### 6. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                                       |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **ARC가 무엇인가?**                               | 컴파일러가 삽입한 retain/release로 참조 카운트를 관리, 0이 되면 즉시 해제. 값 타입은 대상이 아님    |
| **ARC와 GC의 차이는?**                            | 결정적 해제·낮은 오버헤드 vs 순환 참조를 스스로 못 끊음                                             |
| **weak와 unowned의 차이는?**                      | 둘 다 카운트 미증가. weak는 해제 시 nil(옵셔널 필수), unowned는 크래시(비옵셔널 가능)               |
| **언제 unowned를 쓰나?**                          | 상대의 수명이 나와 같거나 더 길다고 **보장**될 때. 확신이 없으면 weak                               |
| **delegate를 weak로 두는 이유는?**                | 부모가 자식을 강하게 소유하므로 자식 → 부모까지 강하면 순환 발생                                    |
| **모든 클로저에 [weak self]가 필요한가?**         | 아니다. 저장되지 않고 즉시 실행되는 클로저는 순환이 없다. 저장·장기 실행 클로저에만 필요            |

- 순환 참조의 정체는 "**서로를 강하게 붙잡는 두 참조 타입**"이며, 클로저는 참조 타입이라 `self` 캡처로 쉽게 순환을 만듦
- 해결의 핵심은 소유 방향을 정하고 **역방향을 weak/unowned로 바꾸는 것**
- 옵셔널이 된 `weak self`를 안전하게 다루는 방법은 **unit03(옵셔널 처리 전략)**에서 이어서 다룸
