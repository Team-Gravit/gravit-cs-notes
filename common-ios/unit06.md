## 네트워크 계층

대부분의 iOS 앱은 서버와 통신하는 코드가 가장 많고 가장 자주 깨진다. 이 유닛은 `URLSession`의 구조와 동작, `Codable` 기반 응답 디코딩, 평문 통신을 막는 **ATS(App Transport Security)**, 그리고 `URLCache`와 HTTP 캐시 헤더가 만드는 **응답 캐싱**을 동작 원리 중심으로 정리한다.

<br>

### 1. URLSession의 구조

```
URLSessionConfiguration ── (.default / .ephemeral / .background)
          │  타임아웃, 헤더, 캐시 정책, 쿠키 저장소 등
          ▼
      URLSession ─────────── delegate (인증, 리다이렉트, 진행률, 백그라운드 완료)
          │
          ├─ dataTask       ── 메모리로 응답 수신 (JSON API)
          ├─ downloadTask   ── 파일로 저장 (대용량, 백그라운드 가능)
          └─ uploadTask     ── 본문 업로드 (파일·멀티파트)
```

| **구성(Configuration)** | **특징**                                                        | **적합한 상황**                          |
| ----------------------- | --------------------------------------------------------------- | ---------------------------------------- |
| **default**             | 디스크 캐시·쿠키·자격 증명을 영구 저장                            | 일반 API 호출                            |
| **ephemeral**           | 캐시·쿠키·자격 증명을 **메모리에만** 보관, 세션 종료 시 폐기        | 시크릿 모드, 민감한 요청                  |
| **background**          | 별도 시스템 프로세스가 전송을 대행 → **앱이 정지·종료돼도 계속**    | 대용량 다운로드·업로드                    |

- `URLSession.shared`는 편의용 싱글턴으로, 델리게이트와 세부 설정을 지정할 수 없음. 인증 처리·진행률·백그라운드가 필요하면 직접 세션을 만들어야 함
- 세션은 생성 비용이 있으므로 **요청마다 만들지 말고** 앱 전역 또는 계층 단위로 하나를 재사용함
- 백그라운드 세션은 앱이 종료된 뒤 완료되면 시스템이 앱을 다시 깨워 `application(_:handleEventsForBackgroundURLSession:completionHandler:)`를 호출함 (앱 생명주기는 unit01 참고)

<br>

### 2. 요청 보내기 — 콜백에서 async/await로

```swift
// 안티패턴: 상태 코드 무시, 강제 언래핑, 메인 스레드 보장 없음
URLSession.shared.dataTask(with: url) { data, response, error in
    let users = try! JSONDecoder().decode([User].self, from: data!)
    self.tableView.reloadData()      // 백그라운드 스레드에서 UI 갱신
}.resume()
```

```swift
// 개선: 상태 코드 검증, 오류 타입화, 메인 액터에서 UI 갱신
enum APIError: Error {
    case invalidResponse
    case httpStatus(Int)
    case decoding(Error)
}

final class APIClient {
    private let session: URLSession
    private let decoder: JSONDecoder

    init(session: URLSession = .shared) {
        self.session = session
        decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase
        decoder.dateDecodingStrategy = .iso8601
    }

    func fetch<T: Decodable>(_ type: T.Type, from request: URLRequest) async throws -> T {
        let (data, response) = try await session.data(for: request)
        guard let http = response as? HTTPURLResponse else { throw APIError.invalidResponse }
        guard (200..<300).contains(http.statusCode) else { throw APIError.httpStatus(http.statusCode) }
        do { return try decoder.decode(T.self, from: data) }
        catch { throw APIError.decoding(error) }
    }
}

// 호출부 (뷰 컨트롤러·뷰모델)
Task { @MainActor in
    do {
        users = try await client.fetch([User].self, from: URLRequest(url: url))
        tableView.reloadData()
    } catch {
        showError(error)
    }
}
```

- `URLSession`의 completion handler는 **백그라운드 큐에서 호출**되므로 UI 갱신 전에 메인으로 돌아와야 함. async/await 버전에서는 호출부를 `@MainActor`로 두면 자연스럽게 해결됨
- `error`가 `nil`이어도 HTTP 4xx·5xx는 성공 콜백으로 들어온다. **전송 오류와 HTTP 오류는 별개**이므로 상태 코드를 반드시 확인함
- 요청 취소는 `Task`를 취소하면 내부 `URLSessionTask`까지 취소되며, 화면이 사라질 때 진행 중인 요청을 정리하는 데 활용함(unit02·unit04 참고)

> 💡 `dataTask`는 `resume()`을 호출해야 시작된다. 콜백 방식에서 "요청이 아예 나가지 않는" 버그의 1순위 원인이다. async 버전(`data(for:)`)은 호출 즉시 시작되므로 이 실수가 사라진다.

<br>

### 3. Codable 디코딩

### 3-1. 키 매핑과 전략

```swift
struct User: Decodable {
    let id: Int
    let displayName: String        // JSON: "display_name" → convertFromSnakeCase로 자동 매핑
    let joinedAt: Date             // JSON: "2024-03-01T09:00:00Z" → iso8601 전략
    let avatarURL: URL?            // 없을 수 있는 필드는 옵셔널

    enum CodingKeys: String, CodingKey {
        case id, displayName, joinedAt
        case avatarURL = "img"         // 이름이 전혀 다른 키만 직접 지정
    }
}
```

> ⚠️ `keyDecodingStrategy`와 `CodingKeys`를 함께 쓸 때는 **전략이 먼저 JSON 키를 변환한 뒤** `CodingKeys`의 원시값과 비교한다. 예를 들어 `.convertFromSnakeCase` 상태에서 `case avatarURL = "avatar_url"`로 지정하면 JSON 키가 이미 `avatarUrl`로 바뀌어 매칭에 실패한다. 이런 경우에는 프로퍼티 이름을 `avatarUrl`로 맞추거나 전략 없이 `CodingKeys`만 사용한다.

```swift
// 참고: 전략 없이 CodingKeys만으로 매핑하는 형태
struct Profile: Decodable {
    let avatarURL: URL?
    enum CodingKeys: String, CodingKey {
        case avatarURL = "avatar_url"
    }
}
```

| **상황**                              | **대응**                                                         |
| ------------------------------------- | ---------------------------------------------------------------- |
| 필드가 가끔 누락됨                     | 옵셔널 프로퍼티 (`let x: Int?`)                                     |
| 필드 이름이 다름                       | `CodingKeys` 또는 `keyDecodingStrategy`                             |
| 날짜 형식이 다양함                     | `dateDecodingStrategy = .custom` 으로 여러 포맷 시도                  |
| 서버가 숫자를 문자열로 보냄            | `init(from:)` 직접 구현해 두 타입 모두 수용                            |
| 응답이 `{ "data": {...}, "meta": ... }` 로 감싸짐 | 제네릭 래퍼 `struct Envelope<T: Decodable>: Decodable { let data: T }` |

<br>

### 3-2. 디코딩 실패 원인 파악

`DecodingError`는 **어느 키에서, 어떤 타입 불일치로** 실패했는지 상세 정보를 담고 있다. 이를 로그로 남기지 않으면 "서버가 이상해요"로 끝나기 쉽다.

```swift
catch let DecodingError.keyNotFound(key, context) {
    print("누락된 키: \(key.stringValue), 경로: \(context.codingPath.map(\.stringValue))")
} catch let DecodingError.typeMismatch(type, context) {
    print("타입 불일치: \(type), 경로: \(context.codingPath.map(\.stringValue))")
}
```

> ⚠️ 서버가 필드 하나만 추가·삭제해도 디코딩 전체가 실패해 화면이 비는 앱은 취약하다. **없어도 되는 필드는 옵셔널로**, 열거형은 알 수 없는 값을 받을 `unknown` 케이스를 두어 **서버 변경에 관대한(tolerant) 모델**을 만든다.

<br>

### 4. ATS(App Transport Security)

ATS는 iOS 9부터 적용된 정책으로, 앱의 네트워크 연결이 **HTTPS와 현대적인 TLS 설정**을 사용하도록 강제한다. 평문 HTTP 요청은 기본적으로 차단되어 오류로 실패한다.

- 요구 사항: TLS 1.2 이상, 순방향 비밀성(forward secrecy)을 지원하는 암호 스위트, SHA-256 이상 서명 인증서 등 (세부 조건은 OS 버전에 따라 조정될 수 있음)
- 예외는 `Info.plist`의 `NSAppTransportSecurity`에 선언하며, **도메인 단위 예외**가 원칙임

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <key>legacy.example.com</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSIncludesSubdomains</key>
            <true/>
        </dict>
    </dict>
</dict>
```

| **키**                               | **효과**                                        | **심사 관점**                                   |
| ------------------------------------ | ----------------------------------------------- | ----------------------------------------------- |
| **NSAllowsArbitraryLoads**           | 모든 도메인의 평문 허용                          | 정당한 사유 없이는 **리젝 사유**                  |
| **NSExceptionDomains**               | 특정 도메인만 예외                               | 사유 설명 시 대체로 허용                          |
| **NSAllowsArbitraryLoadsInWebContent** | `WKWebView` 내 콘텐츠만 예외                   | 웹뷰로 임의 사이트를 여는 앱에 적절               |
| **NSAllowsLocalNetworking**          | 로컬 네트워크(IoT 기기 등) 평문 허용             | 해당 기능이 있으면 허용                           |

> 💡 개발 서버가 HTTP라서 `NSAllowsArbitraryLoads`를 켜 두고 그대로 배포하는 사례가 많다. Debug 구성에서만 예외를 적용하도록 `Info.plist`를 빌드 구성별로 분리하거나, 로컬 서버도 자체 서명 인증서로 HTTPS를 쓰는 편이 안전하다. TLS 자체의 원리는 네트워크 챕터 unit19, 인증서 검증과 피닝은 웹 보안 챕터 unit10을 참고할 것.

<br>

### 5. 응답 캐싱

### 5-1. URLCache와 HTTP 캐시 헤더

`URLSession`은 별도 코드 없이도 **HTTP 표준 캐시 규칙**을 따르는 `URLCache`를 갖고 있다. 서버가 보내는 헤더가 캐시 동작을 결정한다.

```
요청 → URLCache 조회
        ├─ 유효한 캐시 있음(Cache-Control: max-age 이내) ──▶ 네트워크 없이 즉시 응답
        ├─ 만료됐지만 ETag / Last-Modified 있음 ──▶ 조건부 요청(If-None-Match)
        │        └─ 304 Not Modified ──▶ 캐시 본문 재사용 (본문 전송 절약)
        └─ 캐시 없음 / no-store ──▶ 일반 요청 후 응답 저장 (저장 가능한 경우)
```

- 캐시 저장 여부와 기간은 `Cache-Control`(`max-age`, `no-cache`, `no-store`), `Expires`, `ETag`, `Last-Modified` 헤더로 결정됨 (HTTP 캐시 상세는 네트워크 챕터 unit18 참고)
- `URLCache.shared`의 기본 용량은 작으므로 이미지가 많은 앱은 `URLCache(memoryCapacity:diskCapacity:)`로 늘려 세션 구성에 지정함
- **POST 응답은 기본적으로 캐시되지 않으며**, 캐시는 GET 요청에 초점을 맞춤

<br>

### 5-2. 요청 단위 캐시 정책

```swift
var request = URLRequest(url: url)
request.cachePolicy = .returnCacheDataElseLoad   // 캐시 있으면 만료 여부와 무관하게 사용, 없으면 로드
request.timeoutInterval = 15

let config = URLSessionConfiguration.default
config.urlCache = URLCache(memoryCapacity: 20 * 1024 * 1024, diskCapacity: 200 * 1024 * 1024)
config.requestCachePolicy = .useProtocolCachePolicy   // 기본값: HTTP 헤더 규칙을 따름
let session = URLSession(configuration: config)
```

| **정책**                            | **동작**                                           | **사용 예**                        |
| ----------------------------------- | -------------------------------------------------- | ---------------------------------- |
| **useProtocolCachePolicy**          | HTTP 헤더 규칙대로 (기본값)                          | 대부분의 API                        |
| **reloadIgnoringLocalCacheData**    | 캐시 무시하고 항상 네트워크                          | 당겨서 새로고침, 결제 상태 조회       |
| **returnCacheDataElseLoad**         | 캐시 우선, 없을 때만 네트워크                        | 변경이 드문 정적 리소스              |
| **returnCacheDataDontLoad**         | 캐시만 사용, 없으면 실패                             | 오프라인 모드                        |

> ⚠️ `URLCache`는 **HTTP 응답 캐시**이지 이미지 디코딩 결과 캐시가 아니다. 목록 스크롤 성능을 위해서는 디코딩된 `UIImage`를 `NSCache`에 따로 보관하는 2단 캐시(메모리 이미지 캐시 + URLCache 디스크 캐시)가 일반적이다(unit03·unit04 참고). 또한 인증 토큰이 포함된 개인화 응답이 디스크 캐시에 남지 않도록 서버가 `Cache-Control: private` 또는 `no-store`를 보내는지 확인해야 한다.

<br>

### 6. 실무에서 함께 고려할 것

- **인증 헤더 주입·토큰 갱신**: 401 응답 시 리프레시 토큰으로 재발급 후 원 요청을 1회 재시도하는 계층을 `APIClient` 안에 둠. 동시 다발 401에 대한 중복 갱신은 단일 갱신 작업을 공유해 방지함 (토큰 저장은 unit05, 인증 방식은 웹 보안 챕터 unit06 참고)
- **재시도와 지수 백오프**: 네트워크 오류(타임아웃·연결 끊김)에만 재시도하고, 4xx는 재시도하지 않음
- **네트워크 상태 감지**: `NWPathMonitor`로 연결 여부·셀룰러 여부를 관찰해 대용량 다운로드를 Wi-Fi에서만 수행하도록 제어함
- **테스트 용이성**: `URLProtocol` 서브클래스로 응답을 가로채면 실제 서버 없이 `APIClient`를 단위 테스트할 수 있음 (의존성 주입 구조는 unit08 참고)

<br>

### 7. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                            |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **URLSession 구성 3종의 차이는?**                  | default(디스크 캐시·쿠키 유지) / ephemeral(메모리만) / background(앱 종료 후에도 전송)       |
| **error가 nil인데 실패하는 이유는?**               | HTTP 4xx·5xx는 전송 성공으로 취급됨. `HTTPURLResponse.statusCode`를 직접 확인해야 함          |
| **completion handler에서 UI 갱신 시 주의점은?**    | 백그라운드 큐에서 호출되므로 메인 큐(`@MainActor`)로 전환                                   |
| **ATS란?**                                        | HTTPS·TLS 1.2 이상을 강제하는 정책. 예외는 도메인 단위로 최소화, 전체 허용은 리젝 사유         |
| **응답 캐시는 어떻게 동작하는가?**                  | `URLCache`가 `Cache-Control`·`ETag` 등 HTTP 헤더 규칙을 따름. 요청별 `cachePolicy`로 조정   |
| **디코딩 실패에 강한 모델을 만들려면?**             | 선택 필드는 옵셔널, 열거형은 `unknown` 케이스, `DecodingError` 로깅                          |
