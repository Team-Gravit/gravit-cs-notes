## Swift Concurrency

**Swift Concurrency**는 Swift 5.5에서 도입된 언어 수준의 동시성 모델로, 콜백 대신 **`async`/`await`**로 비동기 코드를 순차 코드처럼 쓰고, **`Task`**와 **구조적 동시성(Structured Concurrency)**으로 작업의 수명·취소·오류를 트리 구조로 관리한다. GCD가 남긴 콜백 지옥과 스레드 폭발 문제를 컴파일러가 이해하는 규칙으로 대체한 것이 핵심이다.

<br>

### 1. 왜 GCD·콜백만으로는 부족한가

```swift
// 안티패턴: 콜백 중첩 — 오류 처리 누락과 취소 불가, 어느 스레드에서 오는지 불명확
func loadProfile(completion: @escaping (Result<Profile, Error>) -> Void) {
    fetchUser { userResult in
        switch userResult {
        case .failure(let e): completion(.failure(e))
        case .success(let user):
            fetchAvatar(user.avatarURL) { avatarResult in
                switch avatarResult {
                case .failure(let e): completion(.failure(e))
                case .success(let image): completion(.success(Profile(user: user, avatar: image)))
                }
            }
        }
    }
}

// 개선: async/await — 순차 코드처럼 읽히고, 오류는 throws로 자연 전파
func loadProfile() async throws -> Profile {
    let user = try await fetchUser()
    let image = try await fetchAvatar(user.avatarURL)
    return Profile(user: user, avatar: image)
}
```

| **항목**            | **GCD (DispatchQueue)**                          | **Swift Concurrency**                                |
| ------------------- | ------------------------------------------------ | ---------------------------------------------------- |
| **비동기 표현**     | 콜백 클로저                                      | **`async`/`await`** — 순차적 흐름                    |
| **오류 전파**       | 콜백 인자로 수동 전달 (누락 가능)                | **`throws`로 자동 전파** (unit08 참고)               |
| **취소**            | 지원 없음 (`DispatchWorkItem`은 제한적)          | **협력적 취소**가 작업 트리를 따라 전파              |
| **스레드 관리**     | 큐마다 스레드 생성 가능 → **스레드 폭발** 위험   | 코어 수만큼의 **협력적 스레드 풀**, 블로킹 없이 중단 |
| **데이터 경합 검출**| 런타임 도구(TSan)에 의존                         | **컴파일 타임** 검사 (Swift 6, unit07 참고)          |

> 💡 GCD의 `sync`·세마포어로 스레드를 **블로킹**하는 것과 `await`로 **중단(suspend)**하는 것은 완전히 다르다. `await`는 스레드를 반납하고 다른 작업이 그 스레드를 쓰게 한 뒤, 결과가 준비되면 이어서 실행한다. "await는 스레드를 점유하지 않는다"가 면접의 핵심 문장이다.

<br>

### 2. async/await의 동작 원리

- `async` 함수는 **중단 지점(suspension point)**을 가질 수 있는 함수이며, 호출 시 반드시 `await`를 붙여 "여기서 중단될 수 있음"을 표시함
- `await`에 도달하면 현재 함수의 상태(지역 변수 등)가 힙의 **비동기 프레임**에 저장되고 스레드가 반납됨. 이후 결과가 오면 **같은 스레드가 아닐 수도 있는** 풀의 스레드에서 재개됨
- 런타임은 CPU 코어 수만큼의 스레드로 구성된 **협력적 스레드 풀(cooperative thread pool)**을 사용하므로, 스레드 안에서 절대 블로킹(`sleep`, 세마포어 대기, 동기 락 장기 점유)하면 안 됨

```
스레드 1: [taskA 실행] ──await──┐        ┌── [taskA 재개]
                                 │ 반납   │
스레드 1: ─────────────────── [taskB 실행] ┘ ─────────────
                       (I/O 완료 후 taskA는 비어 있는 스레드에서 이어서 실행)
```

**기존 콜백 API 연결하기** — `withCheckedContinuation` 계열로 콜백을 `async`로 감쌀 수 있다. 콜백은 반드시 **정확히 한 번** `resume`을 호출해야 하며, `Checked` 버전은 위반 시 런타임에 경고·크래시로 알려 준다.

```swift
func fetchLegacy() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        legacyFetch { data, error in
            if let error { continuation.resume(throwing: error) }
            else        { continuation.resume(returning: data!) }
        }
    }
}
```

> ⚠️ 동기 함수 안에서는 `await`를 쓸 수 없다. 동기 세계에서 비동기 세계로 들어가는 유일한 문은 **`Task { }`**이며, 반대로 비동기 결과를 동기적으로 "기다리는" 방법(세마포어 등)은 협력적 스레드 풀을 굶겨 데드락을 만들 수 있으므로 금지에 가깝다.

<br>

### 3. Task — 비동기 작업의 단위

`Task`는 비동기 코드가 실행되는 **단위**이자 취소·우선순위·로컬 값을 담는 컨텍스트다.

| **항목**                | **`Task { }`**                                   | **`Task.detached { }`**                        |
| ----------------------- | ------------------------------------------------ | ---------------------------------------------- |
| **액터 컨텍스트 상속**  | **상속함** (`@MainActor` 안에서 만들면 메인에서 실행) | 상속하지 않음                                  |
| **우선순위·로컬 값 상속** | **상속함**                                       | 상속하지 않음 (명시 필요)                      |
| **취소 전파**           | 부모의 취소가 **전파되지 않음** (비구조적)       | 전파되지 않음                                  |
| **용도**                | 동기 컨텍스트(버튼 탭 등)에서 비동기 진입        | 현재 컨텍스트와 무관한 백그라운드 작업 (드묾)  |

- 두 가지 모두 **비구조적(unstructured) 작업**이라 생성한 스코프와 수명이 묶이지 않음. 핸들(`Task<Success, Failure>`)을 보관하고 `task.cancel()`로 직접 취소해야 함
- SwiftUI의 `.task {}` 수정자나 UIKit에서 뷰가 사라질 때 취소하는 패턴처럼, **누군가는 반드시 취소를 책임**져야 누수와 낭비가 없음

```swift
final class SearchViewModel {
    private var searchTask: Task<Void, Never>?

    func search(_ query: String) {
        searchTask?.cancel()                       // 이전 검색 취소
        searchTask = Task { [weak self] in
            try? await Task.sleep(for: .milliseconds(300))   // 디바운스, 취소되면 throw
            guard !Task.isCancelled, let self else { return }
            let results = await self.api.search(query)
            await self.update(results)
        }
    }
}
```

<br>

### 3-1. 협력적 취소(Cooperative Cancellation)

- `cancel()`은 작업을 강제로 멈추지 않고 **취소 플래그만 세움**. 작업 코드가 `Task.isCancelled`를 확인하거나 `try Task.checkCancellation()`으로 `CancellationError`를 던져 스스로 멈춰야 함
- `Task.sleep`·`URLSession`의 `async` API 등 표준 API는 취소를 감지해 오류를 던짐
- 긴 루프에서는 주기적으로 취소를 확인하고, 취소 시 정리 작업은 `withTaskCancellationHandler`로 등록함

> ⚠️ 취소를 확인하지 않는 `Task`는 `cancel()`을 호출해도 끝까지 실행된다. "취소했으니 멈췄겠지"는 가장 흔한 오해다. 자원을 많이 쓰는 반복 작업에는 반드시 취소 확인 지점을 넣어야 한다.

<br>

### 4. 구조적 동시성(Structured Concurrency)

구조적 동시성은 **자식 작업이 부모 스코프를 벗어나 살아남을 수 없다**는 규칙이다. 함수가 반환될 때 모든 자식은 완료·취소되어 있고, 자식의 오류는 부모로 전파되며, 부모의 취소는 자식으로 전파된다. 이 규칙 덕분에 작업 트리가 코드 블록 구조와 일치해 추론이 쉬워진다.

### 4-1. async let — 고정 개수 병렬

```swift
func loadDashboard() async throws -> Dashboard {
    async let user    = fetchUser()          // 즉시 자식 작업 시작
    async let notices = fetchNotices()       // 동시에 시작
    async let banner  = fetchBanner()
    return try await Dashboard(user: user, notices: notices, banner: banner)   // 세 결과를 모아 조립
}
```

- 선언 시점에 자식 작업이 시작되고, `await`에서 결과를 모음. 셋 중 하나가 throw하면 나머지는 **자동 취소**됨
- 병렬로 실행할 작업의 **개수가 컴파일 시점에 정해져 있을 때** 적합함

<br>

### 4-2. TaskGroup — 동적 개수 병렬

```swift
func downloadAll(_ urls: [URL]) async throws -> [URL: Data] {
    try await withThrowingTaskGroup(of: (URL, Data).self) { group in
        for url in urls {
            group.addTask { (url, try await download(url)) }    // 자식 작업 추가
        }
        var result: [URL: Data] = [:]
        for try await (url, data) in group {                     // 완료 순서대로 수집
            result[url] = data
        }
        return result                                            // 여기 도달 = 모든 자식 완료
    }
}
```

```
loadDashboard() / withTaskGroup  (부모)
├── 자식 1  fetchUser()     ─┐
├── 자식 2  fetchNotices()   ├─ 병렬 실행
└── 자식 3  fetchBanner()   ─┘
     ↑ 부모 취소 → 모든 자식 취소 / 자식 오류 → 형제 취소 후 부모로 throw
     ↑ 부모 스코프 종료 시점에 자식은 반드시 완료 상태
```

- 자식 작업 수가 **런타임에 정해질 때** 사용함. 결과는 완료 순으로 도착하므로 순서가 필요하면 인덱스나 키를 함께 반환함
- 그룹 안에서 하나가 throw하면 나머지 자식이 취소되고 그룹이 오류를 던짐. 개별 실패를 허용하려면 자식 안에서 오류를 잡아 `Result`로 반환함(unit08 참고)

> 💡 "왜 `Task {}`를 반복문에서 여러 개 만들지 않고 TaskGroup을 쓰는가?"의 답은 **수명과 취소 보장**이다. 비구조적 `Task` 여러 개는 부모가 죽어도 살아남고 취소가 전파되지 않으며 오류가 사라진다. 구조적 도구(`async let`, `TaskGroup`)를 기본으로 쓰고, `Task {}`는 동기 → 비동기 진입점에서만 쓰는 것이 원칙이다.

<br>

### 5. 실행 컨텍스트와 버전 의존 사항

- `@MainActor`가 아닌 `nonisolated` 비동기 함수는 Swift 5.7(SE-0338) 이후 호출자의 액터가 아니라 **전역 협력 풀**에서 실행됨. 따라서 UI 갱신은 반드시 `@MainActor`로 격리해야 함(unit07 참고)
- Swift 6.2에서는 이 기본 동작을 "호출자의 액터에서 이어서 실행"으로 바꾸는 `NonisolatedNonsendingByDefault` 업커밍 기능(SE-0461)이 도입되었으며, 기존 동작(항상 액터를 벗어나 실행)은 `@concurrent`로 명시함. 플래그 활성화 여부에 따라 실행 스레드가 달라질 수 있으므로 **버전과 빌드 설정을 확인**해야 함
- `Task.sleep(for:)`·`Clock` API는 Swift 5.7(iOS 16+)부터, `async`/`await` 자체는 Xcode 13.2부터 iOS 13까지 백포트되어 사용 가능함(플랫폼 배포 버전에 따라 다름)

<br>

### 6. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                                   |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **await는 스레드를 블로킹하나?**                  | 아니다. 상태를 저장하고 스레드를 반납한 뒤, 재개 시 다른 스레드에서 이어질 수 있다              |
| **Task와 Task.detached의 차이는?**                | 액터 컨텍스트·우선순위 상속 여부. 둘 다 비구조적이라 취소를 직접 관리해야 함                    |
| **구조적 동시성이란?**                            | 자식이 부모 스코프를 넘어 살 수 없음. 취소는 아래로, 오류는 위로 전파                           |
| **async let과 TaskGroup의 선택 기준은?**          | 개수가 고정이면 `async let`, 동적이면 `TaskGroup`                                               |
| **cancel()하면 즉시 멈추나?**                     | 아니다. 협력적 취소 — 작업이 `isCancelled`·`checkCancellation`으로 스스로 멈춰야 함             |
| **콜백 API를 async로 바꾸려면?**                  | `withCheckedThrowingContinuation`, `resume`은 정확히 한 번                                      |

- Swift Concurrency의 세 축은 **`async`/`await`(중단)**, **`Task`(단위·취소)**, **구조적 동시성(수명 트리)**임
- 협력적 스레드 풀 위에서 **블로킹 금지**, 취소는 **협력적**, 병렬은 **구조적 도구 우선**이 실무 원칙
- 여러 작업이 같은 상태에 접근할 때의 안전성은 **unit07(actor와 데이터 격리)**에서 이어서 다룸
