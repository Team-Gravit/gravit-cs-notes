## 리스트 성능

수천 개의 항목을 스크롤하는 목록은 iOS 앱에서 가장 흔한 화면이자 성능 문제가 가장 자주 드러나는 곳이다. UIKit의 `UITableView`·`UICollectionView`는 **셀 재사용(Cell Reuse)**으로, SwiftUI의 `List`·`LazyVStack`은 **지연 생성(Lazy Creation)**으로 화면 밖 항목의 비용을 줄이며, 각 방식의 원리와 함정이 다르다. 60~120fps를 유지하려면 프레임당 예산(약 8~16ms) 안에 셀 구성과 레이아웃이 끝나야 한다.

<br>

### 1. 왜 목록이 느려지는가

- 항목마다 뷰를 새로 만들면 **메모리와 생성 비용이 항목 수에 비례**한다 (1만 개 → 1만 개의 뷰 계층)
- 셀 하나를 구성하는 데 16ms 이상 걸리면 프레임이 떨어져 **끊김(hitch)**이 발생한다
- 스크롤 중 셀 높이를 매번 새로 계산하거나 이미지를 동기 디코딩하면 메인 스레드가 막힌다

```
화면 (5개 보임)            전체 데이터 (10,000개)
┌──────────────┐
│ cell 0       │ ◀── 실제 뷰 객체는 보이는 개수 + 여유분(약 7~8개)만 존재
│ cell 1       │
│ cell 2       │     나머지 9,990개는 "데이터"로만 존재
│ cell 3       │
│ cell 4       │
└──────────────┘
      ▲ 스크롤로 사라진 셀 → 재사용 풀 → 새 인덱스의 데이터로 다시 구성
```

<br>

### 2. UITableView·UICollectionView — 셀 재사용

### 2-1. 재사용 풀의 동작

- `register(_:forCellReuseIdentifier:)`로 셀 타입을 등록하고, `cellForRowAt`에서 `dequeueReusableCell(withIdentifier:for:)`로 꺼낸다
- 화면 밖으로 나간 셀은 **재사용 풀(reuse queue)**에 들어가고, 새 행이 필요할 때 풀에서 꺼내 **데이터만 갈아끼운다**. 풀이 비어 있을 때만 새 셀을 생성한다
- 셀은 재사용 직전에 `prepareForReuse()`를 호출받으며, 여기서 이전 데이터의 흔적(이미지, 선택 상태, 진행 중 작업)을 지운다

```swift
final class PostCell: UITableViewCell {
    private var imageTask: Task<Void, Never>?

    override func prepareForReuse() {
        super.prepareForReuse()
        imageTask?.cancel()                    // 이전 행의 다운로드 취소
        thumbnail.image = nil                  // 이전 이미지 제거 (잔상 방지)
    }

    func configure(with post: Post) {
        titleLabel.text = post.title
        imageTask = Task { [weak self] in
            let image = await ImageLoader.shared.load(post.imageURL)
            guard !Task.isCancelled else { return }
            self?.thumbnail.image = image
        }
    }
}
```

> ⚠️ 재사용의 대표 버그는 **"스크롤하면 엉뚱한 이미지가 보인다"**이다. 셀 A가 이미지를 비동기로 요청한 뒤 재사용되어 셀 B가 됐는데, 뒤늦게 도착한 A의 이미지가 B에 그려지는 경우다. `prepareForReuse`에서 작업을 취소하거나, 응답 도착 시 **현재 셀의 식별자와 비교**해 다르면 버린다.

<br>

### 2-2. 셀 높이와 자기 크기 조정(Self-Sizing)

- `rowHeight = UITableView.automaticDimension` + `estimatedRowHeight`를 설정하면 Auto Layout 제약으로 셀 높이가 계산된다 (unit06 참고)
- 추정 높이가 실제와 크게 다르면 스크롤 인디케이터가 튀고, 모든 셀 높이를 미리 계산하게 만들면 초기 로딩이 느려진다
- 높이가 항상 같다면 **고정 `rowHeight`**를 주는 것이 가장 빠르다

<br>

### 2-3. 성능 체크리스트

| **항목**                          | **문제**                                             | **대응**                                                     |
| --------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| **`cellForRowAt` 무거움**         | 포매팅·정렬·이미지 디코딩을 매번 수행               | 뷰모델에서 **미리 계산**, 셀은 대입만                        |
| **오프스크린 렌더링**             | `cornerRadius`+`masksToBounds`, `shadow` without path | `shadowPath` 지정, 이미지 자체를 둥글게 처리                 |
| **동기 이미지 로딩**              | 메인 스레드 블로킹                                   | 백그라운드 디코딩 + 캐시, `prepareForReuse`에서 취소         |
| **데이터 갱신 시 `reloadData()`** | 전체 재구성, 애니메이션 없음                         | `UITableViewDiffableDataSource`(iOS 13+)로 **차이만 적용**   |
| **프리페치 미사용**               | 스크롤 직전 항목의 데이터 준비 지연                  | `UITableViewDataSourcePrefetching`으로 미리 요청             |

> 💡 `UICollectionView`는 iOS 13+ **컴포지셔널 레이아웃(Compositional Layout)**과 **Diffable Data Source**를 조합하는 것이 현재 표준이다. 테이블뷰 스타일의 목록도 `UICollectionLayoutListConfiguration`으로 구성할 수 있으며, 셀 내용을 SwiftUI로 작성할 때는 `UIHostingConfiguration`(iOS 16+)을 쓴다 (unit05 참고).

<br>

### 3. SwiftUI — List와 LazyVStack

### 3-1. List

- `List`는 내부적으로 UIKit 컬렉션 뷰(iOS 16+, 이전 버전은 테이블뷰)를 사용해 **셀 재사용과 지연 생성**을 모두 제공한다
- 스와이프 액션, 구분선, 섹션 헤더, 편집 모드, 선택 등 **목록 기능이 내장**되어 있다
- 행의 SwiftUI 뷰는 정체성(unit01 참고)을 기준으로 관리되므로, `ForEach`의 `id`가 불안정하면 상태가 뒤섞인다

```swift
struct FeedList: View {
    let posts: [Post]                      // Identifiable

    var body: some View {
        List(posts) { post in              // id: post.id
            PostRow(post: post)            // 가벼운 별도 struct (unit04 참고)
        }
        .listStyle(.plain)
    }
}
```

<br>

### 3-2. LazyVStack (ScrollView 안)

- `ScrollView { LazyVStack { ForEach(...) } }`는 항목 뷰를 **화면에 가까워질 때 생성**한다. 하지만 UIKit식 **재사용 풀은 없으며**, 한 번 생성된 뷰가 화면 밖으로 나가도 해제가 보장되지 않는다(버전에 따라 다를 수 있음)
- 목록 기능(스와이프, 구분선 등)이 없어 **자유로운 레이아웃**이 필요할 때 적합하다
- 일반 `VStack`은 모든 항목을 **즉시 생성**하므로 수백 개 이상에는 쓰지 않는다

```swift
// 안티패턴: 수천 개 항목을 VStack으로 → 모든 행이 즉시 생성되어 첫 렌더가 수 초
ScrollView {
    VStack {
        ForEach(posts) { PostRow(post: $0) }
    }
}
```

```swift
// 개선: LazyVStack으로 지연 생성, 이미지는 캐시된 로더로 비동기 로딩
ScrollView {
    LazyVStack(spacing: 12) {
        ForEach(posts) { post in
            PostRow(post: post)
                .onAppear { if post == posts.last { loadMore() } }   // 페이지네이션 트리거
        }
    }
}
```

<br>

### 3-3. 비교

| **항목**             | **VStack**            | **LazyVStack**                        | **List**                                     |
| -------------------- | --------------------- | ------------------------------------- | -------------------------------------------- |
| **생성 시점**        | 전부 즉시             | **보일 때**                           | 보일 때                                      |
| **셀 재사용**        | 없음                  | 없음                                  | **있음** (UIKit 기반)                        |
| **내장 목록 기능**   | 없음                  | 없음                                  | 스와이프·구분선·섹션·편집·선택               |
| **레이아웃 자유도**  | 높음                  | 높음                                  | 제한적 (`listRowInsets` 등으로 조정)         |
| **적합 규모**        | 수십 개 이하          | 수백~수천 (메모리 주의)               | 수천 개 이상, 표준 목록 UI                   |

<br>

### 4. SwiftUI 목록의 성능 함정

- **행 뷰가 무겁다**: 행 안에서 날짜 포매팅·정렬·`AnyView` 사용은 스크롤마다 반복된다. 표시용 문자열은 모델에서 미리 만든다
- **불안정한 id**: `ForEach(items.indices, id: \.self)`나 `UUID()`를 body에서 생성하면 스크롤마다 새 뷰로 인식돼 재사용이 무의미해진다
- **상위 상태 변경으로 전체 목록 재평가**: 검색어 `@State`가 목록과 같은 뷰에 있으면 타이핑마다 `List` 전체가 재평가된다. 행을 별도 `struct`로 분리하고 상태를 내려보낸다 (unit01·unit04 참고)
- **`ObservableObject` 하나를 모든 행이 구독**: 한 행의 변경이 전체 행을 갱신한다. iOS 17+ `@Observable`로 바꾸면 읽은 프로퍼티만 추적된다 (unit03 참고)
- **이미지**: `AsyncImage`는 기본 캐시가 제한적이므로 대량 목록에서는 자체 캐시 로더(또는 검증된 라이브러리)를 쓴다

> ⚠️ `List` 행 안의 `@State`는 행이 재사용·재생성될 때 초기화될 수 있다. 확장/접힘 같은 **행별 UI 상태는 모델이나 `Set<ID>` 형태로 부모가 보관**하고, 행은 바인딩으로 받는 것이 안전하다.

<br>

### 5. 선택 기준

| **상황**                                                | **권장**                                                 |
| ------------------------------------------------------- | -------------------------------------------------------- |
| 표준적인 목록(설정·피드·채팅)이며 SwiftUI 앱           | **`List`**                                               |
| 카드형·수평 캐러셀·격자 등 자유로운 레이아웃            | `ScrollView` + **`LazyVStack`/`LazyVGrid`**              |
| 수만 개 항목, 정밀한 스크롤 제어, 복잡한 셀 애니메이션  | **`UICollectionView`** + Diffable + (선택) `UIHostingConfiguration` |
| 기존 UIKit 화면 유지, 셀 UI만 현대화                    | `UITableView`/`UICollectionView` + `UIHostingConfiguration` |

<br>

### 6. 정리

- UIKit 목록의 핵심은 **재사용 풀 + `prepareForReuse` + 비동기 작업 취소**이며, 셀 구성은 대입만 하도록 가볍게 유지한다
- SwiftUI `List`는 재사용과 목록 기능을 모두 제공하고, `LazyVStack`은 지연 생성만 제공하며, `VStack`은 소규모 전용이다
- 양쪽 모두 **안정적인 id·가벼운 행·상태 범위 최소화·이미지 캐시**가 성능의 대부분을 결정한다
- 셀 높이 계산 원리는 unit06, 셀 안에 SwiftUI를 넣는 방법은 unit05를 참고할 것
