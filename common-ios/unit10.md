## 권한과 프라이버시

iOS는 카메라·위치·사진·연락처처럼 샌드박스 밖의 자원(unit07 참고)에 접근할 때 **반드시 사용자의 명시적 허락**을 요구하며, 앱 심사와 개인정보 규제는 해가 갈수록 엄격해지고 있다. 이 유닛은 권한 요청의 동작 방식과 **요청 시점을 정하는 기준**, 민감 정보를 저장할 때의 원칙, 그리고 앱 추적 투명성(ATT)·프라이버시 매니페스트 같은 **정책 대응 사항**을 정리한다.

<br>

### 1. 권한 모델의 기본 원리

- 권한이 필요한 API를 호출하면 시스템이 **한 번만** 권한 대화상자를 띄우고, 사용자의 선택을 기억함
- 한 번 거부된 권한은 앱이 다시 대화상자를 띄울 수 없고, 사용자가 **설정 앱에서 직접 바꿔야** 함 → 첫 요청의 승인율이 그 기능의 성패를 좌우함
- 각 권한에는 `Info.plist`의 **사용 목적 설명 키**(`NS...UsageDescription`)가 필수이며, 없으면 요청 시점에 앱이 **즉시 종료**됨

```
앱이 권한 필요 API 호출
        │
        ▼
Info.plist 에 UsageDescription 있음? ── 없음 ──▶ 크래시 (심사에서도 리젝)
        │ 있음
        ▼
이전에 결정된 상태? ── notDetermined ──▶ 시스템 대화상자 표시 → 허용 / 거부 저장
        │ 이미 결정됨
        ▼
authorized ──▶ 즉시 동작
denied / restricted ──▶ 대화상자 없이 실패 → 앱이 설정 이동 안내 필요
```

| **권한**            | **Info.plist 키**                                                 | **비고**                                                 |
| ------------------- | ----------------------------------------------------------------- | -------------------------------------------------------- |
| **카메라**          | `NSCameraUsageDescription`                                        |                                                          |
| **마이크**          | `NSMicrophoneUsageDescription`                                    |                                                          |
| **사진 라이브러리** | `NSPhotoLibraryUsageDescription`, `NSPhotoLibraryAddUsageDescription` | iOS 14부터 **제한된 접근(선택한 사진만)** 선택지 있음     |
| **위치**            | `NSLocationWhenInUseUsageDescription`, `NSLocationAlwaysAndWhenInUseUsageDescription` | 정확한 위치 / 대략적 위치 선택 가능(iOS 14+)  |
| **연락처·캘린더**   | `NSContactsUsageDescription`, `NSCalendarsFullAccessUsageDescription` 등 | 캘린더는 iOS 17부터 쓰기 전용·전체 접근으로 세분화        |
| **알림**            | (키 없음, `UNUserNotificationCenter`로 요청)                       | unit09 참고                                              |
| **추적(ATT)**       | `NSUserTrackingUsageDescription`                                   | iOS 14.5부터 광고 식별자(IDFA) 접근에 필수                 |

> 💡 사용 목적 설명은 심사 항목이자 **승인율을 좌우하는 카피**다. "카메라 접근이 필요합니다" 같은 동어 반복 대신 "영수증을 촬영해 자동으로 지출을 기록하기 위해 카메라를 사용합니다"처럼 **사용자가 얻는 이익**을 구체적으로 적는다.

<br>

### 2. 권한 요청 시점 — 언제 물어볼 것인가

### 2-1. 원칙

| **원칙**                          | **설명**                                                                          |
| --------------------------------- | --------------------------------------------------------------------------------- |
| **맥락 안에서 요청(Just-in-Time)** | 사용자가 해당 기능을 **직접 실행하려는 순간**에 요청. 앱 첫 실행 시 일괄 요청은 최악    |
| **요청 전 상태 확인**             | `notDetermined`일 때만 시스템 대화상자가 뜨므로, 상태별로 다른 흐름을 준비               |
| **사전 설명 화면(Pre-permission)** | 거부되면 되돌리기 어려운 권한(알림·위치 항상)은 앱 자체 화면으로 먼저 설명하고, 동의한 사용자에게만 시스템 대화상자 노출 |
| **거부 후 복구 경로**             | 거부 상태에서 기능을 시도하면 설정 앱으로 이동하는 버튼을 제공                          |
| **최소 범위 요청**                | 위치는 "사용 중"부터, 사진은 "선택한 사진만"으로 충분하면 전체 접근을 요구하지 않음        |

<br>

### 2-2. 안티패턴과 개선

```swift
// 안티패턴: 앱 시작 시 상태 확인 없이 일괄 요청
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    AVCaptureDevice.requestAccess(for: .video) { _ in }
    CLLocationManager().requestAlwaysAuthorization()
    UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound]) { _, _ in }
    return true
}
```

```swift
// 개선: 기능 진입 시점에 상태별로 분기
import AVFoundation

@MainActor
final class CameraPermission {
    enum Outcome { case granted, deniedNeedsSettings }

    func ensureAccess() async -> Outcome {
        switch AVCaptureDevice.authorizationStatus(for: .video) {
        case .authorized:
            return .granted
        case .notDetermined:
            let granted = await AVCaptureDevice.requestAccess(for: .video)   // 이때만 시스템 대화상자
            return granted ? .granted : .deniedNeedsSettings
        case .denied, .restricted:
            return .deniedNeedsSettings
        @unknown default:
            return .deniedNeedsSettings
        }
    }

    func openSettings() {
        if let url = URL(string: UIApplication.openSettingsURLString) {
            UIApplication.shared.open(url)
        }
    }
}

// 호출부: "영수증 촬영" 버튼을 눌렀을 때
Task {
    switch await cameraPermission.ensureAccess() {
    case .granted:
        presentCamera()
    case .deniedNeedsSettings:
        showAlert(title: "카메라 권한이 필요합니다",
                  message: "설정 > 개인정보 보호에서 카메라를 허용해 주세요.",
                  action: cameraPermission.openSettings)
    }
}
```

> ⚠️ `restricted`는 사용자가 거부한 것이 아니라 **보호자 제한·MDM 정책** 등으로 막힌 상태다. 설정 앱으로 보내도 사용자가 바꿀 수 없으므로, "이 기기에서는 사용할 수 없습니다"처럼 다른 안내가 필요하다.

<br>

### 3. 위치 권한의 특수성

위치는 권한 단계가 가장 세분화되어 있어 별도로 이해해야 한다.

| **단계**                     | **의미**                                       | **요청 방법**                                          |
| ---------------------------- | ---------------------------------------------- | ------------------------------------------------------ |
| **사용 중 허용(When In Use)** | 앱이 포그라운드일 때만 위치 수신                 | `requestWhenInUseAuthorization()`                       |
| **항상 허용(Always)**        | 백그라운드에서도 수신                            | 사용 중 허용을 받은 뒤 `requestAlwaysAuthorization()` → 시스템이 **나중에** 사용자에게 재확인 |
| **정확한 위치 / 대략적 위치** | iOS 14+. 사용자가 정확도를 낮출 수 있음          | 필요 시 `requestTemporaryFullAccuracyAuthorization`으로 일시 요청 |

- "항상 허용"은 요청해도 즉시 부여되지 않으며, 시스템이 일정 기간 사용 패턴을 본 뒤 **사용자에게 계속 허용할지 재확인**함. 실제로 백그라운드 위치가 필요한 기능(내비게이션·지오펜스)이 아니면 요청하지 않는 것이 좋음
- 대략적 위치를 허용한 사용자에게 정확한 위치를 강요하는 UI는 승인율만 떨어뜨림. 반경 수 km 수준의 정확도로도 동작하도록 설계하는 것이 우선임

<br>

### 4. 민감 정보 저장 원칙

권한을 받아 얻은 데이터와 인증 정보는 **저장 위치·보호 등급·수명**을 정해 두고 다뤄야 한다. 저장소별 특성은 unit05를 참고할 것.

| **데이터**                       | **저장 위치**                              | **원칙**                                                    |
| -------------------------------- | ------------------------------------------ | ----------------------------------------------------------- |
| **액세스·리프레시 토큰, 비밀번호** | **Keychain** (`WhenUnlockedThisDeviceOnly`) | `UserDefaults`·plist·로그 금지. 로그아웃 시 즉시 삭제          |
| **주민번호·카드번호 등 식별 정보** | 가능하면 **저장하지 않음**                    | 서버 토큰화(카드 → 결제 토큰)로 대체. 저장 시 Keychain + 최소 기간 |
| **위치·건강·사진 등 권한 데이터** | 처리 후 파기, 보관 시 Data Protection 파일    | 수집 목적 외 사용 금지, 서버 전송 시 최소 필드만                 |
| **캐시·임시 파일**               | Caches / tmp                               | 개인화 응답이 디스크 캐시에 남지 않도록 `no-store` 확인(unit06)   |

```swift
// 안티패턴: 토큰을 UserDefaults에, 요청·응답 전문을 로그에
UserDefaults.standard.set(token, forKey: "accessToken")
print("response: \(String(data: data, encoding: .utf8) ?? "")")   // 개인정보가 콘솔·크래시 리포트에 남음
```

```swift
// 개선: Keychain 저장 + 로그는 식별 정보 마스킹 + 화면 캡처 대비
try KeychainStore.save(Data(token.utf8), service: "auth", account: "accessToken")   // unit05의 래퍼

func maskedLog(_ user: User) {
    Logger.network.debug("user id=\(user.id, privacy: .public) email=\(user.email, privacy: .private)")
}

// 앱 전환기 스냅샷에 민감 화면이 남지 않도록 백그라운드 진입 시 가리기 (unit01 참고)
func sceneWillResignActive(_ scene: UIScene) {
    PrivacyCover.show(in: window)
}
```

- `os.Logger`의 `privacy: .private`는 릴리스 빌드의 통합 로그에서 값을 `<private>`로 치환해 줌
- 텍스트 필드에 `isSecureTextEntry`를 켜면 비밀번호 자동 완성·화면 녹화 보호가 함께 적용됨
- 탈옥 기기·디버거 감지 등 추가 방어는 요구 수준에 따라 검토하되, **Keychain과 Data Protection이 기본선**임

> 💡 "민감 정보는 애초에 기기에 두지 않는다"가 가장 강력한 원칙이다. 카드번호는 결제 대행사의 토큰으로, 사용자 식별은 서버가 발급한 불투명한 토큰으로 대체하면 유출 시 피해 범위가 크게 줄어든다. 비밀번호 저장 원리는 웹 보안 챕터 unit09를 참고할 것.

<br>

### 5. 개인정보 정책 대응 사항

### 5-1. 앱 추적 투명성(ATT)

- iOS 14.5부터 **광고 식별자(IDFA) 접근이나 타 앱·웹사이트 간 사용자 추적**에는 `ATTrackingManager.requestTrackingAuthorization`으로 허락을 받아야 함
- 추적 없이 동작하는 앱은 요청하지 않아도 되며, 요청한다면 사용 목적 설명(`NSUserTrackingUsageDescription`)이 필수
- 요청 시점은 앱이 Active 상태여야 하며(unit01), 실행 직후 다른 대화상자와 겹치면 표시되지 않을 수 있으므로 **첫 화면이 안정된 뒤** 호출함

```swift
import AppTrackingTransparency

func requestTrackingIfNeeded() async {
    guard ATTrackingManager.trackingAuthorizationStatus == .notDetermined else { return }
    let status = await ATTrackingManager.requestTrackingAuthorization()
    AnalyticsConfig.setTrackingAllowed(status == .authorized)
}
```

<br>

### 5-2. 프라이버시 매니페스트와 앱 프라이버시 정보

| **항목**                          | **내용**                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------- |
| **프라이버시 매니페스트**         | `PrivacyInfo.xcprivacy` 파일에 수집 데이터 유형, 추적 여부, **필수 사유 API** 사용 근거를 선언. Xcode 15 이후 도입되었으며, 특정 시점부터 심사에 필수화됨 (세부 요건은 Apple 공지에 따라 변동) |
| **필수 사유 API(Required Reason API)** | `UserDefaults`, 파일 타임스탬프, 시스템 부팅 시간, 디스크 공간 등 **핑거프린팅에 악용 가능한 API**. 사용 시 승인된 사유 코드를 매니페스트에 기재 |
| **서드파티 SDK**                  | 광고·분석 SDK 등 Apple이 지정한 목록의 SDK는 자체 매니페스트와 서명을 포함해야 함        |
| **앱 프라이버시 정보(영양 라벨)** | App Store Connect에서 수집 데이터 유형·목적·연결 여부를 신고. 실제 동작과 다르면 리젝·삭제 사유 |
| **계정 삭제 기능**                | 계정 생성을 지원하는 앱은 **앱 안에서 계정 삭제**를 제공해야 함                         |
| **국내 법규**                     | 개인정보 보호법·정보통신망법에 따른 수집·이용 동의, 처리방침 고지, 만 14세 미만 처리 제한 등 |

> ⚠️ 프라이버시 매니페스트 요건은 **자주 갱신되는 정책**이다. 어떤 API·SDK가 대상인지, 언제부터 필수인지는 Apple 개발자 문서의 최신 공지를 기준으로 확인해야 하며, 여기서는 개념과 구조만 이해하는 것을 목표로 한다.

<br>

### 6. 권한·프라이버시 설계 체크리스트

```
기능 기획 단계
 ├─ 이 기능에 정말 그 권한이 필요한가? (사진 한 장 → PHPicker 는 권한 불필요)
 ├─ 최소 범위로 충분한가? (위치: 사용 중 / 대략적, 사진: 선택 항목만)
 └─ 수집한 데이터를 어디에 얼마나 보관하는가?

구현 단계
 ├─ Info.plist 사용 목적 설명 (이익 중심 문구)
 ├─ 상태 확인 → notDetermined 일 때만 요청, 기능 진입 시점에
 ├─ 거부·restricted 시 대체 흐름과 설정 이동 안내
 └─ 토큰은 Keychain, 로그는 마스킹, 백그라운드 스냅샷 가리기

출시 단계
 ├─ 프라이버시 매니페스트·필수 사유 API 점검, SDK 매니페스트 포함
 ├─ App Store Connect 프라이버시 정보와 실제 수집 항목 일치 확인
 └─ 개인정보 처리방침 URL, 계정 삭제 경로, 국내 동의 절차 확인
```

- `PHPickerViewController`·`UIImagePickerController`(카메라 제외)·`UIDocumentPickerViewController`처럼 **시스템이 대신 선택해 주는 UI**는 권한 없이 사용자가 고른 항목만 앱에 전달함. 권한을 요청하기 전에 이런 대안이 있는지 먼저 확인할 것

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                             |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **권한 요청은 언제 하는가?**                            | 기능을 실행하려는 맥락에서, `notDetermined`일 때만. 첫 실행 일괄 요청은 승인율을 떨어뜨림       |
| **거부된 권한을 다시 요청할 수 있는가?**                 | 없음. 설정 앱으로 안내(`openSettingsURLString`). `restricted`는 사용자가 바꿀 수 없음          |
| **UsageDescription을 빠뜨리면?**                       | 요청 시점에 크래시, 심사 리젝                                                                |
| **토큰은 어디에 저장하는가?**                            | Keychain(`WhenUnlockedThisDeviceOnly`). `UserDefaults`·로그 금지                             |
| **ATT란?**                                            | iOS 14.5+ 앱 간 추적·IDFA 접근에 필요한 사용자 동의. 추적하지 않으면 요청 불필요                 |
| **프라이버시 매니페스트란?**                            | 수집 데이터·추적·필수 사유 API 사용 근거를 선언하는 파일. 서드파티 SDK도 포함해야 함             |
| **권한 없이 사진을 받는 방법은?**                        | `PHPickerViewController` 등 시스템 선택 UI는 권한 없이 선택 항목만 전달                        |

- 권한으로 얻은 데이터의 저장소 선택은 **unit05**, 알림 권한과 푸시 등록의 관계는 **unit09**, 샌드박스 경계는 **unit07**을 참고할 것
