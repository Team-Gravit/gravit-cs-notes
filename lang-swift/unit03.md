## 옵셔널 처리 전략

**옵셔널(Optional)**은 "값이 있을 수도, 없을 수도 있다"를 타입 시스템에 명시해 널 참조 오류를 컴파일 시점에 잡아내는 Swift의 핵심 장치다. 이 유닛에서는 옵셔널의 정체(`enum`)와 언래핑 도구인 **바인딩·체이닝·guard·nil 병합**의 동작을 살펴보고, **강제 언래핑(`!`)이 왜 위험한지**와 상황별 선택 기준을 정리한다.

<br>

### 1. 옵셔널의 정체 — 열거형

`Int?`는 문법 설탕(syntactic sugar)일 뿐, 실제로는 제네릭 열거형 `Optional<Int>`이다. 값이 없음을 `nil` 리터럴로 쓰지만 이는 `.none` 케이스이고, 값이 있으면 `.some(값)`으로 감싸져 있다.

```swift
// 표준 라이브러리의 정의 (요약)
enum Optional<Wrapped> {
    case none
    case some(Wrapped)
}

let a: Int? = 5            // Optional.some(5)
let b: Int? = nil          // Optional.none
print(a)                   // Optional(5)  ← 값 자체가 아니라 상자에 담긴 상태
```

```
       Optional<Int>
   ┌──────────────────┐
   │ .none            │  → nil
   │ .some(Wrapped)   │  → 상자 안에 5
   └──────────────────┘
   언래핑 = 상자를 열어 Wrapped 값을 꺼내는 행위
```

- 옵셔널 값은 상자에 담긴 상태이므로 그대로 산술이나 메서드 호출을 할 수 없음 → 반드시 **언래핑(unwrapping)** 과정을 거쳐야 함
- Objective-C의 `nil` 메시지 무시와 달리, Swift는 "없을 수 있음"을 **타입에 드러내어** 처리 누락을 컴파일 오류로 만듦

> 💡 "옵셔널이 왜 안전한가?"라는 질문에는 "값의 부재 가능성을 타입으로 표현하므로 컴파일러가 언래핑을 강제하고, 그 결과 널 참조가 런타임 크래시가 아니라 컴파일 오류로 드러난다"고 답하면 된다.

<br>

### 2. 언래핑 도구

### 2-1. 옵셔널 바인딩 — if let / guard let

값이 있으면 새 상수에 꺼내 담고, 없으면 분기한다. Swift 5.7부터는 같은 이름일 때 `if let name` 축약이 가능하다.

```swift
func greet(_ name: String?) {
    if let name {                     // Swift 5.7+ 축약 (이전: if let name = name)
        print("안녕하세요, \(name)")
    } else {
        print("이름이 없습니다")
    }
}
```

`guard let`은 **조기 탈출(early exit)** 전용이다. 바인딩된 상수가 `guard` 문 **이후 스코프 전체**에서 유효하므로 중첩이 사라지고 정상 흐름이 왼쪽에 정렬된다.

```swift
// 안티패턴: if let 중첩 → 피라미드
func process(_ input: String?) -> Int? {
    if let text = input {
        if let number = Int(text) {
            if number > 0 {
                return number * 2
            }
        }
    }
    return nil
}

// 개선: guard로 실패 조건을 위에서 걸러내고 정상 흐름은 평탄하게
func process(_ input: String?) -> Int? {
    guard let text = input,
          let number = Int(text),
          number > 0 else { return nil }
    return number * 2
}
```

> ⚠️ `guard`의 `else` 블록은 반드시 스코프를 벗어나야 한다(`return`, `throw`, `continue`, `break`, 또는 `fatalError`). 빠져나가지 않으면 `'guard' body must not fall through` 컴파일 오류가 난다.

<br>

### 2-2. 옵셔널 체이닝

`?.`을 사용하면 중간에 하나라도 `nil`이면 **전체 식이 nil**로 평가되고 이후 호출은 건너뛴다. 결과 타입은 항상 옵셔널이며, 체인이 길어도 옵셔널이 중첩되지 않고 한 겹으로 평탄화된다.

```swift
struct Address { var city: String? }
struct User { var address: Address? }

let user: User? = User(address: Address(city: "Seoul"))
let city = user?.address?.city        // String? — 어느 하나라도 nil이면 nil
let count = user?.address?.city?.count   // Int? (String?.count가 아니라 평탄화됨)
```

- 메서드 호출에도 적용됨: `delegate?.didFinish()`는 delegate가 nil이면 호출 자체를 생략함 (unit02의 weak delegate와 짝을 이룸)
- 반환값이 `Void`인 메서드에 체이닝하면 `Void?`가 되어, `if delegate?.didFinish() != nil`처럼 호출 성공 여부를 확인할 수 있음

<br>

### 2-3. nil 병합 연산자와 옵셔널 패턴

- **`??`(nil-coalescing)**: 값이 없을 때 **기본값**을 제공함. 오른쪽 피연산자는 `@autoclosure`라 필요할 때만 평가됨
- **`switch`의 `case .some` / `case let x?`**: 여러 옵셔널을 조합해 분기할 때 유용함
- **`map` / `flatMap`**: 언래핑 없이 옵셔널 안의 값을 변환함. 변환 결과가 다시 옵셔널이면 `flatMap`으로 평탄화

```swift
let port = Int(ProcessInfo.processInfo.environment["PORT"] ?? "") ?? 8080

let text: String? = "42"
let doubled = text.map { $0 + $0 }          // String? → "4242"
let number = text.flatMap { Int($0) }       // Int? — Int(String)이 옵셔널이므로 flatMap

switch (text, number) {
case let (t?, n?): print("둘 다 있음: \(t), \(n)")
case (nil, _):     print("텍스트 없음")
default:           print("숫자 변환 실패")
}
```

<br>

### 3. 강제 언래핑의 위험

`value!`는 "여기엔 반드시 값이 있다"는 단언이며, 실제로 `nil`이면 `Fatal error: Unexpectedly found nil while unwrapping an Optional value`로 앱이 **즉시 종료**된다. 컴파일러가 잡아 주던 안전성을 개발자가 스스로 포기하는 행위다.

```swift
// 안티패턴: 서버 응답 구조를 신뢰하고 강제 언래핑
let json: [String: Any] = ["name": "kim"]
let age = json["age"] as! Int          // 키가 없거나 타입이 다르면 크래시

// 개선: 조건부 캐스팅 + 기본값 또는 조기 탈출
let safeAge = json["age"] as? Int ?? 0
```

- **암시적 언래핑 옵셔널(IUO, `Int!`)**: 접근할 때마다 자동으로 `!`가 붙는 옵셔널. `@IBOutlet`처럼 "초기화 직후에는 nil이지만 사용 시점에는 반드시 있다"고 보장되는 곳에서만 사용함
- `as!`(강제 다운캐스팅)과 `try!`(오류 무시)도 같은 부류의 위험한 단언임. `try!`는 unit08 참고

<br>

### 3-1. 강제 언래핑이 허용되는 경우

| **상황**                                           | **허용 여부** | **이유·대안**                                                       |
| -------------------------------------------------- | ------------- | ------------------------------------------------------------------- |
| **직전 줄에서 nil 검사를 마친 값**                 | 지양          | 그냥 `if let`으로 바인딩하는 편이 리팩터링에 안전함                 |
| **`@IBOutlet` 등 프레임워크가 주입을 보장하는 값** | **허용**      | IUO 관례. 단, 뷰 로드 전에 접근하지 않도록 주의                     |
| **컴파일 타임에 값이 확정된 리터럴**               | **허용**      | `URL(string: "https://example.com")!` — 잘못됐다면 개발 중 즉시 발견 |
| **네트워크·파일·사용자 입력에서 온 값**            | **금지**      | 외부 데이터는 언제든 형식이 깨질 수 있음. `guard let` 필수          |
| **"nil이면 버그"라 확신하는 내부 불변 조건**       | 조건부 허용   | `precondition`이나 `guard ... else { assertionFailure() }`로 의도 명시 |

> ⚠️ `!`를 써서 크래시가 났을 때 문제는 크래시 자체가 아니라 **원인이 코드 어디에 있는지 드러나지 않는다**는 점이다. 스택 트레이스는 언래핑 지점만 가리키고, nil이 만들어진 근본 원인은 다른 곳에 있다. 크래시가 나야 한다면 `guard let ... else { preconditionFailure("userId가 nil — 로그인 상태 확인 필요") }`처럼 **메시지가 있는 실패**로 대체하자.

<br>

### 4. 도구별 선택 기준

| **도구**                | **값이 없을 때 동작**        | **적합한 상황**                                              |
| ----------------------- | ---------------------------- | ------------------------------------------------------------ |
| **`if let`**            | else 분기 실행               | 값의 유무에 따라 **양쪽 모두 처리 로직**이 있을 때           |
| **`guard let`**         | 스코프 탈출                  | 없으면 더 진행할 수 없는 **전제 조건** 검사 (함수 초입)      |
| **`?.` 체이닝**         | 식 전체가 nil                | 깊은 프로퍼티 접근, 선택적 delegate 호출                     |
| **`??`**                | 기본값 사용                  | 합리적 **기본값**이 존재할 때                                |
| **`map` / `flatMap`**   | nil 유지                     | 언래핑 없이 **변환**만 하고 옵셔널로 계속 흘려보낼 때        |
| **`switch` 패턴**       | 케이스별 분기                | 여러 옵셔널 조합, 열거형 연관값과 함께 분기                  |
| **`!` 강제 언래핑**     | **크래시**                   | 리터럴·프레임워크 보장값 등 극히 제한적                      |

**설계 관점의 원칙**

- 옵셔널은 **경계(boundary)에서 최대한 빨리 해소**하고, 내부 도메인 모델은 비옵셔널로 유지하는 것이 좋음. 디코딩·파싱 단계에서 `guard let`으로 검증하면 이후 계층은 옵셔널을 몰라도 됨
- "값이 없음"이 정말 정상 상태인지 되묻자. 없을 수 없는 값이라면 옵셔널 대신 **비옵셔널 + 생성자 검증**이나 **오류 throw**가 의도를 더 잘 드러냄(unit08 참고)
- 옵셔널의 옵셔널(`Int??`)은 대개 설계 오류의 신호이므로 `flatMap`이나 `??`로 평탄화하거나 타입을 재설계함

> 💡 `weak self`를 캡처한 클로저에서 `guard let self else { return }`이 관용구가 된 이유도 같은 원칙이다. 클로저 초입에서 한 번 해소하고, 이후 본문은 `self`가 확실히 살아 있다는 전제로 평탄하게 작성한다(unit02 참고).

<br>

### 5. 면접·실무 체크포인트

| **질문**                                     | **핵심 답변**                                                                          |
| -------------------------------------------- | -------------------------------------------------------------------------------------- |
| **옵셔널의 실제 타입은?**                    | `enum Optional<Wrapped>` — `.none`과 `.some(Wrapped)`                                  |
| **if let과 guard let의 차이는?**             | 스코프. `guard`는 이후 전체에서 유효하며 else에서 반드시 탈출해야 함                   |
| **옵셔널 체이닝의 결과 타입은?**             | 항상 옵셔널이며, 체인이 길어도 한 겹으로 평탄화됨                                      |
| **`??`의 오른쪽은 항상 평가되나?**           | 아니다. `@autoclosure`라 왼쪽이 nil일 때만 평가됨                                      |
| **강제 언래핑은 언제 허용되나?**             | 컴파일 타임에 확정된 리터럴, IUO 관례 정도. 외부 입력에는 금지                         |
| **map과 flatMap의 차이는?**                  | 변환 결과가 옵셔널이면 `flatMap`으로 `T??`를 `T?`로 평탄화                             |

- 옵셔널은 **부재 가능성을 타입으로 강제**하는 장치이며, 언래핑 도구는 "없을 때 무엇을 할 것인가"에 따라 고른다
- `guard let`으로 **경계에서 조기 해소**, 내부는 비옵셔널로 — 이것이 옵셔널 피로를 줄이는 핵심 전략
- `!`·`as!`·`try!`는 컴파일러의 보호를 포기하는 단언이므로, 써야 한다면 이유를 주석이나 `precondition` 메시지로 남긴다
