## 상태 관리 전략

unit03이 "상태를 어디에 둘 것인가"였다면, 이 유닛은 **트리 전체에 걸친 상태를 어떤 도구로 관리할 것인가**를 다룬다. Context의 리렌더 전파 특성과 한계, 서버 상태와 클라이언트 상태를 분리해야 하는 이유, 외부 스토어를 선택하는 기준을 정리한다.

<br>

### 1. 왜 전역 상태 관리가 필요한가

- 상태 끌어올리기(unit03)를 반복하면 중간 컴포넌트가 자신은 쓰지 않는 props를 아래로 전달만 하는 **props drilling**이 생김
- 테마·로그인 사용자·언어처럼 트리 곳곳에서 읽는 값은 최상단에서 아래까지 props로 흘리기가 비현실적임
- 도구를 고르기 전에 그 상태가 **어떤 종류인지** 먼저 분류해야 함. 종류가 다르면 적합한 도구도 다름

| **종류**            | **예시**                               | **특징**                                        | **적합한 도구**                       |
| ------------------- | -------------------------------------- | ----------------------------------------------- | ------------------------------------- |
| **지역 UI 상태**    | 입력값, 모달 열림, 탭 선택             | 한두 컴포넌트만 사용                            | `useState`, `useReducer`              |
| **전역 UI 상태**    | 테마, 언어, 사이드바 접힘              | 자주 안 바뀜, 트리 곳곳에서 읽음                | Context                               |
| **클라이언트 도메인 상태** | 장바구니, 다단계 폼, 편집기 문서 | 자주 바뀌고 여러 곳에서 쓰며 갱신 로직이 복잡함 | 외부 스토어(Zustand·Redux·Jotai 등)   |
| **서버 상태**       | 사용자 목록, 상품 상세, 알림           | 원본이 서버에 있음, 비동기, 캐시·재검증 필요    | 서버 상태 라이브러리(TanStack Query·SWR 등) |
| **URL 상태**        | 검색어, 페이지 번호, 필터              | 새로고침·공유·뒤로가기에 살아남아야 함          | 라우터의 쿼리스트링                   |

> 💡 "상태 관리 라이브러리 뭐 써봤어요?"라는 질문의 의도는 도구 이름이 아니라 **상태를 종류별로 분류하고 각기 다른 도구를 배정할 수 있는가**다. 위 표를 기준으로 답하면 된다.

<br>

### 2. Context의 동작 원리

**Context**는 트리의 상위에서 값을 제공(Provider)하면 깊이에 상관없이 하위에서 `useContext`로 읽을 수 있게 하는 **의존성 주입 통로**다.

```tsx
const ThemeContext = createContext<"light" | "dark">("light");

function App() {
  const [theme, setTheme] = useState<"light" | "dark">("light");
  return (
    <ThemeContext value={theme}>      {/* React 19: <ThemeContext.Provider> 대신 Context 자체를 Provider로 사용 가능 */}
      <Layout />
    </ThemeContext>
  );
}

function ThemedButton() {
  const theme = useContext(ThemeContext);  // 중간 컴포넌트를 거치지 않고 바로 읽음
  return <button className={theme}>버튼</button>;
}
```

- Provider의 `value`가 `Object.is`로 이전과 다르면 그 Context를 **구독하는 모든 하위 컴포넌트**가 리렌더됨
- 중간 컴포넌트가 `React.memo`로 감싸져 있어도 소비자의 리렌더는 막히지 않음(Context는 props가 아니므로)
- React 19부터는 `<Context.Provider>` 대신 `<Context>`를 직접 Provider로 쓸 수 있고, `useContext(Ctx)` 대신 `use(Ctx)`도 가능함(조건부 호출 허용)

<br>

### 3. Context의 리렌더 전파 한계

### 3-1. 값의 일부만 써도 전체가 리렌더된다

`useContext`는 **선택적 구독(selector)**을 지원하지 않는다. `{ user, theme, cart }`를 하나의 Context에 넣으면 `cart`만 바뀌어도 `theme`만 쓰는 컴포넌트까지 리렌더된다.

```
AppContext value = { user, theme, cart }
                    cart 변경
                       ↓
  ┌────────────┬────────────┬────────────┐
  │ Header     │ Sidebar    │ CartBadge  │   ← 셋 다 리렌더 (Header·Sidebar는 cart를 안 씀)
  │ (theme만)  │ (user만)   │ (cart)     │
  └────────────┴────────────┴────────────┘
```

### 3-2. value 객체가 렌더마다 새로 만들어진다

```tsx
// 안티패턴: Provider를 가진 컴포넌트가 리렌더될 때마다 새 객체 → 모든 소비자 리렌더
<AuthContext value={{ user, login, logout }}>
```

```tsx
// 개선 ①: value를 useMemo로 고정 (user가 바뀔 때만 새 객체)
const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
<AuthContext value={value}>

// 개선 ②: 자주 바뀌는 값과 안 바뀌는 값을 별도 Context로 분리
<UserContext value={user}>
  <AuthActionsContext value={actions}>   {/* login·logout은 거의 안 바뀜 → 액션만 쓰는 컴포넌트는 리렌더 없음 */}
```

### 3-3. Context가 적합한 경우와 아닌 경우

| **기준**                    | **Context 적합**                              | **Context 부적합**                                      |
| --------------------------- | --------------------------------------------- | ------------------------------------------------------- |
| **변경 빈도**               | 드묾(테마, 로케일, 로그인 사용자)             | 잦음(입력값, 마우스 위치, 실시간 데이터)                |
| **소비자 수 × 변경 빈도**   | 작음                                          | 큼 → 리렌더 폭풍                                        |
| **부분 구독 필요**          | 불필요                                        | 필요(대형 객체 중 일부 필드만 사용)                     |
| **본질**                    | **의존성 주입** (어떤 값을 아래로 전달)       | **상태 저장소** (자주 갱신되는 데이터의 관리)           |

> ⚠️ Context는 "전역 상태 관리 도구"가 아니라 **값 전달 통로**다. 자주 바뀌는 상태를 Context에 넣고 리렌더 문제를 `memo`로 막으려 하면 코드는 복잡해지고 효과는 제한적이다. 이 시점이 외부 스토어를 고려할 신호다.

<br>

### 4. 외부 스토어와 선택적 구독

Zustand·Redux·Jotai 같은 **외부 스토어(External Store)**는 상태를 React 트리 밖에 두고, 각 컴포넌트가 **필요한 조각만 선택(selector)**해 구독한다. 선택한 값이 바뀔 때만 그 컴포넌트가 리렌더된다.

```tsx
// Zustand 예시: cart.count만 구독 → user가 바뀌어도 이 컴포넌트는 리렌더되지 않음
const useStore = create<Store>((set) => ({
  user: null,
  cart: { items: [], count: 0 },
  addItem: (item) => set((s) => ({ cart: { items: [...s.cart.items, item], count: s.cart.count + 1 } })),
}));

function CartBadge() {
  const count = useStore((s) => s.cart.count);
  return <span>{count}</span>;
}
```

- 이런 라이브러리는 내부적으로 React 18의 **`useSyncExternalStore`**를 사용해 동시성 렌더링 중에도 찢어짐(tearing, unit08) 없이 외부 값을 읽음
- 직접 스토어를 만들 때도 `useSyncExternalStore(subscribe, getSnapshot)`로 안전하게 연결할 수 있음

| **항목**          | **Context**                       | **외부 스토어**                          |
| ----------------- | --------------------------------- | ---------------------------------------- |
| **구독 단위**     | Context 전체                      | selector로 고른 조각                     |
| **리렌더 범위**   | 모든 소비자                       | 선택값이 바뀐 소비자만                   |
| **트리 밖 접근**  | 불가(컴포넌트 안에서만)           | 가능(이벤트 핸들러·유틸 함수에서도)      |
| **DevTools·미들웨어** | 없음                          | 대부분 지원(로깅, 영속화, 타임트래블)    |
| **추가 의존성**   | 없음                              | 있음                                     |

<br>

### 5. 서버 상태와 클라이언트 상태의 분리

**서버 상태(Server State)**는 원본이 서버에 있는 데이터의 **로컬 캐시**다. 클라이언트가 소유한 상태와 성격이 근본적으로 다르다.

| **항목**            | **클라이언트 상태**            | **서버 상태**                                              |
| ------------------- | ------------------------------ | ---------------------------------------------------------- |
| **소유자**          | 클라이언트                     | 서버 (클라이언트는 사본을 가짐)                            |
| **동기·비동기**     | 동기                           | 비동기 (로딩·에러·재시도 필요)                             |
| **신선도**          | 항상 최신                      | 시간이 지나면 **낡음(stale)** → 재검증 필요                |
| **공유**            | 이 브라우저 탭만               | 다른 사용자·탭이 바꿀 수 있음                              |
| **필요한 기능**     | 읽기·쓰기                      | 캐시, 중복 요청 제거, 백그라운드 갱신, 낙관적 업데이트     |

```tsx
// 안티패턴: 서버 데이터를 useState + useEffect로 직접 관리
function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    let cancelled = false;
    fetch("/api/users").then((r) => r.json()).then((d) => { if (!cancelled) { setUsers(d); setLoading(false); } });
    return () => { cancelled = true; };
  }, []);  // 캐시 없음, 다른 화면에서 같은 요청 반복, 탭 복귀 시 갱신 없음, 에러 처리 누락
  // ...
}
```

```tsx
// 개선: 서버 상태 라이브러리에 위임 (TanStack Query 예시)
function UserList() {
  const { data: users, isPending, error } = useQuery({
    queryKey: ["users"],
    queryFn: () => fetch("/api/users").then((r) => r.json() as Promise<User[]>),
    staleTime: 60_000,  // 1분 동안은 캐시를 신선한 것으로 간주
  });
  if (isPending) return <Spinner />;
  if (error) return <ErrorView error={error} />;
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

- 서버 상태를 전역 스토어(Redux 등)에 복사해 두면 **캐시 무효화·재검증 로직을 직접 구현**해야 하고, 그 코드가 스토어의 대부분을 차지하게 됨
- 서버 상태를 분리하고 나면 진짜 클라이언트 전역 상태는 생각보다 적어 Context나 작은 스토어로 충분한 경우가 많음

> 💡 "Redux 없이도 되나요?"라는 질문에는 "서버 상태를 서버 상태 라이브러리로 빼고 나면 남는 클라이언트 전역 상태가 얼마나 되는지"로 답한다. 남은 것이 테마·인증 정도면 Context로 충분하고, 복잡한 도메인 상태가 남으면 스토어를 도입한다.

<br>

### 6. 도구 선택 흐름

```
이 상태의 원본이 서버인가?
 ├─ 예 → 서버 상태 라이브러리 (캐시·재검증·낙관적 업데이트)
 └─ 아니오
      ├─ URL에 남아야 하는가? → 예 → 쿼리스트링·라우터 상태
      ├─ 한두 컴포넌트만 쓰는가? → 예 → useState / useReducer (unit03)
      ├─ 드물게 바뀌고 여러 곳에서 읽는가? → 예 → Context (value는 useMemo, 필요시 분리)
      └─ 자주 바뀌고 부분 구독·트리 밖 접근이 필요한가? → 예 → 외부 스토어
```

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                       |
| ----------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Context의 한계는?**                                 | 선택적 구독이 없어 value 일부만 바뀌어도 모든 소비자가 리렌더됨. `memo`로도 소비자 리렌더는 못 막음  |
| **Context 리렌더를 줄이는 방법은?**                   | value를 `useMemo`로 고정, 변경 빈도별로 Context 분리, 그래도 부족하면 외부 스토어                    |
| **외부 스토어가 리렌더를 줄이는 원리는?**             | selector로 필요한 조각만 구독하고 `useSyncExternalStore`로 안전하게 읽음                              |
| **서버 상태를 따로 관리하는 이유는?**                 | 원본이 서버에 있어 낡을 수 있고 비동기이므로 캐시·재검증·중복 제거가 필요. 이를 직접 구현하면 스토어가 비대해짐 |
| **Context vs Redux 중 무엇을 쓰는가?**                | 목적이 다름. Context는 의존성 주입, 스토어는 자주 갱신되는 상태의 관리. 상태 종류별로 도구를 배정     |

- 상태를 **서버·URL·지역·전역 UI·도메인**으로 먼저 분류하고, 종류마다 다른 도구를 쓴다
- Context에는 **드물게 바뀌는 값**만 넣고, value는 반드시 참조를 안정화한다
- 동시성 렌더링과 `useSyncExternalStore`의 관계는 **unit08**, 리렌더 최적화 판단은 **unit06**을 참고할 것
