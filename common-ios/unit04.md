## 셀 재사용과 리스트 성능

`UITableView`·`UICollectionView`는 수천 개의 항목이 있어도 **화면에 보이는 개수만큼의 셀만 만들어 돌려쓰는** 재사용 메커니즘으로 성능을 확보한다. 이 유닛은 재사용의 동작 원리, `prepareForReuse`의 역할, 그리고 비동기 이미지가 **엉뚱한 셀에 표시되는 고전적인 버그**의 원인과 해결책을 다룬다.

<br>

### 1. 셀 재사용이 필요한 이유

- 항목 1만 개짜리 목록에 셀 1만 개를 만들면 메모리와 생성 비용이 감당되지 않음 (unit03의 메모리 한계 참고)
- 화면에 동시에 보이는 셀은 많아야 10~20개이므로, **화면 밖으로 나간 셀을 큐에 넣었다가 새로 들어올 항목에 재활용**하면 셀 개수를 상수로 유지할 수 있음
- 스크롤 중 프레임 드롭(16.6ms 예산 초과)의 대부분은 셀 생성이 아니라 **셀 구성(configure) 단계의 무거운 작업** 때문이며, 재사용을 이해해야 어디를 최적화할지 판단할 수 있음

<br>

### 2. 재사용 큐의 동작 원리

```
          화면 위쪽으로 스크롤 아웃
          ┌──────────┐
  row 0   │  cell A  │ ──────────────┐
          └──────────┘               │  reuse queue
          ┌──────────┐               ▼  ┌───────────────┐
  row 1   │  cell B  │  (화면)        │  cell A         │
  row 2   │  cell C  │               └───────┬─────────┘
  row 3   │  cell D  │                       │ dequeue
          ┌──────────┐                       ▼
  row 4   │  cell A' │ ◀── 같은 인스턴스, 데이터만 row 4 로 교체
          └──────────┘
```

1. `register(_:forCellReuseIdentifier:)`로 셀 클래스와 식별자를 등록함
2. `cellForRowAt`에서 `dequeueReusableCell(withIdentifier:for:)`를 호출하면, 큐에 남는 셀이 있으면 그것을, 없으면 새 인스턴스를 돌려줌
3. 큐에서 꺼낸 셀은 **이전 항목의 상태(텍스트·이미지·선택 상태·토글)를 그대로 가지고 있음**
4. 셀이 큐로 돌아가기 직전(정확히는 dequeue 직전)에 `prepareForReuse()`가 호출되어 초기화 기회를 줌

> 💡 `dequeueReusableCell(withIdentifier:)`(indexPath 없는 버전)는 등록된 셀이 없으면 `nil`을 돌려주지만, `for: indexPath` 버전은 등록이 되어 있으면 **항상 셀을 보장**하고 크기까지 맞춰 준다. 새 코드에서는 후자를 쓰고, 등록을 빠뜨리면 즉시 크래시가 나므로 오히려 디버깅이 쉽다.

<br>

### 3. prepareForReuse의 역할과 한계

`prepareForReuse`는 **이전 항목의 흔적을 지우는** 곳이지, 새 데이터를 넣는 곳이 아니다.

| **prepareForReuse에서 할 일**               | **하지 말아야 할 일**                             |
| ------------------------------------------- | ------------------------------------------------- |
| 이미지 뷰를 `nil` 또는 플레이스홀더로 초기화      | 새 항목의 데이터 세팅 (`cellForRowAt`의 역할)        |
| 진행 중인 이미지 다운로드·비동기 작업 **취소**     | 무거운 뷰 재생성                                    |
| 토글·선택·확장 상태 등 **상태 플래그 초기화**       | 레이아웃 제약 조건 재설정 (init에서 1회면 충분)        |
| 클로저 콜백(`onTap` 등) `nil` 처리              | 데이터 소스나 네트워크 호출                          |

```swift
final class ProductCell: UITableViewCell {
    @IBOutlet private weak var thumbnail: UIImageView!
    @IBOutlet private weak var titleLabel: UILabel!
    private var imageTask: Task<Void, Never>?
    var onFavoriteTap: (() -> Void)?

    override func prepareForReuse() {
        super.prepareForReuse()
        imageTask?.cancel()          // 이전 항목의 다운로드 취소
        imageTask = nil
        thumbnail.image = nil        // 이전 이미지 제거
        titleLabel.text = nil
        onFavoriteTap = nil          // 이전 항목의 콜백 제거
        accessoryType = .none
    }
}
```

> ⚠️ `prepareForReuse`는 셀이 **재사용될 때만** 호출된다. 처음 생성된 셀에서는 호출되지 않으므로, 여기서만 초기 상태를 설정하면 첫 화면의 셀은 초기화되지 않은 채 표시된다. 초기값은 `init`·`awakeFromNib`에서, 재사용 정리는 `prepareForReuse`에서, 항목별 값은 `configure`에서 설정한다.

<br>

### 4. 비동기 이미지가 잘못된 셀에 표시되는 문제

### 4-1. 문제가 발생하는 과정

```
시간 →
row 3 에 cell X 배정 ── 이미지 A 요청 시작 ─────────────────── 응답 A 도착 → cell X 에 A 표시 (❌ 지금 cell X 는 row 9)
                              │
사용자가 빠르게 스크롤       │
row 9 에 cell X 재배정 ───────┴── 이미지 B 요청 시작 ─── 응답 B 도착 → cell X 에 B 표시
                                                          (A가 B보다 늦게 오면 최종 화면은 A ❌)
```

- 셀은 재사용되므로 요청을 보낸 시점의 셀과 응답이 도착한 시점의 셀이 **같은 인스턴스지만 다른 항목**을 가리킴
- 네트워크 응답 순서는 요청 순서와 무관하므로, 느린 응답이 나중에 도착해 최신 이미지를 덮어쓰는 **경쟁 상태(race condition)**가 발생함
- 목록을 빠르게 스크롤할 때 이미지가 깜빡이며 바뀌거나, 전혀 다른 상품의 사진이 보이는 증상으로 나타남

<br>

### 4-2. 안티패턴과 개선

```swift
// 안티패턴: 응답이 도착했을 때 셀이 아직 같은 항목인지 확인하지 않음
func configure(with product: Product) {
    titleLabel.text = product.name
    URLSession.shared.dataTask(with: product.imageURL) { data, _, _ in
        guard let data, let image = UIImage(data: data) else { return }
        DispatchQueue.main.async {
            self.thumbnail.image = image   // 이 시점의 self는 다른 상품을 표시 중일 수 있음
        }
    }.resume()
}
```

```swift
// 개선: 요청 URL을 기억해 두고, 응답 시점에 일치할 때만 반영 + 재사용 시 취소
private var currentImageURL: URL?

func configure(with product: Product) {
    titleLabel.text = product.name
    currentImageURL = product.imageURL

    if let cached = ImageStore.shared.image(for: product.imageURL) {
        thumbnail.image = cached           // 캐시 히트: 즉시 표시, 깜빡임 없음
        return
    }
    thumbnail.image = UIImage(named: "placeholder")

    imageTask = Task { [weak self] in
        guard let image = try? await ImageLoader.shared.load(product.imageURL) else { return }
        guard !Task.isCancelled, self?.currentImageURL == product.imageURL else { return }
        self?.thumbnail.image = image
        ImageStore.shared.store(image, for: product.imageURL)
    }
}
```

**해결 원칙 세 가지**

1. **식별자 비교**: 요청 시 URL(또는 indexPath·항목 ID)을 저장하고, 응답 시 여전히 같은지 확인한 뒤에만 반영함
2. **취소**: `prepareForReuse`에서 진행 중인 작업을 취소해 불필요한 네트워크·디코딩을 줄임
3. **캐시**: 메모리 캐시(`NSCache`)를 앞단에 두어 재방문 시 즉시 표시하고, 요청 자체를 줄임

> 💡 indexPath로 비교하는 방식은 항목 삽입·삭제로 indexPath가 바뀌면 틀릴 수 있다. **항목의 고유 ID나 이미지 URL**을 기준으로 비교하는 편이 안전하다.

<br>

### 5. 리스트 성능을 좌우하는 요소들

| **문제**                          | **원인**                                                        | **해결**                                                                  |
| --------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **스크롤 끊김**                   | `cellForRowAt`에서 동기 디코딩·복잡한 문자열 처리                    | 백그라운드에서 미리 계산, 다운샘플링(unit03), 셀 구성은 값 대입만              |
| **높이 계산 지연**                | 셀마다 `heightForRowAt` 실측                                      | `estimatedRowHeight` + `UITableView.automaticDimension`으로 지연 계산        |
| **오프스크린 렌더링**              | `cornerRadius` + `masksToBounds`, 그림자, 마스크                   | 그림자는 `shadowPath` 지정, 둥근 모서리는 이미지 자체를 가공                  |
| **투명 레이어 합성 비용**          | 반투명 배경이 겹침                                                | 셀·서브뷰를 불투명(`isOpaque = true`)으로, 불필요한 알파 제거                  |
| **네트워크 대기**                 | 셀이 보일 때 비로소 이미지 요청                                     | `UITableViewDataSourcePrefetching`으로 다음 화면 항목을 미리 요청              |
| **전체 reloadData**               | 항목 하나 바뀌어도 전체 갱신                                       | Diffable Data Source로 변경분만 애니메이션 적용                               |

```swift
extension ProductListViewController: UITableViewDataSourcePrefetching {
    func tableView(_ tableView: UITableView, prefetchRowsAt indexPaths: [IndexPath]) {
        for path in indexPaths {
            ImageLoader.shared.prefetch(products[path.row].imageURL)
        }
    }

    func tableView(_ tableView: UITableView, cancelPrefetchingForRowsAt indexPaths: [IndexPath]) {
        for path in indexPaths {
            ImageLoader.shared.cancelPrefetch(products[path.row].imageURL)
        }
    }
}
```

<br>

### 6. Diffable Data Source와 최신 셀 구성 방식

iOS 13부터 제공되는 `UITableViewDiffableDataSource`·`UICollectionViewDiffableDataSource`는 **스냅샷을 비교해 변경분만 적용**하므로, `insertRows`·`deleteRows`의 인덱스 불일치 크래시를 원천적으로 피할 수 있다.

```swift
enum Section { case main }

lazy var dataSource = UITableViewDiffableDataSource<Section, Product.ID>(tableView: tableView) {
    tableView, indexPath, productID in
    let cell = tableView.dequeueReusableCell(withIdentifier: "ProductCell", for: indexPath) as! ProductCell
    if let product = self.store.product(id: productID) {
        cell.configure(with: product)
    }
    return cell
}

func apply(_ products: [Product]) {
    var snapshot = NSDiffableDataSourceSnapshot<Section, Product.ID>()
    snapshot.appendSections([.main])
    snapshot.appendItems(products.map(\.id))
    dataSource.apply(snapshot, animatingDifferences: true)
}
```

- 스냅샷의 항목은 `Hashable`이어야 하며, 모델 전체보다 **ID만 넣는 편**이 갱신 판단이 정확하고 메모리도 적음
- iOS 14 이후 `UICollectionView.CellRegistration`과 컴포지셔널 레이아웃을 함께 쓰면 문자열 식별자와 강제 캐스팅 없이 셀을 구성할 수 있음
- SwiftUI의 `List`도 내부적으로 같은 재사용 원리로 동작하므로, 행 뷰 안에서 비동기 작업을 할 때는 `.task(id:)`처럼 **항목 ID에 바인딩된 작업**을 사용해야 동일한 문제를 피할 수 있음

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                          |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **셀 재사용이란?**                                    | 화면 밖 셀을 큐에 보관했다가 새 항목에 재활용해 셀 개수를 상수로 유지하는 메커니즘             |
| **prepareForReuse는 언제, 무엇을?**                    | 재사용 직전 호출. 이전 항목의 이미지·상태·콜백·진행 중 작업을 정리 (새 데이터 세팅은 아님)     |
| **비동기 이미지가 다른 셀에 보이는 이유는?**             | 요청 시점과 응답 시점의 셀이 다른 항목을 가리키며, 응답 순서가 보장되지 않기 때문             |
| **해결 방법은?**                                      | 요청 식별자(URL·ID) 비교, `prepareForReuse`에서 취소, `NSCache` 캐시                       |
| **스크롤 성능을 높이려면?**                            | 셀 구성은 값 대입만, 다운샘플링·프리페칭, 오프스크린 렌더링 제거, Diffable로 부분 갱신         |
| **reloadData 대신 Diffable을 쓰는 이유는?**             | 변경분만 애니메이션 적용, 인덱스 불일치 크래시 방지, 상태 관리 단순화                         |

- 이미지 메모리와 캐시 정책은 **unit03**, 네트워크 요청과 응답 캐싱은 **unit06**을 참고할 것
