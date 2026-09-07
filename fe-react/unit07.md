## ref와 DOM 접근

React는 선언적으로 UI를 기술하지만, **포커스·스크롤·측정·서드파티 라이브러리 연동**처럼 DOM을 직접 만져야 하는 순간이 있다. 이 유닛은 `useRef`의 두 가지 용도, 부모가 자식의 DOM에 접근하는 `forwardRef`(React 19에서는 불필요), 그리고 명령형 처리가 정당한 경우를 다룬다.

<br>

### 1. useRef의 두 가지 용도

`useRef(initial)`은 `{ current: initial }` 객체를 돌려주며, 이 객체는 **컴포넌트가 살아 있는 동안 같은 참조를 유지**한다. `current`를 바꿔도 **리렌더가 일어나지 않는다**는 점이 `useState`와의 핵심 차이다.

| **항목**             | **useState**                          | **useRef**                                          |
| -------------------- | ------------------------------------- | --------------------------------------------------- |
| **변경 시 리렌더**   | **있음**                              | **없음**                                            |
| **값을 읽는 시점**   | 렌더의 스냅샷(unit02)                 | 항상 최신 `current`                                 |
| **변경 방법**        | setter (다음 렌더에 반영)             | `ref.current = x` (즉시 반영)                       |
| **렌더 중 읽기·쓰기** | 읽기 가능                             | **금지** (초기화 예외) — 렌더 순수성 위반            |
| **용도**             | 화면에 보이는 값                      | 화면과 무관한 값, DOM 노드                          |

### 1-1. 용도 ① — 렌더와 무관한 값 보관

타이머 id, 이전 값, 스크롤 위치, 웹소켓 인스턴스처럼 **바뀌어도 화면을 다시 그릴 필요가 없는 값**은 ref에 둔다.

```tsx
// 안티패턴: 타이머 id를 state로 → 불필요한 리렌더, 스냅샷 때문에 clearInterval 대상이 낡을 수 있음
const [timerId, setTimerId] = useState<number | null>(null);

// 개선: ref에 보관. 값이 바뀌어도 리렌더 없음, 항상 최신
const timerRef = useRef<number | null>(null);
function start() { timerRef.current = window.setInterval(tick, 1000); }
function stop() { if (timerRef.current) clearInterval(timerRef.current); }
```

**최신 값을 콜백에 전달하기(latest ref 패턴)** — 이펙트를 재구성하지 않고도 최신 props·state를 읽고 싶을 때 쓴다(unit02의 stale closure 해결책 중 하나).

```tsx
const onMessageRef = useRef(onMessage);
useEffect(() => { onMessageRef.current = onMessage; });  // 매 렌더 후 최신 콜백으로 갱신

useEffect(() => {
  const socket = connect(roomId);
  socket.on("message", (m) => onMessageRef.current(m));  // 항상 최신 onMessage 호출
  return () => socket.close();
}, [roomId]);  // onMessage가 바뀌어도 재연결하지 않음
```

### 1-2. 용도 ② — DOM 노드 참조

JSX의 `ref` 속성에 ref 객체를 넘기면 커밋 시 `current`에 **실제 DOM 노드**가 들어온다. 언마운트되면 `null`로 돌아간다.

```tsx
function SearchBox() {
  const inputRef = useRef<HTMLInputElement>(null);
  useEffect(() => { inputRef.current?.focus(); }, []);  // 마운트 후 자동 포커스
  return <input ref={inputRef} />;
}
```

> ⚠️ 렌더 도중(`return` 위의 본문)에 `ref.current`를 읽거나 쓰면 안 된다. 렌더 단계에서는 DOM ref가 아직 `null`이고, 값 ref를 렌더 중 바꾸면 StrictMode·동시성 렌더링에서 결과가 달라진다. ref는 **이벤트 핸들러나 이펙트 안**에서만 다룬다.

<br>

### 2. ref가 채워지는 시점

```
렌더 단계        : ref.current === null (아직 DOM 없음)
   ↓
커밋 — DOM 반영
   ↓
ref 연결         : ref.current = <input>  (useLayoutEffect 실행 직전)
   ↓
useLayoutEffect  : DOM 측정 가능 (페인트 전)
   ↓
브라우저 페인트
   ↓
useEffect        : DOM 접근 가능 (페인트 후)
   ↓
언마운트 커밋    : ref.current = null
```

- 조건부 렌더링으로 요소가 사라졌다 나타나면 `current`도 `null` ↔ 노드로 바뀌므로 **옵셔널 체이닝**(`ref.current?.focus()`)이 안전함
- 콜백 ref(`ref={(node) => …}`)는 노드가 연결·해제될 때마다 호출되어, "요소가 나타나는 순간"을 잡거나 여러 노드를 Map에 모을 때 유용함. React 19부터는 콜백 ref가 **클린업 함수를 반환**할 수 있음

<br>

### 3. 부모가 자식의 DOM에 접근하기 — forwardRef와 React 19

함수 컴포넌트에 `ref`를 넘겨도 기본적으로는 컴포넌트 함수에 전달되지 않았다(React 18까지). 자식이 내부 DOM에 ref를 **넘겨주도록(forward)** 명시해야 한다.

```tsx
// React 18 이하: forwardRef로 감싸야 부모의 ref가 내부 input에 도달
const TextField = forwardRef<HTMLInputElement, TextFieldProps>(function TextField(props, ref) {
  return <input ref={ref} className="field" {...props} />;
});
```

```tsx
// React 19: ref가 일반 prop으로 전달됨 → forwardRef 불필요
function TextField({ ref, ...props }: TextFieldProps & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} className="field" {...props} />;
}

// 사용 (두 버전 모두 동일)
function Form() {
  const fieldRef = useRef<HTMLInputElement>(null);
  return <TextField ref={fieldRef} onFocus={() => fieldRef.current?.select()} />;
}
```

| **항목**                | **React 18 이하**                       | **React 19**                                        |
| ----------------------- | --------------------------------------- | --------------------------------------------------- |
| **함수 컴포넌트의 ref** | props에 포함되지 않음 → `forwardRef` 필요 | **일반 prop으로 전달**, `forwardRef` 불필요 (deprecated 예정) |
| **`ref` 콜백의 클린업** | 미지원                                  | 반환한 함수가 해제 시 호출됨                        |
| **Provider 표기**       | `<Ctx.Provider>`                        | `<Ctx>` 직접 사용 가능                              |

> 💡 React 19에서도 `forwardRef`는 당장 동작하지만 향후 제거가 예고되어 있으므로, 새 코드는 `ref`를 prop으로 받는 형태로 작성한다. 라이브러리처럼 여러 버전을 지원해야 하면 당분간 `forwardRef`를 유지한다.

<br>

### 4. useImperativeHandle — 노출할 것만 노출하기

부모에게 DOM 노드 전체를 넘기면 부모가 자식의 내부 구조에 의존하게 된다. `useImperativeHandle`은 ref를 통해 **제한된 명령형 API만** 노출한다.

```tsx
type PlayerHandle = { play: () => void; pause: () => void };

function VideoPlayer({ ref, src }: { ref?: React.Ref<PlayerHandle>; src: string }) {
  const videoRef = useRef<HTMLVideoElement>(null);
  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
  }), []);
  return <video ref={videoRef} src={src} />;
}

// 부모: playerRef.current.play() 는 되지만 videoRef 내부 DOM에는 접근 불가
```

- 부모가 자식의 `<video>`가 `<div>` 안에 있든 없든 신경 쓰지 않게 되어 **캡슐화**가 유지됨
- 자주 쓰면 데이터 흐름이 불투명해지므로 **props로 표현할 수 없는 명령**(포커스·재생·스크롤)에 한정함

<br>

### 5. 명령형 처리가 정당한 경우

React의 기본은 **선언적**이다. "상태 → 화면"으로 표현할 수 있는 것은 state와 props로 하고, 아래처럼 **상태로 표현할 수 없는 일회성 동작이나 React 바깥의 API**만 ref로 처리한다.

| **작업**                                    | **선언적으로 가능한가?**                 | **판단**                                           |
| ------------------------------------------- | ---------------------------------------- | -------------------------------------------------- |
| **포커스 이동, 텍스트 선택**                | 불가 (일회성 동작)                       | ref + `focus()` / `select()`                       |
| **스크롤 위치 이동·복원**                   | 불가                                     | ref + `scrollIntoView()` / `scrollTop`             |
| **요소 크기·위치 측정**                     | 불가                                     | ref + `getBoundingClientRect()` (useLayoutEffect)  |
| **미디어 재생·정지**                        | 불가                                     | ref + `play()` / `pause()`                         |
| **차트·지도·에디터 등 서드파티 위젯 마운트** | 불가 (자체 DOM을 관리함)                 | ref로 컨테이너 전달, useEffect로 초기화·클린업     |
| **클래스 토글, 텍스트 변경, 표시·숨김**     | **가능**                                 | state·props로 처리. ref로 DOM을 바꾸면 React와 충돌 |

```tsx
// 안티패턴: React가 관리하는 DOM을 ref로 직접 수정 → 다음 렌더에서 덮어써지거나 상태와 불일치
function Toggle() {
  const ref = useRef<HTMLDivElement>(null);
  return <div ref={ref} onClick={() => ref.current!.classList.toggle("on")}>…</div>;
}

// 개선: 화면에 보이는 것은 상태로 표현
function Toggle() {
  const [on, setOn] = useState(false);
  return <div className={on ? "on" : ""} onClick={() => setOn((v) => !v)}>…</div>;
}
```

> ⚠️ React가 렌더하는 노드의 **자식을 ref로 추가·삭제**하면 React의 재조정(unit01)과 충돌해 "removeChild 실패" 같은 런타임 오류가 난다. 서드파티 위젯은 **React가 자식을 렌더하지 않는 빈 컨테이너**에만 마운트한다.

<br>

### 6. 서드파티 위젯 연동 패턴

```tsx
function Chart({ data }: { data: number[] }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<ChartInstance | null>(null);

  useEffect(() => {
    chartRef.current = createChart(containerRef.current!);  // ① 마운트: 위젯 생성
    return () => { chartRef.current?.destroy(); chartRef.current = null; };  // ③ 언마운트: 해제
  }, []);

  useEffect(() => {
    chartRef.current?.setData(data);  // ② data 변경: 위젯에 동기화 (unit05의 동기화 관점)
  }, [data]);

  return <div ref={containerRef} />;  // React는 이 div의 자식을 건드리지 않음
}
```

- **생성·해제**와 **데이터 동기화**를 별도 이펙트로 분리하면 data가 바뀔 때 위젯이 재생성되지 않음
- 위젯 인스턴스는 화면에 직접 보이는 값이 아니므로 state가 아닌 ref에 보관함

<br>

### 7. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                                      |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **useRef와 useState의 차이는?**                   | ref는 바꿔도 리렌더가 없고 항상 최신값을 읽음. 화면에 보이는 값은 state, 무관한 값·DOM은 ref          |
| **useRef의 두 용도는?**                           | ① 렌더와 무관한 값 보관(타이머 id, 인스턴스, latest ref) ② DOM 노드 참조                             |
| **forwardRef는 왜 필요했고 React 19에선 어떤가?** | 18까지 함수 컴포넌트는 ref를 prop으로 받지 못해 forwardRef로 전달. 19부터 ref가 일반 prop이 되어 불필요 |
| **useImperativeHandle의 목적은?**                 | DOM 전체 대신 제한된 명령형 API만 노출해 캡슐화 유지                                                 |
| **ref로 DOM을 직접 바꾸면 안 되는 경우는?**       | React가 관리하는 속성·자식. 상태로 표현 가능한 것은 state로, ref는 포커스·스크롤·측정·외부 위젯에 한정 |

- ref는 **렌더 중에 읽거나 쓰지 않는다** — 이벤트 핸들러와 이펙트에서만
- 명령형 처리는 **"상태로 표현할 수 없는가?"**를 먼저 묻고 나서 선택한다
- DOM 측정 후 보정의 타이밍(`useLayoutEffect`)은 **unit05**, latest ref로 stale closure를 피하는 맥락은 **unit02**를 참고할 것
