## 에러 경계와 예외 처리

컴포넌트 하나에서 던져진 예외가 처리되지 않으면 React는 **트리 전체를 언마운트**해 빈 화면을 남긴다. **에러 경계(Error Boundary)**는 이를 막는 장치지만 **잡을 수 있는 범위가 제한적**이다. 이 유닛은 에러 경계가 무엇을 잡고 무엇을 못 잡는지, 못 잡는 비동기 에러는 어떻게 다루는지를 정리한다.

<br>

### 1. 처리되지 않은 렌더 에러의 결과

```
<App>
 └ <Layout>
    └ <Feed>
       └ <Post>  ← 렌더 중 throw
                     ↓ 에러 경계가 없으면
React 18+: 루트 전체 언마운트 → 빈 화면 (React 16~17도 동일)
           콘솔에 에러 출력, 사용자는 새로고침 외에 복구 수단 없음
```

- 렌더 중 예외는 부분 손상된 UI를 남기는 것보다 아예 지우는 것이 낫다는 판단으로 React 16부터 **전체 언마운트**가 기본 동작임
- 따라서 프로덕션 앱에서는 **최소한 하나의 에러 경계**가 루트 근처에 있어야 함

> 💡 React 19는 처리되지 않은 에러를 **다시 던지지(rethrow) 않고** `console.error`로 한 번만 보고하며, `createRoot`의 `onUncaughtError`·`onCaughtError`·`onRecoverableError` 옵션으로 로깅 훅을 제공한다. 에러 리포팅 도구(Sentry 등)와 연동할 때 이 옵션을 활용한다.

<br>

### 2. 에러 경계란

**에러 경계**는 하위 트리에서 발생한 렌더 에러를 잡아 **fallback UI를 대신 보여주는 컴포넌트**다. `try/catch`의 컴포넌트 버전이라고 볼 수 있다.

```tsx
import { Component, type ErrorInfo, type ReactNode } from "react";

type Props = { fallback: ReactNode; children: ReactNode };
type State = { hasError: boolean };

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };  // 렌더 단계: 다음 렌더에서 fallback을 그리도록 상태 변경
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    reportError(error, info.componentStack);  // 커밋 단계: 로깅 등 부수효과
  }

  render() {
    return this.state.hasError ? this.props.fallback : this.props.children;
  }
}
```

- `getDerivedStateFromError`와 `componentDidCatch`는 **클래스 컴포넌트에만** 존재함. 훅 버전은 없으므로 에러 경계는 React 19에서도 클래스로 작성해야 함(또는 `react-error-boundary` 같은 라이브러리 사용)
- 에러가 발생하면 경계 **아래 트리 전체**가 언마운트되고 fallback으로 교체됨. 경계 위의 트리는 영향받지 않음

```tsx
// 사용: 위젯 단위로 격리
<ErrorBoundary fallback={<p>차트를 불러올 수 없습니다.</p>}>
  <Chart />
</ErrorBoundary>
```

<br>

### 3. 에러 경계가 잡는 범위

에러 경계는 **React가 컴포넌트를 실행하는 동안** 던져진 에러만 잡는다. 이것이 가장 자주 오해되는 부분이다.

| **에러 발생 위치**                        | **잡히는가?** | **이유**                                                    |
| ----------------------------------------- | ------------- | ----------------------------------------------------------- |
| **렌더 중** (컴포넌트 함수 본문, JSX)     | **예**        | React가 호출 스택 안에서 실행 중                            |
| **생명주기·이펙트 본문** (`useEffect` 동기 부분) | **예**  | 커밋 단계에서 React가 실행                                  |
| **자식 컴포넌트의 생성자·렌더**           | **예**        | 하위 트리 전체가 대상                                       |
| **`use(promise)`의 거부**                 | **예**        | 렌더 중 throw로 변환됨(unit08)                              |
| **이벤트 핸들러** (`onClick` 등)          | **아니오**    | React 스택 밖 — 브라우저가 호출. 렌더에 영향 없으므로 직접 `try/catch` |
| **비동기 콜백** (`setTimeout`, `Promise.then`, `fetch`) | **아니오** | 렌더가 끝난 뒤 별도 태스크에서 실행됨                |
| **서버 사이드 렌더링**                    | **아니오**    | 클라이언트 경계는 서버 렌더에 개입 못 함(프레임워크별 처리) |
| **에러 경계 자신의 렌더**                 | **아니오**    | 자기 자신은 못 잡음 — 상위 경계로 전파                      |

```
        ErrorBoundary
        ┌─────────────────────────────────────┐
        │  <Child />                           │
        │   ├─ render() throw       → 잡힘     │
        │   ├─ useEffect(() => throw) → 잡힘   │
        │   ├─ onClick={() => throw} → 안 잡힘 │ → window.onerror
        │   └─ fetch().then(throw)   → 안 잡힘 │ → unhandledrejection
        └─────────────────────────────────────┘
```

> ⚠️ "에러 경계를 뒀으니 모든 에러가 처리된다"는 착각이 가장 위험하다. 실무에서 가장 많은 에러는 **이벤트 핸들러와 비동기 요청**에서 나오는데, 이들은 에러 경계가 전혀 보지 못한다.

<br>

### 4. 에러 경계 배치 전략

| **배치 위치**              | **효과**                                                      | **적합한 경우**                          |
| -------------------------- | ------------------------------------------------------------- | ---------------------------------------- |
| **루트 (App 바로 아래)**   | 빈 화면 방지, "문제가 발생했습니다 + 새로고침" 전역 fallback   | 모든 앱의 최후 방어선                    |
| **라우트·페이지 단위**     | 한 페이지 오류가 내비게이션·헤더를 살려둠                     | 대부분의 앱                              |
| **위젯·카드 단위**         | 차트 하나가 죽어도 나머지 대시보드는 정상                     | 독립적인 위젯이 많은 화면                |
| **너무 세밀하게**          | fallback이 화면 곳곳에 흩어져 오히려 혼란                     | 피할 것                                  |

- **여러 층으로 중첩**하는 것이 일반적임. 안쪽 경계가 잡으면 바깥은 관여하지 않고, 안쪽이 없으면 바깥으로 전파됨
- 에러 경계는 **Suspense 경계와 짝**으로 배치하면 자연스러움. 같은 단위로 "로딩 중"과 "실패"를 표현하기 때문

<br>

### 5. 복구(reset) 설계

fallback만 보여주고 끝나면 사용자는 새로고침밖에 할 수 없다. **복구 경로**를 함께 설계한다.

```tsx
// 방법 ①: key로 경계를 리마운트 → 내부 상태 초기화 (unit01의 key 원리)
function Page() {
  const [attempt, setAttempt] = useState(0);
  return (
    <ErrorBoundary key={attempt} fallback={<button onClick={() => setAttempt((a) => a + 1)}>다시 시도</button>}>
      <Feed />
    </ErrorBoundary>
  );
}
```

```tsx
// 방법 ②: react-error-boundary 라이브러리 — resetKeys·onReset 제공
<ErrorBoundary
  FallbackComponent={({ error, resetErrorBoundary }) => (
    <div role="alert">
      <p>{error.message}</p>
      <button onClick={resetErrorBoundary}>다시 시도</button>
    </div>
  )}
  onReset={() => queryClient.invalidateQueries()}   // 재시도 시 서버 상태 캐시도 갱신
  resetKeys={[userId]}                              // userId가 바뀌면 자동 복구
>
  <Profile userId={userId} />
</ErrorBoundary>
```

- 복구 시 **원인이 된 데이터도 함께 무효화**해야 같은 에러가 반복되지 않음
- 라우트 이동 시 자동 복구되도록 `resetKeys`에 경로를 넣는 패턴이 흔함

<br>

### 6. 에러 경계 밖의 에러 처리

### 6-1. 이벤트 핸들러

```tsx
async function handleSubmit() {
  try {
    await saveOrder(cart);
    toast.success("저장되었습니다");
  } catch (e) {
    toast.error(e instanceof ApiError ? e.message : "알 수 없는 오류");  // UI 유지, 사용자에게 피드백
    reportError(e);
  }
}
```

### 6-2. 비동기 에러를 렌더 에러로 승격시키기

비동기 에러가 "이 화면을 더 이상 보여줄 수 없는" 수준이라면, **상태에 담아 렌더 중 throw**해 에러 경계로 넘길 수 있다.

```tsx
function Feed() {
  const [error, setError] = useState<Error | null>(null);
  useEffect(() => {
    load().catch(setError);  // 비동기 에러를 상태로 포착
  }, []);
  if (error) throw error;    // 렌더 중 throw → 가장 가까운 에러 경계가 잡음
  // ...
}
```

- 서버 상태 라이브러리(unit04)는 이 패턴을 `throwOnError`(TanStack Query) 같은 옵션으로 제공함. 쿼리 실패를 자동으로 에러 경계에 위임
- React 19의 `use(promise)`는 거부된 Promise를 **렌더 에러로 변환**하므로 별도 승격 없이 경계에서 잡힘

### 6-3. 전역 안전망

```typescript
window.addEventListener("error", (e) => reportError(e.error));                 // 동기 예외
window.addEventListener("unhandledrejection", (e) => reportError(e.reason));   // 처리 안 된 Promise 거부
```

- 에러 경계·try/catch를 빠져나간 것을 **로깅 목적**으로 잡는 마지막 층. UI 복구는 하지 못함

| **에러 종류**                 | **1차 처리**                        | **UI 표현**                       | **로깅**                                  |
| ----------------------------- | ----------------------------------- | --------------------------------- | ----------------------------------------- |
| **렌더 에러**                 | 에러 경계                           | fallback + 재시도                 | `componentDidCatch` / `onCaughtError`     |
| **이벤트 핸들러 에러**        | `try/catch`                         | 토스트·인라인 메시지, UI 유지     | catch 블록에서 리포트                     |
| **데이터 요청 실패**          | 서버 상태 라이브러리 `error` 상태   | 인라인 에러 또는 `throwOnError`로 경계 위임 | 라이브러리 `onError`             |
| **복구 불가 비동기 에러**     | 상태로 포착 후 렌더 중 throw        | 에러 경계 fallback                | 경계에서 리포트                           |
| **예상 못 한 모든 것**        | `window.onerror` / `unhandledrejection` | 없음(안전망)                  | 전역 리포트                               |

> 💡 판단 기준은 **"이 에러가 났을 때 현재 화면을 계속 보여줘도 되는가"**다. 계속 보여줘도 되면 `try/catch` + 토스트로 UI를 유지하고, 보여주면 안 되면(데이터가 깨져 렌더 불가) 에러 경계로 화면 단위 교체를 한다.

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                   |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **에러 경계란?**                                      | 하위 트리의 렌더·생명주기 에러를 잡아 fallback을 보여주는 클래스 컴포넌트. 훅 버전은 없음           |
| **에러 경계가 못 잡는 것은?**                         | 이벤트 핸들러, `setTimeout`·Promise 등 비동기 콜백, SSR, 자기 자신의 에러                          |
| **이벤트 핸들러 에러는 왜 안 잡히는가?**              | React 렌더 스택 밖에서 브라우저가 호출하므로. 렌더에 영향이 없어 `try/catch`로 충분                  |
| **비동기 에러를 경계로 보내려면?**                    | 상태에 담아 렌더 중 throw. 서버 상태 라이브러리의 `throwOnError`, React 19의 `use`가 같은 원리       |
| **에러 경계는 어디에 두는가?**                        | 루트(최후 방어) + 페이지·위젯 단위로 중첩. Suspense 경계와 같은 단위로 배치                         |
| **복구는 어떻게 하는가?**                             | key 변경으로 리마운트하거나 `resetKeys`·`resetErrorBoundary` 사용. 원인 데이터도 함께 무효화         |

- 에러 경계는 **렌더 단계의 안전망**이며, 실무 에러의 다수는 그 밖(핸들러·비동기)에서 난다
- "화면을 계속 보여줘도 되는가"로 **UI 유지(try/catch) vs 화면 교체(경계)**를 가른다
- `use` 훅과 Suspense는 **unit08**, 서버 상태 에러 처리는 **unit04**를 참고할 것
