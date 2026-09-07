## Observation 프레임워크

**Observation**은 iOS 17(Swift 5.9)에서 도입된 관찰 프레임워크로, `@Observable` 매크로 하나로 클래스의 프로퍼티 변화를 추적한다. 기존 `ObservableObject`·`@Published` 조합이 **객체 단위**로 뷰를 갱신했던 것과 달리 **실제로 읽은 프로퍼티 단위**로 갱신하므로 불필요한 `body` 재평가가 크게 줄고, 프로퍼티 래퍼 구성도 단순해진다.

<br>

### 1. 기존 방식의 한계 — ObservableObject

`ObservableObject`는 Combine 기반이다. `@Published` 프로퍼티가 바뀌면 `objectWillChange` 퍼블리셔가 발행되고, 이 객체를 구독하는 **모든 뷰의 body가 재평가**된다.

```swift
final class ProfileViewModel: ObservableObject {
    @Published var name = ""
    @Published var avatarURL: URL?
    @Published var draftMemo = ""      // 타이핑마다 발행
}

struct HeaderView: View {
    @ObservedObject var vm: ProfileViewModel
    var body: some View {
        Text(vm.name)                  // name만 읽지만 draftMemo가 바뀌어도 재평가됨
    }
}
```

- 뷰가 어떤 프로퍼티를 읽었는지와 무관하게 **객체 전체가 갱신 단위**다
- `@Published`를 붙인 프로퍼티만 감지되고, 중첩 객체의 변경은 전파되지 않는다
- 래퍼가 `@StateObject`·`@ObservedObject`·`@EnvironmentObject` 세 종류로 나뉘어 소유권 실수가 잦다 (unit02 참고)

<br>

### 2. @Observable 매크로의 동작 원리

`@Observable`은 Swift 매크로다. 컴파일 시점에 클래스를 확장해 **모든 저장 프로퍼티의 접근자에 추적 코드를 삽입**한다.

```swift
import Observation

@Observable
final class ProfileViewModel {
    var name = ""
    var avatarURL: URL?
    var draftMemo = ""
    @ObservationIgnored var cache: [String: Data] = [:]   // 추적 제외
}
```

매크로가 생성하는 코드를 개념적으로 펼치면 다음과 같다.

```swift
// 매크로 확장 결과(개념)
final class ProfileViewModel: Observable {
    private let _$observationRegistrar = ObservationRegistrar()
    private var _name = ""
    var name: String {
        get { _$observationRegistrar.access(self, keyPath: \.name); return _name }
        set { _$observationRegistrar.withMutation(of: self, keyPath: \.name) { _name = newValue } }
    }
    // ... 나머지 프로퍼티도 동일
}
```

```
[body 평가 시작]
   │  vm.name 읽기 ──▶ registrar.access(\.name) ──▶ "이 뷰는 name에 의존" 기록
   │
[body 평가 종료] ──▶ 의존 집합 = { \.name }
   │
[vm.draftMemo = "..."] ──▶ withMutation(\.draftMemo) ──▶ 의존 집합에 없음 → 무시
[vm.name = "..."]      ──▶ withMutation(\.name)      ──▶ HeaderView 무효화
```

- **읽기(access)**가 의존성을 등록하고, **쓰기(withMutation)**가 등록된 관찰자에게 알린다
- `@Published` 같은 표시가 필요 없고, 계산 프로퍼티는 내부에서 읽는 저장 프로퍼티를 통해 자동 추적된다
- SwiftUI 밖에서는 `withObservationTracking(_:onChange:)`로 같은 메커니즘을 직접 사용할 수 있다

> 💡 `withObservationTracking`의 `onChange`는 **변경 직전에 한 번만** 호출되고 이후 자동 해제된다(willSet 성격). 계속 관찰하려면 `onChange` 안에서 다시 등록해야 한다. 이 특성을 모르면 "첫 변경만 감지된다"는 버그로 보인다.

<br>

### 3. 갱신 범위 비교 — 객체 단위 vs 프로퍼티 단위

| **항목**            | **ObservableObject + @Published**            | **@Observable** (iOS 17+)                      |
| ------------------- | -------------------------------------------- | ---------------------------------------------- |
| **기반**            | Combine (`objectWillChange`)                 | Observation 프레임워크 (매크로 + 레지스트라)   |
| **갱신 단위**       | **객체** — 어느 프로퍼티든 바뀌면 전체 구독 뷰 | **프로퍼티** — body가 읽은 것만               |
| **추적 대상 표시**  | `@Published` 필요                            | 기본 추적, 제외 시 `@ObservationIgnored`       |
| **중첩 객체**       | 전파 안 됨 (내부 객체를 따로 구독)           | 내부 객체도 `@Observable`이면 읽은 경로만 추적 |
| **계산 프로퍼티**   | 감지 불가                                    | 내부 저장 프로퍼티 통해 자동 추적              |
| **최소 버전**       | iOS 13                                       | iOS 17                                         |
| **뷰 래퍼**         | @StateObject / @ObservedObject / @EnvironmentObject | @State / 일반 프로퍼티 / @Environment / @Bindable |

<br>

### 4. SwiftUI에서의 사용 — 래퍼 대응 관계

| **역할**                       | **ObservableObject 시절**       | **@Observable**                          |
| ------------------------------ | ------------------------------- | ---------------------------------------- |
| 이 뷰가 **소유·생성**          | `@StateObject`                  | **`@State`**                             |
| 외부에서 받아 **읽기만**       | `@ObservedObject`               | **일반 프로퍼티** (`let vm: VM`)         |
| 외부에서 받아 **바인딩 필요**  | `@ObservedObject` + `$vm.x`     | **`@Bindable var vm`** + `$vm.x`         |
| 뷰 트리로 **주입**             | `.environmentObject(obj)` / `@EnvironmentObject` | `.environment(obj)` / **`@Environment(VM.self)`** |

```swift
struct ProfileScreen: View {
    @State private var vm = ProfileViewModel()      // 소유: @State로 충분

    var body: some View {
        VStack {
            HeaderView(vm: vm)                      // 읽기 전용 전달
            MemoEditor(vm: vm)                      // 바인딩 필요
        }
        .environment(vm)                            // 하위 트리 주입
    }
}

struct MemoEditor: View {
    @Bindable var vm: ProfileViewModel              // $vm.draftMemo 사용 가능
    var body: some View {
        TextEditor(text: $vm.draftMemo)             // draftMemo에만 의존
    }
}

struct FooterView: View {
    @Environment(ProfileViewModel.self) private var vm
    var body: some View {
        @Bindable var vm = vm                       // 환경에서 꺼낸 객체를 바인딩할 때
        Toggle("공개", isOn: $vm.isPublic)
    }
}
```

> ⚠️ `@Observable` 객체를 `@State`가 아닌 **일반 프로퍼티에 담아 뷰 안에서 생성**하면(`let vm = VM()`), 뷰 값이 재생성될 때마다 객체가 새로 만들어진다. `@ObservedObject var vm = X()`와 같은 실수다. 생성하는 곳은 반드시 `@State`, 전달받는 곳은 일반 프로퍼티다.

<br>

### 5. 흔한 함정

### 5-1. 컬렉션 요소의 변경

- `vm.items.append(x)`는 `items` 프로퍼티 자체의 변경이므로 추적된다
- `vm.items[0].title = "..."`은 `items`가 **구조체 배열**이면 배열 전체가 바뀐 것으로 추적된다
- `items`가 **클래스 배열**이면 요소 객체도 `@Observable`이어야 하고, 뷰가 `item.title`을 직접 읽어야 그 프로퍼티에 의존한다

<br>

### 5-2. body 밖에서 읽은 값

- 의존성은 **body 평가 중에 읽은 프로퍼티**에만 등록된다. `onAppear`나 버튼 액션 클로저 안에서만 읽은 프로퍼티는 뷰를 갱신시키지 않는다
- 반대로 `body`에서 읽지 않은 값이 바뀌었는데 화면이 안 바뀐다면 버그가 아니라 **의도된 동작**이다

<br>

### 5-3. 스레드와 메인 액터

- `@Observable`은 스레드 안전성을 보장하지 않는다. UI에 바인딩되는 모델은 `@MainActor`로 격리해 백그라운드에서의 변경이 UI를 깨뜨리지 않게 한다

```swift
@MainActor
@Observable
final class FeedViewModel {
    private(set) var posts: [Post] = []
    func load() async {
        posts = await api.fetchPosts()       // 메인 액터에서 대입
    }
}
```

<br>

### 6. UIKit과 Observation (iOS 26+)

- iOS 26부터 UIKit도 `layoutSubviews()`·`updateProperties()` 같은 **갱신 메서드 안에서 읽은 `@Observable` 프로퍼티를 자동 추적**한다. 프로퍼티가 바뀌면 해당 뷰가 무효화되어 갱신 메서드가 다시 실행되므로 `setNeedsLayout()`을 수동으로 부를 필요가 없다
- `updateProperties()`는 크기·위치에 영향 없는 속성(텍스트·색상·이미지) 갱신용으로 `layoutSubviews()` 직전에 호출된다
- Info.plist의 `UIObservationTrackingEnabled` 키로 iOS 18까지 백포트할 수 있다(버전·세부 동작은 릴리스에 따라 다를 수 있음)

```swift
final class ProfileCell: UITableViewCell {
    var vm: ProfileViewModel?                       // @Observable 객체
    override func updateProperties() {
        super.updateProperties()
        textLabel?.text = vm?.name                  // 여기서 읽은 name이 바뀌면 자동 재호출
    }
}
```

> 💡 Swift 6.2/iOS 26에서는 `Observations { ... }`로 `@Observable` 프로퍼티 변화를 **AsyncSequence**로 받는 API도 추가됐다. Combine 없이 `for await`로 모델 변화를 소비할 수 있어 unit09에서 다루는 "Combine vs async/await" 선택에도 영향을 준다.

<br>

### 7. 마이그레이션 체크리스트

- `ObservableObject` 채택 제거 → `@Observable` 추가, `@Published` 제거
- `@StateObject` → `@State`, `@ObservedObject` → 일반 프로퍼티(바인딩 필요 시 `@Bindable`), `@EnvironmentObject` → `@Environment(Type.self)`
- `.environmentObject(obj)` → `.environment(obj)`
- `objectWillChange`를 직접 구독하던 코드는 `withObservationTracking` 또는 `Observations`로 대체
- 배포 최소 버전이 iOS 16 이하이면 두 방식을 혼용해야 하므로, 모델 계층에서 어느 쪽을 표준으로 삼을지 먼저 정한다

<br>

### 8. 정리

| **질문**                                     | **핵심 답변**                                                                          |
| -------------------------------------------- | -------------------------------------------------------------------------------------- |
| **@Observable과 ObservableObject의 차이는?** | 갱신 단위가 **객체 → 프로퍼티**로 세분화됨. Combine 의존 제거, 래퍼 단순화             |
| **어떻게 프로퍼티 단위 추적이 가능한가?**    | 매크로가 접근자에 **access/withMutation**을 삽입해 body가 읽은 키 경로만 등록          |
| **@StateObject는 무엇으로 바뀌나?**          | **@State**. 전달은 일반 프로퍼티, 바인딩은 **@Bindable**, 주입은 `@Environment(T.self)` |
| **왜 화면이 갱신되지 않는가?**               | body에서 **읽지 않은** 프로퍼티이거나 `@ObservationIgnored`, 또는 요소 객체가 비관찰   |
| **UIKit에서도 쓸 수 있나?**                  | iOS 26+ `updateProperties()`·`layoutSubviews()`에서 자동 추적, iOS 18 백포트 키 존재   |

- 신규 프로젝트(iOS 17+)는 `@Observable`을 기본으로 하고, 하위 호환이 필요할 때만 `ObservableObject`를 유지한다
- 뷰 정체성과 재평가 원리는 unit01, 기존 래퍼의 소유권 규칙은 unit02, Combine 기반 바인딩은 unit09를 참고할 것
