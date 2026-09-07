## 에러 처리

Swift의 에러 처리는 **`throws`로 실패 가능성을 시그니처에 드러내고 `try`로 호출부에 확인을 강제**하는 방식이며, 실패를 값으로 보관해야 할 때는 **`Result`**를 사용한다. 이 유닛에서는 `Error`·`throws`·`do-catch`의 동작 원리, `Result`와의 선택 기준, Swift 6의 **타입 지정 throws**, 그리고 "어디서 잡고 어디까지 올릴 것인가"라는 **전파 전략**을 정리한다.

<br>

### 1. Swift 에러의 특징

- 에러는 `Error` 프로토콜을 채택한 **아무 타입**이나 될 수 있으며, 관례적으로 열거형으로 정의함
- `throws` 함수는 실패할 수 있음을 **시그니처에 명시**하고, 호출부는 반드시 `try`를 붙여야 함 → 실패 가능성을 "잊는" 것이 컴파일 오류가 됨
- Java·C++의 예외와 달리 **스택 풀기(unwinding)가 없음**. 컴파일러가 에러를 특별한 반환 값처럼 처리하므로 오버헤드가 작고, 어디서 던져질 수 있는지가 코드에 드러남
- 던질 수 있는 에러의 종류는 기본적으로 `any Error`이며, Swift 6부터는 `throws(SomeError)`로 **타입을 지정**할 수 있음(3절 참고)

```swift
enum NetworkError: Error {
    case invalidURL
    case httpStatus(Int)
    case decoding(underlying: Error)     // 원인 에러를 감싸는 연관값
}

func fetchUser(id: Int) async throws -> User {
    guard let url = URL(string: "https://api.example.com/users/\(id)") else {
        throw NetworkError.invalidURL
    }
    let (data, response) = try await URLSession.shared.data(from: url)
    guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
        throw NetworkError.httpStatus((response as? HTTPURLResponse)?.statusCode ?? -1)
    }
    do {
        return try JSONDecoder().decode(User.self, from: data)
    } catch {
        throw NetworkError.decoding(underlying: error)   // 저수준 에러를 도메인 에러로 변환
    }
}
```

> 💡 "Swift 에러는 예외(exception)인가?"라는 질문에는 "문법은 비슷하지만 **런타임 스택 풀기가 없는 값 기반 전파**이며, `throws` 표시와 `try` 강제로 실패 경로가 정적으로 추적된다"고 답하면 된다. 배열 인덱스 초과·강제 언래핑 실패 같은 **프로그래머 오류는 throw되지 않고 즉시 크래시**한다는 점도 함께 짚자.

<br>

### 2. try의 세 가지 형태와 do-catch

| **형태**    | **에러 발생 시 동작**                           | **적합한 상황**                                                    |
| ----------- | ----------------------------------------------- | ------------------------------------------------------------------ |
| **`try`**   | 현재 스코프의 `catch`로 가거나, 함수 밖으로 **전파** | 기본 선택. 처리하거나 위로 올릴 때                                 |
| **`try?`**  | 에러를 버리고 **nil** 반환 (결과는 옵셔널)      | 실패 이유가 중요하지 않고 "없으면 말고"가 자연스러울 때            |
| **`try!`**  | **크래시**                                      | 실패가 곧 프로그래머 오류인 경우(번들 리소스 로드 등). 외부 입력에는 금지 |

```swift
do {
    let user = try await fetchUser(id: 1)
    print(user.name)
} catch NetworkError.httpStatus(let code) where code == 404 {
    print("사용자를 찾을 수 없음")                     // 패턴 매칭 + where 절
} catch let error as NetworkError {
    print("네트워크 계층 실패: \(error)")
} catch {
    print("알 수 없는 오류: \(error)")                 // 암묵적 error 상수
}
```

- `catch` 절은 위에서 아래로 매칭되며, 모든 경우를 잡지 못하면 바깥 함수가 `throws`여야 함
- `defer` 블록은 에러가 던져져도 **스코프를 벗어날 때 반드시 실행**되므로 파일 닫기·락 해제 같은 정리 작업에 사용함
- `rethrows`는 "전달받은 클로저가 throw할 때만 나도 throw한다"는 뜻으로, `map`·`filter` 같은 고차 함수에 쓰임. 던지지 않는 클로저를 넘기면 호출부에 `try`가 필요 없음

> ⚠️ `try?`는 편리하지만 **실패 원인을 삼킨다**. 디코딩 실패로 화면이 비어 있는데 로그조차 없는 상황이 여기서 나온다. `try?`를 쓴다면 최소한 실패가 사용자 경험에 영향이 없는 경우인지 확인하고, 그렇지 않으면 `do-catch`로 로그를 남긴다. `try!`의 위험은 unit03의 강제 언래핑과 동일하다.

<br>

### 3. 타입 지정 throws (Swift 6)

Swift 6(SE-0413)부터 `throws(MyError)`로 **던질 수 있는 에러 타입을 하나로 제한**할 수 있다. `catch` 절에서 `error`가 `any Error`가 아닌 구체 타입으로 잡히므로 캐스팅과 기본 `catch`가 사라진다.

```swift
enum ParseError: Error { case empty, invalidFormat }

func parseAge(_ text: String) throws(ParseError) -> Int {
    guard !text.isEmpty else { throw .empty }              // 타입이 확정되어 .empty로 축약 가능
    guard let n = Int(text) else { throw .invalidFormat }
    return n
}

do {
    let age = try parseAge("abc")
} catch {
    switch error {                                          // error: ParseError — switch가 완전해야 함
    case .empty:         print("빈 입력")
    case .invalidFormat: print("형식 오류")
    }
}
```

| **비교**                | **`throws` (기존)**                        | **`throws(E)` (Swift 6)**                                  |
| ----------------------- | ------------------------------------------ | ---------------------------------------------------------- |
| **에러 타입**           | `any Error` (existential)                  | **구체 타입 E**                                            |
| **catch에서 분기**      | `as?` 캐스팅 + 기본 `catch` 필수           | `switch`로 **완전성 검사** 가능                            |
| **성능**                | 박싱 비용                                  | 박싱 없음 (임베디드 Swift 등에서 유리)                     |
| **API 진화**            | 새 에러 추가가 자유로움                    | 에러 케이스 추가가 **호출부 깨짐** (`switch` 완전성)       |
| **권장 용도**           | 공개 라이브러리·다양한 실패 원인           | 모듈 내부, 실패 종류가 닫혀 있는 함수, 제네릭 에러 전달    |

> ⚠️ Apple 공식 가이드도 **기본은 여전히 타입 미지정 `throws`**를 권장한다. 공개 API에 `throws(E)`를 쓰면 에러 케이스를 하나 추가하는 것이 소스 호환성을 깨는 변경이 되기 때문이다. `throws(Never)`는 `throws`가 없는 것과 같고, `rethrows`는 제네릭 타입 지정 throws로 표현할 수 있다(`throws(E)` where E는 클로저의 에러 타입).

<br>

### 4. Result — 실패를 값으로 다루기

`Result<Success, Failure: Error>`는 성공 또는 실패를 담는 열거형이다. `throws`가 **제어 흐름**이라면 `Result`는 **값**이므로, 저장·전달·나중에 처리가 가능하다.

```swift
// 콜백 API: throws를 쓸 수 없으므로 Result로 실패를 값으로 전달
func load(completion: @escaping (Result<User, NetworkError>) -> Void) { ... }

// throws → Result 변환: 실패를 나중에 처리하거나 여러 결과를 모을 때
let result = Result { try parseAge("42") }          // Result<Int, any Error>

// Result → throws 변환: 다시 제어 흐름으로 되돌림
let age = try result.get()

// 변환 체인: 언래핑 없이 성공 값만 가공
let label = result.map { "\($0)세" }.mapError { NetworkError.decoding(underlying: $0) }
```

| **기준**                 | **`throws` / `try`**                            | **`Result`**                                                  |
| ------------------------ | ----------------------------------------------- | ------------------------------------------------------------- |
| **성격**                 | 제어 흐름 — 즉시 처리하거나 전파               | **값** — 보관·전달·지연 처리                                  |
| **비동기 코드**          | `async throws`로 자연스럽게 표현                | `async`/`await` 이후에는 필요성이 크게 감소                   |
| **여러 결과 수집**       | 하나가 throw하면 흐름이 끊김                    | `[Result<T, E>]`로 **부분 실패** 표현 가능 (TaskGroup에서 유용) |
| **에러 타입 명시**       | Swift 6 `throws(E)`                             | `Failure` 제네릭으로 처음부터 명시                            |
| **적합한 상황**          | 대부분의 동기·비동기 함수                       | 콜백 API, 결과 캐싱, 부분 실패 수집, 재시도 큐                |

> 💡 `async`/`await` 도입 전에는 콜백에 실패를 실어 보내기 위해 `Result`가 널리 쓰였지만, 이제 **기본은 `throws`**이고 `Result`는 "실패를 값으로 보관해야 하는 특수 상황"에만 쓴다. "언제 Result를 쓰나?"에 "여러 작업의 부분 실패를 모아야 할 때, 콜백 기반 레거시 API를 감쌀 때"라고 답하면 충분하다(unit06 TaskGroup 참고).

<br>

### 5. 전파 전략 — 어디서 잡을 것인가

```
        [UI 계층]           사용자에게 보여줄 메시지로 최종 처리 (catch)
             ▲  도메인 에러 (표시 가능한 형태)
        [도메인/서비스 계층]  저수준 에러 → 도메인 에러로 변환 (catch + throw)
             ▲  NetworkError, DatabaseError …
        [인프라 계층]         URLSession·파일 IO 에러를 그대로 throw (전파)
```

- **전파(propagate)**: 현재 계층에서 의미 있게 처리할 수 없으면 `throws`로 그대로 올림. 대부분의 저수준 함수가 여기에 해당
- **변환(translate)**: 계층 경계에서 저수준 에러를 **도메인 에러로 감싸서** 상위가 구현 세부(URLSession, SQLite)를 몰라도 되게 함. 원인은 연관값(`underlying:`)으로 보존해 디버깅 정보를 잃지 않음
- **복구(recover)**: 캐시 폴백, 재시도, 기본값 대체처럼 대안이 있는 곳에서만 잡음. 잡았으면 반드시 **무언가를 해야** 하며, 빈 `catch {}`는 에러를 삼키는 안티패턴
- **최종 처리(handle)**: UI 계층에서 사용자 메시지·로그·분석 이벤트로 마무리함. `LocalizedError`를 채택하면 `errorDescription`으로 표시 문구를 제공할 수 있음
- **프로그래머 오류는 throw하지 않음**: 잘못된 인덱스, 위반된 불변식은 `precondition`·`fatalError`로 **즉시 실패**시켜 원인 지점에서 발견함. 복구 가능한 실패(네트워크·입력)만 `throws` 대상임

```swift
// 안티패턴: 삼키기 — 실패했는데 아무 일도 없었던 것처럼 진행
func loadSettings() -> Settings {
    (try? decoder.decode(Settings.self, from: data)) ?? Settings()   // 왜 기본값이 됐는지 아무도 모름
}

// 개선: 복구는 하되 원인은 기록하고, 복구 불가능한 경우는 위로 올림
func loadSettings() throws -> Settings {
    do {
        return try decoder.decode(Settings.self, from: data)
    } catch let error as DecodingError {
        logger.warning("설정 디코딩 실패, 기본값 사용: \(error)")
        return Settings()                                             // 명시적 복구 + 로그
    }                                                                 // 그 외 에러는 자동 전파
}
```

> ⚠️ `Task { }` 안에서 던져진 에러는 아무도 `await task.value`로 꺼내지 않으면 **조용히 사라진다**. 비구조적 작업의 마지막에는 반드시 `do-catch`로 처리하거나, 결과가 필요한 곳에서 `try await task.value`로 받아야 한다(unit06 참고).

<br>

### 6. 면접·실무 체크포인트

| **질문**                                     | **핵심 답변**                                                                            |
| -------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Swift 에러와 Java 예외의 차이는?**         | 스택 풀기 없는 값 기반 전파, `throws`·`try`로 실패 경로가 정적으로 드러남                |
| **try? / try!의 차이는?**                    | `try?`는 nil로 삼킴, `try!`는 크래시. 외부 입력에는 둘 다 신중                            |
| **throws와 Result의 선택 기준은?**           | 기본은 `throws`. 실패를 값으로 보관·수집·지연 처리할 때만 `Result`                       |
| **타입 지정 throws의 장단점은?**             | `switch` 완전성·성능 이점 vs 에러 추가 시 호출부 깨짐. 공개 API에는 기존 `throws` 권장   |
| **에러를 어디서 잡아야 하나?**               | 의미 있게 복구·변환·표시할 수 있는 계층에서. 그 외는 전파                                |
| **rethrows란?**                              | 인자 클로저가 throw할 때만 throw하는 함수. 고차 함수에 사용                              |
| **defer는 에러가 나도 실행되나?**            | 실행된다. 스코프 종료 시 항상 실행되므로 정리 코드에 적합                                |

- `throws`는 **실패를 시그니처에 드러내고 처리를 강제**하는 장치, `Result`는 **실패를 값으로 보관**하는 장치
- 전파 전략의 원칙은 "**잡았으면 처리하라, 처리 못 하면 올려라, 올릴 때는 도메인 언어로 번역하라**"
- 복구 가능한 실패와 프로그래머 오류를 구분하고, 후자는 `precondition`으로 즉시 드러내는 것이 unit03의 옵셔널 원칙과 같은 맥락임
