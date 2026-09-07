## Reflow와 Repaint

unit01(브라우저 렌더링 파이프라인)의 후속 문서로, DOM이나 스타일이 바뀌었을 때 **파이프라인의 어느 단계부터 다시 실행되는지**를 다룬다. 어떤 속성이 레이아웃(Reflow)을, 어떤 속성이 페인트(Repaint)만을, 어떤 속성이 합성(Composite)만을 유발하는지 구분할 수 있어야 애니메이션과 DOM 조작 성능을 설명할 수 있다.

<br>

### 1. 변경이 파이프라인에 미치는 영향

렌더링 파이프라인은 **한 번 그리고 끝나는 것이 아니라** 변경이 생길 때마다 필요한 단계부터 다시 실행된다. 어떤 단계에서 다시 시작하느냐에 따라 비용이 크게 달라진다.

```
변경 종류                 다시 실행되는 단계
─────────────────────    ────────────────────────────────────────────
기하(크기·위치) 변경  ──▶ [스타일] → [레이아웃] → [페인트] → [합성]   가장 비쌈
색상·배경 변경        ──▶ [스타일] ────────────▶ [페인트] → [합성]
transform·opacity 변경 ─▶ [스타일] ──────────────────────▶ [합성]     가장 저렴
```

| **용어**               | **다른 이름**       | **의미**                                             |
| ---------------------- | ------------------- | ---------------------------------------------------- |
| **Reflow**             | Layout, Relayout    | 요소의 **위치·크기를 다시 계산**함. 자식·형제·조상으로 연쇄됨 |
| **Repaint**            | Redraw              | 레이아웃 변화 없이 **픽셀만 다시 그림**               |
| **Composite only**     | 합성 전용 업데이트  | 레이아웃·페인트 없이 **레이어 위치·투명도만 갱신**    |

> 💡 Reflow가 일어나면 Repaint는 반드시 뒤따르지만, Repaint가 일어난다고 Reflow가 일어나는 것은 아니다. "Reflow ⊃ Repaint" 관계를 먼저 답하고 속성 예시를 붙이면 정확한 답변이 된다.

<br>

### 2. Reflow(레이아웃)를 유발하는 변경

레이아웃은 한 요소의 기하 정보가 바뀌면 그 영향이 트리 전체로 퍼지기 때문에 가장 비싸다.

**Reflow를 유발하는 대표 속성·행위**

- 크기·여백: `width`, `height`, `padding`, `margin`, `border-width`
- 위치·배치: `top`, `left`, `position`, `display`, `float`, `flex`·`grid` 관련 속성
- 글꼴·텍스트: `font-size`, `font-family`, `line-height`, 텍스트 내용 변경
- DOM 구조: 노드 추가·삭제·이동, `innerHTML` 교체
- 창·뷰포트: 브라우저 창 크기 변경, 스크롤바 등장, 폰트 로딩 완료
- **기하 정보 읽기**: `offsetWidth`, `clientHeight`, `scrollTop`, `getBoundingClientRect()`, `getComputedStyle()`의 일부 값

```
Reflow 전파 범위
html
└── body
    └── main
        ├── article   ◀── article의 height가 바뀌면
        └── aside     ◀── 형제 aside의 y좌표가 밀리고
            └── ...   ◀── 그 자손도 모두 다시 계산됨
```

> ⚠️ 마지막 항목(기하 정보 읽기)이 가장 자주 놓치는 함정이다. 값을 **읽기만** 해도 브라우저는 최신 결과를 보장하기 위해 예약된 레이아웃을 즉시 실행한다(강제 동기 레이아웃). 값을 바꾸는 코드가 없더라도 반복문 안에서 `offsetHeight`를 읽으면 매 반복마다 레이아웃이 돌 수 있다.

<br>

### 3. Repaint만 유발하는 변경

기하 정보는 그대로이고 **보이는 모습만** 바뀌는 속성은 레이아웃을 건너뛰고 페인트부터 다시 실행된다.

- `color`, `background-color`, `background-image`
- `border-color`, `border-style`, `outline`
- `box-shadow`, `visibility`
- `border-radius`(크기가 변하지 않으므로 페인트만 다시 함)

```css
/* 호버 시 배경만 바꿈 → Reflow 없이 Repaint만 발생 */
.button:hover {
  background-color: #2563eb;
}

/* 호버 시 크기가 바뀜 → 주변 요소가 밀리며 Reflow 발생 */
.button-bad:hover {
  padding: 12px 24px;
}
```

Repaint는 Reflow보다 저렴하지만, `box-shadow`·`filter`처럼 픽셀 연산이 무거운 속성이 큰 영역에 걸쳐 있으면 여전히 프레임을 떨어뜨릴 수 있다.

<br>

### 4. transform·opacity가 합성만 일으키는 이유

`transform`과 `opacity`는 특별하다. 이 두 속성은 요소가 **자체 합성 레이어**로 분리되어 있다면 레이아웃도 페인트도 건드리지 않는다.

**동작 원리**

- 레이어는 이미 래스터화된 **비트맵(텍스처)**으로 GPU에 올라가 있음
- `transform: translateX(100px)`는 그 비트맵을 **어디에 배치할지**만 바꾸며, 비트맵 내용은 그대로임
- `opacity: 0.5`는 비트맵을 합성할 때 **얼마나 투명하게 섞을지**만 바꿈
- 따라서 합성 스레드가 GPU에 "이 텍스처를 저 좌표에, 이 투명도로" 명령만 다시 내리면 끝남

```
일반 속성 (left) 애니메이션           transform 애니메이션
────────────────────────            ────────────────────────
메인 스레드: 스타일→레이아웃→페인트   메인 스레드: (관여 없음)
   ↓ 프레임마다 반복                  합성 스레드: 텍스처 좌표만 갱신
합성 스레드: 합성                        ↓ JS가 바빠도 60fps 유지
```

```css
/* 안티패턴: left/top 애니메이션 → 프레임마다 Reflow + Repaint */
.slide-bad {
  transition: left 300ms ease;
}
.slide-bad.open { left: 240px; }

/* 개선: transform 애니메이션 → 합성만 발생 */
.slide-good {
  transition: transform 300ms ease;
  will-change: transform;          /* 애니메이션 직전에만 부여하는 것이 이상적 */
}
.slide-good.open { transform: translateX(240px); }
```

> 💡 "왜 `left` 대신 `transform`을 쓰나요?"라는 질문의 정답은 "GPU 가속이라서"가 아니라 **"레이아웃과 페인트를 건너뛰고 합성 단계만 다시 실행되기 때문"**이다. 여기에 "합성은 메인 스레드와 분리된 합성 스레드에서 처리되므로 JS 실행 중에도 끊기지 않는다"를 덧붙이면 완성된 답변이 된다.

❗️**레이어가 없으면 합성 전용이 아니다**: `transform`을 처음 적용하는 순간에는 레이어를 새로 만들기 위해 페인트가 한 번 발생한다. 또한 `filter`, `clip-path` 등은 브라우저·조건에 따라 페인트를 다시 유발할 수 있으므로 DevTools의 Performance 패널로 실제 동작을 확인한다.

<br>

### 5. 레이아웃 스래싱(Layout Thrashing)

JavaScript에서 **읽기와 쓰기를 번갈아 수행**하면 브라우저가 변경을 모아 처리하지 못하고 매번 강제 동기 레이아웃을 실행한다. 이를 **레이아웃 스래싱**이라 한다.

```javascript
// 안티패턴: 읽기(offsetWidth) → 쓰기(style.width) → 읽기 → 쓰기 ...
// 매 반복마다 레이아웃이 강제 실행됨 (요소 100개면 레이아웃 100번)
const items = document.querySelectorAll('.item');
items.forEach((el) => {
  const w = el.offsetWidth;            // 읽기 → 직전 쓰기 때문에 레이아웃 강제 실행
  el.style.width = `${w + 10}px`;      // 쓰기 → 레이아웃 무효화
});

// 개선: 읽기를 모두 끝낸 뒤 쓰기를 한 번에 수행 (레이아웃 1번)
const widths = Array.from(items, (el) => el.offsetWidth);   // 읽기 일괄
items.forEach((el, i) => {
  el.style.width = `${widths[i] + 10}px`;                    // 쓰기 일괄
});
```

**스래싱을 줄이는 방법**

- 읽기 구간과 쓰기 구간을 분리하고, 쓰기는 `requestAnimationFrame` 콜백으로 미룸
- 여러 스타일 변경은 개별 `style.xxx` 대신 **클래스 토글** 한 번으로 처리함
- 많은 노드를 추가할 때는 `DocumentFragment`에 모아서 한 번에 삽입함
- 대량 변경 전 `display: none`으로 떼어냈다가 되돌리면 중간 변경이 레이아웃에 반영되지 않음

<br>

### 6. 영향 범위 제한 — contain과 content-visibility

브라우저는 변경이 트리 어디까지 영향을 주는지 알 수 없어 보수적으로 넓게 다시 계산한다. CSS `contain` 속성은 개발자가 "이 요소의 내부 변경은 바깥에 영향이 없다"고 **선언**해서 범위를 좁히는 수단이다.

| **값**                  | **의미**                                                        | **효과**                                   |
| ----------------------- | --------------------------------------------------------------- | ------------------------------------------ |
| **contain: layout**     | 내부 레이아웃이 외부에 영향을 주지 않음                         | 내부 Reflow가 바깥으로 전파되지 않음       |
| **contain: paint**      | 내부 콘텐츠가 요소 경계 밖으로 그려지지 않음                    | 페인트 영역을 요소 박스로 제한             |
| **contain: strict**     | `size layout paint style`을 한꺼번에 적용                       | 가장 강한 격리, 크기를 직접 지정해야 함    |
| **content-visibility: auto** | 뷰포트 밖 요소의 렌더링을 **건너뜀**                       | 긴 목록·피드에서 초기 렌더 비용 대폭 절감  |

```css
/* 카드 하나의 내부 변경이 목록 전체 Reflow로 번지지 않게 격리 */
.feed-card {
  contain: layout paint;
}

/* 화면 밖 카드는 렌더링을 미루되, 스크롤바 계산용 예상 높이는 알려줌 */
.feed-card {
  content-visibility: auto;
  contain-intrinsic-size: auto 320px;
}
```

> ⚠️ `content-visibility: auto`는 요소가 뷰포트에 들어올 때 비로소 레이아웃을 수행하므로, `contain-intrinsic-size`를 지정하지 않으면 스크롤 도중 높이가 갑자기 바뀌어 **레이아웃 이동(CLS)**이 발생한다. 성능 지표와의 관계는 unit10을 참고할 것.

<br>

### 7. 측정과 진단

- **Chrome DevTools → Performance**: 녹화 후 보라색 **Layout**·초록색 **Paint** 블록의 길이와 횟수를 확인함. "Forced reflow is a likely performance bottleneck" 경고가 강제 동기 레이아웃 지점을 알려줌
- **Rendering 탭 → Paint flashing**: 다시 그려지는 영역이 초록색으로 깜빡임. 의도하지 않은 영역이 깜빡이면 페인트 범위를 줄여야 함
- **Rendering 탭 → Layer borders**: 합성 레이어 경계를 주황색으로 표시함. 레이어가 지나치게 많으면 메모리 낭비를 의심함
- **Layers 패널**: 레이어별 메모리 사용량과 분리 이유(compositing reason)를 확인함

<br>

### 8. 정리

| **변경 대상**                         | **재실행 단계**            | **상대 비용** | **권장 대안**                         |
| ------------------------------------- | -------------------------- | ------------- | ------------------------------------- |
| **width·height·margin·top·left**      | 레이아웃 → 페인트 → 합성   | 높음          | `transform`으로 이동·확대             |
| **DOM 추가·삭제, 텍스트 변경**        | 레이아웃 → 페인트 → 합성   | 높음          | `DocumentFragment`, 일괄 처리         |
| **color·background·box-shadow**       | 페인트 → 합성              | 중간          | 페인트 영역 최소화, 레이어 분리       |
| **transform·opacity**                 | 합성                       | 낮음          | 애니메이션에 우선 사용                |

- **Reflow**는 기하 정보 재계산으로 트리 전체에 연쇄되며, **Repaint**는 픽셀만 다시 그림
- `transform`·`opacity`는 이미 래스터화된 레이어의 **배치·투명도만 바꾸므로** 합성만 발생함
- 기하 정보를 **읽는 것만으로도** 강제 동기 레이아웃이 발생하며, 읽기·쓰기 교차가 **레이아웃 스래싱**을 만듦
- `contain`·`content-visibility`로 변경의 **영향 범위를 선언적으로 제한**할 수 있음
- 이벤트 루프에서 스타일 변경이 언제 화면에 반영되는지는 unit03을 참고할 것
