## 레이아웃 시스템

iOS의 레이아웃은 UIKit의 **Auto Layout(제약 기반)**과 SwiftUI의 **크기 협상(제안-응답 기반)**이라는 서로 다른 모델 위에 서 있다. 두 모델 모두 "누가 크기를 결정하는가"를 이해하면 대부분의 레이아웃 버그(잘리는 텍스트, 늘어나는 뷰, 충돌 경고)를 설명할 수 있으며, 상호운용(unit05 참고) 시에는 두 모델의 경계에서 크기가 어떻게 넘어가는지도 알아야 한다.

<br>

### 1. Auto Layout — 제약으로 방정식을 푼다

### 1-1. 제약(Constraint)의 구조

Auto Layout의 제약은 **선형 방정식(또는 부등식)**이다. 엔진(Cassowary 알고리즘 기반)이 모든 제약을 동시에 만족하는 프레임을 계산한다.

```
item1.attribute  =  multiplier × item2.attribute  +  constant
label.leading    =  1.0 × superview.leading       +  16
imageView.width  =  1.5 × imageView.height        +  0
```

```swift
let label = UILabel()
label.translatesAutoresizingMaskIntoConstraints = false   // 코드로 제약을 줄 때 필수
view.addSubview(label)
NSLayoutConstraint.activate([
    label.leadingAnchor.constraint(equalTo: view.safeAreaLayoutGuide.leadingAnchor, constant: 16),
    label.trailingAnchor.constraint(lessThanOrEqualTo: view.trailingAnchor, constant: -16),
    label.centerYAnchor.constraint(equalTo: view.centerYAnchor),
])
```

- 한 축을 결정하려면 **위치 + 크기**(또는 양 끝) 정보가 충분해야 한다. 부족하면 **모호(ambiguous)**, 넘치면 **충돌(unsatisfiable)**이다
- 크기가 명시되지 않은 뷰는 자신의 **`intrinsicContentSize`**(레이블·버튼·이미지뷰가 콘텐츠로부터 계산)로 크기를 제안한다

> ⚠️ `translatesAutoresizingMaskIntoConstraints`를 `false`로 두지 않으면 기존 프레임이 자동 제약으로 변환되어 개발자 제약과 충돌한다. 콘솔의 "Unable to simultaneously satisfy constraints" 경고 대부분이 이 설정 누락 또는 필수 제약 중복에서 나온다.

<br>

### 1-2. 우선순위(Priority)와 콘텐츠 크기 우선순위

모든 제약은 **1~1000의 우선순위**를 가진다. 1000(`required`)은 반드시 만족해야 하고, 그 이하는 "가능하면" 만족한다. 충돌이 나면 낮은 우선순위가 양보한다.

| **우선순위 상수**   | **값**   | **의미**                                   |
| ------------------- | -------- | ------------------------------------------ |
| **`.required`**     | **1000** | 위반 시 런타임 경고, 반드시 만족           |
| **`.defaultHigh`**  | **750**  | 콘텐츠 압축 저항의 기본값                  |
| **`.defaultLow`**   | **250**  | 콘텐츠 허깅의 기본값                       |

`intrinsicContentSize`를 가진 뷰는 두 가지 **콘텐츠 크기 우선순위**로 "늘어남·줄어듦"에 대한 태도를 표현한다.

| **항목**                            | **질문**                          | **높을수록**                     | **기본값** |
| ----------------------------------- | --------------------------------- | -------------------------------- | ---------- |
| **Content Hugging** (허깅)          | 콘텐츠보다 **커지기 싫은가**       | 늘어나지 않으려 함               | 250        |
| **Compression Resistance** (압축 저항) | 콘텐츠보다 **작아지기 싫은가**  | 잘리지 않으려 함                 | 750        |

```swift
// 제목은 잘리면 안 되고(압축 저항↑), 부제목은 남는 공간을 채워도 됨(허깅↓)
titleLabel.setContentCompressionResistancePriority(.required, for: .horizontal)
subtitleLabel.setContentHuggingPriority(.defaultLow, for: .horizontal)
titleLabel.setContentHuggingPriority(.defaultHigh, for: .horizontal)
```

```
[UIStackView 폭 300] ── 두 레이블의 intrinsic 폭 합 200 → 남는 100은 누가 가져가나?
  titleLabel  hugging 750 ─┐
  subtitleLabel hugging 250 ─┴─▶ 허깅이 낮은 subtitleLabel이 늘어남
[UIStackView 폭 150] ── intrinsic 합 200 → 50은 누가 잘리나?
  titleLabel  resistance 1000 ─┐
  subtitleLabel resistance 750 ─┴─▶ 저항이 낮은 subtitleLabel이 잘림
```

> 💡 "두 레이블을 나란히 뒀는데 어느 쪽이 잘릴지 모호하다"는 경고가 나오면 **허깅·압축 저항 우선순위가 같아서** 엔진이 결정할 수 없기 때문이다. 한쪽을 1만큼이라도 높이면 해결된다. 면접에서 허깅과 압축 저항의 차이를 묻는 이유가 이것이다.

<br>

### 1-3. 레이아웃 패스와 갱신

- 제약을 바꿔도 즉시 프레임이 바뀌지 않는다. `setNeedsLayout()`으로 **무효화 표시**만 하고, 다음 런루프의 레이아웃 패스에서 `layoutSubviews()`가 실행된다
- 애니메이션과 함께 즉시 반영하려면 제약 변경 후 `UIView.animate { self.view.layoutIfNeeded() }`로 **강제 패스**를 돌린다
- `UIStackView`는 자식들의 제약을 자동 생성해주는 컨테이너로, 축·분배(`distribution`)·정렬(`alignment`)만 지정하면 허깅/압축 저항으로 나머지를 결정한다

<br>

### 2. SwiftUI — 부모가 제안하고 자식이 결정한다

### 2-1. 3단계 협상

SwiftUI에는 제약이 없다. 대신 부모와 자식이 **크기를 협상**한다.

```
① 부모 → 자식 : "너에게 이 정도 공간을 제안한다"  (proposed size, nil 가능)
② 자식 → 부모 : "나는 이 크기를 쓰겠다"           (자식이 최종 결정, 제안을 무시할 수도 있음)
③ 부모        : 자식이 보고한 크기로 위치를 잡는다 (배치)

  VStack(제안 390×800)
    ├ Text ─────▶ "필요한 만큼만" 200×20    (제안보다 작게 응답)
    ├ Image ────▶ "원본 크기" 1000×600      (제안보다 크게 응답 → 넘침)
    └ Color ────▶ "주는 대로" 390×180       (제안을 전부 사용)
```

- **자식이 크기의 최종 결정권**을 가진다. 부모는 자식을 강제로 줄일 수 없고 그저 배치만 한다
- `Text`·`Image`처럼 고유 크기를 가진 뷰는 필요한 만큼만 쓰고, `Color`·`Rectangle`·`Spacer`처럼 유연한 뷰는 제안을 다 쓴다
- 모디파이어가 중첩된 뷰(unit04 참고)에서는 제안이 **바깥에서 안쪽으로** 전달되고 응답이 **안쪽에서 바깥으로** 돌아온다. 이것이 `.frame().background()`와 `.background().frame()`이 다른 이유다

<br>

### 2-2. 크기 협상을 조정하는 모디파이어

| **모디파이어**                     | **역할**                                                                 |
| ---------------------------------- | ------------------------------------------------------------------------ |
| **`.frame(width:height:)`**        | 자식에게 **고정 크기**를 제안하고 자신도 그 크기로 응답                  |
| **`.frame(maxWidth: .infinity)`**  | 제안된 공간을 **최대한** 차지 (배경·정렬 컨테이너 역할)                  |
| **`.fixedSize()`**                 | 제안을 무시하고 **이상적 크기(ideal size)**로 응답 → 텍스트 줄임 방지    |
| **`.layoutPriority(n)`**           | 스택에서 공간 분배 시 **우선순위**가 높은 뷰가 먼저 필요한 만큼 가져감   |
| **`Spacer()`**                     | 남는 공간을 모두 흡수하는 유연한 뷰                                      |
| **`GeometryReader`**               | 제안된 공간을 **전부 차지**하고 그 크기를 클로저로 알려줌                |

```swift
// 안티패턴: 긴 제목이 "..."으로 잘리는데 원인을 몰라 frame을 남발
HStack {
    Text(longTitle)                           // HStack이 나눠준 공간에 맞춰 줄임
    Text(date).font(.caption)
}
```

```swift
// 개선: 우선순위와 fixedSize로 협상 규칙을 명시
HStack {
    Text(longTitle)
        .layoutPriority(1)                    // 제목이 먼저 필요한 공간을 가져감
    Text(date)
        .font(.caption)
        .fixedSize()                          // 날짜는 절대 줄이지 않음
}
```

> ⚠️ `GeometryReader`는 제안된 공간을 **모두 차지**하므로, 텍스트 하나의 크기를 재려고 감싸면 레이아웃 전체가 망가진다. 크기 측정은 `.background(GeometryReader { ... })`처럼 **배경에 숨겨** 쓰거나, iOS 16+ `Layout` 프로토콜, iOS 17+ `onGeometryChange`(버전에 따라 가용성 다름)를 사용한다.

<br>

### 2-3. 스택의 공간 분배 규칙

`HStack`·`VStack`은 자식에게 공간을 나눠줄 때 다음 순서로 동작한다.

- 각 자식의 **유연성(최소~최대 크기 범위)**을 조사한다
- **유연성이 가장 낮은 자식부터** 남은 공간을 균등 분할한 크기를 제안한다 (고정 크기 뷰가 먼저 확정)
- 자식이 응답한 크기를 빼고 남은 공간을 다음 자식에게 제안한다
- `layoutPriority`가 높은 자식은 이 순서에서 앞으로 당겨진다

이 규칙 때문에 `HStack { Text; Spacer(); Text }`에서 `Spacer`는 항상 마지막에 남은 공간만 받는다.

<br>

### 2-4. 커스텀 레이아웃 — Layout 프로토콜 (iOS 16+)

```swift
struct FlowLayout: Layout {
    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        // 제안 폭 안에서 줄바꿈하며 필요한 높이 계산
        let maxWidth = proposal.width ?? .infinity
        var x: CGFloat = 0, y: CGFloat = 0, rowHeight: CGFloat = 0
        for sub in subviews {
            let size = sub.sizeThatFits(.unspecified)
            if x + size.width > maxWidth { x = 0; y += rowHeight; rowHeight = 0 }
            x += size.width; rowHeight = max(rowHeight, size.height)
        }
        return CGSize(width: maxWidth, height: y + rowHeight)
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
        var x = bounds.minX, y = bounds.minY, rowHeight: CGFloat = 0
        for sub in subviews {
            let size = sub.sizeThatFits(.unspecified)
            if x + size.width > bounds.maxX { x = bounds.minX; y += rowHeight; rowHeight = 0 }
            sub.place(at: CGPoint(x: x, y: y), proposal: .unspecified)
            x += size.width; rowHeight = max(rowHeight, size.height)
        }
    }
}
```

- `sizeThatFits`가 협상 ②단계(응답), `placeSubviews`가 ③단계(배치)에 정확히 대응한다
- 태그 클라우드·불규칙 그리드처럼 스택으로 표현하기 어려운 레이아웃을 `GeometryReader` 없이 구현할 수 있다

<br>

### 3. 두 모델 비교

| **항목**            | **Auto Layout (UIKit)**                          | **크기 협상 (SwiftUI)**                          |
| ------------------- | ------------------------------------------------ | ------------------------------------------------ |
| **모델**            | 제약 방정식을 엔진이 **동시에** 해결             | 부모 제안 → 자식 결정 → 부모 배치의 **재귀**     |
| **크기 결정권**     | 제약 집합 전체 (전역)                            | **자식** (지역적, 단방향)                        |
| **충돌 처리**       | 우선순위로 양보, 위반 시 경고                    | 충돌 개념 없음 — 자식이 넘치면 그냥 넘침         |
| **콘텐츠 크기**     | `intrinsicContentSize` + 허깅/압축 저항          | 이상적 크기 + `fixedSize`/`layoutPriority`       |
| **디버깅**          | 경고 로그, `hasAmbiguousLayout`, 뷰 디버거       | `.border()`로 프레임 확인, `Self._printChanges()` |
| **경계 통과**       | `UIHostingController.sizingOptions`, `sizeThatFits(in:)` | `UIViewRepresentable.sizeThatFits` (iOS 16+) |

> 💡 상호운용 시 UIKit 쪽 `intrinsicContentSize`가 SwiftUI의 "이상적 크기"로, SwiftUI의 응답 크기가 UIKit의 호스팅 뷰 크기로 번역된다. 호스팅 뷰가 늘어나거나 0 크기로 잡히면 **어느 쪽이 크기를 결정하기로 했는지**부터 확인한다 (unit05 참고).

<br>

### 4. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                          |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **허깅과 압축 저항의 차이는?**                        | 허깅은 **커지기 싫음**(기본 250), 압축 저항은 **작아지기 싫음**(기본 750)               |
| **제약 충돌 경고가 나는 대표 원인은?**                | 필수(1000) 제약 중복, `translatesAutoresizingMaskIntoConstraints` 미설정              |
| **SwiftUI에서 크기는 누가 정하는가?**                 | 부모가 **제안**하고 **자식이 결정**, 부모는 배치만                                      |
| **텍스트가 "..."으로 잘릴 때 대처는?**                 | `layoutPriority`로 순서 조정, `fixedSize`로 줄임 방지, 불필요한 `frame` 제거            |
| **GeometryReader 남용이 문제인 이유는?**              | 제안된 공간을 **전부 차지**해 주변 레이아웃을 깨뜨림. `Layout` 프로토콜이 대안         |

- Auto Layout은 **제약 + 우선순위**, SwiftUI는 **제안 + 응답 + 우선순위**로 정리한다
- 모디파이어 순서가 레이아웃을 바꾸는 이유(unit04)와 셀 높이 계산(unit07)은 모두 이 협상 규칙의 응용이다
