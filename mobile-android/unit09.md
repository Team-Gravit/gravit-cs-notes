## 리스트 성능

피드·채팅·검색 결과처럼 수천 개 항목을 스크롤하는 리스트는 모바일 앱 성능 문제의 대부분이 발생하는 곳이다. View 시스템의 **RecyclerView(ViewHolder 패턴·DiffUtil)**와 Compose의 **LazyColumn**은 구현은 다르지만 "보이는 만큼만 만들고, **재사용**하며, **바뀐 항목만** 갱신한다"는 같은 원칙 위에 서 있다.

<br>

### 1. 왜 리스트가 성능의 병목인가

- 화면은 초당 60프레임(120Hz 기기는 120프레임)을 그려야 하므로 **프레임당 예산은 약 16ms(120Hz는 약 8ms)**임
- 항목 하나를 만드는 비용 = 레이아웃 인플레이트(또는 컴포지션) + 측정·배치 + 데이터 바인딩. 스크롤 중 새 항목이 나타날 때마다 이 비용을 전부 치르면 예산을 초과해 **프레임 드랍(Jank)**이 발생함
- 항목이 1,000개인데 화면에는 10개만 보인다면, 1,000개를 모두 만드는 것은 메모리와 시간 모두 낭비임

```
화면 (보이는 영역)              메모리에 실제 존재하는 항목
┌──────────────┐
│ item 20      │  ─┐
│ item 21      │   │  보이는 10개 + 위아래 여유분 몇 개만 생성
│ ...          │   │  나머지 990개는 "데이터"로만 존재
│ item 29      │  ─┘
└──────────────┘
 스크롤 ↓ → item 20이 화면 밖으로 → 그 View/컴포지션을 item 30에 재사용
```

> 💡 "ScrollView 안에 항목 1,000개를 넣으면 왜 느린가요?"에 대한 답은 **모든 항목을 즉시 생성해 메모리에 올리고 측정까지 하기 때문**이다. RecyclerView·LazyColumn은 보이는 것만 만들고 나머지는 지연(Lazy) 생성한다.

<br>

### 2. RecyclerView와 ViewHolder 패턴

### 2-1. ViewHolder 패턴의 원리

- **ViewHolder**: 항목 View와 그 안의 자식 View 참조(`TextView`, `ImageView` 등)를 **미리 찾아 보관**하는 객체
- 과거 `ListView`에서는 `getView()`마다 `findViewById()`를 반복해 느렸고, ViewHolder 패턴은 이를 한 번만 수행하도록 고안됨. RecyclerView는 이 패턴을 **강제**함
- RecyclerView는 화면 밖으로 나간 ViewHolder를 **재활용 풀(RecycledViewPool)**에 넣고, 새 항목이 필요하면 풀에서 꺼내 데이터만 다시 바인딩함

```
어댑터 호출 흐름
   새 항목 필요
      │
      ├─ 풀에 같은 viewType의 ViewHolder 있음? ── 예 → onBindViewHolder() 만 호출 (데이터만 교체)
      │
      └─ 없음 → onCreateViewHolder() (인플레이트, 비쌈) → onBindViewHolder()
```

<br>

### 2-2. 기본 어댑터 구현

```kotlin
class UserAdapter : RecyclerView.Adapter<UserAdapter.UserViewHolder>() {
    private var items: List<User> = emptyList()

    // 자식 View 참조를 생성 시 한 번만 찾아 둠
    class UserViewHolder(private val binding: ItemUserBinding) : RecyclerView.ViewHolder(binding.root) {
        fun bind(user: User) {
            binding.name.text = user.name
            binding.email.text = user.email
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder =
        UserViewHolder(ItemUserBinding.inflate(LayoutInflater.from(parent.context), parent, false))

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) = holder.bind(items[position])

    override fun getItemCount() = items.size
}
```

- 헤더·광고처럼 레이아웃이 다른 항목은 `getItemViewType()`으로 구분해 **유형별로 풀을 분리**함. 유형이 다른데 같은 viewType을 쓰면 잘못된 레이아웃에 바인딩되어 크래시가 남
- `onBindViewHolder`는 스크롤 중 매우 자주 호출되므로 **가볍게** 유지해야 함. 날짜 포매팅·이미지 디코딩 같은 무거운 작업은 데이터 준비 단계(ViewModel)나 이미지 라이브러리(Coil·Glide)에 맡김

> ⚠️ ViewHolder는 재사용되므로 **바인딩 시 모든 상태를 명시적으로 설정**해야 한다. 어떤 항목에서만 아이콘을 `VISIBLE`로 바꾸고 다른 항목에서 `GONE`으로 되돌리지 않으면, 재활용된 뷰에 이전 항목의 아이콘이 남는 고전적인 버그가 생긴다.

<br>

### 3. DiffUtil — 바뀐 항목만 갱신하기

- 목록이 바뀔 때 `notifyDataSetChanged()`를 부르면 **모든 항목을 다시 바인딩**하고 애니메이션도 없음
- **DiffUtil**은 이전·새 목록을 비교(Eugene Myers 차이 알고리즘)해 삽입·삭제·이동·변경을 계산하고, 필요한 `notifyItemXxx()`만 호출함
- `ListAdapter`는 DiffUtil 계산을 **백그라운드 스레드**에서 수행한 뒤 결과를 메인 스레드에 반영하는 어댑터로, 실무 표준임

```kotlin
object UserDiffCallback : DiffUtil.ItemCallback<User>() {
    // 같은 항목인가? (정체성) → id 비교
    override fun areItemsTheSame(oldItem: User, newItem: User) = oldItem.id == newItem.id
    // 내용이 같은가? (변경 여부) → data class equals
    override fun areContentsTheSame(oldItem: User, newItem: User) = oldItem == newItem
}

class UserAdapter : ListAdapter<User, UserViewHolder>(UserDiffCallback) {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder = /* 동일 */
    override fun onBindViewHolder(holder: UserViewHolder, position: Int) = holder.bind(getItem(position))
}

// 사용: 새 목록을 넘기면 차이를 계산해 필요한 부분만 갱신·애니메이션
adapter.submitList(newUsers)
```

- `areItemsTheSame`은 **정체성(ID)**, `areContentsTheSame`은 **내용**을 비교함. 둘을 뒤섞으면 이동 감지가 깨지거나 변경이 무시됨
- `submitList`에 **같은 리스트 인스턴스를 변경해서 다시 넘기면** 이전과 참조가 같아 갱신이 무시됨. 항상 새 리스트를 만들어 넘겨야 함

<br>

### 4. Compose LazyColumn

### 4-1. 재사용 원리

- `LazyColumn`은 화면에 보이는 항목(과 약간의 사전 로딩 항목)만 **컴포지션**하고, 스크롤로 벗어난 항목의 컴포지션은 폐기함
- 폐기된 항목의 컴포지션 노드는 **재사용 풀**에 들어가 같은 `contentType`의 새 항목에 재사용됨 → RecyclerView의 ViewHolder 풀에 해당하는 최적화가 내부에 존재함
- 항목마다 컴포저블 함수를 실행해야 하므로 인플레이트 비용은 없지만, **항목 컴포저블이 무겁거나 불안정하면** 스크롤 시 재구성 비용이 그대로 드러남

<br>

### 4-2. 필수 설정 — key와 contentType

```kotlin
// 안티패턴: key 없음 → 삽입·삭제 시 항목 상태가 어긋나고 전체가 재구성됨
LazyColumn {
    items(feed) { post -> PostCard(post) }
}

// 개선
LazyColumn(
    modifier = Modifier.fillMaxSize(),
    contentPadding = PaddingValues(vertical = 8.dp),
) {
    stickyHeader { FeedHeader() }
    items(
        items = feed,
        key = { it.id },                                   // 정체성 → 상태 보존·애니메이션·최소 재구성
        contentType = { it.type },                         // 유형별 재사용 풀 (viewType에 해당)
    ) { post ->
        PostCard(post, modifier = Modifier.animateItem())
    }
}
```

- `key`의 역할과 규칙(유일성, Bundle 저장 가능 타입)은 unit04 참고. DiffUtil의 `areItemsTheSame`과 같은 정체성 개념임
- **내용 비교(`areContentsTheSame`)에 해당하는 것은 Compose의 건너뛰기 규칙**임. `Post`가 안정적 타입이고 값이 같으면 `PostCard`는 재실행되지 않음 → 항목 데이터 클래스를 불변으로 설계해야 함

<br>

### 4-3. LazyColumn 특유의 함정

- **`LazyColumn` 안에 `LazyColumn`(세로 중첩)**: 안쪽 높이가 무한대로 측정되어 크래시. 중첩 대신 하나의 `LazyColumn`에 항목 유형을 섞어 넣음
- **`Column` + `verticalScroll` + 많은 항목**: 지연 생성이 아니므로 ScrollView와 같은 문제. 항목이 많으면 반드시 `LazyColumn`
- **항목 안에서 목록 변환**(`posts.filter { }`): 스크롤·재구성마다 새 리스트 생성. ViewModel에서 미리 가공
- **`remember`로 항목별 상태 보관**: 화면 밖으로 나가면 사라짐. 유지해야 하면 `rememberSaveable`(key 필요)이나 ViewModel로 올림

> 💡 "LazyColumn에 DiffUtil이 없는데 어떻게 바뀐 항목만 갱신하나요?"라는 질문에는 **`key`로 정체성을 보존하고, 안정적 인자에 대한 건너뛰기(Skipping)가 내용 비교 역할을 한다**고 답하면 된다. 별도 비교 알고리즘 없이 Compose 런타임의 일반 규칙으로 해결한다.

<br>

### 5. RecyclerView vs LazyColumn 비교

| **항목**                | **RecyclerView (View)**                          | **LazyColumn (Compose)**                                |
| ----------------------- | ------------------------------------------------ | ------------------------------------------------------- |
| **항목 생성**           | XML 인플레이트 + ViewHolder                      | 컴포저블 함수 실행(컴포지션)                            |
| **재사용 단위**         | `viewType`별 ViewHolder 풀                       | `contentType`별 컴포지션 노드 풀                        |
| **정체성**              | `DiffUtil.areItemsTheSame`(ID)                   | `key` 람다                                              |
| **변경 감지**           | `DiffUtil.areContentsTheSame` + 백그라운드 비교  | 안정적 인자 기반 **건너뛰기**                           |
| **부분 갱신·애니메이션** | `notifyItemXxx` + `ItemAnimator`                | `key` + `Modifier.animateItem()`                        |
| **초기 표시 속도**      | 인플레이트 비용 있음                             | 인플레이트 없음, 대신 첫 컴포지션 비용                  |
| **코드량**              | Adapter·ViewHolder·XML·DiffCallback              | `items()` 블록 하나                                     |

- 두 방식 모두 **보이는 것만 생성·재사용·최소 갱신**이라는 동일 원칙을 따르며, 성능 차이보다 **항목 컴포저블/바인딩의 무게**가 실제 체감 성능을 좌우함

<br>

### 6. 공통 최적화 체크리스트

- **데이터는 미리 가공**: 포매팅·정렬·필터링은 ViewModel에서 끝내고 UI 모델(불변 data class)로 내려보냄
- **이미지는 라이브러리에 위임**: Coil·Glide는 크기 조절·캐시·취소를 처리함. 항목 크기에 맞는 해상도를 요청해 메모리를 아낌
- **항목 높이 고정**: 높이가 정해져 있으면 측정이 줄어듦. RecyclerView는 `setHasFixedSize(true)`, Compose는 `Modifier.height()` 고정
- **릴리스 빌드로 측정**: 디버그 빌드(특히 Compose)는 실제보다 훨씬 느림. Macrobenchmark와 Baseline Profile을 적용한 릴리스 빌드에서 판단 (unit04 참고)
- **큰 목록은 페이징**: 수만 건을 한 번에 메모리에 올리지 말고 Paging 라이브러리로 페이지 단위 로딩

❗️**스크롤 성능은 항목 하나의 비용 × 화면에 보이는 개수다**: 리스트 프레임워크가 아무리 잘 재사용해도 항목 하나가 5ms 걸리면 화면에 10개가 보이는 순간 예산을 넘긴다. 최적화의 첫 대상은 항상 **항목 레이아웃·바인딩·컴포저블 자체**다.

<br>

### 7. 정리

- 리스트 성능의 원칙은 **보이는 만큼만 생성, 화면 밖 항목 재사용, 바뀐 항목만 갱신**이며 프레임 예산은 약 16ms임
- **ViewHolder 패턴**은 자식 View 참조를 한 번만 찾고, RecyclerView는 `viewType`별 풀에서 ViewHolder를 재활용함. 재사용되므로 바인딩 시 모든 상태를 명시함
- **DiffUtil·ListAdapter**는 ID(정체성)와 내용을 구분해 비교하고 필요한 갱신만 백그라운드 계산으로 수행함. `submitList`에는 새 리스트 인스턴스를 넘김
- **LazyColumn**은 `key`로 정체성을, 안정적 인자의 **건너뛰기**로 내용 비교를 대신하며 `contentType`으로 재사용 풀을 나눔
- 세로 중첩 LazyColumn, `Column`+`verticalScroll` 남용, 항목 안 목록 변환은 대표적 함정임
- 항목 자체의 무게가 체감 성능을 결정하므로 데이터 가공은 ViewModel에서, 이미지는 라이브러리에서, 측정은 릴리스 빌드에서 수행함
