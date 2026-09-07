## 앱 생명주기와 상태 전이

iOS 앱은 사용자가 홈으로 나가거나 전화를 받는 순간 시스템에 의해 **실행 상태가 강제로 바뀌며**, 개발자는 그 전이 시점에 맞춰 저장·정리·재개 작업을 배치해야 한다. 이 유닛에서는 앱의 5개 상태와 전이 흐름, 그리고 iOS 13 이후 **AppDelegate와 SceneDelegate로 역할이 나뉜 이유**를 정리한다.

<br>

### 1. 왜 앱 생명주기를 알아야 하는가

- iOS는 포그라운드 앱 하나에 자원을 집중하므로, 백그라운드로 내려간 앱은 **언제든 정지(Suspended)되거나 종료**될 수 있음
- 사용자 입력 중인 데이터, 진행 중인 네트워크 요청, 타이머 등을 **어느 시점에 저장하고 정리할지**는 전적으로 생명주기 콜백에 달려 있음
- "홈 버튼을 누르면 어떤 콜백이 순서대로 호출되는가"는 iOS 면접의 단골 질문이며, 콜백 이름만이 아니라 **각 시점에 어떤 작업을 넣어야 하는지**까지 답할 수 있어야 함

> 💡 iOS 앱은 안드로이드와 달리 "백그라운드에서 계속 동작하는 앱"이 예외적이다. 기본값은 **백그라운드 진입 후 짧은 유예 시간 뒤 정지**이며, 오디오 재생·위치 추적 등 명시적으로 백그라운드 모드를 선언한 앱만 계속 실행된다.

<br>

### 2. 앱의 5개 상태

| **상태**          | **의미**                                                   | **코드 실행** | **화면 표시** |
| ----------------- | ---------------------------------------------------------- | ------------- | ------------- |
| **Not Running**   | 아직 실행되지 않았거나 시스템·사용자에 의해 종료됨         | 없음          | 없음          |
| **Inactive**      | 포그라운드에 있지만 이벤트를 받지 않음 (전화 수신, 앱 전환 중, 제어 센터 노출 등) | **실행 중**   | 표시됨        |
| **Active**        | 포그라운드에서 이벤트를 정상적으로 받는 상태               | **실행 중**   | 표시됨        |
| **Background**    | 화면에서 사라졌지만 잠시 코드가 실행되는 상태              | **제한적 실행** | 없음        |
| **Suspended**     | 메모리에는 남아 있지만 코드가 실행되지 않는 상태           | **없음**      | 없음          |

- Inactive는 대개 **Active와 Background 사이를 지나가는 짧은 순간**이지만, 전화 수신·알림 센터 노출처럼 사용자가 머무는 경우도 있음
- Suspended 상태의 앱은 시스템이 메모리가 부족하면 **아무 통보 없이 종료**함 (unit03 참고)
- `UIApplication.State` 열거형에는 `active`·`inactive`·`background` 세 값만 있음. Not Running과 Suspended는 **앱 코드가 실행되지 않는 상태**이므로 앱이 스스로 관찰할 수 없기 때문임

<br>

### 3. 상태 전이 흐름

```
[Not Running]
      │ 실행 (아이콘 탭, 푸시, URL)
      ▼
 [Inactive] ──── 첫 화면 준비 완료 ────▶ [Active]
      ▲                                    │
      │ 포그라운드 복귀                     │ 홈 이동, 전화 수신, 앱 전환
      │                                    ▼
 [Background] ◀──────────────────────── [Inactive]
      │ 유예 시간 종료(약 수 초~수십 초)
      ▼
 [Suspended] ── 메모리 부족 시 조용히 종료 ──▶ [Not Running]
```

**앱 실행 → 활성화**

1. 시스템이 프로세스를 만들고 `application(_:didFinishLaunchingWithOptions:)`를 호출함
2. 씬(Scene)이 연결되며 `scene(_:willConnectTo:options:)`에서 윈도우와 루트 뷰 컨트롤러를 구성함
3. `sceneWillEnterForeground` → `sceneDidBecomeActive` 순으로 호출되어 Active가 됨

**홈 이동 → 백그라운드**

1. `sceneWillResignActive`가 호출되어 Inactive가 됨 (타이머 일시 정지, 게임 멈춤 등)
2. `sceneDidEnterBackground`가 호출되어 Background가 됨 (**데이터 저장, 민감 화면 가리기**)
3. 유예 시간이 지나면 Suspended로 넘어가며, 이 시점에는 어떤 콜백도 오지 않음

> ⚠️ `sceneDidEnterBackground`(또는 `applicationDidEnterBackground`)는 **앱이 살아 있는 동안 마지막으로 확실히 호출되는 콜백**이다. `applicationWillTerminate`는 Suspended 상태에서 시스템이 종료할 때는 호출되지 않으므로, 저장 로직을 여기에만 두면 데이터가 유실된다.

<br>

### 4. AppDelegate와 SceneDelegate의 역할 분리

### 4-1. 분리된 배경

iOS 12까지는 AppDelegate 하나가 **프로세스 생명주기와 UI 생명주기를 모두** 담당했다. iOS 13에서 iPadOS 멀티 윈도우가 도입되면서 **한 프로세스가 여러 개의 UI 인스턴스(씬)를 가질 수 있게 되었고**, 그에 맞춰 UI 생명주기가 SceneDelegate로 분리되었다.

```
iOS 12 이하                       iOS 13 이상
┌─────────────────┐               ┌─────────────────┐
│   AppDelegate   │               │   AppDelegate   │ ← 프로세스 단위 (1개)
│  프로세스 + UI   │               └────────┬────────┘
└─────────────────┘                        │ 씬 세션 관리
                                  ┌────────┴────────┐
                                  │  SceneDelegate  │ ← UI 단위 (N개 가능)
                                  │  SceneDelegate  │
                                  └─────────────────┘
```

<br>

### 4-2. 담당 범위 비교

| **항목**          | **AppDelegate**                                          | **SceneDelegate**                                           |
| ----------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| **단위**          | **프로세스** (앱당 1개)                                   | **UI 인스턴스(씬)** (여러 개 가능)                          |
| **시작 콜백**     | `didFinishLaunchingWithOptions`                          | `scene(_:willConnectTo:options:)`                           |
| **상태 전이**     | (iOS 13 이상에서는 씬으로 위임)                            | `sceneDidBecomeActive`, `sceneWillResignActive`, `sceneDidEnterBackground`, `sceneWillEnterForeground` |
| **종료·정리**     | `applicationWillTerminate`, `didDiscardSceneSessions`    | `sceneDidDisconnect`                                        |
| **적합한 작업**   | SDK 초기화, 푸시 토큰 등록(unit09), 전역 의존성 구성      | 윈도우 생성, 딥링크 처리(unit07), 화면 상태 저장·복원        |

- 씬을 사용하는 앱에서는 AppDelegate의 `applicationDidBecomeActive` 같은 UI 관련 콜백이 **호출되지 않고** SceneDelegate의 대응 메서드가 대신 호출됨
- `sceneDidDisconnect`는 사용자가 앱 전환기에서 씬을 닫거나 시스템이 자원을 회수할 때 호출되며, 프로세스 종료를 뜻하지는 않음

> 💡 SwiftUI 앱은 `@main struct`의 `App` 프로토콜이 이 역할을 대신하며, `@Environment(\.scenePhase)`로 `active`·`inactive`·`background` 전이를 관찰한다. 필요하면 `@UIApplicationDelegateAdaptor`로 AppDelegate를 붙여 푸시 등록 같은 프로세스 단위 작업을 처리한다.

<br>

### 5. 코드로 보는 상태 전이 처리

```swift
import UIKit

final class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession,
               options connectionOptions: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }
        let window = UIWindow(windowScene: windowScene)
        window.rootViewController = UINavigationController(rootViewController: HomeViewController())
        window.makeKeyAndVisible()
        self.window = window
    }

    func sceneWillResignActive(_ scene: UIScene) {
        // Inactive 진입: 타이머·게임 루프 일시 정지
        GameClock.shared.pause()
    }

    func sceneDidEnterBackground(_ scene: UIScene) {
        // Background 진입: 사용자 데이터 저장, 민감 정보 화면 가리기
        DraftStore.shared.saveAll()
        PrivacyCover.show(in: window)
    }

    func sceneWillEnterForeground(_ scene: UIScene) {
        PrivacyCover.hide()
        SessionRefresher.refreshIfNeeded()   // 토큰 만료 확인 등
    }

    func sceneDidBecomeActive(_ scene: UIScene) {
        GameClock.shared.resume()
    }
}
```

씬 전이는 `NotificationCenter`로도 관찰할 수 있어 뷰 컨트롤러나 뷰모델이 델리게이트 없이 대응할 수 있다.

```swift
NotificationCenter.default.addObserver(
    forName: UIScene.didEnterBackgroundNotification,
    object: nil, queue: .main
) { _ in
    // 씬이 백그라운드로 내려갈 때 실행할 정리 작업
}
```

<br>

### 6. 백그라운드 유예 시간 다루기

Background 상태에서 곧바로 정지되면 업로드 같은 작업이 끊길 수 있다. 이때 **백그라운드 작업 요청**으로 시간을 조금 더 확보할 수 있다.

```swift
var taskID: UIBackgroundTaskIdentifier = .invalid

func sceneDidEnterBackground(_ scene: UIScene) {
    taskID = UIApplication.shared.beginBackgroundTask(withName: "flushUpload") {
        // 만료 핸들러: 시간이 다 되면 반드시 종료를 알려야 함
        UIApplication.shared.endBackgroundTask(self.taskID)
        self.taskID = .invalid
    }
    Uploader.shared.flush {
        UIApplication.shared.endBackgroundTask(self.taskID)
        self.taskID = .invalid
    }
}
```

- 허용되는 시간은 공식 문서에 고정값으로 명시되어 있지 않으며, iOS 13 이후 **대략 30초 안팎**으로 알려져 있음. 버전과 시스템 상황에 따라 달라질 수 있으므로 `backgroundTimeRemaining`으로 확인하는 것이 안전함
- 몇 분 이상 걸리는 다운로드는 `URLSession`의 백그라운드 세션(unit06)을, 주기적 갱신은 `BackgroundTasks` 프레임워크를 사용해야 함

> ⚠️ `beginBackgroundTask`를 호출하고 `endBackgroundTask`를 빠뜨리면 시스템이 앱을 **강제 종료**할 수 있다. 만료 핸들러에서도 반드시 종료를 호출해야 한다.

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                 |
| ----------------------------------------------------- | ----------------------------------------------------------------------------- |
| **앱의 5개 상태는?**                                  | Not Running · Inactive · Active · Background · Suspended                       |
| **홈 버튼을 누르면 호출되는 콜백 순서는?**             | `sceneWillResignActive` → `sceneDidEnterBackground` → (유예 후) Suspended      |
| **데이터 저장은 어디에?**                             | `sceneDidEnterBackground`. `applicationWillTerminate`는 호출되지 않을 수 있음   |
| **AppDelegate와 SceneDelegate를 나눈 이유는?**         | 한 프로세스가 여러 UI 인스턴스를 가질 수 있어 **프로세스 단위와 UI 단위를 분리**  |
| **Inactive는 언제 발생하는가?**                       | 전화 수신·앱 전환·제어 센터 노출 등 포그라운드지만 이벤트를 받지 못할 때         |
| **Suspended에서 종료되면 앱은 알 수 있는가?**           | 알 수 없음. 다음 실행은 `didFinishLaunching`부터 다시 시작됨                    |

- 씬 전이에 맞춘 뷰 컨트롤러 단위의 작업 배치는 **unit02(뷰 컨트롤러 생명주기)**, 백그라운드 종료와 메모리 경고는 **unit03**을 참고할 것
