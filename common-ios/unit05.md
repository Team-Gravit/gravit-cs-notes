## 데이터 영속화 선택

iOS 앱은 종료되어도 남아야 하는 데이터를 어디에 어떻게 저장할지 선택해야 한다. 이 유닛은 `UserDefaults`·`Keychain`·파일·`Core Data`·`SwiftData`의 **동작 원리와 보안 특성**을 비교하고, "이 데이터는 어디에 저장해야 하는가"를 판단하는 기준을 정리한다.

<br>

### 1. 영속화 선택이 중요한 이유

- 저장소마다 **크기 한계·보안 수준·쿼리 능력·동기화 지원**이 다르며, 잘못 고르면 성능 문제나 보안 취약점으로 직결됨
- 대표적인 실수: 인증 토큰을 `UserDefaults`에 저장(평문 노출), 수천 건의 목록을 `UserDefaults`에 통째로 저장(매번 전체 직렬화), 캐시를 Documents 폴더에 저장(iCloud 백업 용량 낭비)
- "토큰은 어디에 저장하나요?"와 "Core Data와 SwiftData의 차이는?"은 iOS 면접에서 거의 매번 등장함

<br>

### 2. 저장소 한눈에 비교

| **저장소**       | **형태**                       | **적합한 데이터**                              | **보안**                       | **쿼리** | **비고**                                   |
| ---------------- | ------------------------------ | ---------------------------------------------- | ------------------------------ | -------- | ------------------------------------------ |
| **UserDefaults** | 키-값 plist                    | 설정값, 플래그, 마지막 탭 등 **작고 단순한 값**   | **없음** (평문)                 | 없음     | 앱 실행 시 통째로 메모리에 로드됨            |
| **Keychain**     | 시스템 암호화 저장소           | 토큰, 비밀번호, 인증서 등 **민감 정보**          | **하드웨어 기반 암호화**         | 없음     | 앱 삭제 후에도 남을 수 있음                  |
| **파일**         | Documents·Caches·tmp 디렉터리  | 이미지, 문서, JSON 스냅샷 등 **덩어리 데이터**    | Data Protection 클래스로 제어    | 없음     | 디렉터리별 백업·정리 정책이 다름              |
| **Core Data**    | 객체 그래프 (기본 SQLite 저장)  | **관계가 있는 구조화 데이터**, 대량 목록          | 파일 보호 + 선택적 암호화        | **있음** | 성숙하고 강력하지만 학습 비용 높음            |
| **SwiftData**    | Core Data 위의 Swift 전용 API   | Core Data와 동일, **SwiftUI 중심 새 프로젝트**    | Core Data와 동일                | **있음** | iOS 17 이상, 매크로 기반 선언                |

> 💡 선택 기준을 한 줄로 요약하면 **"민감하면 Keychain, 작고 단순하면 UserDefaults, 크면 파일, 구조와 관계가 있으면 Core Data/SwiftData"**다. 외부 라이브러리(Realm, SQLite 래퍼, GRDB 등)는 이 기준 위에서 팀의 요구에 따라 추가로 검토한다.

<br>

### 3. UserDefaults

- 내부적으로 `Library/Preferences/<번들ID>.plist` 파일에 저장되며, 앱 실행 시 **전체 내용이 메모리에 캐시**되므로 읽기가 빠름
- 저장할 수 있는 타입은 plist 호환 타입(`String`·`Int`·`Bool`·`Data`·`Date`·배열·딕셔너리)뿐이며, 커스텀 타입은 `Codable`로 `Data`로 바꿔 저장함
- 크기 제한은 명시되어 있지 않지만 **수백 KB를 넘기기 시작하면 기동 시간과 디스크 쓰기에 영향**을 줌

```swift
struct DisplaySettings: Codable {
    var isDarkMode: Bool
    var fontScale: Double
}

enum SettingsStore {
    private static let key = "displaySettings"

    static func save(_ settings: DisplaySettings) {
        if let data = try? JSONEncoder().encode(settings) {
            UserDefaults.standard.set(data, forKey: key)
        }
    }

    static func load() -> DisplaySettings {
        guard let data = UserDefaults.standard.data(forKey: key),
              let settings = try? JSONDecoder().decode(DisplaySettings.self, from: data)
        else { return DisplaySettings(isDarkMode: false, fontScale: 1.0) }
        return settings
    }
}
```

> ⚠️ `UserDefaults`는 **암호화되지 않은 plist**다. 탈옥 기기나 백업 파일에서 그대로 읽을 수 있으므로 토큰·비밀번호·개인정보를 넣으면 안 된다. 익스텐션과 값을 공유하려면 `UserDefaults(suiteName:)`과 App Group을 사용한다(unit07 참고).

<br>

### 4. Keychain

Keychain은 Security 프레임워크가 제공하는 **시스템 수준의 암호화 저장소**로, 기기의 Secure Enclave와 연계된 키로 보호된다.

- 항목은 `kSecClass`(비밀번호·인터넷 비밀번호·인증서·키 등)와 `kSecAttrService`·`kSecAttrAccount` 조합으로 식별함
- **접근 가능 시점**을 `kSecAttrAccessible`로 지정함. 기본 권장값은 `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`(잠금 해제 상태에서만, 기기 이전 시 미포함)
- 같은 팀 ID의 앱끼리는 **Keychain Access Group**으로 항목을 공유할 수 있음
- C 스타일 API(`SecItemAdd`·`SecItemCopyMatching`)가 불편하므로 실무에서는 얇은 래퍼를 만들어 사용함

```swift
import Security

enum KeychainStore {
    static func save(_ value: Data, service: String, account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account
        ]
        SecItemDelete(query as CFDictionary)   // 기존 항목이 있으면 교체
        var attributes = query
        attributes[kSecValueData as String] = value
        attributes[kSecAttrAccessible as String] = kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        let status = SecItemAdd(attributes as CFDictionary, nil)
        guard status == errSecSuccess else { throw KeychainError.unhandled(status) }
    }

    static func read(service: String, account: String) throws -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        if status == errSecItemNotFound { return nil }
        guard status == errSecSuccess else { throw KeychainError.unhandled(status) }
        return result as? Data
    }
}

enum KeychainError: Error { case unhandled(OSStatus) }
```

> 💡 Keychain 항목은 **앱을 삭제해도 남아 있을 수 있다**. 재설치한 사용자가 "로그아웃했는데 다시 로그인되어 있다"고 느끼는 원인이 되므로, 첫 실행을 감지해 Keychain을 초기화하는 로직을 두는 팀이 많다.

<br>

### 5. 파일 시스템과 디렉터리 정책

앱 샌드박스 안의 디렉터리는 **백업 여부와 시스템의 자동 정리 여부**가 다르다 (샌드박스 구조는 unit07 참고).

| **디렉터리**          | **iCloud·iTunes 백업** | **시스템 자동 삭제**     | **용도**                                   |
| --------------------- | ---------------------- | ------------------------ | ------------------------------------------ |
| **Documents**         | **포함**               | 없음                     | 사용자가 만든 문서, 재생성 불가능한 데이터    |
| **Library/Application Support** | 포함         | 없음                     | 앱이 만든 DB 파일, 설정 등 사용자에게 안 보이는 데이터 |
| **Library/Caches**    | 제외                   | **저장 공간 부족 시 삭제 가능** | 재다운로드 가능한 이미지·응답 캐시          |
| **tmp**               | 제외                   | **앱 미실행 시 삭제 가능**  | 처리 중 임시 파일                           |

- 재생성 가능한 데이터를 Documents에 두면 사용자의 iCloud 용량을 낭비하고 앱 심사에서 지적받을 수 있음
- 파일 단위 보호는 `FileProtectionType`(`.complete`, `.completeUnlessOpen`, `.completeUntilFirstUserAuthentication`)으로 지정하며, 잠금 상태에서 백그라운드 작업이 파일에 접근해야 한다면 `.complete`는 피해야 함

<br>

### 6. Core Data와 SwiftData

### 6-1. Core Data의 구조

Core Data는 데이터베이스가 아니라 **객체 그래프 관리 프레임워크**다. 저장 방식은 SQLite가 기본이지만, 개발자는 SQL이 아니라 관리 객체(`NSManagedObject`)를 다룬다.

```
NSPersistentContainer
 ├─ NSManagedObjectModel      ── 엔티티·속성·관계 정의 (.xcdatamodeld)
 ├─ NSPersistentStoreCoordinator ── 저장소(SQLite 파일)와의 연결
 └─ NSManagedObjectContext    ── 작업 공간. viewContext(메인) / background context
        │
        ▼ fetch / insert / delete → save()
   NSManagedObject (엔티티 인스턴스)
```

- **컨텍스트는 스레드에 묶임**: `viewContext`는 메인 스레드 전용이며, 대량 삽입·갱신은 `performBackgroundTask`로 백그라운드 컨텍스트에서 수행한 뒤 병합함
- **지연 로딩(faulting)**: 관계 객체는 실제 접근 시점까지 로드하지 않아 메모리를 아낌
- 스키마 변경 시 **경량 마이그레이션**(속성 추가 등)은 자동, 구조가 크게 바뀌면 매핑 모델이 필요함

```swift
let request: NSFetchRequest<Note> = Note.fetchRequest()
request.predicate = NSPredicate(format: "isArchived == NO AND updatedAt >= %@", since as NSDate)
request.sortDescriptors = [NSSortDescriptor(key: "updatedAt", ascending: false)]
request.fetchLimit = 50

container.performBackgroundTask { context in
    let notes = (try? context.fetch(request)) ?? []
    // 백그라운드 컨텍스트에서 처리 후 필요 시 objectID로 메인에 전달
}
```

<br>

### 6-2. SwiftData

SwiftData(iOS 17 이상)는 Core Data의 저장 계층을 그대로 쓰면서 **Swift 매크로와 타입 안전한 API**로 감싼 것이다.

```swift
import SwiftData

@Model
final class Note {
    var title: String
    var updatedAt: Date
    var isArchived: Bool
    @Relationship(deleteRule: .cascade) var attachments: [Attachment] = []

    init(title: String, updatedAt: Date = .now, isArchived: Bool = false) {
        self.title = title
        self.updatedAt = updatedAt
        self.isArchived = isArchived
    }
}

// 조회: #Predicate 매크로로 컴파일 타임 검증
let descriptor = FetchDescriptor<Note>(
    predicate: #Predicate { !$0.isArchived },
    sortBy: [SortDescriptor(\.updatedAt, order: .reverse)]
)
let notes = try modelContext.fetch(descriptor)
```

| **항목**          | **Core Data**                            | **SwiftData**                                  |
| ----------------- | ---------------------------------------- | ---------------------------------------------- |
| **모델 정의**     | `.xcdatamodeld` 에디터 + `NSManagedObject` | `@Model` 매크로가 붙은 Swift 클래스              |
| **조회 조건**     | 문자열 `NSPredicate` (오타는 런타임에 발견) | `#Predicate` 매크로 (컴파일 타임 검증)            |
| **UI 연동**       | `NSFetchedResultsController`              | SwiftUI `@Query`로 자동 갱신                     |
| **최소 버전**     | 오래전부터 지원                           | **iOS 17 이상**                                 |
| **성숙도**        | 매우 높음, 자료 풍부                      | 비교적 새로움, 버전에 따라 기능·안정성 차이 있음   |

> ⚠️ SwiftData는 SwiftUI와 결합했을 때 가장 강력하지만, UIKit 중심 프로젝트나 iOS 16 이하를 지원해야 하는 앱에서는 선택지가 아니다. 또한 초기 버전에서는 복잡한 마이그레이션·동시성 처리가 Core Data만큼 세밀하지 않다는 보고가 있으므로, **요구 사항이 복잡하면 Core Data가 여전히 안전한 선택**이다.

<br>

### 7. 선택 판단 흐름

```
저장할 데이터가 민감한가? (토큰·비밀번호·개인 식별 정보)
   │ 예 ──▶ Keychain
   │ 아니오
   ▼
값이 작고 단순한가? (설정·플래그·최근 선택)
   │ 예 ──▶ UserDefaults
   │ 아니오
   ▼
덩어리 파일인가? (이미지·문서·응답 스냅샷)
   │ 예 ──▶ 파일 (재생성 가능하면 Caches, 아니면 Documents/Application Support)
   │ 아니오
   ▼
관계·검색·정렬이 필요한 구조화 데이터
   ├─ iOS 17+ & SwiftUI 중심 ──▶ SwiftData
   └─ 그 외 / 복잡한 마이그레이션 ──▶ Core Data (또는 팀 표준 DB 라이브러리)
```

<br>

### 8. 면접·실무 체크포인트

- **인증 토큰 저장 위치**: Keychain. `UserDefaults`는 평문 plist라 부적합함
- **UserDefaults가 느려지는 경우**: 큰 데이터를 넣으면 실행 시 전체 로드·매 저장 시 전체 직렬화가 발생함
- **Caches vs Documents**: 재생성 가능 여부와 백업 포함 여부로 구분함
- **Core Data는 DB인가?**: 아니다. 객체 그래프 관리 프레임워크이며 SQLite는 기본 저장 방식일 뿐임
- **Core Data 스레드 규칙**: 컨텍스트는 생성된 스레드(큐)에서만 사용, 백그라운드 작업은 별도 컨텍스트
- **SwiftData를 쓰지 못하는 조건**: iOS 16 이하 지원, 복잡한 마이그레이션 요구, UIKit 중심 아키텍처
- 샌드박스 디렉터리 구조와 앱 간 데이터 공유는 **unit07**, 민감 정보 취급 원칙은 **unit10**을 참고할 것
