## 동시성 렌더링

React 18에서 도입된 **동시성 렌더링(Concurrent Rendering)**은 렌더 작업을 **중단·재개·폐기할 수 있게** 만들어, 급한 갱신(입력)이 느린 갱신(대량 목록)에 가로막히지 않도록 한다. 이 유닛은 그 기반 위에 세워진 `Suspense`, `transition`, 스트리밍 SSR, 그리고 React 19의 `use` 훅을 다룬다.

<br>

### 1. 왜 동시성이 필요한가

React 17까지의 렌더는 **동기적이고 중단 불가**였다. 큰 트리를 렌더하는 동안 메인 스레드가 점유되어 입력·애니메이션이 멈췄다.

```
동기 렌더 (React 17)
키 입력 ──▶ [ 무거운 목록 렌더 300ms ─────────────────── ] ──▶ 입력창 갱신
                    ↑ 이 동안 타이핑이 화면에 안 보임 (버벅임)

동시성 렌더 (React 18+, createRoot)
키 입력 ──▶ [입력창 갱신 2ms] ──▶ [목록 렌더 일부] ─ 다음 키 입력 → 목록 렌더 폐기 ──▶ [입력창 갱신] ──▶ [목록 렌더 재시작] …
             긴급(urgent)        전환(transition) — 쪼개서 진행, 더 급한 일이 오면 양보
```

- 동시성은 **기능이 아니라 내부 메커니즘**임. 렌더 단계를 작은 단위로 나눠 실행하고, 사이사이에 브라우저에게 제어권을 돌려줌
- 렌더가 중단·재시도될 수 있으므로 **렌더 단계는 반드시 순수**해야 함(unit01). 렌더 중 부수효과가 있으면 폐기된 렌더의 효과가 남음
- `ReactDOM.render` 대신 **`createRoot`**를 써야 동시성 기능이 활성화됨. 그 외에는 자동으로 켜지지 않으며, `startTransition`·`Suspense` 같은 API를 쓸 때만 해당 갱신이 동시적으로 처리됨

> 💡 "동시성 = 멀티스레드"가 아니다. JS는 여전히 단일 스레드이며, React가 렌더 작업을 **협력적으로 양보(yield)**하는 스케줄링을 할 뿐이다. OS의 선점형 멀티태스킹이 아니라 **협력형 멀티태스킹**에 가깝다.

<br>

### 2. transition — 긴급하지 않은 갱신 표시

**transition**은 "이 상태 갱신은 급하지 않으니, 더 급한 갱신이 오면 양보하고 결과가 준비될 때까지 이전 화면을 유지해도 된다"고 React에 알리는 API다.

### 2-1. useTransition

```tsx
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setQuery(e.target.value);                     // 긴급: 입력창은 즉시 반영
    startTransition(() => {
      setResults(filterHeavy(e.target.value));    // 전환: 느려도 되고, 새 입력이 오면 폐기
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ul style={{ opacity: isPending ? 0.5 : 1 }}>{results.map((r) => <li key={r.id}>{r.name}</li>)}</ul>
    </>
  );
}
```

- `startTransition` 안의 setter는 **전환 갱신**으로 표시됨. 렌더가 백그라운드에서 진행되고 완료될 때까지 화면은 이전 결과를 유지
- `isPending`으로 진행 중임을 표시할 수 있음
- React 19부터 `startTransition`에 **async 함수**를 넘길 수 있어, 서버 요청 → 상태 갱신 흐름 전체를 하나의 transition으로 묶을 수 있음(`useActionState`·`useOptimistic`과 함께 "Actions"로 불림)

### 2-2. useDeferredValue

값 자체를 "지연된 버전"으로 받아 사용한다. 갱신 코드를 통제할 수 없을 때(부모에서 받은 props 등) 유용하다.

```tsx
function List({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query);  // query가 급히 바뀌어도 deferredQuery는 여유 있을 때 따라옴
  const items = useMemo(() => filterHeavy(deferredQuery), [deferredQuery]);
  return <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

| **항목**            | **useTransition**                              | **useDeferredValue**                              | **디바운스/스로틀**                       |
| ------------------- | ---------------------------------------------- | ------------------------------------------------- | ----------------------------------------- |
| **적용 대상**       | 상태 **갱신 함수** 호출                        | **값**                                            | 이벤트 발생 빈도                          |
| **필요 조건**       | setter를 직접 호출할 수 있어야 함              | 값만 있으면 됨(props도 가능)                      | 없음                                      |
| **대기 시간**       | 고정 없음 — 기기 성능에 따라 적응              | 고정 없음                                         | 고정(예: 300ms) — 빠른 기기도 기다림       |
| **진행 중 표시**    | `isPending`                                    | `value !== deferredValue` 비교                    | 직접 구현                                 |
| **적합한 경우**     | 탭 전환·필터처럼 갱신을 통제할 때              | 자식이 받은 props 지연, 검색 결과 표시            | 네트워크 요청 횟수 제한(렌더 문제가 아닐 때) |

> ⚠️ transition은 **렌더 비용**이 문제일 때 쓰는 것이지, **네트워크 요청 횟수**를 줄이는 도구가 아니다. 키 입력마다 API를 호출하는 문제는 디바운스나 서버 상태 라이브러리의 중복 제거로 해결한다.

<br>

### 3. Suspense — 준비되지 않은 UI의 선언적 대기

`<Suspense fallback={…}>`는 하위 트리가 **아직 렌더할 준비가 안 됐을 때**(코드 분할 로딩, 데이터 로딩) fallback을 대신 보여준다.

```tsx
const Chart = lazy(() => import("./Chart"));  // 코드 분할

function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <Chart />            {/* 청크가 로딩되는 동안 Spinner 표시 */}
      <UserPanel />        {/* 같은 경계 안의 형제도 함께 대기 → 한꺼번에 나타남 */}
    </Suspense>
  );
}
```

```
동작 원리
컴포넌트가 렌더 중 "아직 준비 안 됨" 신호(Promise throw 또는 use(promise))
   ↓
가장 가까운 Suspense 경계까지 렌더 중단 → fallback 커밋
   ↓
Promise 완료 → 경계 아래를 다시 렌더 → 실제 콘텐츠로 교체
```

- 경계를 **어디에 두느냐**가 UX를 결정함. 하나의 큰 경계는 "전부 준비될 때까지 전부 대기", 여러 작은 경계는 "준비된 부분부터 표시"
- transition 중에 Suspense가 발생하면 React는 fallback을 보여주는 대신 **이전 화면을 유지**함. 이미 보이는 콘텐츠가 스피너로 바뀌는 불쾌한 경험을 막음

### 3-1. React 19의 use 훅

`use(promise)`와 `use(context)`는 **조건문·이른 반환 뒤에서도 호출할 수 있는** 특별한 훅이다. Promise를 넘기면 완료될 때까지 가장 가까운 Suspense로 중단된다.

```tsx
function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise);  // 준비 안 됐으면 Suspense로 중단, 거부되면 에러 경계(unit09)로
  return <ul>{comments.map((c) => <li key={c.id}>{c.text}</li>)}</ul>;
}
```

> ⚠️ 렌더 안에서 `use(fetch(...))`처럼 **매 렌더 새 Promise를 만들면** 재렌더마다 다시 중단되어 무한 로딩이 된다. Promise는 서버 컴포넌트·부모·캐시(서버 상태 라이브러리)에서 만들어 **안정된 참조**로 넘겨야 한다. 클라이언트 데이터 페칭은 unit04의 라이브러리를 통하는 것이 안전하다.

<br>

### 4. 스트리밍 SSR

전통적 SSR은 서버가 **모든 데이터를 준비한 뒤** 완성된 HTML을 한 번에 보냈다. 느린 데이터 하나가 전체 응답을 지연시켰다. React 18의 **스트리밍 SSR**(`renderToPipeableStream`)은 Suspense 경계 단위로 HTML을 **나눠서 순차 전송**한다.

```
전통 SSR
서버: [데이터 A][데이터 B(느림)………………][전체 HTML 생성] ──▶ 브라우저: 빈 화면 …… 전체 표시

스트리밍 SSR
서버: [셸 HTML + A] ──▶ 브라우저: 셸과 A 즉시 표시, B 자리엔 fallback
서버: ……[B 준비] ──▶ <div hidden>B HTML</div><script>fallback을 B로 교체</script> ──▶ B 자리 채워짐
```

- **선택적 하이드레이션(Selective Hydration)**: 스트리밍된 조각을 순서와 무관하게, 사용자가 **먼저 상호작용한 부분부터** 하이드레이션함. 전체 JS 로딩을 기다리지 않음
- Suspense 경계는 서버에서는 "스트리밍 청크의 단위", 클라이언트에서는 "하이드레이션의 단위"가 됨
- Next.js App Router 등 프레임워크는 이를 기본으로 사용함. React 19.2는 여기에 **부분 사전 렌더링(Partial Pre-rendering)**과 SSR Suspense 배치를 추가함

> 💡 "SSR인데 왜 Suspense가 필요한가?"에는 "**느린 데이터를 기다리지 않고 셸을 먼저 보내고, 준비된 조각부터 스트리밍·하이드레이션하기 위한 경계**"라고 답한다. TTFB와 TTI를 함께 줄이는 것이 목적이다.

<br>

### 5. 찢어짐(tearing)과 useSyncExternalStore

렌더가 중단·재개되는 사이에 **외부 스토어 값이 바뀌면**, 한 화면 안에 이전 값과 새 값이 섞여 렌더되는 **찢어짐(tearing)**이 생길 수 있다.

```
렌더 시작 (store = 1) → 컴포넌트 A 렌더: 1 → [양보] → 스토어 갱신: 2 → 컴포넌트 B 렌더: 2
결과: A는 1, B는 2 → 같은 커밋에 서로 다른 값 (찢어짐)
```

- React가 관리하는 `useState`·Context는 렌더 시작 시점의 값으로 고정되므로 찢어지지 않음
- 외부 스토어(Redux·Zustand, 브라우저 API)는 **`useSyncExternalStore(subscribe, getSnapshot)`**로 구독해야 함. React는 렌더 중 스냅샷이 바뀌면 그 렌더를 버리고 **동기적으로 다시 렌더**해 일관성을 보장함
- 대부분의 상태 라이브러리는 내부적으로 이 훅을 사용함(unit04)

<br>

### 6. 도구 선택 요약

| **문제**                                          | **도구**                                        |
| ------------------------------------------------- | ----------------------------------------------- |
| **입력은 즉시, 결과 목록은 느려도 됨**            | `useTransition` (setter를 통제할 때) / `useDeferredValue` (값만 있을 때) |
| **코드 분할·데이터 로딩 중 대체 UI**              | `Suspense` + `lazy` / `use` / 서버 상태 라이브러리의 suspense 모드 |
| **탭 전환 시 스피너 대신 이전 화면 유지**         | `startTransition` 안에서 탭 상태 갱신           |
| **느린 데이터가 SSR 전체를 막음**                 | 스트리밍 SSR + Suspense 경계 배치               |
| **외부 스토어 값이 화면에서 불일치**              | `useSyncExternalStore`                          |
| **네트워크 요청이 너무 잦음**                     | 디바운스·서버 상태 라이브러리 (동시성 도구 아님) |

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                          |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **동시성 렌더링이란?**                                | 렌더를 중단·재개·폐기할 수 있는 스케줄링. 단일 스레드에서 협력적으로 양보해 긴급 갱신을 우선 처리          |
| **transition과 디바운스의 차이는?**                   | transition은 렌더 우선순위를 낮추고 기기 성능에 적응. 디바운스는 고정 시간 대기이며 요청 횟수 제한에 적합    |
| **useTransition vs useDeferredValue?**                | 전자는 setter 호출을 감쌈, 후자는 값을 지연. 갱신을 통제할 수 없으면 후자                                 |
| **Suspense의 동작 원리는?**                           | 하위가 "준비 안 됨"을 알리면 가장 가까운 경계까지 중단하고 fallback 표시, 준비되면 재렌더                   |
| **스트리밍 SSR의 이점은?**                            | 셸을 먼저 보내고 Suspense 단위로 HTML을 순차 전송, 선택적 하이드레이션으로 상호작용 부분부터 활성화          |
| **tearing이란?**                                      | 렌더 중단 사이에 외부 값이 바뀌어 한 화면에 다른 버전이 섞이는 현상. `useSyncExternalStore`로 방지          |

- 동시성 기능은 `createRoot` + **순수한 렌더**가 전제다. 렌더 중 부수효과가 있으면 어떤 최적화도 안전하지 않다
- Suspense 경계는 **UX 단위**로 배치한다 — "무엇이 함께 나타나야 하는가"를 기준으로
- `use`로 던진 Promise가 거부됐을 때의 처리는 **unit09(에러 경계)**, 서버 상태 캐시는 **unit04**를 참고할 것
