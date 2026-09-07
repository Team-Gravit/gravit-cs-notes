## 메모리 압력과 백그라운드 종료

iOS에는 스왑(swap) 영역이 없어 물리 메모리가 부족해지면 시스템이 **앱을 직접 종료해 메모리를 회수**한다. 이 유닛은 메모리 경고가 오는 원리, 경고를 받았을 때 무엇을 먼저 버려야 하는지, 앱과 익스텐션이 쓸 수 있는 메모리 한계, 그리고 백그라운드에서 조용히 종료되는 상황에 대비하는 방법을 정리한다.

<br>

### 1. iOS 메모리 관리의 전제

- 데스크톱 OS는 메모리가 부족하면 디스크로 페이지를 내보내지만, iOS는 **플래시 수명과 성능을 이유로 스왑을 사용하지 않음**
- 대신 커널의 **Jetsam**이라는 메커니즘이 메모리 압력을 감시하다가 우선순위가 낮은 프로세스부터 종료함
- 종료 우선순위는 대략 **백그라운드 앱 → 포그라운드 앱 → 시스템 프로세스** 순이며, 같은 그룹 안에서는 메모리를 많이 쓰는 프로세스가 먼저 희생됨
- 따라서 "우리 앱이 메모리를 얼마나 쓰는가"는 자기 앱의 안정성뿐 아니라 **백그라운드에 있을 때 살아남을 확률**을 결정함

> 💡 Jetsam에 의한 종료는 크래시 리포트에 일반적인 스택 트레이스가 없고, 대신 `EXC_RESOURCE` 유형의 이벤트나 설정 앱의 "분석 데이터"에 JetsamEvent 로그로 남는다. 크래시 도구에 잡히지 않는 "이유 없는 재시작"의 대부분이 여기에 해당한다.

<br>

### 2. 메모리 풋프린트의 구성

시스템이 판단하는 앱의 메모리 사용량은 단순 할당량이 아니라 **풋프린트(footprint)**다.

| **종류**                | **설명**                                                        | **Jetsam 계산 포함** |
| ----------------------- | --------------------------------------------------------------- | -------------------- |
| **Dirty 메모리**        | 앱이 기록한 힙 객체, 디코딩된 이미지 버퍼, 캐시 등                  | **포함**             |
| **Compressed 메모리**   | 한동안 접근하지 않아 압축된 Dirty 페이지                            | **포함**             |
| **Clean 메모리**        | 실행 파일 코드, 매핑된 파일 등 디스크에서 다시 읽을 수 있는 페이지     | 미포함 (회수 가능)   |

- 이미지 하나를 `UIImage(named:)`로 화면에 그리면 파일 크기가 아니라 **가로 × 세로 × 4바이트**의 디코딩 버퍼가 Dirty 메모리로 잡힘. 4000×3000 사진 한 장이 약 48MB에 해당함
- 압축된 메모리를 다시 접근하면 해제 비용이 들므로, 백그라운드 진입 전에 **큰 버퍼를 아예 해제**하는 편이 압축에 맡기는 것보다 유리함

<br>

### 3. 메모리 경고가 전달되는 경로

```
        커널(Jetsam)이 메모리 압력 감지
                    │
                    ▼
   UIApplication.didReceiveMemoryWarningNotification
                    │
        ┌───────────┴────────────┐
        ▼                        ▼
 AppDelegate                모든 UIViewController
 applicationDidReceive      didReceiveMemoryWarning()
 MemoryWarning(_:)          (뷰가 로드된 인스턴스에 호출)
        │                        │
        └───── 응답 없으면 ──────┘
                    │
                    ▼
        압력이 계속되면 Jetsam이 프로세스 종료
```

- 경고는 "지금 정리하지 않으면 곧 종료된다"는 마지막 기회이며, 응답에 쓸 수 있는 시간은 매우 짧음
- `DispatchSource.makeMemoryPressureSource(eventMask: [.warning, .critical])`를 쓰면 UIKit 콜백보다 세분화된 단계(`warning`·`critical`)를 백그라운드 큐에서 받을 수 있음
- 뷰 컨트롤러의 `didReceiveMemoryWarning`은 예전처럼 화면 밖 뷰를 자동 해제하지 않으므로, 개발자가 명시적으로 정리해야 함

<br>

### 4. 메모리 경고 시 조치 우선순위

정리 대상은 **"버려도 다시 만들 수 있고, 크기가 큰 것"**부터 순서대로 처리한다.

| **우선순위** | **대상**                                          | **이유**                                             |
| ------------ | ------------------------------------------------- | ---------------------------------------------------- |
| **1**        | 이미지·데이터 캐시 (`NSCache`, 직접 만든 딕셔너리 캐시) | 가장 크고, 다시 받거나 디코딩하면 복원 가능             |
| **2**        | 화면에 보이지 않는 뷰 컨트롤러가 붙잡은 대용량 데이터    | 사용자가 돌아올 때 다시 로드하면 됨                     |
| **3**        | 미리 계산해 둔 결과, 프리페치한 다음 페이지 데이터       | 성능 최적화용이라 없어도 기능은 동작함                  |
| **4**        | 재생·편집 중인 미디어 버퍼                            | 사용자 경험에 직접 영향을 주므로 마지막에 축소            |

```swift
// 안티패턴: 직접 만든 딕셔너리 캐시는 경고가 와도 스스로 비워지지 않음
final class ImageStore {
    static let shared = ImageStore()
    var images: [URL: UIImage] = [:]
}
```

```swift
// 개선: NSCache는 메모리 압력 시 자동 축출되고, 비용 한도도 설정할 수 있음
final class ImageStore {
    static let shared = ImageStore()
    private let cache = NSCache<NSURL, UIImage>()

    init() {
        cache.totalCostLimit = 100 * 1024 * 1024   // 약 100MB
        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil, queue: .main
        ) { [weak self] _ in
            self?.cache.removeAllObjects()           // 경고 시 즉시 전량 비움
        }
    }

    func image(for url: URL) -> UIImage? { cache.object(forKey: url as NSURL) }

    func store(_ image: UIImage, for url: URL) {
        let cost = Int(image.size.width * image.size.height * image.scale * image.scale * 4)
        cache.setObject(image, forKey: url as NSURL, cost: cost)
    }
}
```

> ⚠️ `NSCache`가 자동으로 비워진다고 해서 안심하면 안 된다. 축출 시점과 양은 시스템 재량이며, 경고를 받은 직후에도 캐시가 남아 있을 수 있다. 경고 알림에서 **명시적으로 비우는 코드**를 함께 두는 것이 안전하다.

<br>

### 5. 앱과 익스텐션의 메모리 한계

메모리 한계는 공식 문서에 수치로 명시되어 있지 않고 **기기·OS 버전에 따라 다르다**. 다만 커뮤니티에서 반복적으로 확인된 경향은 다음과 같다.

| **대상**                    | **알려진 한계(대략)**                        | **비고**                                        |
| --------------------------- | -------------------------------------------- | ----------------------------------------------- |
| **포그라운드 앱**           | 기기 RAM의 상당 부분 (수 GB 기기에서 GB 단위)   | `os_proc_available_memory()`로 남은 양 조회 가능 |
| **백그라운드 앱**           | 포그라운드보다 훨씬 낮은 우선순위로 먼저 종료됨   | 큰 버퍼를 들고 있으면 최우선 희생 대상            |
| **위젯 익스텐션**           | **약 30MB**                                   | 이미지 몇 장으로도 초과 가능                      |
| **공유 익스텐션**           | **약 120MB**                                  | 원본 이미지를 그대로 디코딩하면 초과 위험          |

- `os_proc_available_memory()`(iOS 13 이상)는 현재 프로세스가 추가로 사용할 수 있는 메모리를 바이트 단위로 돌려주며, 대용량 처리 전에 분기 기준으로 활용할 수 있음
- 익스텐션은 한계가 매우 낮으므로 **이미지 다운샘플링**(원본을 메모리에 다 올리지 않고 축소본만 생성)이 사실상 필수임

```swift
import ImageIO

func downsampledImage(at url: URL, maxPixel: CGFloat) -> UIImage? {
    let options: [CFString: Any] = [kCGImageSourceShouldCache: false]
    guard let source = CGImageSourceCreateWithURL(url as CFURL, options as CFDictionary) else { return nil }
    let thumbOptions: [CFString: Any] = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceThumbnailMaxPixelSize: maxPixel
    ]
    guard let cgImage = CGImageSourceCreateThumbnailAtIndex(source, 0, thumbOptions as CFDictionary) else { return nil }
    return UIImage(cgImage: cgImage)
}
```

> 💡 `UIImage(contentsOfFile:)`로 원본을 열고 `draw`로 줄이는 방식은 축소 과정에서 **원본 크기의 버퍼가 잠시 생긴다**. `ImageIO`의 썸네일 생성은 원본을 디코딩하지 않고 축소본만 만들므로 메모리 피크가 훨씬 낮다.

<br>

### 6. 백그라운드 종료에 대비하기

Suspended 상태의 앱이 Jetsam으로 종료되면 **어떤 콜백도 호출되지 않는다**(unit01 참고). 따라서 "종료될 때 저장한다"는 전략은 성립하지 않으며, **백그라운드 진입 시점에 이미 저장이 끝나 있어야** 한다.

- `sceneDidEnterBackground`에서 입력 중인 초안, 스크롤 위치, 진행 중인 화면 식별자를 저장함
- 다음 실행 시 `didFinishLaunchingWithOptions`에서 저장된 상태를 읽어 **사용자가 보던 화면을 복원**함. UIKit의 상태 복원 API(`NSUserActivity` 기반)를 활용할 수 있음
- 네트워크 업로드처럼 중단되면 안 되는 작업은 프로세스와 무관하게 동작하는 **백그라운드 `URLSession`**(unit06)에 위임함
- 백그라운드 진입 시 큰 이미지 버퍼와 캐시를 미리 줄여 두면 **Jetsam 우선순위에서 밀려나 살아남을 확률**이 높아짐

```swift
func sceneDidEnterBackground(_ scene: UIScene) {
    DraftStore.shared.save(editor.currentDraft)      // 1. 사용자 데이터 먼저
    RestorationState.save(currentRoute: router.route) // 2. 복원용 화면 정보
    ImageStore.shared.trimToMinimum()                 // 3. 큰 캐시 축소
}
```

<br>

### 7. 진단 도구와 예방 습관

- **Xcode Memory Gauge**: 디버그 실행 중 실시간 사용량 확인. 화면을 오갈 때 계속 우상향하면 누수 의심
- **Memory Graph Debugger**: 해제되지 않은 인스턴스와 참조 경로 시각화 (순환 참조는 unit02 참고)
- **Instruments – Allocations / Leaks**: 할당 추이와 누수 지점 추적
- **Instruments – VM Tracker**: Dirty·Compressed·Clean 비율 확인
- 예방 습관: 이미지는 표시 크기로 다운샘플링, 셀에서는 `prepareForReuse`로 참조 해제(unit04), 큰 배열은 필요한 페이지만 로드, 클로저에서 `[weak self]` 습관화

<br>

### 8. 정리

| **항목**                          | **핵심**                                                                        |
| --------------------------------- | ------------------------------------------------------------------------------- |
| **iOS 메모리 관리의 특징**         | 스왑 없음 → Jetsam이 우선순위 낮은 프로세스부터 종료                               |
| **경고 수신 경로**                | `didReceiveMemoryWarningNotification` → AppDelegate·모든 뷰 컨트롤러             |
| **조치 우선순위**                 | 캐시 → 화면 밖 대용량 데이터 → 프리페치 결과 → 미디어 버퍼                          |
| **메모리 한계**                   | 기기·버전 의존. 위젯 약 30MB, 공유 익스텐션 약 120MB 수준으로 알려짐                 |
| **백그라운드 종료 대비**          | 종료 콜백에 의존하지 말고 `sceneDidEnterBackground`에서 저장 완료                    |
| **가장 효과적인 절감법**          | 이미지 다운샘플링과 `NSCache` 기반 캐시 + 경고 시 명시적 비우기                      |
