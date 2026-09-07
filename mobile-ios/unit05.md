## UIKit과 SwiftUI 상호운용

현업 iOS 앱 대부분은 UIKit 코드베이스 위에 SwiftUI를 점진적으로 도입하거나, SwiftUI 앱 안에서 SwiftUI가 아직 제공하지 않는 UIKit 컴포넌트(`WKWebView`, `MKMapView`, 고급 텍스트 편집 등)를 써야 한다. **`UIViewRepresentable`(UIKit → SwiftUI 안으로)**과 **`UIHostingController`(SwiftUI → UIKit 안으로)**가 두 세계를 잇는 다리이며, 갱신 주기와 소유권이 다른 두 프레임워크를 섞을 때 생기는 함정을 알아야 한다.

<br>

### 1. 두 방향의 다리

```
SwiftUI 뷰 트리 안에 UIKit을 넣기            UIKit 계층 안에 SwiftUI를 넣기
┌────────────────────────────┐             ┌──────────────────────────────┐
│ SwiftUI VStack             │             │ UINavigationController       │
│  ├ Text                    │             │  └ UIHostingController       │
│  └ UIViewRepresentable ────┼─▶ UIView    │      └ rootView: SwiftUI View │
│       (Coordinator)        │             │  └ UITableViewCell           │
└────────────────────────────┘             │      └ UIHostingConfiguration │
                                           └──────────────────────────────┘
```

| **방향**            | **도구**                                                   | **대표 용도**                               |
| ------------------- | ---------------------------------------------------------- | ------------------------------------------- |
| **UIKit → SwiftUI** | `UIViewRepresentable`, `UIViewControllerRepresentable`     | 웹뷰·지도·카메라·서드파티 UIKit 컴포넌트    |
| **SwiftUI → UIKit** | `UIHostingController`, `UIHostingConfiguration` (iOS 16+)  | 기존 UIKit 앱에 SwiftUI 화면·셀 도입        |

<br>

### 2. UIViewRepresentable — UIKit 뷰를 SwiftUI에 넣기

### 2-1. 생명주기 메서드

| **메서드**                                | **호출 시점**                                | **할 일**                                          |
| ----------------------------------------- | -------------------------------------------- | -------------------------------------------------- |
| **`makeCoordinator()`**                   | 가장 먼저 **한 번**                          | 델리게이트·타깃 액션을 받을 브리지 객체 생성       |
| **`makeUIView(context:)`**                | 뷰 정체성이 생길 때 **한 번**                | UIView 생성, 델리게이트 연결, 초기 설정            |
| **`updateUIView(_:context:)`**            | 생성 직후 + SwiftUI 상태가 바뀔 때마다 **반복** | SwiftUI 상태를 UIView에 반영 (멱등하게)         |
| **`sizeThatFits(_:uiView:context:)`**     | 레이아웃 협상 시 (iOS 16+)                   | 제안 크기에 대한 원하는 크기 반환                  |
| **`dismantleUIView(_:coordinator:)`**     | 제거될 때 (static)                           | 옵저버 해제, 리소스 정리                           |

```swift
struct SearchTextField: UIViewRepresentable {
    @Binding var text: String
    var placeholder: String

    func makeCoordinator() -> Coordinator { Coordinator(text: $text) }

    func makeUIView(context: Context) -> UITextField {
        let field = UITextField()
        field.placeholder = placeholder
        field.delegate = context.coordinator
        field.addTarget(context.coordinator,
                        action: #selector(Coordinator.editingChanged(_:)),
                        for: .editingChanged)
        return field
    }

    func updateUIView(_ uiView: UITextField, context: Context) {
        if uiView.text != text { uiView.text = text }   // 같은 값이면 대입하지 않음 (커서 튐 방지)
        uiView.placeholder = placeholder
    }

    final class Coordinator: NSObject, UITextFieldDelegate {
        var text: Binding<String>
        init(text: Binding<String>) { self.text = text }

        @objc func editingChanged(_ sender: UITextField) {
            text.wrappedValue = sender.text ?? ""        // UIKit → SwiftUI 방향 전달
        }
    }
}
```

<br>

### 2-2. Coordinator의 역할

- UIKit의 **델리게이트·데이터소스·타깃 액션**은 참조 타입(`NSObject`)이 필요하다. 구조체인 Representable은 이 역할을 할 수 없으므로 `Coordinator` 클래스가 대신 받는다
- `Coordinator`는 Representable의 **정체성 수명 동안 하나만** 유지되며, `context.coordinator`로 접근한다
- 데이터 흐름은 두 갈래다: **SwiftUI → UIKit**은 `updateUIView`, **UIKit → SwiftUI**는 `Coordinator`가 `Binding`이나 클로저를 통해 전달

> ⚠️ `updateUIView`는 부모 뷰가 재평가될 때마다 호출된다(unit01 참고). 여기서 **매번 무조건 값을 대입**하면 `UITextField`의 커서가 튀거나 `WKWebView`가 같은 페이지를 다시 로드하는 문제가 생긴다. 항상 "달라졌을 때만" 반영하는 **멱등한 갱신**으로 작성한다.

<br>

### 2-3. UIViewControllerRepresentable

뷰 컨트롤러 단위(`UIImagePickerController`, `SFSafariViewController`, 커스텀 VC)를 넣을 때 사용하며, 메서드 이름만 `makeUIViewController`·`updateUIViewController`로 바뀌고 구조는 동일하다. VC 생명주기 자체는 common-ios 챕터에서 다루므로 여기서는 생략한다.

<br>

### 3. UIHostingController — SwiftUI를 UIKit에 넣기

`UIHostingController<Content: View>`는 SwiftUI 뷰를 루트로 가지는 `UIViewController`다. UIKit 내비게이션에 push하거나, 자식 VC로 임베드하거나, `view`를 꺼내 서브뷰로 붙일 수 있다.

```swift
// 안티패턴: 상태를 바꾸기 위해 매번 새 HostingController를 만들어 교체
func showProfile(user: User) {
    let vc = UIHostingController(rootView: ProfileView(user: user))
    navigationController?.pushViewController(vc, animated: true)   // 갱신 때마다 push되어 화면이 쌓임
}
```

```swift
// 개선: 하나의 HostingController를 두고 rootView 또는 관찰 객체를 갱신
final class ProfileContainerVC: UIViewController {
    private let model = ProfileModel()                   // @Observable 또는 ObservableObject
    private lazy var host = UIHostingController(rootView: ProfileView(model: model))

    override func viewDidLoad() {
        super.viewDidLoad()
        addChild(host)                                   // 자식 VC 계약 준수
        view.addSubview(host.view)
        host.view.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            host.view.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            host.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            host.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            host.view.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])
        host.didMove(toParent: self)
    }

    func update(user: User) { model.user = user }        // SwiftUI가 알아서 재평가
}
```

- **크기**: 기본적으로 호스팅 뷰는 주어진 프레임을 채운다. 콘텐츠 크기에 맞추려면 iOS 16+ `sizingOptions = [.intrinsicContentSize]`를 설정하거나 `sizeThatFits(in:)`을 사용한다
- **세이프 에어리어**: 호스팅 뷰가 세이프 에어리어를 이중으로 적용해 위아래 여백이 생기는 경우가 있다. iOS 16.4+ `safeAreaRegions`로 조정할 수 있다(버전에 따라 동작이 다를 수 있음)
- **셀 안의 SwiftUI**: `UITableViewCell`·`UICollectionViewCell`에는 별도 호스팅 컨트롤러 대신 iOS 16+ `UIHostingConfiguration`을 쓰는 것이 재사용·크기 계산 면에서 안전하다 (unit07 참고)

```swift
cell.contentConfiguration = UIHostingConfiguration {
    OrderRow(order: order)                               // 셀 재사용 시 configuration만 교체
}
```

<br>

### 4. 혼용 시 주의점

### 4-1. 갱신 주기와 소유권이 다르다

| **항목**         | **UIKit**                                  | **SwiftUI**                                   |
| ---------------- | ------------------------------------------ | --------------------------------------------- |
| **뷰 수명**      | 개발자가 생성·보관·해제                    | 프레임워크가 정체성 기준으로 관리             |
| **갱신 트리거**  | 명령형 호출 (`label.text = ...`)           | 상태 변경 → body 재평가                       |
| **상태 저장소**  | 객체 프로퍼티                              | `@State` 등 프레임워크 저장소 (unit02 참고)   |
| **경계 통과**    | `Representable`은 값 복사, `Coordinator`가 참조 유지 | `rootView` 교체 또는 관찰 객체 변경 |

- `Representable` 구조체 자체에 저장한 값은 **재생성될 때 사라진다**. 유지해야 할 것은 `Coordinator`나 `@State`에 둔다
- UIKit 객체가 SwiftUI 뷰를 **강한 참조**로 오래 붙잡으면(클로저 캡처 등) 메모리 누수·오래된 상태 참조가 생긴다. 클로저에서는 `[weak self]`, 값 전달은 최신 상태를 읽는 방향으로 설계한다

<br>

### 4-2. 뷰 갱신 중 상태 변경 금지

- `updateUIView` 안에서 `Binding`에 값을 쓰면 "Modifying state during view update" 경고와 함께 정의되지 않은 동작이 된다. UIKit → SwiftUI 방향 갱신은 반드시 **델리게이트/액션 콜백(Coordinator)**에서 수행한다
- 부득이하게 갱신 중 상태를 바꿔야 한다면 `DispatchQueue.main.async` 또는 `Task { @MainActor in ... }`로 다음 런루프로 미룬다

> 💡 환경값(`@Environment`)은 `Representable`에는 `context.environment`로 전달되지만, `UIHostingController`로 **UIKit을 한 번 거치면 SwiftUI 환경 체인이 끊긴다**. 호스팅 컨트롤러의 `rootView`에 `.environment(...)`·`.environmentObject(...)`를 다시 붙여 주입해야 한다.

<br>

### 4-3. 내비게이션과 프레젠테이션

- SwiftUI 안의 `Representable`에서 UIKit 방식으로 `present`하려면 `UIViewController`가 필요하다. `UIViewControllerRepresentable`로 감싸거나 `.sheet`·`.fullScreenCover` 같은 SwiftUI 프레젠테이션을 우선 사용한다
- UIKit 내비게이션 스택 위에 `UIHostingController`를 push한 경우, SwiftUI 안에서 `NavigationStack`을 또 만들면 **내비게이션 바가 이중**으로 생긴다. 흐름 제어를 어느 쪽이 소유할지 먼저 정한다 (unit08 참고)

> ⚠️ `UIHostingController`를 컨테이너에 붙일 때 `addChild` → `addSubview` → `didMove(toParent:)` 계약을 지키지 않으면 화면 회전·세이프 에어리어·`viewWillAppear` 전달이 깨진다. `view`만 떼어 붙이는 것은 임시 프로토타입에서만 허용한다.

<br>

### 5. 선택 기준

| **상황**                                        | **권장 접근**                                                   |
| ----------------------------------------------- | --------------------------------------------------------------- |
| 기존 UIKit 앱에 새 화면 추가                    | 화면 단위로 `UIHostingController`, 흐름은 기존 Coordinator 유지 |
| SwiftUI 앱에서 웹뷰·지도·카메라 필요            | `UIViewRepresentable` + `Coordinator`, 갱신은 멱등하게          |
| 테이블/컬렉션 셀 내부만 SwiftUI로               | `UIHostingConfiguration` (iOS 16+)                              |
| 매우 큰 목록·복잡한 제스처·정밀 스크롤 제어     | UIKit 컴포넌트 유지, SwiftUI는 셀 내용만 담당                   |
| 양쪽에서 같은 모델 공유                         | `@Observable`/`ObservableObject` 객체를 **한 곳이 소유**하고 참조 전달 |

<br>

### 6. 정리

- **`UIViewRepresentable`**은 `make`(한 번)·`update`(반복)·`Coordinator`(참조 브리지) 세 축으로 UIKit 뷰를 SwiftUI에 넣는다
- **`UIHostingController`**는 SwiftUI 뷰를 UIViewController로 감싸며, 갱신은 컨트롤러 교체가 아니라 **rootView 또는 관찰 객체 변경**으로 한다
- 혼용 시 핵심은 **소유권을 한쪽에 두기**, **갱신을 멱등하게**, **갱신 중 상태 변경 금지**, **환경값 재주입**이다
- 셀 재사용과의 조합은 unit07, 내비게이션 소유권 문제는 unit08에서 이어진다
