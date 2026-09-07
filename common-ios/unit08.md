## 앱 아키텍처

앱이 커질수록 "네트워크·상태·화면 전환 코드를 어디에 둘 것인가"가 유지보수 비용을 좌우한다. 이 유닛은 iOS에서 가장 널리 쓰이는 **MVC·MVVM·MVVM-C**를 책임 분리 관점에서 비교하고, 어떤 아키텍처를 쓰든 **테스트 가능한 코드**를 만드는 핵심 수단인 의존성 주입(Dependency Injection)을 정리한다.

<br>

### 1. 아키텍처를 고민하는 이유

- UIKit의 `UIViewController`는 뷰 생명주기(unit02), 사용자 입력, 데이터 로드, 화면 전환을 모두 받을 수 있어 **모든 코드가 뷰 컨트롤러로 모이는 구조**가 되기 쉬움
- 비대해진 뷰 컨트롤러는 단위 테스트가 불가능하고(UIKit 없이는 인스턴스도 못 만듦), 한 화면을 고치면 다른 기능이 깨지는 회귀가 잦음
- 아키텍처의 목적은 패턴 이름을 맞추는 것이 아니라 **역할을 분리해 각 조각을 독립적으로 이해·교체·테스트할 수 있게 만드는 것**임

> 💡 면접에서 "MVVM을 썼다"고 답하면 반드시 "왜 MVC가 아니라 MVVM인가", "뷰모델은 어떻게 테스트했는가"가 이어진다. 패턴 이름보다 **어떤 문제를 어떻게 풀었는지**를 답할 수 있어야 한다.

<br>

### 2. MVC — Apple 방식의 MVC와 Massive View Controller

```
       사용자 입력                     갱신
  View ───────────▶ Controller ◀─────────── Model
   ▲                    │                     ▲
   └──── UI 갱신 ───────┘─── 요청·변경 ────────┘
```

- Apple의 MVC는 View와 Model이 직접 통신하지 않고 **Controller가 양쪽을 중재**함. UIKit에서 Controller는 곧 `UIViewController`
- 문제는 뷰 컨트롤러가 뷰의 생명주기까지 소유하므로 **View와 Controller의 경계가 사실상 없어지고**, 남은 로직이 모두 여기로 쏟아진다는 점 → 이른바 **Massive View Controller**
- 화면이 단순하고 수명이 짧은 프로젝트에서는 여전히 가장 빠른 선택임

```swift
// 안티패턴: 네트워크·파싱·포맷팅·화면 전환이 모두 뷰 컨트롤러에
final class OrderListViewController: UIViewController {
    var orders: [Order] = []

    override func viewDidLoad() {
        super.viewDidLoad()
        URLSession.shared.dataTask(with: URL(string: "https://api.example.com/orders")!) { data, _, _ in
            guard let data, let orders = try? JSONDecoder().decode([Order].self, from: data) else { return }
            self.orders = orders.filter { $0.status != .cancelled }
            DispatchQueue.main.async { self.tableView.reloadData() }
        }.resume()
    }

    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let detail = OrderDetailViewController(order: orders[indexPath.row])
        navigationController?.pushViewController(detail, animated: true)
    }
}
```

이 코드는 서버 없이는 필터링 규칙조차 테스트할 수 없고, `OrderDetailViewController`를 직접 생성하므로 목록 화면이 상세 화면의 존재를 알아야 한다.

<br>

### 3. MVVM — 표현 로직을 ViewModel로

### 3-1. 구조

```
  View / ViewController ──── 입력 전달 ────▶ ViewModel ──── 요청 ────▶ Model / Service
          ▲                                     │                          │
          └──────── 바인딩(상태 관찰) ──────────┘◀──────── 결과 ──────────┘
```

- **ViewModel**은 화면에 표시할 상태(목록·로딩 여부·오류 메시지)와 사용자 액션 처리를 담당하며, **UIKit을 import하지 않음** → 순수 Swift 객체로 단위 테스트 가능
- **View(ViewController)**는 ViewModel의 상태를 관찰해 그리기만 하고, 입력을 ViewModel에 전달함
- 바인딩 수단은 클로저, `Combine`의 `@Published`, `Observation`(iOS 17+) 등 프로젝트 선택에 따름. 바인딩 방식이 MVVM의 본질은 아님

```swift
protocol OrderService {
    func fetchOrders() async throws -> [Order]
}

@MainActor
final class OrderListViewModel {
    @Published private(set) var rows: [OrderRow] = []
    @Published private(set) var isLoading = false
    @Published private(set) var errorMessage: String?

    private let service: OrderService

    init(service: OrderService) {          // 의존성 주입: 구체 타입이 아니라 프로토콜을 받음
        self.service = service
    }

    func load() async {
        isLoading = true
        defer { isLoading = false }
        do {
            let orders = try await service.fetchOrders()
            rows = orders
                .filter { $0.status != .cancelled }
                .map { OrderRow(id: $0.id, title: $0.productName, priceText: $0.price.formatted(.currency(code: "KRW"))) }
        } catch {
            errorMessage = "주문을 불러오지 못했습니다."
        }
    }
}
```

```swift
// ViewController: 관찰과 그리기만 담당
final class OrderListViewController: UIViewController {
    private let viewModel: OrderListViewModel
    private var cancellables = Set<AnyCancellable>()

    init(viewModel: OrderListViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        viewModel.$rows
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in self?.tableView.reloadData() }
            .store(in: &cancellables)
        Task { await viewModel.load() }
    }
}
```

<br>

### 3-2. MVVM의 흔한 함정

- **ViewModel이 UIKit을 알게 되는 것**: `UIImage`·`UIColor`를 ViewModel에서 만들면 테스트가 다시 어려워짐. 이미지 이름·색상 토큰 같은 **값**만 넘기고 변환은 View가 담당
- **Massive ViewModel**: 뷰 컨트롤러의 코드가 그대로 옮겨 가면 문제가 위치만 바뀜. 네트워크·저장은 Service/Repository 계층으로, 화면 전환은 Coordinator로 분리
- **화면 전환의 소유권**: ViewModel이 `navigationController`를 참조하는 순간 UIKit 의존이 생김 → MVVM-C의 등장 배경

> ⚠️ `@Published` 프로퍼티를 백그라운드 스레드에서 갱신하면 UI 갱신이 메인 스레드 밖에서 일어나 크래시하거나 경고가 발생한다. ViewModel을 `@MainActor`로 격리하거나 `receive(on: DispatchQueue.main)`을 반드시 두어야 한다(네트워크 콜백 스레드는 unit06 참고).

<br>

### 4. MVVM-C — 화면 전환을 Coordinator로

**Coordinator**는 "어떤 화면 다음에 어떤 화면이 오는가"라는 **내비게이션 흐름**을 전담하는 객체다. 뷰 컨트롤러와 ViewModel은 "상세로 가고 싶다"는 의도만 알리고, 실제 push·present는 Coordinator가 수행한다.

```
AppCoordinator
 ├─ AuthCoordinator ─── LoginVC → SignUpVC
 └─ MainCoordinator ─── TabBar
        ├─ OrderCoordinator ─── OrderListVC → OrderDetailVC → RefundVC
        └─ ProfileCoordinator ── ProfileVC → SettingsVC
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
        let viewModel = OrderListViewModel(service: container.orderService)
        viewModel.onSelectOrder = { [weak self] orderID in       // 의도만 전달
            self?.showDetail(orderID: orderID)
        }
        navigation.pushViewController(OrderListViewController(viewModel: viewModel), animated: false)
    }

    private func showDetail(orderID: Order.ID) {
        let viewModel = OrderDetailViewModel(orderID: orderID, service: container.orderService)
        navigation.pushViewController(OrderDetailViewController(viewModel: viewModel), animated: true)
    }
}
```

- 각 화면은 다음 화면의 존재를 모르므로 **재사용과 흐름 변경(A/B 테스트, 딥링크 진입)**이 쉬워짐. 푸시 알림(unit09)·Universal Link(unit07)로 특정 화면에 진입할 때도 Coordinator에 "이 화면으로 가라"고 요청하면 됨
- 자식 Coordinator는 부모가 `childCoordinators`에 보관해 해제되지 않게 하고, 흐름이 끝나면 제거해야 함. 이 관리를 빠뜨리면 **메모리 누수 또는 조기 해제**가 발생함

<br>

### 5. 세 패턴 비교

| **항목**               | **MVC**                          | **MVVM**                                   | **MVVM-C**                                        |
| ---------------------- | -------------------------------- | ------------------------------------------ | ------------------------------------------------- |
| **표현 로직 위치**     | ViewController                   | **ViewModel** (UIKit 비의존)                | ViewModel                                         |
| **화면 전환 위치**     | ViewController                   | ViewController (또는 ViewModel이 침범)       | **Coordinator**                                   |
| **단위 테스트 범위**   | 거의 불가                        | ViewModel·Service                          | ViewModel·Service·**흐름(Coordinator)**             |
| **보일러플레이트**     | 최소                             | 중간 (바인딩 코드)                          | 많음 (Coordinator 계층)                            |
| **적합한 규모**        | 프로토타입, 화면 수 적은 앱        | 대부분의 상용 앱                            | 흐름이 복잡하고 딥링크·A/B 테스트가 많은 앱          |
| **SwiftUI 적용**       | 부자연스러움                     | `ObservableObject`/`Observable`로 자연스러움 | `NavigationStack` 경로 관리 객체로 변형해 적용       |

> 💡 VIPER·Clean Architecture·TCA(The Composable Architecture) 같은 선택지도 있지만, 공통 원리는 같다. **UI 프레임워크에 의존하지 않는 계층을 최대한 넓히고, 의존 방향을 한쪽(바깥 → 안쪽)으로 고정**하는 것이다. 팀 규모와 학습 비용을 고려해 필요 이상으로 계층을 늘리지 않는 것도 아키텍처 판단의 일부다.

<br>

### 6. 의존성 주입과 테스트 용이성

### 6-1. 주입 방식

의존성 주입은 객체가 필요한 협력자를 **직접 만들거나 싱글턴으로 찾지 않고 외부에서 받는** 기법이다. 테스트에서 실제 네트워크·DB 대신 가짜 구현을 끼워 넣을 수 있게 하는 것이 핵심 목적이다.

| **방식**                 | **형태**                                   | **장점**                               | **단점**                                    |
| ------------------------ | ------------------------------------------ | -------------------------------------- | ------------------------------------------- |
| **생성자 주입**          | `init(service: OrderService)`               | 의존성이 명시적, 불변, 누락 시 컴파일 오류 | 생성 지점의 인자가 늘어남                     |
| **프로퍼티 주입**        | `var service: OrderService!`               | 스토리보드·시스템 생성 객체에 적용 가능   | 주입 전 접근 시 크래시, 선택적 의존에만 사용   |
| **컨테이너(조립 루트)**  | `DependencyContainer`가 생성 담당           | 조립 지점이 한 곳, 수명 관리 용이        | 컨테이너를 서비스 로케이터처럼 남용하기 쉬움   |

- 싱글턴(`Service.shared`)을 코드 곳곳에서 직접 참조하면 **숨은 의존성**이 되어 테스트에서 교체할 수 없음. 싱글턴을 쓰더라도 **주입 경로를 통해 전달**하면 문제가 사라짐
- 프로토콜로 추상화하는 대상은 **외부 세계와 닿는 경계**(네트워크·저장소·시간·난수)에 집중하고, 순수 계산 로직까지 모두 프로토콜로 감싸지는 않음

<br>

### 6-2. 테스트 예시

```swift
struct StubOrderService: OrderService {
    var result: Result<[Order], Error>
    func fetchOrders() async throws -> [Order] { try result.get() }
}

@MainActor
final class OrderListViewModelTests: XCTestCase {
    func test_취소된_주문은_목록에서_제외된다() async {
        let orders = [
            Order(id: 1, productName: "A", price: 1000, status: .paid),
            Order(id: 2, productName: "B", price: 2000, status: .cancelled)
        ]
        let sut = OrderListViewModel(service: StubOrderService(result: .success(orders)))

        await sut.load()

        XCTAssertEqual(sut.rows.map(\.id), [1])
        XCTAssertNil(sut.errorMessage)
    }

    func test_네트워크_실패시_오류_메시지를_노출한다() async {
        let sut = OrderListViewModel(service: StubOrderService(result: .failure(URLError(.notConnectedToInternet))))

        await sut.load()

        XCTAssertEqual(sut.errorMessage, "주문을 불러오지 못했습니다.")
        XCTAssertTrue(sut.rows.isEmpty)
    }
}
```

- 테스트는 서버·시뮬레이터 UI 없이 수 밀리초에 끝나며, 필터 규칙·오류 처리·포맷팅을 검증함
- `URLSession` 자체를 테스트해야 할 때는 `URLProtocol`을 서브클래싱해 응답을 가로채는 방법을 사용함(unit06 참고)

<br>

### 7. 면접·실무 체크포인트

- **Massive View Controller의 원인**: 뷰 컨트롤러가 뷰 생명주기·입력·데이터·전환을 모두 받는 UIKit 구조
- **MVVM에서 ViewModel의 조건**: UIKit 비의존, 화면 상태와 액션 처리 담당, 단위 테스트 가능
- **Coordinator를 도입하는 이유**: 화면 전환 소유권 분리 → 화면 재사용, 딥링크·푸시 진입 처리 단순화
- **의존성 주입의 목적**: 협력자를 외부에서 받아 테스트에서 가짜로 교체. 생성자 주입이 기본
- **프로토콜 추상화 대상**: 네트워크·저장소·시간 등 외부 경계. 모든 것을 프로토콜로 만들지 않음
- **아키텍처 선택 기준**: 팀 규모·화면 수·흐름 복잡도·테스트 요구. 과한 계층은 그 자체로 비용
- 화면 전환의 진입점이 되는 푸시 알림 처리는 **unit09**, 링크 진입은 **unit07**을 참고할 것
