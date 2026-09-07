## 뷰 모디파이어와 재사용 설계

SwiftUI의 **뷰 모디파이어(View Modifier)**는 뷰를 "수정"하는 것이 아니라 **기존 뷰를 감싼 새로운 뷰를 반환**한다. 그래서 적용 순서에 따라 결과가 달라지고, 자주 쓰는 조합은 커스텀 모디파이어로 묶어 재사용한다. 이 유닛은 모디파이어의 동작 원리, 순서 함정, 커스텀 모디파이어 작성법, 그리고 **뷰를 언제·어떻게 분리할지**의 기준을 다룬다.

<br>

### 1. 모디파이어는 뷰를 감싼다

`.padding()`, `.background()` 같은 모디파이어는 `View` 프로토콜의 메서드이며, 호출하면 `ModifiedContent<원본, 모디파이어>` 타입의 **새로운 뷰**를 반환한다. 원본 뷰는 바뀌지 않는다.

```swift
let v = Text("안녕")
    .padding()                  // ModifiedContent<Text, _PaddingLayout>
    .background(.yellow)        // ModifiedContent<ModifiedContent<Text, _PaddingLayout>, _BackgroundModifier>
```

```
바깥 ◀─────────────────────────── 안쪽
background( padding( Text("안녕") ) )

      ┌ background(.yellow) ┐
      │  ┌ padding() ┐      │
      │  │  Text     │      │
      │  └───────────┘      │
      └─────────────────────┘
```

- 체인에서 **나중에 쓴 모디파이어일수록 바깥쪽**에 위치한다
- 레이아웃은 바깥에서 안쪽으로 크기를 제안하고 안쪽에서 바깥으로 크기를 보고하므로(unit06 참고), **순서가 곧 레이아웃 결과**가 된다

<br>

### 2. 적용 순서에 따른 결과 차이

```swift
// A: 패딩 → 배경  : 배경이 패딩 영역까지 칠해짐 (노란 상자 안에 여백 있는 텍스트)
Text("A").padding().background(.yellow)

// B: 배경 → 패딩  : 텍스트 크기만큼만 배경, 그 바깥에 투명한 여백
Text("B").background(.yellow).padding()

// C: frame → background : 배경이 200×80 전체
Text("C").frame(width: 200, height: 80).background(.yellow)

// D: background → frame : 배경은 텍스트 크기, 200×80 안에 가운데 배치
Text("D").background(.yellow).frame(width: 200, height: 80)
```

| **조합**                           | **결과**                                        | **자주 쓰는 의도**                       |
| ---------------------------------- | ----------------------------------------------- | ---------------------------------------- |
| `.padding().background()`          | 배경이 **여백 포함** 영역을 채움                | 버튼·태그 형태의 칩 UI                   |
| `.background().padding()`          | 배경은 콘텐츠만, 바깥 여백은 투명               | 다른 뷰와의 간격 확보                    |
| `.frame().background()`            | 배경이 **프레임 전체**                          | 고정 크기 카드                           |
| `.cornerRadius()` 후 `.border()`   | 테두리는 각진 채로 남음                         | → `.overlay(RoundedRectangle().stroke())` 사용 |
| `.clipped()` 후 `.shadow()`        | 그림자가 잘림                                   | → `.shadow()`를 클리핑 **밖**에 적용     |

> ⚠️ 탭 영역도 순서에 좌우된다. `.onTapGesture{}.padding()`은 패딩 영역이 탭에 반응하지 않고, `.padding().onTapGesture{}`는 여백까지 탭된다. 배경이 투명한 영역은 기본적으로 히트 테스트에서 제외되므로, 빈 영역까지 탭을 받으려면 `.contentShape(Rectangle())`을 제스처 **안쪽**에 둔다.

<br>

### 3. 커스텀 모디파이어

### 3-1. ViewModifier 프로토콜

같은 모디파이어 조합이 세 곳 이상 반복되면 `ViewModifier`로 추출한다. `body(content:)`에서 `content`가 원본 뷰다.

```swift
struct CardStyle: ViewModifier {
    var cornerRadius: CGFloat = 12

    func body(content: Content) -> some View {
        content
            .padding(16)
            .background(Color(.secondarySystemBackground))
            .clipShape(RoundedRectangle(cornerRadius: cornerRadius))
            .shadow(color: .black.opacity(0.08), radius: 8, y: 2)
    }
}

extension View {
    func cardStyle(cornerRadius: CGFloat = 12) -> some View {
        modifier(CardStyle(cornerRadius: cornerRadius))
    }
}

// 사용
VStack { /* ... */ }.cardStyle()
```

- `ViewModifier`는 `@State`·`@Environment`를 가질 수 있어, **환경값에 반응하는 스타일**(다크 모드별 그림자 등)을 캡슐화하기 좋다
- 단순 조합이고 상태가 없다면 `extension View { func cardStyle() -> some View { self.padding()... } }`처럼 확장만으로도 충분하다

<br>

### 3-2. 조건부 모디파이어의 함정

```swift
// 안티패턴: 조건에 따라 뷰 타입이 갈라져 정체성이 바뀜 (상태 초기화·애니메이션 단절, unit01 참고)
extension View {
    @ViewBuilder
    func `if`<T: View>(_ cond: Bool, transform: (Self) -> T) -> some View {
        if cond { transform(self) } else { self }
    }
}
Text("hi").if(isHighlighted) { $0.foregroundStyle(.red) }
```

```swift
// 개선: 항상 같은 모디파이어를 적용하고 "값"만 조건으로 바꿈
Text("hi").foregroundStyle(isHighlighted ? .red : .primary)
Text("hi").opacity(isVisible ? 1 : 0)            // if로 뷰를 빼는 대신
```

> 💡 "조건부로 뷰를 감쌀지 말지"보다 "항상 감싸되 **값을 조건부**로"가 SwiftUI의 기본 원칙이다. 뷰 타입이 갈라지면 `if/else`와 똑같이 구조적 정체성이 바뀐다.

<br>

### 4. 뷰 분리 기준

`body`가 길어지면 나누고 싶어지지만, **어떤 단위로 어떻게 나누느냐**가 성능과 정체성에 영향을 준다.

### 4-1. 분리 방법 세 가지 비교

| **방법**                     | **형태**                                  | **정체성·상태**                     | **재평가 최적화**                | **적합한 경우**                   |
| ---------------------------- | ----------------------------------------- | ----------------------------------- | -------------------------------- | --------------------------------- |
| **별도 `struct` 뷰**         | `struct RowView: View`                    | 독립된 정체성, `@State` 소유 가능   | 입력이 같으면 **body 건너뜀**    | 재사용, 상태 소유, 무거운 뷰      |
| **계산 프로퍼티**            | `var header: some View { ... }`           | 부모의 일부 (정체성 공유)           | 부모와 **함께 항상 재평가**      | 가독성용 단순 분할                |
| **`@ViewBuilder` 메서드**    | `func row(_ item: Item) -> some View`     | 부모의 일부                         | 부모와 함께 재평가               | 매개변수가 필요한 단순 분할       |

- 계산 프로퍼티·메서드는 **코드 정리**일 뿐 렌더 단위를 나누지 않는다. 재평가 범위를 줄이려면 별도 `struct`가 필요하다
- 별도 `struct`는 자신이 읽는 상태에만 의존하므로, 부모의 다른 상태가 바뀌어도 `body`가 건너뛰어진다

<br>

### 4-2. 분리해야 하는 신호

- 뷰가 **자기만의 상태**(`@State`, 포커스, 애니메이션 값)를 가질 때 → 반드시 `struct`로 분리
- 같은 UI가 **두 곳 이상**에서 쓰일 때
- `body`가 한 화면을 넘거나 **컴파일러가 타입 추론에 실패**("expression too complex")할 때
- 특정 상태 변경 시 **무거운 하위 뷰가 함께 재평가**되는 것이 프로파일링에서 확인될 때

```swift
// 분리 전: 타이핑마다 전체 body 재평가, ChartView도 매번 값 생성
struct DashboardView: View {
    @State private var query = ""
    @State private var data: [Point] = []
    var body: some View {
        VStack {
            TextField("검색", text: $query)
            ChartView(data: data)          // 별도 struct라면 data가 같을 때 body 건너뜀
        }
    }
}
```

```swift
// 분리 후: 상태를 사용하는 곳으로 내려보내고, 재사용 단위는 struct로
struct DashboardView: View {
    @State private var data: [Point] = []
    var body: some View {
        VStack {
            SearchField()                  // query를 내부에서 소유 → 타이핑이 Dashboard를 건드리지 않음
            ChartView(data: data)
        }
    }
}
```

<br>

### 5. 재사용 가능한 컨테이너 뷰 만들기

여러 화면에서 쓰는 "껍데기" 뷰는 **제네릭 `Content`와 `@ViewBuilder`**로 내용을 주입받게 설계한다.

```swift
struct SectionCard<Content: View>: View {
    let title: String
    @ViewBuilder let content: () -> Content     // 호출부에서 여러 뷰를 자연스럽게 나열 가능

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            Text(title).font(.headline)
            content()
        }
        .cardStyle()
    }
}

// 사용
SectionCard(title: "최근 주문") {
    OrderRow(order: a)
    OrderRow(order: b)
}
```

- 스타일 옵션이 많아지면 매개변수 대신 **환경값(`EnvironmentKey`)**으로 전달해 하위 트리 전체에 일관되게 적용할 수 있다
- `ButtonStyle`·`LabelStyle` 같은 **스타일 프로토콜**은 "뷰 구조는 두고 외형만 바꾸는" 공식 확장 지점이므로, 버튼 계열은 커스텀 모디파이어보다 `ButtonStyle`이 우선이다

> 💡 재사용 뷰의 API는 UIKit의 서브클래싱이 아니라 **조합(composition)**으로 설계한다. "상속으로 변형"이 아니라 "내용을 주입하고 환경으로 스타일링"이 SwiftUI의 관용구다.

<br>

### 6. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                     |
| ----------------------------------------------------- | --------------------------------------------------------------------------------- |
| **모디파이어 순서가 왜 결과를 바꾸는가?**             | 모디파이어는 **뷰를 감싼 새 뷰**를 반환하므로, 순서가 곧 중첩 구조이자 레이아웃    |
| **`.padding().background()`와 반대 순서의 차이는?**   | 전자는 배경이 여백을 포함, 후자는 콘텐츠만 배경                                    |
| **커스텀 모디파이어는 언제 만드는가?**                | 같은 조합이 반복되거나 환경값에 반응하는 스타일을 캡슐화할 때 `ViewModifier` 사용 |
| **계산 프로퍼티로 나눈 뷰와 struct로 나눈 뷰의 차이?** | 계산 프로퍼티는 **부모와 함께 재평가**, struct는 독립 정체성과 재평가 최적화       |
| **조건부 모디파이어 확장(`.if`)이 위험한 이유는?**     | 뷰 타입이 갈라져 **정체성이 바뀌고** 상태·애니메이션이 끊김                        |

- 스타일은 **모디파이어·스타일 프로토콜**로, 구조는 **제네릭 컨테이너 + @ViewBuilder**로 재사용한다
- 상태를 가진 단위와 무거운 단위는 `struct`로 분리해 무효화 범위를 좁힌다 (unit01 참고)
- 크기 협상 관점의 순서 의미는 unit06(레이아웃 시스템)에서 이어진다
