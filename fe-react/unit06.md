## 렌더링 최적화 판단

`React.memo`·`useMemo`·`useCallback`은 리렌더 비용을 줄이는 도구지만, **잘못 쓰면 비용만 늘고 효과는 없다**. 이 유닛은 세 도구의 정확한 동작, 오남용 기준, 그리고 "추측이 아니라 프로파일러로 검증"하는 절차를 다룬다.

<br>

### 1. 최적화 전에 확인할 것

React는 기본적으로 **충분히 빠르다**. 대부분의 리렌더는 수 밀리초 안에 끝나고, 사용자는 16ms(60fps) 안에 끝나는 작업을 느끼지 못한다. 최적화는 다음 두 조건이 모두 성립할 때 의미가 있다.

- 리렌더가 **실제로 느리다** — 프로파일러에서 특정 커밋이 수십 ms 이상
- 그 리렌더가 **불필요하다** — props·state가 사실상 같은데 다시 그려짐

```
① 느리다고 "느껴진다"        → 프로파일러로 측정 (3절)
② 느린 커밋의 원인 컴포넌트 식별 → 왜 렌더됐는지 확인 (부모? props? state? context?)
③ 구조적 해결이 가능한가?      → 상태 내리기 / children으로 분리 (2절)
④ 그래도 남으면 memo 계열 적용  → 적용 후 다시 측정해 효과 확인
```

> 💡 "성능 최적화 어떻게 하셨어요?"에 "useMemo·useCallback을 썼다"고만 답하면 약하다. **무엇을 측정했고, 원인이 무엇이었고, 어떤 대안을 비교한 뒤 왜 그 방법을 택했는지**를 말해야 한다.

<br>

### 2. 메모이제이션 전에 구조를 먼저 본다

리렌더의 원인은 대부분 "상태가 너무 위에 있어서" 넓은 서브트리가 함께 다시 그려지는 것이다. 이 경우 메모 훅보다 **컴포넌트 구조 변경**이 근본적이고 비용도 없다.

### 2-1. 상태를 아래로 내리기

```tsx
// 안티패턴: 입력값 하나 때문에 무거운 목록까지 매 키 입력마다 리렌더
function Page() {
  const [query, setQuery] = useState("");
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <HeavyList />   {/* query와 무관하지만 Page가 리렌더되어 함께 렌더됨 */}
    </>
  );
}
```

```tsx
// 개선: 입력 상태를 쓰는 부분만 별도 컴포넌트로 → HeavyList는 리렌더되지 않음
function Page() {
  return (
    <>
      <SearchInput />
      <HeavyList />
    </>
  );
}
function SearchInput() {
  const [query, setQuery] = useState("");
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

### 2-2. children으로 분리하기

상태를 가진 컴포넌트가 무거운 자식을 **감싸야만** 하는 경우, 자식을 `children`으로 받으면 부모가 리렌더돼도 `children` 엘리먼트는 **같은 참조**이므로 React가 건너뛴다.

```tsx
function ScrollTracker({ children }: { children: React.ReactNode }) {
  const [y, setY] = useState(0);
  useEffect(() => { /* 스크롤 구독 → setY */ }, []);
  return <div style={{ opacity: y > 100 ? 0.5 : 1 }}>{children}</div>;
}

// <ScrollTracker><HeavyList /></ScrollTracker>
// ScrollTracker가 setY로 리렌더돼도 HeavyList 엘리먼트는 App이 만든 그대로 → 렌더 생략
```

<br>

### 3. 세 도구의 정확한 동작

| **도구**            | **무엇을 기억하는가**      | **비교 기준**                          | **막는 것**                                     |
| ------------------- | -------------------------- | -------------------------------------- | ----------------------------------------------- |
| **`React.memo`**    | 컴포넌트의 마지막 렌더 결과 | props 얕은 비교(`Object.is`, 필드별)   | **부모 리렌더로 인한** 자식 리렌더              |
| **`useMemo`**       | 계산 결과 값                | 의존성 배열 얕은 비교                  | 비싼 계산 반복, 객체·배열 참조 변경             |
| **`useCallback`**   | 함수 참조                  | 의존성 배열 얕은 비교                  | 함수 참조 변경 (= `useMemo(() => fn, deps)`)   |

- `memo`는 **자신의 state·context 변경**으로 인한 리렌더는 막지 못함. 오직 "부모가 리렌더됐지만 props는 같다"는 경우만 건너뜀
- `useMemo`·`useCallback`은 **그 자체로는 리렌더를 막지 않음**. 참조를 고정해 `memo`된 자식이나 이펙트 의존성이 "같다"고 판단하게 만드는 보조 도구임
- 셋 다 **보장이 아니라 힌트**임. React는 메모리 압박 등으로 캐시를 버릴 수 있으므로 정확성이 메모에 의존하면 안 됨

```tsx
// memo + useCallback이 함께 있어야 효과가 나는 예
const Row = memo(function Row({ item, onSelect }: RowProps) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});

function List({ items }: { items: Item[] }) {
  const [selected, setSelected] = useState<number | null>(null);
  const handleSelect = useCallback((id: number) => setSelected(id), []);  // 이게 없으면 매 렌더 새 함수 → memo 무력화
  return <ul>{items.map((i) => <Row key={i.id} item={i} onSelect={handleSelect} />)}</ul>;
}
```

<br>

### 4. 오남용 기준

### 4-1. 효과가 없는 경우

- **렌더가 매우 싼 컴포넌트에 memo**: `<li>{name}</li>` 수준의 렌더는 props 비교 비용과 큰 차이가 없어 이득이 없고, 코드만 복잡해짐
- **props 중 하나라도 매번 새 참조(인라인 객체·배열·함수·children JSX)**인데 memo: 얕은 비교가 항상 실패해 memo 비용만 추가됨
- **싼 계산에 useMemo**: `a + b`, 짧은 배열 `filter` 같은 계산은 메모 오버헤드(의존성 비교, 클로저 생성, 캐시 저장)가 계산보다 큼
- **자식이 memo가 아니고 이펙트 의존성에도 안 쓰이는 함수에 useCallback**: 함수 참조가 안정적이어도 아무도 그 안정성을 활용하지 않음

### 4-2. 효과가 있는 경우

| **상황**                                                   | **도구**                            |
| ---------------------------------------------------------- | ----------------------------------- |
| **부모가 자주 리렌더되고 자식 렌더가 비싸며 props가 대체로 같음** | `memo` (+ 참조 props는 `useMemo`/`useCallback`) |
| **수만 개 항목 정렬·필터 등 눈에 띄게 비싼 계산**          | `useMemo`                           |
| **객체·배열을 memo된 자식 props나 이펙트 의존성으로 전달** | `useMemo`                           |
| **함수를 memo된 자식 props나 이펙트 의존성으로 전달**      | `useCallback`                       |
| **Context value 객체 안정화(unit04)**                      | `useMemo`                           |

```tsx
// 안티패턴: 메모가 무의미한 예
const label = useMemo(() => `${first} ${last}`, [first, last]);   // 문자열 결합은 메모보다 싸다
const style = useMemo(() => ({ color }), [color]);                // 받는 자식이 memo가 아니면 의미 없음
const onClick = useCallback(() => setOpen(true), []);             // <button>에 넘기는 함수는 안정성이 필요 없음
```

> ⚠️ "일단 다 감싸면 손해는 없겠지"는 틀렸다. 모든 메모는 **의존성 비교 + 캐시 저장 메모리 + 코드 가독성**이라는 비용을 지불하며, 의존성 배열을 잘못 적으면 낡은 값을 보여주는 **정확성 버그**까지 만든다. 측정 없이 적용된 메모는 부채다.

<br>

### 5. 프로파일러로 검증하기

React DevTools의 **Profiler** 탭은 "어떤 컴포넌트가, 얼마나 오래, 왜 렌더됐는지"를 커밋 단위로 보여준다.

```
Profiler 사용 절차
① 설정(⚙) → "Record why each component rendered while profiling" 켜기
② 녹화(●) 시작 → 느리다고 의심되는 상호작용 재현 → 녹화 중지
③ Flamegraph: 커밋별로 렌더된 컴포넌트와 소요 시간(ms). 회색은 렌더 안 됨
④ Ranked: 이번 커밋에서 오래 걸린 순서
⑤ 컴포넌트 클릭 → "Why did this render?" (props 변경 / state 변경 / 부모 렌더 / hooks 변경 / context 변경)
```

| **"Why did this render?" 결과**      | **의미**                                   | **대응**                                                 |
| ------------------------------------ | ------------------------------------------ | -------------------------------------------------------- |
| **The parent component rendered**    | props는 같은데 부모 때문에 렌더됨          | 상태 내리기·children 분리 → 남으면 `memo`                |
| **Props changed: (onClick, style)**  | 참조형 props가 매번 새로 생김              | 부모에서 `useCallback`/`useMemo`, 또는 인라인 제거       |
| **Hooks changed**                    | 자신의 state·reducer 변경                  | 정당한 렌더. 렌더 자체가 느리면 계산을 `useMemo`         |
| **Context changed**                  | 구독한 Context value 변경                  | Context 분리·value 안정화(unit04), 외부 스토어           |

- 프로덕션 빌드는 개발 빌드보다 훨씬 빠르므로, 최종 판단은 **프로덕션 프로파일링 빌드**(`react-dom/profiling`) 또는 실제 배포 환경에서 함
- 코드로 측정하려면 `<Profiler id="List" onRender={(id, phase, actualDuration) => …}>`로 감싸 렌더 시간을 수집할 수 있음

> 💡 개발 모드에서 "Highlight updates" 깜빡임이 많다고 곧바로 문제로 단정하면 안 된다. **깜빡임 = 렌더 발생**일 뿐이며, 그 렌더가 느린지(ms)와 불필요한지(why)는 Profiler로 확인해야 한다.

<br>

### 6. React Compiler와 자동 메모이제이션

React 19와 함께 공개된 **React Compiler**는 빌드 시점에 컴포넌트와 훅을 분석해 `useMemo`·`useCallback`·`memo`에 해당하는 최적화를 **자동으로** 삽입한다.

- 훅의 규칙과 렌더 순수성을 지킨 코드에만 적용됨. 규칙 위반이 감지된 컴포넌트는 건너뜀
- 수동 메모 코드는 대부분 제거할 수 있지만, 컴파일러 도입 여부와 세부 동작은 **프로젝트 설정·버전에 따라 다를 수 있음**
- 컴파일러가 있어도 "상태를 어디에 둘지"(unit03)와 "느린 계산 자체"는 해결해 주지 않으므로 구조적 최적화는 여전히 개발자의 몫임

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                   |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **memo·useMemo·useCallback의 차이는?**                | memo는 컴포넌트 렌더 결과, useMemo는 값, useCallback은 함수 참조를 기억. 뒤의 둘은 memo를 돕는 보조 도구 |
| **useCallback을 쓰면 항상 빨라지는가?**               | 아니오. 받는 쪽이 memo이거나 이펙트 의존성일 때만 의미가 있고, 그 외엔 비교·메모리 비용만 발생       |
| **memo가 효과 없는 대표 상황은?**                     | 인라인 객체·함수·children JSX가 props로 들어와 얕은 비교가 항상 실패할 때                           |
| **최적화 순서는?**                                    | 측정 → 원인 식별 → 상태 내리기·children 분리 등 구조 개선 → 남으면 메모 → 재측정                    |
| **프로파일러에서 무엇을 보는가?**                     | 커밋별 렌더 시간(ms)과 "why did this render"로 불필요한 렌더의 원인을 분류                            |

- 메모이제이션은 **측정된 병목**에만, **구조 개선 이후**에 적용한다
- 메모는 힌트이지 보장이 아니므로 **정확성이 메모에 의존하지 않게** 작성한다
- 리렌더 트리거의 원리는 **unit01**, Context 리렌더 문제는 **unit04**를 참고할 것
