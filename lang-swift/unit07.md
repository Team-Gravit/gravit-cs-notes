## actor와 데이터 격리

**actor**는 자신의 가변 상태를 **한 번에 하나의 작업만** 접근하도록 격리(isolation)하는 참조 타입으로, Swift Concurrency에서 데이터 경합(data race)을 막는 핵심 도구다. 여기에 "경계를 넘어도 안전한 타입"을 표시하는 **`Sendable`**과 UI 상태를 메인 스레드에 묶는 **`@MainActor`**가 더해져, Swift 6 언어 모드에서는 데이터 경합이 **컴파일 오류**로 검출된다.

<br>

### 1. 데이터 경합과 격리의 필요성

- **데이터 경합**: 두 개 이상의 스레드가 같은 메모리에 동시에 접근하고 그중 하나 이상이 쓰기일 때 발생. 결과가 비결정적이고 재현이 어려워 가장 악명 높은 버그 유형임
- 전통적 해법은 락(`NSLock`)·직렬 큐(`DispatchQueue`)로 **개발자가 규약을 지키는 것**이었으나, 한 곳이라도 빠뜨리면 컴파일러는 알지 못함
- Swift Concurrency는 "**가변 상태는 격리 영역(isolation domain) 안에서만 만진다**"는 규칙을 언어에 내장하고, 영역 사이를 오가는 값은 `Sendable`이어야 한다고 검사함

```swift
// 안티패턴: 클래스 + 락 — 락을 빼먹은 경로가 하나라도 있으면 경합
final class Counter {
    private var value = 0
    private let lock = NSLock()
    func increment() { lock.lock(); value += 1; lock.unlock() }
    var current: Int { value }      // 락 없이 읽음 → 경합 (컴파일러는 침묵)
}

// 개선: actor — 모든 접근이 자동으로 직렬화되고 외부에서는 await가 강제됨
actor SafeCounter {
    private var value = 0
    func increment() { value += 1 }
    var current: Int { value }
}
let counter = SafeCounter()
await counter.increment()
print(await counter.current)       // 외부 접근은 반드시 await
```

<br>

### 2. actor의 동작 원리

- `actor`는 `class`처럼 참조 타입이지만 상속이 없고, 내부에 **직렬 실행기(serial executor)**와 **메일박스(대기 큐)**를 가짐
- 외부에서 액터의 격리된 멤버(저장 프로퍼티·메서드)에 접근하면 메시지가 큐에 들어가고, 액터는 **한 번에 하나씩** 처리함 → 상태를 동시에 만지는 일이 구조적으로 불가능
- 액터 내부에서 자기 상태에 접근할 때는 `await`가 필요 없고, **다른 액터**나 **외부**에서 접근할 때만 `await`가 필요함
- `nonisolated`를 붙인 멤버는 격리 밖에 있으며, 불변 `let`이나 상태에 접근하지 않는 메서드에만 쓸 수 있음

```
         호출자 A ──┐
         호출자 B ──┼──→ [메일박스: 요청 큐] ──→ actor (직렬 실행기)
         호출자 C ──┘                              │
                                                   └─ 격리된 상태: value, cache …
       await 지점에서 호출자는 중단되고, 액터가 순서대로 하나씩 처리
```

<br>

### 2-1. 재진입(Reentrancy) 함정

액터는 **중단 지점(`await`)에서 재진입**을 허용한다. 즉, 액터 메서드 안에서 `await`로 기다리는 동안 **다른 요청이 끼어들어 상태를 바꿀 수 있다**. "액터라서 안전하다"고 믿고 `await` 전후의 상태가 같다고 가정하면 논리 오류가 난다.

```swift
actor ImageCache {
    private var cache: [URL: Data] = [:]

    func image(for url: URL) async throws -> Data {
        if let cached = cache[url] { return cached }
        let data = try await download(url)      // ← 여기서 중단: 다른 호출이 같은 url을 또 다운로드할 수 있음
        cache[url] = data
        return data
    }
}
```

- 위 코드는 경합은 없지만 **같은 URL을 중복 다운로드**할 수 있음. 해결책은 진행 중인 `Task`를 캐시에 함께 저장해 두 번째 호출자가 그 `Task`의 결과를 기다리게 하는 것
- 원칙: **`await` 이후에는 상태를 다시 확인**하고, 불변식(invariant)이 깨지지 않도록 상태 변경을 `await` 사이에 걸치지 않게 설계함

> ⚠️ 재진입은 데드락을 막기 위한 의도된 설계다(액터가 `await` 중에도 다른 요청을 처리할 수 있어야 서로 기다리는 상황이 줄어듦). "액터 = 자동 락"이 아니라 "**동기 구간만 원자적**"이라고 이해해야 한다.

<br>

### 3. Sendable — 격리 경계를 넘을 수 있는 타입

`Sendable`은 "이 값을 다른 격리 영역(스레드·액터)으로 **복사해 보내도 안전하다**"는 표시 프로토콜(marker protocol)이다. 요구 메서드는 없고 컴파일러가 구조를 검사한다.

| **타입 종류**                          | **Sendable 여부**                                    | **비고**                                                        |
| -------------------------------------- | ---------------------------------------------------- | --------------------------------------------------------------- |
| **값 타입 (struct·enum)**              | 모든 저장 프로퍼티가 Sendable이면 **암묵적 준수**    | `public` 타입은 명시 선언 필요                                  |
| **actor**                              | **항상 Sendable**                                    | 상태 접근이 직렬화되므로 참조를 넘겨도 안전                     |
| **final class, 모든 프로퍼티가 `let`** | 명시적으로 `Sendable` 채택 가능                      | 불변이므로 공유해도 안전                                        |
| **가변 프로퍼티가 있는 class**         | **불가**                                             | 락으로 직접 보호했다면 `@unchecked Sendable`로 책임 선언        |
| **클로저**                             | `@Sendable`로 표시 시 캡처 값도 Sendable이어야 함    | `Task { }`·`TaskGroup.addTask`의 클로저는 `@Sendable`           |
| **표준 라이브러리 컬렉션**             | 요소가 Sendable이면 **조건부 준수**                  | `[Int]`는 Sendable, `[NSMutableArray]`는 아님                   |

```swift
struct UserDTO: Sendable {              // 모든 프로퍼티가 Sendable → 준수 가능
    let id: Int
    let name: String
}

final class ThreadSafeLogger: @unchecked Sendable {   // 내부에서 락으로 보호함을 개발자가 보증
    private let lock = NSLock()
    private var lines: [String] = []
    func log(_ s: String) { lock.lock(); lines.append(s); lock.unlock() }
}
```

> 💡 값 타입이 동시성에서 유리한 이유가 여기서 드러난다. `struct`는 복사되어 전달되므로 격리 영역이 달라도 서로 간섭하지 않아 대부분 자동으로 `Sendable`이다(unit01 참고). 반면 `class`는 참조가 공유되므로 컴파일러가 안전을 보장할 수 없다.

> ⚠️ `@unchecked Sendable`은 "컴파일러 검사를 끄겠다"는 선언이다. 실제로 동기화가 되어 있지 않다면 Swift 6가 약속하는 경합 검출이 무력화된다. 마이그레이션 중 임시 조치로만 쓰고, 가능하면 액터나 값 타입으로 재설계한다.

<br>

### 4. @MainActor — 메인 스레드 격리

**`@MainActor`**는 메인 스레드에서 실행되는 **전역 액터(global actor)**다. UIKit·SwiftUI의 뷰와 상태는 메인 스레드에서만 만져야 하므로, 이를 타입 시스템으로 보장한다.

```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    @Published var name = ""                 // 메인 액터 격리 → 백그라운드에서 대입하면 컴파일 오류

    func load() async {
        let dto = await api.fetchProfile()   // 네트워크 호출은 액터 밖(협력 풀)에서 실행
        name = dto.name                      // await 복귀 후 자동으로 메인 액터로 돌아옴
    }
}

// 특정 코드 블록만 메인에서 실행
await MainActor.run { label.text = "완료" }
```

- 타입·메서드·프로퍼티·클로저 어디에나 붙일 수 있으며, 타입에 붙이면 모든 멤버가 메인 액터에 격리됨
- `@MainActor` 안에서 만든 `Task { }`는 컨텍스트를 상속해 메인에서 시작하고, 그 안의 `await` 호출은 필요 시 다른 실행기로 갔다가 **자동으로 메인으로 복귀**함(unit06 참고)
- `nonisolated async` 함수의 실행 위치는 Swift 6.2의 `NonisolatedNonsendingByDefault` 설정에 따라 달라지므로, "무거운 작업이 메인에서 돌지 않는지"는 버전·플래그를 확인해 판단함

> 💡 `DispatchQueue.main.async { }`로 감싸던 UI 갱신 코드는 `@MainActor` 격리로 대체된다. 차이는 **컴파일러가 강제**한다는 점이다. 격리되지 않은 곳에서 `@MainActor` 프로퍼티에 접근하면 실행 전에 오류로 잡힌다.

<br>

### 5. Swift 6 엄격 동시성(Strict Concurrency)

| **항목**                         | **Swift 5 언어 모드**                                  | **Swift 6 언어 모드**                                  |
| -------------------------------- | ------------------------------------------------------ | ------------------------------------------------------ |
| **격리 위반·비Sendable 전달**    | 기본 무시, `-strict-concurrency=complete`로 **경고**   | **컴파일 오류**                                        |
| **전역 가변 변수 (`var`)**       | 허용                                                   | 오류 — 액터 격리하거나 `let`, 또는 `nonisolated(unsafe)` |
| **점진적 적용**                  | 모듈별 경고 확인 후 정리                               | 모듈 단위로 언어 모드 전환 (Swift 6 컴파일러에서 선택) |
| **Swift 6.2 완화 옵션**          | -                                                      | 기본 격리를 `MainActor`로 두는 설정(SE-0466) 등 도입   |

- Swift 6 **컴파일러**와 Swift 6 **언어 모드**는 다르다. Xcode 16+의 Swift 6 컴파일러로도 Swift 5 언어 모드를 유지할 수 있으며, 마이그레이션은 보통 `complete` 경고를 모두 정리한 뒤 모듈별로 6 모드를 켜는 순서로 진행함
- 흔한 오류와 대응: 비Sendable 클래스를 `Task`에 캡처 → 값 타입·액터로 변경, 싱글턴 전역 `var` → `@MainActor` 또는 `actor`로 격리, 콜백 클로저가 메인 격리 상태를 건드림 → `@MainActor` 클로저로 표시

```
격리 영역 지도
┌──────────────┐   Sendable 값만 통과   ┌──────────────┐   Sendable 값만 통과   ┌────────────────┐
│ @MainActor   │ ◀────────────────────▶ │ actor Cache  │ ◀────────────────────▶ │ nonisolated    │
│ UI 상태      │                        │ 격리 상태    │                        │ (협력 스레드 풀)│
└──────────────┘                        └──────────────┘                        └────────────────┘
   비Sendable 참조가 경계를 넘으려 하면 → Swift 6: 컴파일 오류 / Swift 5 complete: 경고
```

<br>

### 6. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                  |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **actor와 class의 차이는?**                           | 상태 접근이 직렬화되고 외부 접근에 `await`가 강제됨. 상속 불가, 항상 Sendable                  |
| **actor는 재진입이 가능한가?**                        | 가능하다. `await` 중 다른 요청이 처리될 수 있으므로 `await` 후 상태를 재확인해야 함            |
| **Sendable이란?**                                     | 격리 경계를 넘어도 안전한 타입임을 나타내는 마커 프로토콜. 값 타입은 대부분 자동 준수          |
| **@unchecked Sendable은 언제 쓰나?**                  | 내부에서 락 등으로 직접 동기화한 클래스. 컴파일러 검사를 끄는 것이므로 최소화                  |
| **@MainActor의 역할은?**                              | 메인 스레드 전역 액터. UI 상태 접근을 컴파일 타임에 메인으로 강제                              |
| **Swift 6에서 달라지는 점은?**                        | 데이터 경합 가능성이 컴파일 오류. 전역 가변 변수·비Sendable 캡처가 대표적 오류                 |

- 격리의 세 도구는 **actor(자체 상태 보호)**, **Sendable(경계 통과 허가)**, **@MainActor(UI 격리)**
- 액터는 "자동 락"이 아니라 "동기 구간만 원자적"이며, **재진입**을 항상 염두에 둠
- 컴파일러가 잡아 주는 오류 대부분은 **참조 타입 공유**에서 오므로, unit01의 "값 타입 기본" 원칙이 동시성 안전의 출발점임
