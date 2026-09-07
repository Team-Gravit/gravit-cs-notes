## 상태 설계

React 애플리케이션의 버그와 복잡도는 대부분 **상태를 어디에, 어떤 형태로 둘지** 잘못 결정한 데서 시작된다. 이 유닛은 상태의 위치를 정하는 원칙, 파생 상태를 별도 상태로 두지 않는 이유, 상태 끌어올리기의 기준을 다룬다.

<br>

### 1. 상태란 무엇이고 무엇이 아닌가

**상태(State)**는 "시간에 따라 바뀌며, 바뀌면 화면이 달라져야 하는 값"이다. 상태를 추가하기 전에 다음 질문으로 걸러낸다.

| **질문**                                              | **예**                        | **결론**                                    |
| ----------------------------------------------------- | ----------------------------- | ------------------------------------------- |
| **시간이 지나도 변하지 않는가?**                      | 설정 상수, 라우트 파라미터    | 상태가 아님 — props·상수로 둠               |
| **부모에게서 props로 받는가?**                        | `user`, `items`               | 상태가 아님 — 복사하지 말고 그대로 사용     |
| **다른 상태·props로 계산할 수 있는가?**               | `total`, `filteredList`       | 상태가 아님 — 렌더 중 계산(파생 값)         |
| **바뀌어도 화면이 그대로인가?**                       | 타이머 id, 이전 스크롤 위치   | 상태가 아님 — `useRef`(unit07)              |
| **위 모두 아니다**                                    | 입력값, 열림 여부, 선택된 탭  | **상태**                                    |

> 💡 "최소한의 상태(minimal state)"가 원칙이다. 상태가 하나 늘 때마다 그 상태를 **동기화해야 할 다른 값**이 함께 늘어나고, 동기화가 어긋나는 순간이 곧 버그다.

<br>

### 2. 파생 상태를 지양하는 이유

**파생 상태(Derived State)**란 다른 상태나 props에서 계산할 수 있는 값을 별도 `useState`에 저장한 것이다. 원본과 사본이 생기면서 **단일 진실 공급원(Single Source of Truth)**이 깨진다.

```tsx
// 안티패턴: items에서 계산 가능한 total을 상태로 중복 보관
function Cart({ items }: { items: Item[] }) {
  const [total, setTotal] = useState(0);
  useEffect(() => {
    setTotal(items.reduce((sum, i) => sum + i.price, 0));
  }, [items]);  // items → 렌더 → 이펙트 → setTotal → 또 렌더 (2회 렌더, 잠깐 낡은 total 노출)
  return <p>{total}</p>;
}
```

```tsx
// 개선: 렌더 중 계산. 항상 items와 일치하며 렌더도 1회
function Cart({ items }: { items: Item[] }) {
  const total = items.reduce((sum, i) => sum + i.price, 0);
  return <p>{total}</p>;
}
```

```
파생 상태 보관 시 데이터 흐름              렌더 중 계산 시 데이터 흐름
items 변경 → 렌더(낡은 total) → 이펙트     items 변경 → 렌더(정확한 total)
          → setTotal → 렌더(새 total)
      원본과 사본이 잠시 불일치                항상 일치, 동기화 코드 없음
```

- 계산 비용이 눈에 띄게 크면 `useMemo`로 감싼다(unit06). 상태로 승격시키는 것이 아님
- "props를 `useState` 초기값으로 복사"하는 것도 파생 상태다. 초기값은 **마운트 시 한 번만** 반영되므로 이후 props 변경이 무시됨

> ⚠️ `useState(props.value)`는 props의 **초기 스냅샷**만 사용한다. 부모가 value를 바꿔도 자식 상태는 그대로다. 정말 "초기값만 받고 이후는 독립적으로 관리"하려는 의도라면 이름을 `initialValue`·`defaultValue`로 지어 의도를 드러내고, 리셋이 필요하면 `key`를 바꾼다(unit01 참고).

<br>

### 3. 상태의 위치 결정

### 3-1. 원칙 — 필요한 컴포넌트들의 가장 가까운 공통 조상

상태는 **그 상태를 읽거나 쓰는 모든 컴포넌트의 가장 가까운 공통 조상**에 둔다. 더 위로 올리면 관계없는 컴포넌트까지 리렌더되고, 더 아래에 두면 형제가 공유할 수 없다.

```
                App
               /   \
        Header      Main            ← 검색어를 SearchBar와 ResultList가 함께 쓴다면
                   /    \             상태는 둘의 공통 조상인 Main에 둔다 (App까지 올릴 필요 없음)
           SearchBar  ResultList
```

### 3-2. 지역 상태를 기본값으로

- 한 컴포넌트만 쓰는 상태(입력 중인 값, 툴팁 열림 여부)는 **그 컴포넌트 안**에 둔다
- "나중에 공유할지도 몰라서" 미리 올리지 않는다. 필요해지면 그때 올리는 비용은 낮고, 미리 올린 상태는 불필요한 리렌더와 결합도를 만든다

<br>

### 4. 상태 끌어올리기(Lifting State Up)

### 4-1. 언제 올리는가

| **신호**                                                    | **판단**                                    |
| ----------------------------------------------------------- | ------------------------------------------- |
| **두 형제 컴포넌트가 같은 값을 보여주거나 바꿔야 함**       | 공통 부모로 올림                            |
| **한 컴포넌트의 변화가 다른 컴포넌트의 표시에 영향을 줌**   | 공통 부모로 올림                            |
| **컴포넌트 밖(부모)에서 값을 읽거나 리셋해야 함**           | 부모로 올려 제어 컴포넌트로 만듦(unit10)    |
| **한 컴포넌트만 쓰는데 "혹시 몰라서"**                      | 올리지 않음                                 |

### 4-2. 끌어올리기 예시 — 아코디언

```tsx
// 안티패턴: 각 패널이 자기 열림 상태를 가짐 → "하나만 열리기"를 구현할 수 없음
function Panel({ title, children }: PanelProps) {
  const [open, setOpen] = useState(false);
  return (
    <section>
      <button onClick={() => setOpen(!open)}>{title}</button>
      {open && children}
    </section>
  );
}
```

```tsx
// 개선: 열린 패널의 index를 부모가 소유하고, 패널은 props로 받음
function Accordion() {
  const [activeIndex, setActiveIndex] = useState(0);
  return (
    <>
      <Panel title="A" isOpen={activeIndex === 0} onOpen={() => setActiveIndex(0)} />
      <Panel title="B" isOpen={activeIndex === 1} onOpen={() => setActiveIndex(1)} />
    </>
  );
}

function Panel({ title, isOpen, onOpen }: { title: string; isOpen: boolean; onOpen: () => void }) {
  return (
    <section>
      <button onClick={onOpen}>{title}</button>
      {isOpen && <p>내용</p>}
    </section>
  );
}
```

- 올린 뒤 자식은 상태를 **소유하지 않고**, `isOpen`(값)과 `onOpen`(변경 요청)만 받는 **제어 컴포넌트**가 됨
- 두 패널의 `open`을 각각 유지하는 대신 `activeIndex` 하나로 표현해 **"동시에 두 개가 열림"이라는 불가능한 상태**를 타입 차원에서 제거함

<br>

### 5. 상태의 형태(Shape) 설계 원칙

- **모순 가능한 상태를 없앤다**: `isLoading`·`isError`·`isSuccess` 세 boolean 대신 `status: "idle" | "loading" | "error" | "success"` 하나로 표현
- **중복을 없앤다**: 선택된 항목 객체 전체(`selectedItem`)가 아니라 `selectedId`만 저장하고 렌더 중 `items.find`로 찾음. 항목이 수정돼도 선택 정보가 낡지 않음
- **깊은 중첩을 피한다**: 트리 구조는 `{ [id]: node }` 형태로 **정규화(normalize)**하면 갱신 코드가 단순해짐
- **함께 바뀌는 값은 묶는다**: `x`, `y`를 따로 두면 한쪽만 갱신하는 실수가 생김. `position: { x, y }` 또는 `useReducer`

```tsx
// 안티패턴: 불가능한 조합(isLoading && isError)이 표현 가능
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [data, setData] = useState<User | null>(null);

// 개선: 판별 유니온으로 가능한 상태만 남김
type Fetch =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "error"; error: Error }
  | { status: "success"; data: User };
const [state, setState] = useState<Fetch>({ status: "idle" });
```

> 💡 여러 상태가 **같은 이벤트에 의해 함께 전이**된다면 `useReducer`가 적합하다. "어떤 액션이 들어왔을 때 상태가 어떻게 바뀌는가"를 한 함수에 모아 두면 불가능한 조합이 생기는 경로 자체를 막을 수 있다.

<br>

### 6. 상태 위치·형태 결정 흐름

```
값이 필요하다
 ├─ 계산 가능한가? ── 예 → 렌더 중 계산 (필요시 useMemo)
 ├─ 화면과 무관한가? ─ 예 → useRef
 └─ 아니오 → 상태
      ├─ 누가 쓰는가?
      │    ├─ 한 컴포넌트 → 그 컴포넌트의 useState
      │    ├─ 형제 몇 개 → 가장 가까운 공통 조상으로 끌어올림
      │    └─ 트리 곳곳 → Context 또는 외부 스토어 (unit04)
      └─ 서버에서 온 데이터인가? → 서버 상태로 분리 (unit04)
```

<br>

### 7. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                                   |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **파생 상태를 useState에 넣으면 안 되는 이유는?** | 원본과 사본이 생겨 동기화 코드가 필요하고, 이펙트로 동기화하면 렌더가 2회 돌며 잠시 불일치가 노출됨 |
| **상태는 어디에 두는가?**                         | 그 상태를 쓰는 모든 컴포넌트의 **가장 가까운 공통 조상**. 기본은 지역 상태, 필요할 때만 올림       |
| **상태 끌어올리기의 기준은?**                     | 형제가 같은 값을 공유하거나 한쪽 변화가 다른 쪽 표시에 영향을 줄 때. "혹시 몰라서"는 기준이 아님    |
| **`useState(props.x)`의 문제는?**                 | 초기 스냅샷만 반영되어 이후 props 변경이 무시됨. 리셋이 필요하면 `key`를 바꿈                      |
| **boolean 여러 개 대신 무엇을 쓰는가?**           | `status` 문자열 유니온 또는 판별 유니온 타입으로 불가능한 조합을 제거                             |

- 상태를 추가하기 전에 **"계산할 수 있는가, props인가, 화면과 무관한가"**를 먼저 묻는다
- 상태 구조는 **불가능한 상태가 표현 불가능하도록** 설계한다
- 전역·서버 상태로의 확장은 **unit04**, 제어·비제어 컴포넌트는 **unit10**을 참고할 것
