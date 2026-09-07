## 마이크로태스크와 매크로태스크

unit01의 이벤트 루프 페이즈는 큰 그림일 뿐, 실제 콜백 실행 순서를 결정하는 것은 **매크로태스크(Macrotask)**와 **마이크로태스크(Microtask)**라는 두 종류의 큐다. `process.nextTick`과 Promise가 왜 타이머보다 먼저 실행되는지, 둘 사이의 우선순위는 어떻게 되는지를 알아야 비동기 코드의 순서를 정확히 예측할 수 있다.

<br>

### 1. 두 종류의 태스크

| **구분**           | **매크로태스크**                                            | **마이크로태스크**                                  |
| ------------------ | ----------------------------------------------------------- | --------------------------------------------------- |
| **대표 API**       | `setTimeout`, `setInterval`, `setImmediate`, I/O 콜백        | `process.nextTick`, `Promise.then/catch/finally`, `queueMicrotask` |
| **큐 위치**        | 이벤트 루프의 **각 페이즈 큐**                               | 페이즈와 무관한 **별도 큐 2개** (nextTick 큐, Promise 큐) |
| **실행 시점**      | 해당 페이즈 차례가 왔을 때 하나씩                            | **현재 실행 중인 JS가 끝난 직후**, 다음 매크로태스크 전  |
| **실행 단위**      | 한 번에 콜백 **하나**                                        | 큐가 **완전히 빌 때까지** 전부                       |
| **새로 추가된 작업** | 다음 루프 순회에서 처리                                     | 지금 비우는 중에도 계속 실행 (기아 가능)             |

- 매크로태스크는 이벤트 루프가 "한 바퀴"를 돌며 페이즈마다 소비함
- 마이크로태스크는 "한 콜백이 끝날 때마다" 끼어들어 큐를 비우므로, 사실상 **가장 높은 우선순위**를 가짐

<br>

### 2. 실행 규칙: 콜 스택이 비면 마이크로태스크부터

```
[동기 코드 실행] ─ 콜 스택 비워짐
      │
      ▼
┌──────────────────────────────┐
│ ① process.nextTick 큐 전부 실행 │◀─┐  새로 쌓이면
├──────────────────────────────┤  │  다시 ①부터
│ ② Promise 마이크로태스크 큐 전부 │──┘
└──────────────────────────────┘
      │  두 큐가 모두 비어야
      ▼
[다음 매크로태스크 콜백 1개 실행] ──▶ 다시 ①②
```

- 순서: **동기 코드 → nextTick 큐 → Promise 큐 → 매크로태스크 1개 → nextTick 큐 → Promise 큐 → …**
- nextTick 큐를 비우는 도중 Promise가 추가되면, nextTick 큐를 다 비운 뒤 Promise 큐로 넘어가고, Promise 콜백이 다시 nextTick을 추가하면 **Promise 큐를 비운 후** 다시 nextTick 큐를 확인함

> 💡 Node.js 11 이전에는 마이크로태스크가 **페이즈가 끝날 때** 한 번만 처리되어 브라우저와 순서가 달랐다. Node 11부터는 브라우저처럼 **타이머·`setImmediate` 콜백 하나하나 사이**에 마이크로태스크를 비우도록 바뀌었다. 오래된 블로그 글의 실행 순서 예제는 이 차이 때문에 현재 버전과 결과가 다를 수 있다.

<br>

### 3. 실행 순서 예제

```javascript
console.log('1 동기');

setTimeout(() => console.log('6 timeout'), 0);
setImmediate(() => console.log('7 immediate'));

Promise.resolve().then(() => {
  console.log('4 promise');
  process.nextTick(() => console.log('5 nextTick in promise'));
});

process.nextTick(() => console.log('3 nextTick'));

console.log('2 동기');
```

실행 결과와 이유는 다음과 같다.

| **순서** | **출력**                 | **이유**                                                       |
| -------- | ------------------------ | -------------------------------------------------------------- |
| **1, 2** | 동기                     | 콜 스택에서 즉시 실행                                          |
| **3**    | nextTick                 | 스택이 비자마자 nextTick 큐가 Promise 큐보다 먼저 처리됨        |
| **4**    | promise                  | nextTick 큐가 빈 뒤 Promise 큐 처리                            |
| **5**    | nextTick in promise      | Promise 큐를 비운 뒤 새로 쌓인 nextTick을 **매크로태스크 전에** 처리 |
| **6, 7** | timeout → immediate      | 메인 모듈에서는 순서가 비결정적일 수 있음 (unit01 참고)         |

<br>

### 4. process.nextTick vs Promise vs setImmediate

이름 때문에 가장 많이 헷갈리는 세 API를 비교하면 다음과 같다. 역설적으로 `setImmediate`가 가장 늦고 `nextTick`이 가장 빠르다.

| **항목**           | **process.nextTick**                    | **Promise.then / queueMicrotask**     | **setImmediate**                    |
| ------------------ | --------------------------------------- | ------------------------------------- | ----------------------------------- |
| **분류**           | 마이크로태스크 (Node 전용 큐)            | 마이크로태스크 (V8 큐)                | **매크로태스크** (check 페이즈)      |
| **실행 시점**      | 현재 작업 직후, **가장 먼저**            | nextTick 큐 다음                      | poll 페이즈 이후, 다음 루프 순회     |
| **I/O 양보 여부**  | 양보하지 않음                            | 양보하지 않음                         | **I/O에 양보함**                    |
| **표준 여부**      | Node.js 고유                             | ECMAScript 표준                       | Node.js 고유 (브라우저 미지원)       |
| **주 용도**        | 콜백 일관성 보장, 생성자 직후 이벤트 발행 | 비동기 결과 처리, `async/await`        | 무거운 작업을 다음 루프로 미루기     |

> ⚠️ 이름이 반대다. `setImmediate`는 "즉시"가 아니라 **check 페이즈**에서 실행되고, `process.nextTick`은 "다음 틱"이 아니라 **지금 이 작업이 끝나자마자** 실행된다. 역사적 이유로 이름이 뒤바뀐 채 굳어졌다.

<br>

### 5. 흔한 함정

### 5-1. 마이크로태스크 재귀로 인한 I/O 기아

마이크로태스크 큐는 **빌 때까지** 실행되므로, 재귀적으로 자신을 추가하면 이벤트 루프가 다음 페이즈로 넘어가지 못한다.

```javascript
// 안티패턴: nextTick 재귀 → poll 페이즈에 영원히 도달하지 못함 (I/O·타이머 기아)
function spin() {
  doSmallWork();
  process.nextTick(spin);
}
spin();

// 개선: setImmediate로 매 반복마다 I/O에 양보
function spinFriendly() {
  doSmallWork();
  setImmediate(spinFriendly); // 다음 루프 순회에서 재개 → 사이에 I/O 콜백 실행됨
}
spinFriendly();
```

- `Promise.resolve().then(spin)` 형태의 Promise 재귀도 동일한 문제를 일으킴
- 큰 배열을 나눠 처리하는 "청크 처리"에는 반드시 **`setImmediate`**를 사용해야 함 (unit03 참고)

<br>

### 5-2. async/await의 실제 실행 순서

`async` 함수는 `await`를 만나는 즉시 호출자에게 제어를 돌려주고, 나머지 코드는 **Promise 마이크로태스크**로 예약된다.

```javascript
async function run() {
  console.log('A');           // 동기적으로 실행
  await null;                 // 여기서 제어 반환 → 나머지는 마이크로태스크
  console.log('C');
}
run();
console.log('B');
// 출력: A → B → C
```

- `await`는 값이 Promise가 아니어도 무조건 한 번 마이크로태스크 큐를 거침
- `async` 함수 안에서 던진 예외는 동기 `throw`가 아니라 **rejected Promise**가 되므로 `try/catch` 위치에 주의해야 함 (unit08 참고)

<br>

### 5-3. 동기·비동기가 섞인 콜백 (Zalgo 문제)

같은 함수가 상황에 따라 콜백을 **동기적으로도, 비동기적으로도** 호출하면 호출 측의 상태 관리가 꼬인다. `process.nextTick`은 이런 경우 "항상 비동기"로 통일하는 용도로 쓰인다.

```javascript
// 안티패턴: 캐시 히트면 동기 호출, 미스면 비동기 호출 → 호출 순서가 상황마다 다름
function getUser(id, cb) {
  if (cache.has(id)) return cb(null, cache.get(id));
  db.find(id, cb);
}

// 개선: 캐시 히트여도 nextTick으로 미뤄 항상 비동기 보장
function getUser(id, cb) {
  if (cache.has(id)) return process.nextTick(cb, null, cache.get(id));
  db.find(id, cb);
}
```

> 💡 `EventEmitter`를 상속한 클래스가 생성자 안에서 바로 `this.emit('ready')`를 호출하면 아직 리스너가 등록되기 전이라 이벤트가 유실된다. `process.nextTick(() => this.emit('ready'))`로 미루면 호출자가 `.on('ready', ...)`를 붙인 뒤에 발행된다. Node.js 내부 API가 nextTick을 쓰는 대표적인 이유다.

<br>

### 6. 선택 기준과 정리

| **상황**                                            | **권장 API**                | **이유**                                       |
| --------------------------------------------------- | --------------------------- | ---------------------------------------------- |
| 비동기 결과를 이어서 처리                            | **`Promise` / `async-await`** | 표준이며 에러 전파가 명확함                     |
| 콜백을 "항상 비동기"로 만들고 싶을 때                 | **`process.nextTick`**      | 현재 작업 직후 가장 먼저 실행, I/O보다 우선     |
| 긴 작업을 잘게 나눠 I/O에 양보                       | **`setImmediate`**          | 매 반복마다 poll 페이즈를 거침                  |
| 일정 시간 뒤 실행                                    | **`setTimeout`**            | 단, "최소" 지연 시간임                          |
| 브라우저와 코드를 공유하는 라이브러리                  | **`queueMicrotask`**        | nextTick은 Node 전용                            |

**핵심 정리**

- 콜 스택이 빌 때마다 **nextTick 큐 → Promise 큐** 순으로 마이크로태스크를 모두 비운 뒤 매크로태스크 하나를 실행함
- 우선순위: **동기 코드 > nextTick > Promise > 타이머·I/O·setImmediate**
- Node 11부터 타이머·immediate 콜백 **하나마다** 마이크로태스크를 처리하여 브라우저와 순서가 같아짐
- 마이크로태스크 재귀는 I/O 기아를 일으키므로, 반복 양보에는 `setImmediate`를 사용함
- `await`는 항상 최소 한 번 마이크로태스크를 거치고, `async` 함수의 예외는 rejected Promise가 됨
- `process.nextTick`은 콜백의 동기·비동기 일관성 보장, 생성자 직후 이벤트 발행에 사용함
- 이벤트 루프의 페이즈 구조는 **unit01**, 비동기 예외 처리는 **unit08**을 참고할 것
