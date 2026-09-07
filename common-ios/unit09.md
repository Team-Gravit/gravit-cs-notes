## 푸시 알림

푸시 알림은 앱이 실행되고 있지 않아도 사용자에게 정보를 전달하는 **거의 유일한 수단**이며, 재방문율과 직결되는 만큼 대부분의 앱이 구현한다. 이 유닛은 로컬 알림과 원격 푸시의 차이, APNs를 거치는 전달 경로, 페이로드 구조와 처리 방법, 그리고 **알림을 탭했을 때 올바른 화면으로 이동시키는 방법**을 정리한다.

<br>

### 1. 로컬 알림 vs 원격 푸시

| **항목**          | **로컬 알림 (Local Notification)**              | **원격 푸시 (Remote Push Notification)**                 |
| ----------------- | ----------------------------------------------- | -------------------------------------------------------- |
| **발신 주체**     | **앱 자신** (기기 안에서 예약)                    | **서버** → APNs → 기기                                     |
| **네트워크**      | 불필요                                          | 필요 (APNs와의 지속 연결은 OS가 관리)                       |
| **트리거**        | 시간 간격, 특정 날짜·시각, 위치 진입·이탈           | 서버가 원하는 시점                                          |
| **데이터 출처**   | 앱이 이미 알고 있는 정보                          | 서버의 최신 정보                                            |
| **사전 준비**     | 권한 요청                                       | 권한 요청 + 디바이스 토큰 등록 + 서버·APNs 인증 구성          |
| **대표 사례**     | 알람, 리마인더, 미완료 작업 재알림                  | 채팅 메시지, 주문 상태 변경, 마케팅                          |

- 두 방식 모두 **`UNUserNotificationCenter`(UserNotifications 프레임워크)**로 권한·표시·탭 처리를 통일해서 다룸
- 시스템이 알림 배너·사운드·배지를 표시하는 것은 같으며, 차이는 "누가 언제 만들었는가"뿐임

```swift
import UserNotifications

// 권한 요청: 알림 종류(배너·사운드·배지)를 지정
func requestNotificationPermission() async -> Bool {
    let center = UNUserNotificationCenter.current()
    return (try? await center.requestAuthorization(options: [.alert, .sound, .badge])) ?? false
}

// 로컬 알림 예약: 30분 뒤 1회
func scheduleReminder(taskID: String, title: String) async throws {
    let content = UNMutableNotificationContent()
    content.title = title
    content.body = "아직 완료하지 않은 작업이 있어요."
    content.sound = .default
    content.userInfo = ["route": "task", "id": taskID]      // 탭 시 이동에 사용

    let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 30 * 60, repeats: false)
    let request = UNNotificationRequest(identifier: "reminder-\(taskID)", content: content, trigger: trigger)
    try await UNUserNotificationCenter.current().add(request)
}
```

> 💡 같은 `identifier`로 다시 `add`하면 기존 예약이 **교체**된다. 작업마다 안정적인 식별자를 쓰면 중복 알림을 막고, `removePendingNotificationRequests(withIdentifiers:)`로 취소도 쉽다. 권한 요청 시점에 관한 UX 원칙은 unit10을 참고할 것.

<br>

### 2. 원격 푸시의 전달 경로

```
① 앱 실행 → registerForRemoteNotifications()
② APNs 가 기기·앱 조합에 대한 디바이스 토큰 발급
      │  didRegisterForRemoteNotificationsWithDeviceToken
      ▼
③ 앱이 토큰을 자사 서버에 전송 (사용자 ID와 매핑)
      │
      ▼
④ 서버가 보낼 일이 생기면 APNs 에 HTTP/2 요청
      (인증: .p8 키 기반 JWT 토큰 또는 인증서, 헤더: apns-topic = 번들 ID)
      │
      ▼
⑤ APNs 가 해당 기기로 전달 ── 기기가 오프라인이면 일정 기간 보관 후 최신 1건 전달
      │
      ▼
⑥ 기기: 시스템이 배너 표시 / 앱이 실행 중이면 delegate 로 전달
```

- **디바이스 토큰은 바뀔 수 있음**(재설치, 백업 복원, OS 업데이트 등). 따라서 앱 실행 시마다 등록하고, 값이 바뀌었으면 서버에 다시 보내야 함
- 서버 → APNs 인증은 **토큰 기반(.p8 키)**이 인증서 방식보다 만료 관리가 쉬워 권장됨
- APNs는 전달을 **최선 노력(best-effort)**으로 보장함. 기기가 오래 오프라인이면 유실될 수 있으므로 중요한 상태 변경은 앱 실행 시 서버 조회로 보완해야 함

```swift
final class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        UNUserNotificationCenter.current().delegate = self     // 반드시 실행 초기에 설정
        application.registerForRemoteNotifications()
        return true
    }

    func application(_ application: UIApplication,
                     didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        let token = deviceToken.map { String(format: "%02x", $0) }.joined()
        PushTokenSyncer.sendIfChanged(token)      // 서버에 사용자 ID와 함께 저장
    }

    func application(_ application: UIApplication,
                     didFailToRegisterForRemoteNotificationsWithError error: Error) {
        // 시뮬레이터, 네트워크 문제, 프로비저닝 프로파일에 Push 권한 없음 등
    }
}
```

> ⚠️ `registerForRemoteNotifications()`는 **권한 요청과 별개**다. 권한이 거부되어도 토큰은 발급되며, 이 토큰으로 **사일런트 푸시**는 받을 수 있다. 반대로 토큰 등록 없이 권한만 받으면 원격 푸시는 오지 않는다. 두 단계를 모두 수행해야 한다.

<br>

### 3. 페이로드 구조

APNs 페이로드는 JSON이며, 시스템이 해석하는 `aps` 딕셔너리와 앱이 자유롭게 쓰는 **커스텀 키**로 구성된다. 크기 한계는 일반 알림 **4KB**다.

```json
{
  "aps": {
    "alert": { "title": "주문 완료", "body": "주문 #123이 결제되었습니다." },
    "badge": 3,
    "sound": "default",
    "thread-id": "orders",
    "category": "ORDER_STATUS",
    "mutable-content": 1
  },
  "route": "order",
  "id": "123"
}
```

| **aps 키**            | **의미**                                                                 |
| --------------------- | ------------------------------------------------------------------------ |
| **alert**             | 배너에 표시할 제목·본문 (문자열 또는 딕셔너리)                               |
| **badge**             | 앱 아이콘 배지 숫자 (0이면 제거). 서버가 정확한 수를 계산해야 함               |
| **sound**             | 재생할 사운드 이름 (`default` 또는 번들 내 파일)                              |
| **content-available** | `1`이면 **사일런트 푸시**. 배너 없이 앱을 백그라운드에서 깨워 데이터 갱신        |
| **mutable-content**   | `1`이면 **Notification Service Extension**이 표시 전에 내용을 수정할 수 있음     |
| **category**          | 알림에 붙일 액션 버튼 세트 식별자 (앱에서 미리 등록)                           |
| **thread-id**         | 같은 값끼리 알림 센터에서 그룹화                                             |

- **사일런트 푸시**(`content-available: 1`)는 `Background Modes → Remote notifications`를 켜야 하며, `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)`로 전달됨. 시스템이 전달 빈도를 조절하므로 **실시간성이 보장되지 않음**
- **Notification Service Extension**은 `mutable-content: 1`인 알림을 가로채 이미지 첨부, 본문 복호화, 다국어 처리 등을 수행함. 별도 프로세스이므로 메모리 한계가 낮고(unit03), 메인 앱과 데이터를 나누려면 App Group이 필요함(unit07)

<br>

### 4. 알림 수신 처리 — 포그라운드와 백그라운드

앱 상태에 따라 알림이 전달되는 경로가 다르다(앱 상태는 unit01 참고).

| **앱 상태**            | **시스템 동작**                                     | **앱에 전달되는 콜백**                                          |
| ---------------------- | --------------------------------------------------- | --------------------------------------------------------------- |
| **포그라운드(Active)** | 기본적으로 **배너를 표시하지 않음**                  | `willPresent` → 앱이 표시 여부를 결정                             |
| **백그라운드·Suspended** | 시스템이 배너·사운드·배지 표시                       | 사용자가 탭하면 `didReceive`                                      |
| **종료(Not Running)**  | 시스템이 배너 표시                                   | 탭하면 앱 실행 후 `didReceive` (delegate가 미리 설정돼 있어야 함)   |

```swift
extension AppDelegate: UNUserNotificationCenterDelegate {
    // 포그라운드에서 알림이 도착했을 때: 어떻게 표시할지 결정
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                willPresent notification: UNNotification) async -> UNNotificationPresentationOptions {
        let userInfo = notification.request.content.userInfo
        if userInfo["route"] as? String == "chat", ChatScreenTracker.isViewing(roomID: userInfo["id"] as? String) {
            return []                          // 보고 있는 채팅방의 알림은 표시하지 않음
        }
        return [.banner, .list, .sound]        // 그 외에는 배너 + 알림 센터 목록 + 소리
    }
}
```

> 💡 포그라운드에서 배너를 무조건 띄우면 사용자가 이미 보고 있는 화면의 알림까지 뜬다. `willPresent`에서 **현재 화면과 알림의 대상이 같은지** 판단해 표시를 생략하는 것이 채팅·알림 목록 화면의 기본 처리다.

<br>

### 5. 탭 시 화면 이동

### 5-1. 처리 흐름

```
사용자가 알림 탭
      │
      ▼
userNotificationCenter(_:didReceive:) ── response.notification.request.content.userInfo
      │
      ▼
userInfo 에서 route·id 추출 ──▶ 딥링크 모델로 변환 (URL Scheme·Universal Link 와 동일한 모델)
      │
      ▼
앱 상태 확인
 ├─ 이미 실행 중 & 루트 화면 준비됨 ──▶ Coordinator/Router 가 즉시 이동
 └─ 방금 실행됨 & 화면 구성 전 ──▶ pendingDeepLink 에 보관 → 루트 화면 준비 후 처리
```

```swift
// 안티패턴: 탭 즉시 뷰 컨트롤러를 직접 생성해 push (앱이 종료 상태였다면 window·navigation이 아직 없음)
func userNotificationCenter(_ center: UNUserNotificationCenter,
                            didReceive response: UNNotificationResponse) async {
    let id = response.notification.request.content.userInfo["id"] as? String ?? ""
    let vc = OrderDetailViewController(orderID: id)
    (UIApplication.shared.windows.first?.rootViewController as? UINavigationController)?
        .pushViewController(vc, animated: true)   // nil이거나 로그인 화면일 수 있음
}
```

```swift
// 개선: 딥링크 모델로 변환해 라우터에 위임, 준비 전이면 보관
enum DeepLink {
    case order(id: String)
    case chat(roomID: String)

    init?(userInfo: [AnyHashable: Any]) {
        guard let route = userInfo["route"] as? String, let id = userInfo["id"] as? String else { return nil }
        switch route {
        case "order": self = .order(id: id)
        case "chat":  self = .chat(roomID: id)
        default:      return nil
        }
    }
}

final class DeepLinkRouter {
    static let shared = DeepLinkRouter()
    private var pending: DeepLink?
    weak var coordinator: AppCoordinator?

    func handle(_ link: DeepLink) {
        guard let coordinator, coordinator.isReady else { pending = link; return }
        coordinator.navigate(to: link)
    }

    func flushIfNeeded() {                     // 루트 화면·로그인 완료 후 호출
        if let link = pending { pending = nil; handle(link) }
    }
}

extension AppDelegate {
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                didReceive response: UNNotificationResponse) async {
        if let link = DeepLink(userInfo: response.notification.request.content.userInfo) {
            DeepLinkRouter.shared.handle(link)
        }
    }
}
```

<br>

### 5-2. 놓치기 쉬운 경우

- **앱이 종료된 상태에서 탭**: `didFinishLaunching`에서 `delegate`를 설정하면 시스템이 곧이어 `didReceive`를 호출해 줌. delegate 설정이 늦으면(예: 첫 화면의 `viewDidLoad`에서 설정) 이 호출을 놓침
- **로그인 전 진입**: 인증이 필요한 화면이면 로그인 완료 후 `flushIfNeeded()`로 이어서 이동
- **탭 후 배지 정리**: 탭한 알림에 해당하는 항목을 읽음 처리하고, 배지 수를 서버 기준으로 재계산해 `setBadgeCount` 등으로 갱신함
- **액션 버튼**: `category`에 등록한 액션을 눌렀다면 `response.actionIdentifier`로 구분해 화면 이동 없이 처리할 수도 있음(예: "읽음 처리")
- URL Scheme·Universal Link 진입(unit07)과 **같은 `DeepLink` 모델·라우터를 공유**하면 진입 경로가 늘어나도 처리 코드는 하나로 유지됨. 라우터를 Coordinator에 연결하는 구조는 unit08 참고

<br>

### 6. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                          |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **로컬 알림과 원격 푸시의 차이는?**                     | 발신 주체(앱 vs 서버)와 네트워크·토큰 필요 여부. 표시·탭 처리는 동일하게 `UNUserNotificationCenter` |
| **디바이스 토큰은 언제 서버에 보내는가?**                | 실행 시마다 등록하고 값이 바뀌면 재전송. 재설치·복원 시 바뀔 수 있음                       |
| **포그라운드에서 알림이 안 보이는 이유는?**              | 기본 동작. `willPresent`에서 표시 옵션을 돌려줘야 배너가 뜸                                |
| **사일런트 푸시란?**                                   | `content-available: 1`. 배너 없이 백그라운드 갱신. 전달 빈도는 시스템이 조절                |
| **앱이 종료된 상태에서 탭하면?**                        | 실행 후 `didReceive` 호출. delegate를 `didFinishLaunching`에서 설정해야 놓치지 않음         |
| **탭 시 화면 이동은 어떻게 설계하는가?**                 | userInfo → 딥링크 모델 → 라우터/Coordinator. 준비 전이면 보관 후 처리                       |
| **페이로드 크기 제한은?**                               | 일반 알림 4KB. 큰 데이터는 ID만 보내고 앱이 서버에서 조회                                   |
