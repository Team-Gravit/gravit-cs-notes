## 컴포넌트 설계

좋은 컴포넌트는 **재사용하기 쉽고, 변경에 강하며, 무엇을 하는지 이름만 봐도 드러난다**. 이 유닛은 React가 상속 대신 합성을 권장하는 이유, 제어·비제어 컴포넌트의 선택 기준, 그리고 반복 로직을 커스텀 훅으로 추출하는 기준을 다룬다.

<br>

### 1. 합성(Composition) vs 상속(Inheritance)

React 공식 문서는 "컴포넌트 간 코드 재사용에 상속 계층을 쓸 이유를 찾지 못했다"고 명시한다. React에서 재사용의 단위는 **props와 children을 통한 합성**이다.

| **항목**             | **상속** (`class Modal extends Dialog`)         | **합성** (`<Dialog><ModalBody/></Dialog>`)             |
| -------------------- | ----------------------------------------------- | ------------------------------------------------------ |
| **재사용 방식**      | 부모 클래스의 동작을 물려받고 오버라이드        | 작은 컴포넌트를 조립                                   |
| **결합도**           | 높음 — 부모 내부 구현에 의존                    | 낮음 — 인터페이스(props)만 의존                        |
| **변형 추가 비용**   | 계층이 깊어지고 다이아몬드 문제                 | 새 조합을 만들면 됨                                    |
| **React에서의 적합성** | 함수 컴포넌트·훅과 맞지 않음                  | **권장**                                               |

### 1-1. children으로 구멍 뚫기

```tsx
// 안티패턴: 변형마다 props가 늘어남 → 조합 폭발
<Card title="공지" showIcon iconType="warning" footerText="확인" footerAlign="right" />

// 개선: 내용은 children·slot으로 받고, Card는 틀만 책임짐
<Card>
  <Card.Header><WarningIcon /> 공지</Card.Header>
  <Card.Body>내용</Card.Body>
  <Card.Footer align="right"><Button>확인</Button></Card.Footer>
</Card>
```

### 1-2. 특수화는 합성으로

```tsx
// "WelcomeDialog는 Dialog의 특수한 경우" → 상속이 아니라 Dialog를 감싸는 컴포넌트
function Dialog({ title, children }: { title: string; children: React.ReactNode }) {
  return <section className="dialog"><h2>{title}</h2>{children}</section>;
}
function WelcomeDialog() {
  return <Dialog title="환영합니다"><p>가입을 축하합니다.</p></Dialog>;
}
```

> 💡 "children을 받는 컴포넌트는 리렌더 최적화에도 유리하다" — 부모가 만든 children 엘리먼트는 부모가 리렌더되지 않는 한 같은 참조이므로 React가 건너뛴다(unit06 2-2절). 합성은 설계와 성능을 동시에 얻는 방법이다.

<br>

### 2. 컴포넌트 분리 기준

```
이 JSX 덩어리를 컴포넌트로 뺄 것인가?
 ├─ 두 곳 이상에서 같은 구조를 쓰는가?              → 예 → 분리 (재사용)
 ├─ 자체 상태·이펙트를 가지는가?                    → 예 → 분리 (관심사 격리, 리렌더 범위 축소)
 ├─ 이름을 붙이면 부모의 JSX가 읽기 쉬워지는가?     → 예 → 분리 (가독성)
 └─ 위 모두 아니고 한 번만 쓰이는 짧은 조각인가?    → 예 → 분리하지 않음 (과도한 추상화)
```

- 분리 기준은 "줄 수"가 아니라 **책임**임. 100줄이어도 한 가지 일을 하면 괜찮고, 20줄이어도 상태·이펙트·표시가 뒤섞이면 나눔
- **표현(presentational)과 로직(container)의 분리**는 이제 컴포넌트 두 개가 아니라 **"컴포넌트 + 커스텀 훅"** 형태로 하는 것이 일반적임(4절)

<br>

### 3. 제어 컴포넌트 vs 비제어 컴포넌트

**제어(Controlled)** 컴포넌트는 값을 **부모의 state**가 소유하고 props로 내려받는다. **비제어(Uncontrolled)** 컴포넌트는 값을 **자기 내부(DOM 또는 지역 state)**가 소유한다.

```tsx
// 제어: 값의 진실이 React state에 있음. 매 입력마다 리렌더
function Controlled() {
  const [name, setName] = useState("");
  return <input value={name} onChange={(e) => setName(e.target.value)} />;
}

// 비제어: 값의 진실이 DOM에 있음. 필요할 때 ref로 읽음 (unit07)
function Uncontrolled() {
  const ref = useRef<HTMLInputElement>(null);
  return (
    <form onSubmit={(e) => { e.preventDefault(); submit(ref.current!.value); }}>
      <input ref={ref} defaultValue="" />
    </form>
  );
}
```

| **항목**                | **제어 컴포넌트**                                     | **비제어 컴포넌트**                                  |
| ----------------------- | ----------------------------------------------------- | ---------------------------------------------------- |
| **값의 소유자**         | 부모 state                                            | DOM / 컴포넌트 내부                                  |
| **props**               | `value` + `onChange`                                  | `defaultValue` (+ ref)                               |
| **실시간 검증·포맷팅**  | 쉬움 (매 입력마다 값을 가로챔)                        | 어려움                                               |
| **여러 입력의 연동**    | 쉬움 (state로 계산)                                   | 어려움                                               |
| **렌더 비용**           | 매 입력마다 리렌더                                    | 없음                                                 |
| **외부에서 값 리셋**    | `setState`                                            | `key` 변경 또는 ref로 직접 조작                      |
| **적합한 경우**         | 검증·마스킹·연동이 필요한 폼, 부모가 값을 알아야 할 때 | 단순 폼, 파일 입력, 대규모 폼의 성능 문제, 외부 폼 라이브러리 |

> ⚠️ 하나의 입력에서 `value`와 `defaultValue`를 함께 쓰거나, 처음엔 `value={undefined}`였다가 나중에 문자열을 넣으면 "비제어에서 제어로 전환" 경고가 난다. 제어로 쓸 거라면 초기값을 `""`처럼 **항상 정의된 값**으로 준다.

### 3-1. 재사용 컴포넌트는 둘 다 지원한다

라이브러리성 컴포넌트(`Select`, `Tabs`, `Accordion`)는 사용자가 상황에 따라 제어·비제어를 고를 수 있도록 **둘 다 지원**하는 경우가 많다.

```tsx
function useControllableState<T>(value: T | undefined, defaultValue: T, onChange?: (v: T) => void) {
  const [internal, setInternal] = useState(defaultValue);
  const isControlled = value !== undefined;
  const current = isControlled ? value : internal;
  const set = (next: T) => {
    if (!isControlled) setInternal(next);  // 비제어면 내부 갱신
    onChange?.(next);                      // 제어면 부모에게 요청만
  };
  return [current, set] as const;
}

function Tabs({ value, defaultValue = "a", onChange }: TabsProps) {
  const [active, setActive] = useControllableState(value, defaultValue, onChange);
  // ...
}
```

<br>

### 4. 커스텀 훅 추출 기준

**커스텀 훅**은 `use`로 시작하는 함수로, 내부에서 다른 훅을 호출해 **상태 로직**을 재사용한다. UI를 재사용하는 것이 아니라 **동작**을 재사용하며, 호출한 컴포넌트마다 **독립적인 상태**를 가진다.

### 4-1. 추출해야 하는 신호

| **신호**                                                  | **예시 훅**                              |
| --------------------------------------------------------- | ---------------------------------------- |
| **같은 useState+useEffect 조합이 두 컴포넌트 이상에 있음** | `useWindowSize`, `useOnlineStatus`       |
| **이펙트의 의도가 코드에서 안 보임 (설명 주석이 필요함)** | `useChatRoom(roomId)`                    |
| **컴포넌트가 "무엇을 보여줄지"보다 "어떻게 얻을지"로 길어짐** | `useUserProfile(id)`, `usePagination` |
| **외부 시스템 연동(소켓·스토리지·타이머)이 컴포넌트에 노출됨** | `useLocalStorage`, `useInterval`    |

```tsx
// 안티패턴: 온라인 상태 감지 로직이 컴포넌트마다 복붙됨
function StatusBar() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  useEffect(() => {
    const on = () => setIsOnline(true), off = () => setIsOnline(false);
    window.addEventListener("online", on); window.addEventListener("offline", off);
    return () => { window.removeEventListener("online", on); window.removeEventListener("offline", off); };
  }, []);
  return <span>{isOnline ? "온라인" : "오프라인"}</span>;
}
```

```tsx
// 개선: 동작을 훅으로, 컴포넌트는 표시만
function useOnlineStatus() {
  return useSyncExternalStore(
    (cb) => { window.addEventListener("online", cb); window.addEventListener("offline", cb);
              return () => { window.removeEventListener("online", cb); window.removeEventListener("offline", cb); }; },
    () => navigator.onLine,
    () => true,  // 서버 스냅샷
  );
}
function StatusBar() {
  const isOnline = useOnlineStatus();
  return <span>{isOnline ? "온라인" : "오프라인"}</span>;
}
```

### 4-2. 추출하지 말아야 하는 경우

- **훅을 호출하지 않는 함수**: 순수 계산이면 그냥 일반 함수(`formatDate`)로 둔다. `use` 접두사는 훅의 규칙(unit02)을 적용받게 만드므로 필요 없을 때 붙이면 오히려 제약만 생김
- **`useMount`, `useUpdateEffect` 같은 생명주기 래퍼**: 이펙트의 "동기화" 의미(unit05)를 가리고 의존성 린트를 우회하게 만듦
- **한 곳에서만 쓰이는데 "혹시 몰라서"**: 훅 이름이 구체적 도메인(`useCheckoutForm`)이면 괜찮지만, 추상적 이름으로 미리 일반화하면 과도한 추상화가 됨

> 💡 커스텀 훅의 이름은 **"무엇을 하는가"**를 담아야 한다. `useData`보다 `useProductSearch(query)`가 낫다. 훅이 반환하는 값도 배열(`[value, setValue]`)은 두 개 이하일 때, 그 이상이면 객체(`{ data, isPending, refetch }`)로 돌려주는 것이 관례다.

<br>

### 5. 설계 패턴 비교

| **패턴**                    | **재사용 대상**  | **형태**                                       | **현재 위치**                                    |
| --------------------------- | ---------------- | ---------------------------------------------- | ------------------------------------------------ |
| **고차 컴포넌트(HOC)**      | 로직 + props 주입 | `withAuth(Component)`                         | 레거시. 래퍼 지옥·props 출처 불명. 훅으로 대체    |
| **렌더 props**              | 로직             | `<Mouse render={(pos) => …} />`                | 훅 이전의 로직 공유. 지금은 훅이 대체             |
| **커스텀 훅**               | 로직(상태·이펙트) | `const pos = useMouse()`                      | **기본 선택**                                    |
| **합성 컴포넌트(Compound)** | UI 구조          | `<Tabs><Tabs.List/><Tabs.Panel/></Tabs>`       | 관련 UI 묶음의 유연한 조립. 내부는 Context 사용   |
| **헤드리스 컴포넌트**       | 동작 + 접근성    | `useSelect()` → 마크업은 사용자가             | 디자인 시스템·UI 라이브러리(Radix, Headless UI 등) |

- 로직 재사용 → **커스텀 훅**, 구조 재사용 → **합성/컴파운드**, 동작만 주고 마크업은 위임 → **헤드리스**
- HOC와 렌더 props를 아는 것은 레거시 코드를 읽기 위해서이고, 새 코드에서는 훅과 합성으로 표현함

<br>

### 6. 컴포넌트 설계 체크리스트

```
□ props 이름이 "무엇을 하는가"가 아니라 "무엇인가"를 말하는가? (onClick보다 onSelect, isOpen보다 open)
□ boolean props가 셋 이상 쌓이면 variant·children·컴파운드로 바꿀 수 있는가?
□ 값의 소유자가 명확한가? (제어면 value+onChange, 비제어면 defaultValue)
□ 상태·이펙트 로직이 표시 코드와 섞여 있다면 커스텀 훅으로 뺄 수 있는가?
□ 컴포넌트가 내부 DOM 구조를 부모에게 노출하는가? (필요하면 useImperativeHandle, unit07)
□ 이 컴포넌트를 삭제해도 다른 컴포넌트가 깨지지 않는가? (결합도)
```

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                       |
| ----------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **React는 왜 상속 대신 합성을 권하는가?**             | props·children으로 조립하면 결합도가 낮고 변형 추가가 쉬움. 상속은 부모 구현에 묶이고 훅과 맞지 않음     |
| **제어·비제어 컴포넌트의 차이와 선택 기준은?**        | 값의 소유자가 React state인지 DOM인지. 실시간 검증·연동이 필요하면 제어, 단순·성능 민감·파일 입력이면 비제어 |
| **커스텀 훅은 언제 추출하는가?**                      | 같은 상태+이펙트 조합이 반복되거나, 컴포넌트가 "어떻게 얻는지"로 길어지거나, 외부 시스템 연동이 노출될 때   |
| **커스텀 훅끼리 상태를 공유하는가?**                  | 아니오. 훅은 로직을 공유하고 상태는 호출한 컴포넌트마다 독립적. 공유하려면 Context·스토어(unit04)       |
| **HOC와 훅의 차이는?**                                | HOC는 컴포넌트를 감싸 props를 주입(출처 불명·래퍼 중첩), 훅은 컴포넌트 안에서 명시적으로 호출              |
| **컴파운드 컴포넌트란?**                              | `Tabs`·`Tabs.List`처럼 관련 컴포넌트를 묶어 사용자가 구조를 조립하게 하고, 내부 상태는 Context로 공유       |

- 재사용은 **로직 → 훅, 구조 → 합성**으로 나눠 생각한다
- 컴포넌트 API는 **값의 소유자**(제어/비제어)와 **책임 범위**(무엇을 보여주는가)를 명확히 드러내야 한다
- 상태의 위치 결정은 **unit03**, ref와 명령형 API 노출은 **unit07**을 참고할 것
