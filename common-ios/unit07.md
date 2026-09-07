## 샌드박스와 앱 간 공유

iOS의 모든 앱은 **자기만의 격리된 저장 공간(샌드박스)** 안에서 실행되며, 다른 앱의 파일이나 메모리에 직접 접근할 수 없다. 이 유닛은 샌드박스의 구조를 살펴본 뒤, 그 벽을 **허가된 방식으로 넘는 세 가지 수단** — App Group(같은 개발자의 앱·익스텐션 간 데이터 공유), URL Scheme(앱 호출), Universal Link(웹 URL로 앱 열기) — 의 동작 원리와 선택 기준을 정리한다.

<br>

### 1. 샌드박스의 구조

```
/var/mobile/Containers/
 ├─ Bundle/Application/<UUID>/MyApp.app     ── 앱 번들 (읽기 전용, 서명 검증됨)
 └─ Data/Application/<UUID>/                ── 데이터 컨테이너 (앱마다 격리)
      ├─ Documents/                         ── 사용자 데이터, 백업 포함
      ├─ Library/
      │    ├─ Application Support/          ── 앱 내부 데이터(DB 등), 백업 포함
      │    ├─ Caches/                       ── 재생성 가능 데이터, 시스템이 삭제 가능
      │    └─ Preferences/                  ── UserDefaults plist
      └─ tmp/                               ── 임시 파일, 앱 미실행 시 삭제 가능

/private/var/mobile/Containers/Shared/AppGroup/<UUID>/   ── App Group 공유 컨테이너
```

- 앱 번들은 코드 서명으로 무결성이 보장되며 **런타임에 수정할 수 없음**. 설정 파일을 번들 안에 쓰려는 시도는 실패함
- 데이터 컨테이너의 UUID는 **재설치·업데이트 시 바뀔 수 있으므로** 절대 경로를 저장하면 안 되고, 항상 `FileManager.default.urls(for:in:)`로 매번 조회해야 함
- 각 디렉터리의 백업·정리 정책은 unit05를 참고할 것

```swift
let documents = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
let caches = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
let fileURL = documents.appendingPathComponent("notes.json")

// 파일 단위 보호 등급: 잠금 해제 상태에서만 접근 가능
try Data("...".utf8).write(to: fileURL, options: [.atomic, .completeFileProtection])
```

> 💡 샌드박스는 "앱 간 격리"뿐 아니라 **권한 모델의 기반**이다. 사진·연락처처럼 샌드박스 밖의 자원은 반드시 시스템 API와 사용자 권한을 거쳐야 하며(unit10 참고), 파일 앱의 문서는 `UIDocumentPickerViewController`처럼 시스템이 대신 열어 주는 창구를 통해서만 접근한다.

<br>

### 2. 앱 간 공유 수단 한눈에 비교

| **수단**              | **무엇을 공유·전달하는가**                   | **대상**                          | **사전 조건**                                      | **위조 가능성**                  |
| --------------------- | -------------------------------------------- | --------------------------------- | -------------------------------------------------- | -------------------------------- |
| **App Group**         | 파일·UserDefaults·Core Data 저장소 **데이터**  | **같은 팀의** 앱·익스텐션           | 동일 Team ID, `group.` 식별자 entitlement            | 없음 (서명 기반)                  |
| **Keychain 공유**     | 토큰·비밀번호 등 **민감 데이터**               | 같은 팀의 앱                       | Keychain Access Group entitlement                    | 없음                              |
| **URL Scheme**        | 앱 **실행 + 짧은 파라미터**                    | 임의의 앱                          | `Info.plist`에 스킴 등록                              | **있음** (같은 스킴을 누구나 등록)  |
| **Universal Link**    | 웹 URL로 앱 실행 + 경로·쿼리                    | 임의의 앱 (도메인 소유자만)          | AASA 파일 + Associated Domains entitlement            | 없음 (도메인 소유 검증)            |

<br>

### 3. App Group — 같은 개발자의 앱·익스텐션 간 데이터 공유

위젯·공유 익스텐션·워치 앱은 **별도 프로세스이자 별도 샌드박스**에서 동작하므로, 메인 앱의 Documents나 UserDefaults를 볼 수 없다. App Group은 이들이 함께 접근할 수 있는 **공유 컨테이너**를 제공한다.

- Xcode의 Signing & Capabilities에서 App Groups를 추가하고 `group.com.example.myapp` 형태의 식별자를 **모든 타깃에 동일하게** 지정함
- 공유 가능한 것: 공유 컨테이너 디렉터리의 파일, `UserDefaults(suiteName:)`, 공유 컨테이너에 둔 Core Data/SQLite 파일

```swift
// 안티패턴: 메인 앱과 위젯이 각자의 UserDefaults.standard를 보며 데이터가 안 맞는다고 당황
UserDefaults.standard.set(todayCount, forKey: "todayCount")   // 위젯에서는 보이지 않음
```

```swift
// 개선: App Group suite로 공유
enum SharedStore {
    static let groupID = "group.com.example.myapp"
    static let defaults = UserDefaults(suiteName: groupID)!

    static var containerURL: URL {
        FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: groupID)!
    }

    static func saveSnapshot(_ data: Data) throws {
        try data.write(to: containerURL.appendingPathComponent("snapshot.json"), options: .atomic)
    }
}

// 메인 앱
SharedStore.defaults.set(todayCount, forKey: "todayCount")
WidgetCenter.shared.reloadAllTimelines()     // 위젯에 갱신 요청

// 위젯 익스텐션
let count = SharedStore.defaults.integer(forKey: "todayCount")
```

> ⚠️ 공유 컨테이너의 파일을 **두 프로세스가 동시에 쓰면 손상**될 수 있다. `.atomic` 옵션으로 쓰거나 `NSFileCoordinator`로 접근을 조정해야 하며, Core Data 저장소를 공유할 때는 각 프로세스가 별도의 `NSPersistentContainer`를 열고 **한쪽만 쓰기**를 담당하도록 역할을 나누는 것이 안전하다. 익스텐션의 메모리 한계도 매우 낮다는 점(unit03)을 함께 기억할 것.

<br>

### 4. URL Scheme — 커스텀 스킴으로 앱 호출

### 4-1. 동작 원리

`myapp://order/123` 같은 **앱 고유 스킴**을 `Info.plist`의 `CFBundleURLTypes`에 등록하면, 다른 앱이나 Safari에서 이 URL을 열 때 시스템이 해당 앱을 실행하고 URL을 전달한다.

```
다른 앱 / Safari / 푸시 알림 / QR 코드
        │  UIApplication.shared.open(URL("myapp://order/123"))
        ▼
   시스템이 "myapp" 스킴을 등록한 앱을 찾아 실행
        │
        ├─ 앱이 실행 중이 아님 → didFinishLaunching → scene(_:willConnectTo:) 의 connectionOptions.urlContexts
        └─ 앱이 실행 중 → scene(_:openURLContexts:)
        ▼
   URL 파싱 → 라우터가 해당 화면으로 이동
```

```swift
func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
    guard let url = URLContexts.first?.url else { return }
    handle(url)
}

func handle(_ url: URL) {
    // myapp://order/123?ref=push
    guard url.scheme == "myapp",
          let host = url.host,
          let components = URLComponents(url: url, resolvingAgainstBaseURL: false)
    else { return }
    let pathID = url.pathComponents.dropFirst().first        // "123"
    let ref = components.queryItems?.first { $0.name == "ref" }?.value
    router.route(to: host, id: pathID, source: ref)
}
```

- 다른 앱의 스킴을 열 수 있는지 확인하려면 `canOpenURL(_:)`을 쓰되, 조회할 스킴을 `LSApplicationQueriesSchemes`에 미리 선언해야 함 (사용자 설치 앱 목록 추적을 막기 위한 제한)
- 앱이 종료된 상태에서 열리면 `scene(_:openURLContexts:)`가 아니라 `willConnectTo`의 `connectionOptions`로 들어오므로 **두 경로를 모두 처리**해야 함 (앱 생명주기는 unit01 참고)

<br>

### 4-2. 한계와 보안

- **스킴 충돌**: 스킴은 전역 등록이 아니어서 **누구나 같은 스킴을 선언**할 수 있고, 어떤 앱이 열릴지 보장되지 않음. 악성 앱이 은행 앱 스킴을 가로채 로그인 콜백을 탈취하는 사례가 보고된 바 있음
- **설치 여부 확인 불가**: 앱이 없으면 아무 일도 일어나지 않아 대체 흐름(스토어 이동)을 만들기 어려움
- 따라서 URL Scheme으로 **민감한 값(토큰·인증 코드)을 전달해서는 안 되며**, OAuth 콜백처럼 보안이 중요한 경우 Universal Link나 `ASWebAuthenticationSession`을 사용해야 함 (인증 흐름은 웹 보안 챕터 unit06 참고)

<br>

### 5. Universal Link — 웹 URL로 앱 열기

### 5-1. 동작 원리

Universal Link는 `https://example.com/order/123` 같은 **일반 웹 URL**을 탭했을 때, 앱이 설치되어 있으면 앱으로, 없으면 Safari로 열리게 한다. 핵심은 **도메인 소유자가 서버에 올린 AASA(apple-app-site-association) 파일**로 앱과 도메인의 관계를 증명한다는 점이다.

```
[앱 설치·업데이트 시]
  Apple CDN ── https://example.com/.well-known/apple-app-site-association 를 가져와 검증
        │       (Team ID + Bundle ID 와 허용 경로 목록)
        ▼
  기기에 "example.com/order/* 는 MyApp 이 처리" 로 등록

[사용자가 링크 탭]
  https://example.com/order/123
        │
        ├─ 등록된 경로와 일치 & 앱 설치됨 ──▶ 앱 실행 → NSUserActivity(webpageURL) 전달
        └─ 불일치 / 앱 없음 ──▶ Safari 에서 웹 페이지 표시 (자연스러운 대체 흐름)
```

**서버 측 AASA 파일 예시** (`Content-Type: application/json`, 확장자 없음, HTTPS 필수)

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["ABCDE12345.com.example.myapp"],
        "components": [
          { "/": "/order/*", "comment": "주문 상세" },
          { "/": "/promo/*", "?": { "campaign": "?*" } },
          { "/": "/help", "exclude": true }
        ]
      }
    ]
  }
}
```

**앱 측 설정과 처리**

- Signing & Capabilities에서 Associated Domains에 `applinks:example.com` 추가
- 링크로 실행되면 `scene(_:continue:)`(실행 중) 또는 `willConnectTo`의 `connectionOptions.userActivities`(종료 상태)로 `NSUserActivity`가 전달됨

```swift
func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL
    else { return }
    handle(url)       // URL Scheme과 같은 라우터로 합류
}
```

<br>

### 5-2. 실무에서 자주 겪는 문제

| **증상**                                     | **원인·확인 사항**                                                         |
| -------------------------------------------- | -------------------------------------------------------------------------- |
| 링크를 탭해도 Safari로만 열림                  | AASA 파일이 HTTPS 아님, 리다이렉트됨, JSON 형식 오류, Team ID·Bundle ID 불일치 |
| 배포 후 한참 뒤에야 동작함                      | AASA는 Apple CDN이 캐시하므로 갱신 반영에 시간이 걸릴 수 있음                   |
| 같은 도메인 페이지 안에서 탭하면 앱이 안 열림     | Safari에서 **같은 도메인 내부 이동**은 의도적으로 앱을 열지 않음                 |
| 앱에서 앱으로의 링크가 웹으로 열림               | 사용자가 상단 배너로 "Safari에서 열기"를 선택해 기기 설정이 바뀜                 |

> 💡 URL Scheme과 Universal Link는 경쟁 관계가 아니라 **보완 관계**다. 실무에서는 Universal Link를 기본으로 하되, 앱 내부·QR 코드·푸시 알림(unit09)처럼 앱 설치가 전제된 경로에서는 URL Scheme을 함께 지원하고, 두 경로가 **하나의 라우터로 합류**하도록 설계한다.

<br>

### 6. 선택 기준 정리

```
무엇을 하려는가?
 ├─ 내 앱과 내 익스텐션·워치 앱이 데이터를 함께 읽고 써야 함
 │      └─ App Group (+ 민감 정보면 Keychain Access Group)
 ├─ 외부(웹·이메일·SNS)에서 링크로 앱의 특정 화면을 열어야 함
 │      └─ Universal Link (앱 없으면 웹으로 자연 대체)
 ├─ 앱 설치가 전제된 환경에서 가볍게 앱을 호출해야 함 (QR, 파트너 앱 간 이동)
 │      └─ URL Scheme (민감 값 전달 금지)
 └─ 사용자에게 파일·텍스트를 다른 앱으로 넘기게 하고 싶음
        └─ UIActivityViewController(공유 시트) / 공유 익스텐션
```

<br>

### 7. 면접·실무 체크포인트

- **샌드박스란?**: 앱마다 격리된 데이터 컨테이너. 다른 앱 파일 접근 불가, 시스템 자원은 권한 API 경유
- **컨테이너 경로를 저장하면 안 되는 이유**: UUID가 재설치·업데이트 시 바뀔 수 있어 매번 `FileManager`로 조회
- **App Group이 필요한 대표 상황**: 위젯·공유 익스텐션·워치 앱과 데이터 공유. `UserDefaults(suiteName:)`과 공유 컨테이너 URL
- **URL Scheme의 보안 문제**: 누구나 같은 스킴 등록 가능 → 하이재킹. 민감 값 전달 금지, OAuth 콜백은 Universal Link 사용
- **Universal Link가 안전한 이유**: 도메인의 AASA 파일과 앱의 Team ID·Bundle ID를 Apple이 교차 검증
- **앱 종료 상태에서 링크로 열릴 때 처리 위치**: `scene(_:willConnectTo:options:)`의 `connectionOptions` (urlContexts / userActivities)
- 저장소 종류별 선택 기준은 **unit05**, 푸시 알림 탭 시 딥링크 처리는 **unit09**를 참고할 것
