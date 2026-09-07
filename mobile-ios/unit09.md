## Combine과 데이터 바인딩

**Combine**은 iOS 13에 도입된 Apple의 선언형 반응형 프레임워크로, 시간에 따라 발생하는 값의 흐름을 **Publisher → Operator → Subscriber** 파이프라인으로 처리한다. `ObservableObject`·`@Published`가 Combine 위에 있으므로(unit03 참고) SwiftUI 데이터 바인딩의 내부 원리이기도 하며, Swift 5.5+ **async/await**와 역할이 겹치는 부분이 많아 "언제 무엇을 쓰는가"가 면접의 단골 질문이다.

<br>

### 1. 세 가지 구성 요소

```
Publisher ──(값·완료·에러)──▶ Operator ──▶ Operator ──▶ Subscriber
   │                                                        │
   │◀────────────── Subscription (수요 요청·취소) ───────────┘

시간 →  ●──●──●──────●──|      ● = 값(Output)   | = 완료(finished) 또는 실패(Failure)
```

| **구성 요소**   | **역할**                                          | **대표 타입**                                                   |
| --------------- | ------------------------------------------------- | --------------------------------------------------------------- |
| **Publisher**   | 값(`Output`)과 완료/에러(`Failure`)를 **발행**    | `Just`, `Future`, `PassthroughSubject`, `CurrentValueSubject`, `@Published` |
| **Operator**    | 값을 변환·필터·결합해 **새 Publisher** 반환       | `map`, `filter`, `debounce`, `combineLatest`, `flatMap`, `catch` |
| **Subscriber**  | 값을 **소비**하고 수요(Demand)를 요청             | `sink`, `assign(to:on:)`                                        |

- `Publisher`는 **타입 수준에서 `Output`과 `Failure`가 고정**된다. 에러가 없으면 `Failure == Never`
- Publisher는 구독 전까지 아무것도 하지 않는다(**지연 실행**). `sink`·`assign`이 붙는 순간 구독이 시작된다
- 구독 결과 `AnyCancellable`을 **보관하지 않으면 즉시 해제**되어 파이프라인이 사라진다

```swift
final class SearchViewModel: ObservableObject {
    @Published var query = ""                       // Publisher: $query
    @Published private(set) var results: [Item] = []
    private var cancellables = Set<AnyCancellable>()

    init(api: SearchAPI) {
        $query
            .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)   // 입력 멈춘 뒤 300ms
            .removeDuplicates()                                                  // 같은 검색어 중복 제거
            .filter { $0.count >= 2 }
            .map { api.search($0).replaceError(with: []) }                       // Publisher<Publisher>
            .switchToLatest()                                                    // 최신 요청만 유지
            .receive(on: DispatchQueue.main)
            .assign(to: &$results)                                               // iOS 14+, cancellable 불필요
    }
}
```

> 💡 검색창 디바운스 예시는 Combine이 가장 잘하는 일을 보여준다. **"시간 조건 + 중복 제거 + 이전 요청 취소"**를 연산자 조합만으로 선언하며, 같은 로직을 async/await로 쓰면 `Task` 취소·보관을 직접 관리해야 한다.

<br>

### 2. 구독 생명주기와 수요(Demand)

```
① subscriber.receive(subscription:)  ← Publisher가 Subscription 객체 전달
② subscription.request(.unlimited)   ← Subscriber가 "몇 개 받을지" 요청 (백프레셔)
③ subscriber.receive(value) → Demand ← 값 전달, 추가 수요 반환
④ subscriber.receive(completion:)    ← finished 또는 failure로 종료
   또는 cancellable.cancel()          ← 소비자가 먼저 끊음
```

- Combine은 **풀(pull) 기반 백프레셔**를 내장한다. `sink`는 `.unlimited`를 요청하므로 실무에서 수요를 직접 다루는 일은 드물지만, "왜 Subscription이 존재하는가"의 답이 이것이다
- `AnyCancellable`은 **`deinit` 시 자동 `cancel()`**한다. 뷰모델이 해제되면 `cancellables`도 해제되어 구독이 정리된다
- 완료(`finished`/`failure`) 후에는 값이 더 전달되지 않는다. **에러가 나면 파이프라인 전체가 종료**되므로 UI 스트림에서는 `catch`·`replaceError`로 에러를 값으로 바꾸거나, `flatMap` 안쪽에서 처리해 바깥 스트림을 살린다

> ⚠️ `@Published`의 프로젝션(`$value`)은 **`willSet` 시점**에 발행한다. `sink` 안에서 `self.value`를 읽으면 아직 **이전 값**이다. 클로저 매개변수로 받은 새 값을 써야 한다. `ObservableObject`의 `objectWillChange`도 같은 이유로 "will"이다.

<br>

### 3. Scheduler — 어디서 실행할 것인가

`Scheduler`는 "코드가 언제·어느 스레드에서 실행되는가"를 추상화한 프로토콜이다. `DispatchQueue`, `RunLoop`, `OperationQueue`, `ImmediateScheduler`가 채택한다.

| **연산자**            | **영향 범위**                                              | **용도**                                    |
| --------------------- | ---------------------------------------------------------- | ------------------------------------------- |
| **`subscribe(on:)`**  | **업스트림** — 구독·수요 요청·발행 시작 작업이 실행될 곳   | 무거운 발행 작업을 백그라운드로             |
| **`receive(on:)`**    | **다운스트림** — 이후 연산자와 Subscriber가 실행될 곳      | UI 갱신 전 메인으로 복귀                    |
| **시간 연산자 인자**  | `debounce`·`throttle`·`delay`·`timeout`의 타이머가 도는 곳 | 타이머 스케줄러 지정                        |

```swift
imageLoader.publisher(for: url)           // 네트워크·디코딩
    .subscribe(on: DispatchQueue.global(qos: .userInitiated))   // 위쪽 작업은 백그라운드
    .map { $0.resized(to: thumbnailSize) }                       // 여전히 백그라운드
    .receive(on: DispatchQueue.main)                             // 여기부터 메인
    .sink { [weak self] image in self?.imageView.image = image }
    .store(in: &cancellables)
```

- `subscribe(on:)`은 위치와 무관하게 **파이프라인의 시작점**에 영향을 주고, `receive(on:)`은 **호출 지점 이후**부터 적용된다. 여러 번 쓰면 마지막 `receive(on:)`이 이후 구간을 결정한다
- `Publisher`가 이미 특정 스레드에서 발행하면(예: `URLSession.dataTaskPublisher`는 백그라운드) `receive(on: .main)` 없이 UI를 갱신하면 안 된다

> ⚠️ `RunLoop.main`과 `DispatchQueue.main`은 다르다. `RunLoop.main` 스케줄러는 **스크롤 등 UI 트래킹 모드 중에는 이벤트를 전달하지 않아** 스크롤이 끝날 때까지 갱신이 지연된다. UI 바인딩에는 일반적으로 `DispatchQueue.main`을 쓴다.

<br>

### 4. 데이터 바인딩 패턴

### 4-1. 양방향 바인딩과 Subject

- **`PassthroughSubject`**: 값을 보관하지 않고 발행만 한다. 버튼 탭·이벤트 스트림에 적합
- **`CurrentValueSubject`**: **현재 값을 보관**하며 구독 즉시 마지막 값을 전달한다. 상태 스트림에 적합
- SwiftUI에서는 `@Published` + `Binding`(unit02 참고)이 이 역할을 대신하므로 Subject를 직접 노출하는 일은 UIKit + MVVM 조합에서 주로 발생한다

```swift
// UIKit + MVVM: 입력(Subject) → 뷰모델 → 출력(Published)
final class LoginViewModel {
    let emailInput = PassthroughSubject<String, Never>()
    let passwordInput = PassthroughSubject<String, Never>()
    @Published private(set) var isSubmitEnabled = false
    private var cancellables = Set<AnyCancellable>()

    init() {
        Publishers.CombineLatest(emailInput, passwordInput)
            .map { email, pw in email.contains("@") && pw.count >= 8 }
            .removeDuplicates()
            .assign(to: &$isSubmitEnabled)
    }
}

// 뷰 컨트롤러
viewModel.$isSubmitEnabled
    .receive(on: DispatchQueue.main)
    .assign(to: \.isEnabled, on: submitButton)         // 대상 객체를 강하게 참조함에 주의
    .store(in: &cancellables)
```

<br>

### 4-2. 메모리 함정

- `assign(to:on:)`은 **대상 객체를 강하게 참조**한다. `self`를 대상으로 쓰고 `cancellables`를 `self`가 보관하면 **순환 참조**가 생긴다. `sink { [weak self] ... }`로 바꾸거나 `assign(to: &$published)` 형태를 쓴다
- `sink` 클로저에서 `self`를 강하게 캡처하고 `cancellables`를 `self`가 보관해도 같은 순환이 생긴다. 항상 `[weak self]`

<br>

### 5. async/await와의 선택

Swift 5.5+(iOS 15+, 일부 백포트 iOS 13+) **async/await**는 "한 번 요청하고 한 번 응답받는" 비동기 작업을 동기 코드처럼 쓰게 한다. Combine의 상당 부분과 겹치지만 **모델이 다르다**.

| **항목**             | **Combine**                                          | **async/await (Swift Concurrency)**                     |
| -------------------- | ---------------------------------------------------- | ------------------------------------------------------- |
| **모델**             | **여러 값**이 시간에 따라 흐르는 스트림              | **하나의 값**을 기다리는 함수 호출 (`AsyncSequence`로 스트림도 가능) |
| **에러 처리**        | `Failure` 타입, `catch`·`retry` 연산자               | `throws`·`do/catch`, 일반 Swift 에러                    |
| **취소**             | `AnyCancellable.cancel()`                            | `Task.cancel()` + 협력적 취소 확인                      |
| **스레드 제어**      | `Scheduler` (`subscribe/receive(on:)`)               | `actor`·`@MainActor`로 격리                             |
| **결합·시간 연산**   | `combineLatest`·`debounce`·`throttle` 등 **풍부**    | 직접 구현 필요 (AsyncAlgorithms 패키지로 보완)          |
| **가독성**           | 파이프라인 길어지면 난해, 디버깅 어려움              | 순차 코드처럼 읽힘, 스택 트레이스 명확                  |
| **가용성**           | iOS 13+                                              | iOS 15+ (언어 기능은 13+ 백포트)                        |

```swift
// 안티패턴: 단발성 요청에 Combine — 보관·취소·에러 매핑이 장황
func loadProfile() {
    api.profilePublisher()
        .receive(on: DispatchQueue.main)
        .sink(receiveCompletion: { [weak self] c in if case .failure(let e) = c { self?.error = e } },
              receiveValue:      { [weak self] p in self?.profile = p })
        .store(in: &cancellables)
}
```

```swift
// 개선: 단발성 요청은 async/await — 순차적이고 에러가 자연스러움
@MainActor
func loadProfile() async {
    do { profile = try await api.fetchProfile() }
    catch { self.error = error }
}
```

- 두 세계는 연결된다: `publisher.values`(iOS 15+)로 Publisher를 `AsyncSequence`로 소비하고, `Future { promise in Task { ... } }`로 async 함수를 Publisher로 감쌀 수 있다
- iOS 17+ Observation(unit03 참고)이 `ObservableObject`의 Combine 의존을 제거했고, Swift 6.2/iOS 26의 `Observations` AsyncSequence가 모델 관찰까지 async 세계로 옮기면서 **신규 코드에서 Combine의 필수 영역은 줄어드는 추세**다 (버전에 따라 가용성 다름)

> 💡 선택 기준 한 줄: **"값이 여러 번, 시간 조건과 결합이 필요하면 Combine, 요청-응답 한 번이면 async/await"**. 실무에서는 네트워크·저장소 계층을 async/await로 쓰고, UI 입력 스트림(디바운스·결합)에만 Combine을 남기는 절충이 흔하다.

<br>

### 6. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                            |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Publisher·Subscriber·Subscription의 관계는?**       | Subscriber가 구독하면 Subscription이 생기고, **수요(Demand)**를 요청한 만큼 값이 흐름     |
| **subscribe(on:)과 receive(on:)의 차이는?**           | 전자는 **업스트림(발행 측)**, 후자는 **호출 이후 다운스트림**의 실행 스케줄러             |
| **AnyCancellable을 보관하지 않으면?**                 | 즉시 해제되어 구독이 취소됨. `deinit` 시 자동 cancel이 정리 메커니즘                     |
| **@Published는 언제 발행하는가?**                     | **willSet** 시점 — 클로저 안에서 self의 프로퍼티를 읽으면 이전 값                        |
| **Combine 대신 async/await를 택하는 기준은?**         | 단발 요청-응답·순차 로직은 async/await, 다중 값·시간 연산·결합은 Combine                 |
| **PassthroughSubject와 CurrentValueSubject의 차이는?** | 현재 값 **보관 여부** — 이벤트 vs 상태                                                   |

- 에러가 나면 스트림이 끝나므로 UI 스트림은 **에러를 값으로 변환**해 살린다
- 메모리 순환은 `[weak self]`와 `assign(to: &$published)`로 끊는다
- 상태 래퍼와의 관계는 unit02, Observation과의 관계는 unit03을 참고할 것
