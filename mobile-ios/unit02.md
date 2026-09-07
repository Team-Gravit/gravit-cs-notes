## 상태 프로퍼티 래퍼

SwiftUI 뷰는 매 갱신마다 새로 생성되는 값이므로(unit01 참고), 유지돼야 할 상태는 **프로퍼티 래퍼(Property Wrapper)**를 통해 프레임워크 저장소에 맡기고 뷰는 이를 참조만 한다. `@State`·`@Binding`·`@ObservedObject`·`@StateObject`·`@Environment`는 각각 **"누가 소유하는가"와 "값 타입인가 참조 타입인가"**로 역할이 갈리며, 이를 잘못 고르면 값이 사라지거나 객체가 매번 재생성되는 버그가 생긴다.

<br>

### 1. 프로퍼티 래퍼가 필요한 이유

- 구조체인 뷰는 `body` 안에서 자기 프로퍼티를 **수정할 수 없다** (`mutating` 불가)
- 뷰 값은 렌더마다 다시 만들어지므로 일반 저장 프로퍼티는 **값이 유지되지 않는다**
- 래퍼는 실제 값을 **뷰 밖(SwiftUI 관리 저장소)**에 두고, 값이 바뀌면 뷰를 무효화해 `body`를 재평가하도록 연결한다

```
        뷰 구조체 (매 렌더마다 새로 생성)
        ┌────────────────────────┐
        │ @State var count ──────┼──▶ SwiftUI 저장소 (정체성에 묶여 유지)
        │ @Binding var text ─────┼──▶ 다른 뷰의 저장소를 가리키는 참조
        │ @StateObject var vm ───┼──▶ 저장소가 소유하는 참조 객체
        │ @ObservedObject var m ─┼──▶ 외부가 소유하는 참조 객체 (구독만)
        │ @Environment(\.x) ─────┼──▶ 뷰 트리를 타고 내려오는 공유 값
        └────────────────────────┘
```

<br>

### 2. 값 타입 상태 — @State와 @Binding

### 2-1. @State: 뷰가 소유하는 지역 상태

- 뷰 **자신만 사용하는 단순 값**(토글 여부, 입력 텍스트, 선택 인덱스)에 사용한다
- 값은 뷰의 **정체성**에 묶여 보존되며, 정체성이 바뀌면 초기화된다
- 외부에서 접근할 이유가 없으므로 관례적으로 `private`으로 선언한다

```swift
struct LoginForm: View {
    @State private var email = ""
    @State private var isAgreed = false

    var body: some View {
        Form {
            TextField("이메일", text: $email)      // $email → Binding<String>
            AgreementRow(isAgreed: $isAgreed)      // 자식에게 쓰기 권한을 넘김
            Button("로그인") { /* ... */ }
                .disabled(email.isEmpty || !isAgreed)
        }
    }
}
```

<br>

### 2-2. @Binding: 소유하지 않고 읽고 쓰는 참조

- 부모의 `@State`(또는 다른 바인딩)를 **소유 없이 읽고 쓰는** 통로다. `$` 접두사로 `Binding<T>`를 얻는다
- 자식이 값을 바꾸면 **부모의 원본이 바뀌고**, 부모가 재평가되면서 자식도 갱신된다 (단일 진실 공급원)

```swift
struct AgreementRow: View {
    @Binding var isAgreed: Bool          // 소유하지 않음, 초기값 없음

    var body: some View {
        Toggle("약관에 동의합니다", isOn: $isAgreed)
    }
}
```

> 💡 값을 "읽기만" 하는 자식에게는 `@Binding`이 아니라 **일반 `let` 프로퍼티**로 넘긴다. 바인딩은 "자식이 부모 상태를 바꿔야 할 때"만 쓰는 것이 데이터 흐름을 읽기 쉽게 만든다. 이 원칙이 SwiftUI의 **단방향 데이터 흐름**이다.

<br>

### 3. 참조 타입 상태 — ObservableObject 계열

값 타입으로 표현하기 어려운 **모델·뷰모델(네트워크, 저장소, 여러 화면이 공유하는 데이터)**은 `ObservableObject` 프로토콜을 채택한 클래스로 만들고, `@Published` 프로퍼티가 바뀔 때 `objectWillChange`가 발행돼 구독 중인 뷰가 갱신된다. (iOS 17+ `@Observable`은 unit03 참고)

```swift
final class CartViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading = false

    func load() async { /* ... */ }
}
```

<br>

### 3-1. @StateObject vs @ObservedObject — 소유권이 갈림

| **항목**          | **@StateObject** (iOS 14+)                     | **@ObservedObject**                              |
| ----------------- | ---------------------------------------------- | ------------------------------------------------ |
| **소유자**        | **이 뷰** — SwiftUI가 생성·보관                | **외부** — 부모나 DI 컨테이너가 생성해 넘김      |
| **생성 시점**     | 뷰 정체성이 생길 때 **한 번**                  | 뷰 값이 생성될 때마다 인자를 받음                |
| **재평가 시 유지** | 유지됨                                         | 넘겨받은 인스턴스를 그대로 사용                  |
| **사용 위치**     | 객체를 처음 만드는 뷰 (화면 루트 등)           | 객체를 전달받는 하위 뷰                          |

```swift
// 안티패턴: 부모가 재평가될 때마다 CartViewModel이 새로 생성되어 items가 사라짐
struct CartScreen: View {
    @ObservedObject var vm = CartViewModel()   // 뷰 값 생성마다 새 객체
    var body: some View { CartList(vm: vm) }
}
```

```swift
// 개선: 처음 만드는 곳은 @StateObject, 전달받는 곳은 @ObservedObject
struct CartScreen: View {
    @StateObject private var vm = CartViewModel()   // 정체성 수명 동안 한 번만 생성
    var body: some View { CartList(vm: vm) }
}

struct CartList: View {
    @ObservedObject var vm: CartViewModel           // 소유하지 않고 구독만
    var body: some View {
        List(vm.items) { Text($0.name) }
    }
}
```

> ⚠️ `@StateObject`의 초기값 표현식은 **지연 평가**돼 정체성이 처음 생길 때만 실행된다. 반면 `@ObservedObject var vm = X()`는 뷰 값이 만들어질 때마다 `X()`가 실행된다. "탭을 바꿨다 오니 목록이 사라졌다", "타이핑할 때마다 네트워크 요청이 다시 나간다"는 대표 증상이다.

<br>

### 3-2. @EnvironmentObject: 뷰 트리를 타고 내려오는 공유 객체

- 여러 계층 아래의 뷰가 같은 객체를 써야 할 때, 매 단계마다 인자로 넘기는 대신 상위에서 `.environmentObject(_:)`로 주입하고 하위에서 `@EnvironmentObject`로 꺼낸다
- 주입되지 않은 타입을 꺼내면 **런타임 크래시**가 나므로 앱 루트(또는 프리뷰)에서 반드시 주입해야 한다

```swift
@main
struct ShopApp: App {
    @StateObject private var session = UserSession()
    var body: some Scene {
        WindowGroup {
            RootView().environmentObject(session)
        }
    }
}

struct ProfileBadge: View {
    @EnvironmentObject var session: UserSession    // 타입으로 조회
    var body: some View { Text(session.user.name) }
}
```

<br>

### 4. @Environment: 시스템·커스텀 환경값

`@Environment`는 객체가 아니라 **키 경로(KeyPath)로 식별되는 값**을 뷰 트리에서 읽는다. 다크 모드, 동적 타입, `dismiss` 같은 시스템 값이 대표적이며, 커스텀 키도 정의할 수 있다.

```swift
struct DetailView: View {
    @Environment(\.dismiss) private var dismiss
    @Environment(\.colorScheme) private var scheme

    var body: some View {
        VStack {
            Text(scheme == .dark ? "다크" : "라이트")
            Button("닫기") { dismiss() }
        }
    }
}
```

- 환경값은 **상위에서 하위로만** 흐르며, `.environment(\.key, value)`로 하위 트리 전체에 덮어쓸 수 있다
- iOS 17+에서는 `@Observable` 객체도 `@Environment(Type.self)`로 주입·조회한다 (unit03 참고)

<br>

### 5. 선택 기준 요약

| **상황**                                        | **래퍼**                | **이유**                                    |
| ----------------------------------------------- | ----------------------- | ------------------------------------------- |
| 뷰 내부에서만 쓰는 단순 값                      | **@State**              | 값 타입, 뷰가 소유                          |
| 부모 상태를 자식이 **수정**해야 함              | **@Binding**            | 소유 없이 읽고 쓰기, 단일 진실 공급원 유지  |
| 자식이 값을 **읽기만** 함                       | 일반 `let`              | 바인딩 불필요, 흐름 단순화                  |
| 참조 객체를 **이 뷰에서 처음 생성**             | **@StateObject**        | 정체성 수명 동안 한 번만 생성               |
| 참조 객체를 **외부에서 받아** 구독              | **@ObservedObject**     | 소유하지 않음, 재생성 방지                  |
| 깊은 하위 뷰 여러 곳에서 같은 객체 공유         | **@EnvironmentObject**  | 프로퍼티 드릴링 제거                        |
| 다크 모드·dismiss 등 시스템/환경 값             | **@Environment**        | 키 경로 기반 조회                           |

> 💡 판단 순서는 **① 값 타입인가 참조 타입인가 → ② 이 뷰가 소유하는가 전달받는가 → ③ 몇 단계나 내려가는가**다. 이 세 질문에 답하면 래퍼가 하나로 결정된다.

<br>

### 6. 흔한 실수 모음

- **`@State`에 클래스 인스턴스를 담기**: 프로퍼티 변경이 감지되지 않는다. `ObservableObject`(또는 iOS 17+ `@Observable`, 이 경우는 `@State` 사용이 정답)로 바꾼다
- **`@Binding`을 초기값과 함께 선언하기**: 바인딩은 외부에서 반드시 주입돼야 하며, 기본값이 필요하면 `Binding.constant(_:)`는 프리뷰에서만 쓴다
- **`@ObservedObject`를 `init`에서 생성하기**: `@StateObject`로 바꾸거나, 외부에서 생성해 넘긴다
- **`@EnvironmentObject` 주입 누락**: 프리뷰·테스트에서 크래시가 가장 자주 나는 지점이다
- **뷰모델에 UI 상태까지 몰아넣기**: 토글·포커스 같은 순수 UI 상태는 `@State`로 두고, 뷰모델은 도메인 상태만 담는다

<br>

### 7. 정리

- 뷰는 값이므로 상태는 **프레임워크 저장소**에 두고 래퍼로 연결하며, 값이 바뀌면 의존하는 뷰가 재평가된다
- **@State/@Binding**은 값 타입 상태의 소유/참조, **@StateObject/@ObservedObject**는 참조 객체의 소유/참조를 담당한다
- `@StateObject`는 정체성 수명 동안 **한 번만** 생성되고, `@ObservedObject`는 **외부에서 받은** 객체를 구독만 한다
- **@EnvironmentObject/@Environment**는 뷰 트리를 통해 아래로 흐르는 공유 값이며, 주입 누락은 크래시로 이어진다
- iOS 17+ Observation 프레임워크에서는 이 래퍼 구성이 `@State`·`@Environment`·`@Bindable` 중심으로 단순화된다 → unit03 참고
