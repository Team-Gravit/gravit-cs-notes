## 뷰 컨트롤러 생명주기

unit01(앱 생명주기)이 프로세스와 씬 단위의 상태 전이를 다뤘다면, 이 유닛은 **화면 하나(UIViewController)가 생성되어 사라질 때까지 호출되는 콜백**을 다룬다. `viewDidLoad`·`viewWillAppear`·`viewDidAppear`에 어떤 작업을 배치해야 하는지 판단하는 기준이 핵심이다.

<br>

### 1. 뷰 컨트롤러 생명주기가 중요한 이유

- UIKit은 뷰 컨트롤러가 화면에 나타나고 사라지는 각 단계를 콜백으로 알려주며, **잘못된 단계에 작업을 넣으면** 중복 실행·잘못된 레이아웃·메모리 누수가 발생함
- "`viewDidLoad`와 `viewWillAppear`의 차이"는 iOS 면접에서 가장 자주 나오는 질문 중 하나이며, 단순한 호출 순서보다 **호출 횟수와 레이아웃 확정 여부**를 기준으로 답해야 함
- SwiftUI만 사용하는 프로젝트라도 `UIViewControllerRepresentable`이나 기존 UIKit 코드와 섞이는 경우가 많아 이해가 필수임

<br>

### 2. 전체 호출 순서

```
init(coder:) / init(nibName:bundle:)
        │
        ▼
     loadView()              ── view 프로퍼티 생성 (직접 만들거나 스토리보드 로드)
        │
        ▼
    viewDidLoad()            ── 단 1회. 뷰 계층은 있으나 크기는 미확정
        │
        ▼ (화면에 나타날 때마다 반복)
  viewWillAppear(_:)
        │
        ▼
  viewIsAppearing(_:)        ── iOS 17+. 트레이트·레이아웃이 결정된 직후
        │
        ▼
 viewWillLayoutSubviews() ─┐
        │                   │ 레이아웃이 바뀔 때마다 반복 (회전, 키보드 등)
 viewDidLayoutSubviews()  ─┘
        │
        ▼
   viewDidAppear(_:)         ── 화면 전환 애니메이션 완료
        │
        ▼ (화면에서 사라질 때)
 viewWillDisappear(_:)
        │
        ▼
  viewDidDisappear(_:)
        │
        ▼
      deinit                 ── 참조가 모두 해제되었을 때만
```

- `viewDidLoad`는 뷰 컨트롤러 인스턴스당 **한 번만** 호출되지만, Appear·Disappear 계열은 **화면에 나타나고 사라질 때마다** 호출됨
- 탭 전환, 네비게이션 push/pop, 모달 dismiss 뒤 복귀 등 모든 경우에 `viewWillAppear`가 다시 호출됨

<br>

### 3. 각 단계의 특징과 배치 기준

| **콜백**                    | **호출 횟수**   | **뷰 크기 확정** | **적합한 작업**                                                       | **부적합한 작업**                          |
| --------------------------- | --------------- | ---------------- | --------------------------------------------------------------------- | ------------------------------------------ |
| **viewDidLoad**             | **1회**         | 아니오           | 서브뷰 추가, 제약 조건 설정, 델리게이트·데이터소스 연결, 최초 데이터 요청 | 프레임 값 계산, 매번 갱신해야 하는 데이터 로드 |
| **viewWillAppear**          | 매번            | 아니오           | 화면 복귀 시 데이터 갱신, 네비게이션 바 스타일, 옵저버 등록              | 애니메이션 시작, 무거운 네트워크 요청       |
| **viewIsAppearing**         | 매번            | **예**           | 크기·트레이트에 의존하는 초기 스크롤 위치, 다크 모드 대응               | (iOS 17 미만에서는 호출되지 않음)          |
| **viewDidLayoutSubviews**   | 레이아웃마다    | **예**           | `cornerRadius`·그라데이션 레이어 프레임 갱신                           | 데이터 로드, 제약 조건 추가                |
| **viewDidAppear**           | 매번            | 예               | 애니메이션 시작, 포커스·키보드 올리기, 화면 노출 로그                   | 초기 UI 구성                               |
| **viewWillDisappear**       | 매번            | 예               | 키보드 내리기, 편집 내용 임시 저장                                     | 화면 해제로 오해하고 리소스 삭제           |
| **viewDidDisappear**        | 매번            | 예               | 옵저버 해제, 타이머 정지, 재생 중단                                     | 화면이 실제로 해제됐다고 가정하는 작업     |

> 💡 `viewDidLoad`에서 `view.bounds`나 `frame`을 읽어 계산하는 코드는 대표적인 함정이다. 이 시점의 크기는 스토리보드 기본값이나 0일 수 있으며, 실제 크기는 **`viewDidLayoutSubviews` 이후에야 확정**된다. 오토 레이아웃 제약 조건은 크기와 무관하므로 `viewDidLoad`에서 설정해도 된다.

<br>

### 4. 흔한 함정과 개선

### 4-1. 옵저버 중복 등록

`viewWillAppear`에서 옵저버를 등록하고 해제하지 않으면, 화면을 오갈 때마다 옵저버가 쌓여 **같은 알림에 핸들러가 여러 번 실행**된다.

```swift
// 안티패턴: 등록만 하고 해제하지 않음
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    NotificationCenter.default.addObserver(
        self, selector: #selector(refresh),
        name: .cartDidChange, object: nil
    )
}
```

```swift
// 개선: 등록과 해제를 대칭으로 배치
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    NotificationCenter.default.addObserver(
        self, selector: #selector(refresh),
        name: .cartDidChange, object: nil
    )
}

override func viewDidDisappear(_ animated: Bool) {
    super.viewDidDisappear(animated)
    NotificationCenter.default.removeObserver(self, name: .cartDidChange, object: nil)
}
```

- 등록·해제는 **짝이 맞는 콜백**에 둔다: `viewWillAppear` ↔ `viewDidDisappear`, `viewDidLoad` ↔ `deinit`
- 클로저 기반 `addObserver(forName:)`를 쓰면 반환된 토큰을 보관했다가 해제해야 하며, 토큰을 잃어버리면 옵저버가 영원히 남음

<br>

### 4-2. 매번 갱신해야 할 데이터를 viewDidLoad에 배치

상세 화면에서 돌아왔을 때 목록이 갱신되지 않는 문제는 대부분 로드 코드가 `viewDidLoad`에만 있기 때문이다. 반대로 `viewWillAppear`에 무거운 요청을 넣으면 탭을 오갈 때마다 요청이 반복된다.

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    configureLayout()            // 1회성: 서브뷰·제약 조건
    viewModel.loadInitial()      // 최초 1회 로드
}

override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    viewModel.refreshIfStale()   // 마지막 로드 후 일정 시간 지났을 때만 갱신
}
```

<br>

### 4-3. 크기에 의존하는 레이어 설정

```swift
override func viewDidLayoutSubviews() {
    super.viewDidLayoutSubviews()
    // 그라데이션 레이어는 오토 레이아웃 대상이 아니므로 직접 프레임을 맞춘다
    gradientLayer.frame = headerView.bounds
}
```

> ⚠️ `viewDidLayoutSubviews`는 회전·키보드·스크롤 등으로 **여러 번 호출**된다. 여기서 서브뷰를 추가하거나 네트워크를 호출하면 중복 실행되므로, 프레임 갱신처럼 **멱등한(여러 번 실행해도 결과가 같은) 작업만** 배치한다.

<br>

### 5. 컨테이너 뷰 컨트롤러와 자식 생명주기

`UINavigationController`·`UITabBarController`처럼 다른 뷰 컨트롤러를 담는 컨테이너는 자식의 Appear 콜백을 **대신 전달**한다. 직접 컨테이너를 만들 때는 아래 순서를 지켜야 자식이 올바른 콜백을 받는다.

```swift
func embed(_ child: UIViewController, in container: UIView) {
    addChild(child)                       // 1. 부모-자식 관계 등록
    container.addSubview(child.view)      // 2. 뷰 계층에 추가
    child.view.frame = container.bounds
    child.didMove(toParent: self)         // 3. 이동 완료 통보
}

func remove(_ child: UIViewController) {
    child.willMove(toParent: nil)         // 1. 제거 예고
    child.view.removeFromSuperview()      // 2. 뷰 계층에서 제거
    child.removeFromParent()              // 3. 관계 해제
}
```

- `addChild` 없이 `view`만 추가하면 자식은 `viewWillAppear` 등을 받지 못하고 회전·트레이트 변화도 전달되지 않음
- `UIPageViewController`처럼 자식이 자주 교체되는 구조에서는 이 규칙을 어기면 메모리 누수와 콜백 누락이 동시에 발생함

<br>

### 6. deinit이 호출되지 않는 경우 — 순환 참조

뷰 컨트롤러가 pop된 뒤에도 `deinit`이 찍히지 않으면 **강한 참조 순환**을 의심해야 한다. 가장 흔한 원인은 클로저가 `self`를 강하게 붙잡는 경우다.

```swift
// 안티패턴: viewModel → onUpdate 클로저 → self → viewModel 순환
viewModel.onUpdate = {
    self.tableView.reloadData()
}

// 개선: 약한 참조로 순환 차단
viewModel.onUpdate = { [weak self] in
    self?.tableView.reloadData()
}
```

- `Timer.scheduledTimer(target:selector:)`는 타겟을 강하게 보유하므로 `viewDidDisappear`에서 `invalidate()`하지 않으면 뷰 컨트롤러가 해제되지 않음
- 델리게이트 프로퍼티는 `weak var delegate: SomeDelegate?`로 선언해 부모가 자식을, 자식이 부모를 동시에 강하게 잡지 않도록 함
- Xcode의 Memory Graph Debugger로 해제되지 않은 인스턴스와 참조 경로를 시각적으로 확인할 수 있음

> 💡 `deinit`에 `print`를 넣어 두는 습관은 순환 참조를 조기에 발견하는 가장 값싼 방법이다. 화면을 닫았는데 로그가 찍히지 않으면 어딘가에서 `self`를 붙잡고 있는 것이다.

<br>

### 7. 면접·실무 체크포인트

| **질문**                                         | **핵심 답변**                                                                     |
| ------------------------------------------------ | --------------------------------------------------------------------------------- |
| **viewDidLoad vs viewWillAppear 차이는?**         | 호출 횟수(1회 vs 매번)와 용도(초기 구성 vs 복귀 시 갱신)                             |
| **viewDidLoad에서 frame을 읽으면 안 되는 이유는?** | 레이아웃이 확정되기 전이라 값이 부정확함. `viewDidLayoutSubviews` 이후에 읽어야 함     |
| **애니메이션은 어디서 시작하는가?**               | `viewDidAppear`. 그 전에는 화면 전환 애니메이션과 겹침                              |
| **옵저버 등록·해제 위치는?**                     | `viewWillAppear` ↔ `viewDidDisappear` 또는 `viewDidLoad` ↔ `deinit`으로 대칭 배치     |
| **viewIsAppearing은 무엇인가?**                  | iOS 17에서 추가된 콜백으로, 트레이트와 레이아웃이 결정된 뒤 `viewDidAppear` 전에 호출됨 |
| **deinit이 안 불리면?**                          | 클로저·타이머·델리게이트의 강한 참조 순환을 의심하고 Memory Graph로 확인             |

- 앱 전체가 백그라운드로 내려갈 때의 처리는 **unit01**, 메모리 경고 시 뷰 컨트롤러가 해야 할 일은 **unit03**, 셀 단위 재사용 생명주기는 **unit04**를 참고할 것
