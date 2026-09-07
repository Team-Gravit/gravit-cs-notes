## 화면 전환과 흐름 제어

화면 전환은 "어떤 화면을 보여줄지"뿐 아니라 **"누가 그 결정을 소유하는가"**의 문제다. UIKit에서는 뷰 컨트롤러가 직접 다음 화면을 push하는 구조가 흐름을 화면에 종속시키기 때문에 **Coordinator 패턴**으로 흐름을 분리하고, SwiftUI에서는 iOS 16+ **`NavigationStack`**과 `NavigationPath`가 내비게이션을 **상태(데이터)**로 표현한다. 딥링크·복귀·테스트 가능성은 모두 이 소유권 설계에서 갈린다.

<br>

### 1. UIKit의 화면 전환 기본

| **방식**                       | **API**                                            | **특징**                                        |
| ------------------------------ | -------------------------------------------------- | ----------------------------------------------- |
| **스택 push/pop**              | `UINavigationController.pushViewController`        | 뒤로 가기 내장, 계층적 탐색                     |
| **모달 present/dismiss**       | `UIViewController.present(_:animated:)`            | 독립 작업 흐름(로그인·작성), `modalPresentationStyle` |
| **탭 전환**                    | `UITabBarController.selectedIndex`                 | 병렬 최상위 흐름                                |
| **컨테이너 교체**              | `addChild`·`removeFromParent`                      | 커스텀 컨테이너, 온보딩 → 메인 전환             |

```swift
// 안티패턴: 화면이 다음 화면을 직접 알고 생성함
final class OrderListVC: UIViewController {
    func didSelect(order: Order) {
        let detail = OrderDetailVC(order: order)              // 의존성 생성까지 떠안음
        navigationController?.pushViewController(detail, animated: true)
    }
}
```

- `OrderListVC`가 `OrderDetailVC`의 존재·생성 방법·전환 방식을 모두 알아야 하므로 **재사용·테스트가 어렵고**, 흐름을 바꾸면 화면 코드를 고쳐야 한다
- VC 생명주기(`viewDidLoad` 등)는 common-ios 챕터에서 다루므로 여기서는 전환 구조에 집중한다

<br>

### 2. Coordinator 패턴

**Coordinator**는 화면(뷰 컨트롤러)에서 **흐름 제어 책임을 분리**한 객체다. 화면은 "무슨 일이 일어났는지"만 알리고, 어디로 갈지는 Coordinator가 결정한다.

```
AppCoordinator
 ├─ AuthCoordinator ──── LoginVC ──▶ SignUpVC
 └─ MainCoordinator (TabBar)
      ├─ OrderCoordinator ── OrderListVC ──▶ OrderDetailVC ──▶ RefundVC
      └─ ProfileCoordinator ─ ProfileVC ──▶ SettingsVC

화면 → (delegate/closure) → Coordinator → 다음 화면 생성·push/present
```

```swift
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    func start()
}

final class OrderCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    private let navigation: UINavigationController
    private let container: DependencyContainer

    init(navigation: UINavigationController, container: DependencyContainer) {
        self.navigation = navigation
        self.container = container
    }

    func start() {
        let list = container.makeOrderListVC()
        list.onSelect = { [weak self] order in self?.showDetail(order) }   // 화면은 이벤트만 알림
        navigation.pushViewController(list, animated: false)
    }

    private func showDetail(_ order: Order) {
        let detail = container.makeOrderDetailVC(order: order)
        detail.onRefund = { [weak self] in self?.showRefund(order) }
        navigation.pushViewController(detail, animated: true)
    }
}
```

- **화면은 흐름을 모른다**: `onSelect` 클로저(또는 델리게이트)만 노출하고 다음 화면을 생성하지 않는다
- **의존성 주입 지점**이 된다: Coordinator가 컨테이너에서 VC를 만들며 필요한 서비스를 주입한다
- **자식 Coordinator**로 흐름을 트리로 나누고, 흐름이 끝나면 부모가 `childCoordinators`에서 제거한다

> ⚠️ Coordinator의 고전적 함정은 **메모리 관리**다. 사용자가 뒤로 가기로 흐름을 빠져나갔는데 부모가 자식 Coordinator를 제거하지 않으면 누수된다. `UINavigationControllerDelegate`의 `didShow`에서 pop을 감지하거나, 화면의 `deinit`/완료 콜백으로 Coordinator 종료를 알려야 한다. 클로저 캡처는 항상 `[weak self]`다.

<br>

### 3. SwiftUI의 내비게이션 — NavigationStack (iOS 16+)

### 3-1. NavigationView에서 NavigationStack으로

- iOS 13~15의 `NavigationView` + `NavigationLink(destination:)`는 **목적지 뷰를 링크에 직접 박아** 넣어 프로그래밍 방식 전환·딥링크가 어려웠고, iOS 16에서 **deprecated** 되었다
- iOS 16+ `NavigationStack`은 **경로(path)를 데이터로 표현**하고, 값 타입별 목적지를 `navigationDestination(for:)`로 선언한다

```swift
struct OrderFlow: View {
    @State private var path = NavigationPath()          // 스택 = 데이터 배열

    var body: some View {
        NavigationStack(path: $path) {
            OrderListView()                              // 루트
                .navigationDestination(for: Order.self) { order in
                    OrderDetailView(order: order)        // Order 값이 push되면 이 뷰
                }
                .navigationDestination(for: RefundRequest.self) { req in
                    RefundView(request: req)
                }
        }
    }
}

// 링크는 "값"만 push
NavigationLink("상세", value: order)

// 프로그래밍 방식 전환: path를 직접 조작
path.append(order)                                       // push
path.removeLast()                                        // pop
path = NavigationPath()                                  // 루트로
```

| **항목**            | **NavigationView + NavigationLink(destination:)** | **NavigationStack + NavigationPath**           |
| ------------------- | ------------------------------------------------- | ---------------------------------------------- |
| **스택 표현**       | 뷰 계층 안에 암묵적                               | **`[Hashable]` 데이터**로 명시적               |
| **프로그래밍 전환** | `isActive` 바인딩, 다단계 push 불안정             | `path.append/removeLast`로 자유롭게            |
| **딥링크**          | 구현 난해                                         | 경로 배열을 조립하면 끝                        |
| **목적지 결정**     | 링크마다 목적지 뷰 지정                           | `navigationDestination(for:)`에서 타입별 한 번 |
| **상태 복원**       | 어려움                                            | `NavigationPath`의 `codable` 표현 저장 가능    |
| **가용 버전**       | iOS 13+ (16에서 deprecated)                       | iOS 16+                                        |

<br>

### 3-2. 모달 프레젠테이션

- `.sheet(item:)`·`.fullScreenCover(item:)`은 **옵셔널 상태**가 `nil`이 아닐 때 표시된다. 표시 여부가 곧 상태이므로 `Bool` 대신 `Identifiable` 값을 쓰면 "무엇을 보여줄지"까지 표현된다
- 시트 안에서 `@Environment(\.dismiss)`로 닫는다 (unit02 참고). 시트 안에 별도의 `NavigationStack`을 두어 독립 흐름을 만들 수 있다

```swift
@State private var editingOrder: Order?

Button("수정") { editingOrder = order }
.sheet(item: $editingOrder) { order in
    NavigationStack { OrderEditView(order: order) }      // 모달 내부 흐름은 별도 스택
}
```

> 💡 iOS 16+에서 `NavigationStack`은 **"내비게이션 상태 = 앱 상태의 일부"**라는 SwiftUI 철학을 완성한다. 화면 전환을 "함수 호출"이 아니라 **"경로 값의 변경"**으로 보면 딥링크·복원·테스트가 같은 코드 경로를 탄다.

<br>

### 4. SwiftUI에서의 Coordinator — Router 객체

`NavigationPath`를 뷰의 `@State`에 두면 뷰가 흐름을 소유하게 된다. 흐름을 뷰에서 분리하려면 **경로를 소유하는 관찰 가능한 Router**를 만들고 뷰 트리에 주입한다.

```swift
enum OrderRoute: Hashable {
    case detail(Order)
    case refund(RefundRequest)
}

@Observable                                   // iOS 17+, 이전에는 ObservableObject + @Published
final class OrderRouter {
    var path: [OrderRoute] = []               // 타입이 하나면 NavigationPath 대신 배열도 가능

    func showDetail(_ order: Order) { path.append(.detail(order)) }
    func popToRoot() { path.removeAll() }

    func handle(deepLink url: URL) {          // gravit://orders/42/refund
        guard let id = url.pathComponents.dropFirst().first.flatMap(Int.init) else { return }
        path = [.detail(Order(id: id)), .refund(RefundRequest(orderID: id))]
    }
}

struct OrderFlowView: View {
    @State private var router = OrderRouter()

    var body: some View {
        NavigationStack(path: $router.path) {
            OrderListView()
                .navigationDestination(for: OrderRoute.self) { route in
                    switch route {
                    case .detail(let order): OrderDetailView(order: order)
                    case .refund(let req):   RefundView(request: req)
                    }
                }
        }
        .environment(router)                  // 하위 뷰는 @Environment(OrderRouter.self)로 접근
        .onOpenURL { router.handle(deepLink: $0) }
    }
}
```

- 화면은 `router.showDetail(order)`만 호출하고 **다음 뷰가 무엇인지 모른다** — UIKit Coordinator와 같은 원칙
- 경로가 `enum` 배열이므로 **단위 테스트에서 화면 없이 흐름을 검증**할 수 있다
- `UIHostingController`로 UIKit 스택에 올린 SwiftUI 화면은 **UIKit 쪽 Coordinator가 흐름을 소유**하고, SwiftUI 안에서 `NavigationStack`을 또 만들지 않는다 (unit05 참고)

> ⚠️ `navigationDestination(for:)`는 **스택 안의 뷰**에 붙여야 하며, 같은 타입에 대해 여러 번 선언하면 경고와 함께 예측 불가능하게 동작한다. 또한 `LazyVStack`·`List` 행 내부처럼 지연 생성되는 곳에 선언하면 등록 시점 문제로 전환이 실패할 수 있으므로 스택 루트 근처에 모아 둔다.

<br>

### 5. 흐름 설계 체크리스트

- **소유권을 한 곳에**: 한 흐름의 경로는 UIKit Coordinator 또는 SwiftUI Router 중 하나만 소유한다
- **화면은 이벤트만 방출**: 클로저·델리게이트·Router 메서드 호출까지만, 목적지 생성은 금지
- **모달은 독립 흐름**: 자체 스택(또는 자식 Coordinator)을 가지며, 완료·취소 결과만 부모에 돌려준다
- **딥링크는 경로 조립**: URL → 경로 값 배열로 변환하는 함수를 Router에 두고 앱 시작·포그라운드 진입 양쪽에서 재사용한다
- **뒤로 가기 동기화**: 시스템 뒤로 가기(스와이프)로 pop되면 경로 상태도 함께 줄어드는지 확인한다 (`NavigationStack`은 자동, UIKit은 델리게이트에서 처리)

<br>

### 6. 정리

| **질문**                                              | **핵심 답변**                                                                          |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **Coordinator 패턴의 목적은?**                        | 화면에서 **흐름 제어와 의존성 생성 책임을 분리**해 재사용·테스트를 가능하게 함          |
| **Coordinator의 대표 함정은?**                        | 자식 Coordinator **미해제로 인한 누수**, 시스템 뒤로 가기와의 동기화                   |
| **NavigationStack이 NavigationView와 다른 점은?**     | 스택을 **데이터(`NavigationPath`)**로 표현해 프로그래밍 전환·딥링크·복원이 쉬움         |
| **SwiftUI에서 흐름을 뷰에서 분리하려면?**             | 경로를 소유한 **Router 객체**를 환경으로 주입, 화면은 Router 메서드만 호출             |
| **UIKit 스택 위의 SwiftUI 화면 전환은 누가?**         | UIKit **Coordinator가 소유**, SwiftUI 안에 중첩 스택을 만들지 않음                     |

- UIKit·SwiftUI 모두 핵심은 **"화면은 흐름을 모른다"**와 **"내비게이션은 상태다"**이다
- 상태 래퍼와 환경 주입은 unit02·unit03, 상호운용 경계는 unit05를 참고할 것
