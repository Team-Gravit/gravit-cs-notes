## 메모리 누수 진단

Node.js 프로세스의 메모리가 **재시작 전까지 계속 증가**하다가 OOM(Out Of Memory)으로 죽는다면 메모리 누수(Memory Leak)다. JavaScript는 가비지 컬렉터(GC)가 메모리를 자동 회수하지만, **더 이상 필요 없는 객체가 어딘가에서 계속 참조되고 있으면** 회수하지 못한다. 이 유닛은 V8 메모리 구조, 클로저·리스너 등 흔한 누수 패턴, 그리고 힙 스냅샷(Heap Snapshot)을 읽어 원인을 찾는 방법을 다룬다.

<br>

### 1. V8의 메모리 구조와 GC

```
┌───────────────────────────── V8 힙 ─────────────────────────────┐
│  New Space (Young 세대, 수 MB)      │  Old Space (Old 세대)         │
│  새 객체 할당 → Scavenge (빠름, 자주) │  오래 살아남은 객체 → Mark-Sweep │
│  두 번 살아남으면 Old로 승격 ────────▶│  (느림, 드묾, 정지 시간 큼)     │
└─────────────────────────────────────┴───────────────────────────┘
   + 힙 밖: Buffer·네이티브 객체 (external), 코드, 스택 (rss에 포함)
```

- **세대별 GC**: 대부분의 객체는 금방 죽는다는 가정으로 New Space를 작고 빠르게 청소(Scavenge)하고, 살아남은 객체만 Old Space로 옮겨 가끔 전체 표시-청소(Mark-Sweep-Compact)함
- 누수 객체는 Old Space에 쌓이므로 **Old Space 크기가 단조 증가**하는 그래프가 누수의 전형적 모습임
- 힙 상한은 `--max-old-space-size=<MB>`로 지정하며, 기본값은 버전과 시스템 메모리에 따라 다름(64비트 기준 대략 2~4GB). 컨테이너 메모리 제한보다 낮게 잡지 않으면 GC 전에 컨테이너가 먼저 죽을 수 있음

```javascript
const m = process.memoryUsage();
console.log({
  rss: m.rss,               // 프로세스 전체 상주 메모리 (힙 + 네이티브 + 코드)
  heapTotal: m.heapTotal,   // V8이 확보한 힙 크기
  heapUsed: m.heapUsed,     // 실제 사용 중인 힙 — 누수 판단의 1차 지표
  external: m.external,     // Buffer 등 V8 밖 C++ 객체
  arrayBuffers: m.arrayBuffers,
});
```

> 💡 `heapUsed`는 평평한데 `rss`만 늘면 힙 밖 누수(Buffer, 네이티브 애드온, 스레드 풀)이거나 단순 메모리 단편화다. 어느 지표가 늘어나는지에 따라 진단 도구가 달라진다.

<br>

### 2. 누수란 무엇인가: 도달 가능성

GC는 "쓰이지 않는 객체"가 아니라 **"GC 루트에서 도달할 수 없는 객체"**를 회수한다. 루트는 전역 객체, 현재 실행 중인 스택, 활성 타이머·핸들의 콜백 등이다. 따라서 누수는 언제나 "**필요 없어진 객체로 향하는 참조 경로가 남아 있는 것**"이며, 진단은 그 경로(Retainer Path)를 찾는 일이다.

```
GC 루트 (global, 모듈 스코프, 활성 타이머 ...)
   │
   └─▶ 캐시 Map ──▶ 요청 컨텍스트 객체 ──▶ 응답 버퍼 (수 MB)
                    ↑ 요청은 끝났지만 Map이 잡고 있어 회수 불가
```

<br>

### 3. 흔한 누수 패턴

**패턴 ① 전역·모듈 스코프에 무한히 쌓이는 컬렉션**

```javascript
// 안티패턴: 요청마다 Map에 넣고 지우지 않음 → 요청 수에 비례해 증가
const sessions = new Map();
app.post('/login', (req, res) => {
  sessions.set(req.body.token, { user: req.body.user, createdAt: Date.now() });
});

// 개선: 크기·시간 상한이 있는 캐시(LRU + TTL) 또는 외부 저장소(Redis)
const { LRUCache } = require('lru-cache');
const sessions = new LRUCache({ max: 10_000, ttl: 30 * 60 * 1000 });
```

- 모듈 스코프 변수는 모듈 캐시(unit04) 때문에 프로세스가 살아 있는 한 GC 루트에서 도달 가능함
- "임시로 넣어 두는" 배열·Map·객체 캐시가 가장 흔한 원인이며, 반드시 **상한(max)·만료(TTL)·삭제 경로**가 있어야 함

**패턴 ② 클로저가 잡고 있는 큰 참조**

클로저는 자신이 정의된 스코프의 변수를 **참조로** 유지한다. 오래 살아남는 클로저(타이머 콜백, 리스너, 캐시된 함수)가 큰 데이터를 담은 스코프에서 만들어지면, 그 데이터는 클로저와 수명을 같이한다.

```javascript
// 안티패턴: 5MB 버퍼를 읽은 스코프에서 만든 타이머 콜백이 스코프 전체를 유지
function handle(req, res) {
  const big = fs.readFileSync('template.html');        // 5MB
  const stats = { path: req.url };
  setInterval(() => report(stats), 60_000);            // 이 클로저 때문에 big도 회수되지 않음
  res.end(render(big));
}

// 개선: 오래 사는 콜백은 필요한 값만 담은 별도 스코프에서 만들고, 해제 경로를 둠
function handle(req, res) {
  const big = fs.readFileSync('template.html');
  res.end(render(big));
  scheduleReport({ path: req.url });                   // big이 없는 스코프
}
function scheduleReport(stats) {
  const t = setInterval(() => report(stats), 60_000);
  return () => clearInterval(t);                       // 호출자가 정리할 수 있게
}
```

- V8은 같은 스코프의 클로저들이 **컨텍스트 객체를 공유**하므로, 한 클로저만 `big`을 써도 다른 클로저가 살아 있으면 `big`이 남을 수 있음
- `setInterval`·`setTimeout`의 콜백은 타이머가 해제될 때까지 GC 루트에서 도달 가능함

**패턴 ③ 이벤트 리스너 누적**

```javascript
// 안티패턴: 요청마다 싱글톤 emitter에 리스너 추가, 제거하지 않음
app.get('/events', (req, res) => {
  bus.on('update', (data) => res.write(`data: ${JSON.stringify(data)}\n\n`));
  // 클라이언트가 끊어도 리스너는 남아 res(와 소켓 버퍼)를 계속 붙잡음
});

// 개선: 연결 종료 시 반드시 제거, 또는 AbortSignal로 일괄 해제
app.get('/events', (req, res) => {
  const onUpdate = (data) => res.write(`data: ${JSON.stringify(data)}\n\n`);
  bus.on('update', onUpdate);
  req.on('close', () => bus.off('update', onUpdate));
});
```

- 리스너는 emitter가 참조하는 함수이며, 그 함수는 클로저로 `req`·`res`를 참조함 → 요청 하나가 통째로 남음
- 같은 emitter에 리스너가 **11개**를 넘으면 Node.js가 `MaxListenersExceededWarning`을 출력함. 이 경고는 누수의 가장 이른 신호이므로 `setMaxListeners`로 끄지 말고 원인을 찾아야 함
- Node 20+에서는 `events.on(emitter, 'x', { signal })`·`addEventListener(..., { signal })`처럼 `AbortSignal`로 여러 리스너를 한 번에 해제할 수 있음

> ⚠️ 위 패턴들의 공통점은 "**짧게 살아야 할 객체(요청)가 오래 사는 객체(싱글톤, 전역, 타이머, emitter)에 매달리는 것**"이다. NestJS에서 DEFAULT 스코프 서비스의 필드에 요청 데이터를 저장하는 실수(unit06)도 정확히 같은 구조의 누수다.

<br>

### 4. 진단 도구

| **도구**                                     | **용도**                                       | **비고**                                          |
| -------------------------------------------- | ---------------------------------------------- | ------------------------------------------------- |
| **`process.memoryUsage()` + 메트릭**          | 시간에 따른 `heapUsed`·`rss` 추이 관찰           | 누수 여부 판단의 첫 단계. APM·Prometheus로 수집     |
| **`--trace-gc`**                             | GC 발생 시점·전후 힙 크기 로그                   | Mark-Sweep 후에도 힙이 줄지 않으면 누수 의심         |
| **`--inspect` + Chrome DevTools Memory 탭**   | 힙 스냅샷 촬영·비교, 할당 타임라인               | 개발·스테이징에서 가장 강력                          |
| **`v8.writeHeapSnapshot()` / `--heapsnapshot-signal=SIGUSR2`** | 운영 프로세스에서 파일로 스냅샷 저장 | 촬영 순간 **이벤트 루프가 멈추고** 힙 크기의 수 배 메모리·시간 필요 |
| **`--heap-prof`**                            | 할당 위치별 샘플링 프로파일                      | 어떤 코드가 많이 할당하는지                          |
| **clinic.js (doctor·heapprofiler)**          | 종합 진단 리포트                                  | 원인 후보를 자동 제시                                |

```bash
node --heapsnapshot-signal=SIGUSR2 --max-old-space-size=1024 server.js   # 실행 시 플래그 필요
kill -USR2 <pid>    # Heap.<날짜>.heapsnapshot 파일 생성 → DevTools에서 Load
```

- 로컬에서 재현되지 않으면 부하 도구(`autocannon`)로 트래픽을 흘리며 `heapUsed`가 GC 이후에도 계단식으로 오르는지 확인함

<br>

### 5. 힙 스냅샷 해석

### 5-1. 3-스냅샷 기법

```
시간 ─▶
 [부하 전] 스냅샷 1 ── 부하(N회 요청) ── [GC] 스냅샷 2 ── 부하(N회) ── [GC] 스냅샷 3
                                            │                              │
             스냅샷 2를 기준(Comparison)으로 3에서 "새로 생겨 남아 있는" 객체를 본다
```

- 스냅샷 1과 2의 차이에는 초기화(JIT, 캐시 워밍) 객체가 섞여 있으므로, **2 → 3의 증가분**을 보는 것이 정확함
- 촬영 전에 DevTools의 GC 버튼(휴지통)으로 강제 수집해 회수 가능한 객체를 제거함
- 부하를 정확히 N회로 통제하면 "N개씩 늘어나는 생성자"가 곧 누수 지점임

<br>

### 5-2. 뷰와 컬럼 읽기

| **항목**             | **의미**                                                     | **활용**                                                   |
| -------------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| **Summary 뷰**       | 생성자(Constructor)별로 객체 묶음                              | `(closure)`, `(array)`, `(string)`, 도메인 클래스명 확인     |
| **Comparison 뷰**    | 두 스냅샷 사이의 **# New / # Deleted / # Delta**               | Delta가 요청 수와 비례하는 생성자를 찾음                     |
| **Shallow Size**     | 객체 **자체**의 크기                                          | 작은 객체가 많으면 여기선 안 보임                              |
| **Retained Size**    | 이 객체가 사라지면 **함께 회수될 총량**                         | 누수의 진짜 크기. 정렬 기준으로 사용                          |
| **Distance**         | GC 루트로부터의 참조 거리                                       | 짧을수록 전역·모듈 스코프에 가깝게 매달려 있음                 |
| **Retainers 패널**   | 선택한 객체를 **누가 참조하는가** (역방향 경로)                   | 위로 따라가면 `Map → module → global` 같은 루트까지의 경로가 보임 |

- 진단 순서: **Comparison에서 Delta가 큰 생성자 선택 → 인스턴스 하나 클릭 → Retainers를 루트 방향으로 추적 → 코드 위치 특정**
- `(closure)`가 늘면 패턴 ②, `EventEmitter`의 `_events` 아래 배열이 늘면 패턴 ③, `Map`·`Array`가 늘면 패턴 ①일 가능성이 높음
- `(string)`이 큰 경우 로그 버퍼, 응답 본문 캐시, 대용량 JSON 문자열을 의심함

> 💡 Retained Size로 정렬했을 때 최상단은 보통 `(GC roots)`·`system` 같은 내부 항목이다. 이들은 무시하고 **애플리케이션 클래스명이나 `(closure)`가 처음 나타나는 행**부터 보면 시간이 절약된다.

<br>

### 6. 예방 도구: 약한 참조

- **`WeakMap` / `WeakSet`**: 키 객체가 다른 곳에서 참조되지 않으면 항목이 자동 회수됨. "객체 → 부가 정보" 매핑(요청별 메타데이터 등)에 적합하며, 키는 객체만 가능하고 순회 불가
- **`WeakRef`**: 객체를 약하게 참조하고 `deref()`로 꺼냄. 캐시 값이 GC되어도 되는 상황에 사용
- **`FinalizationRegistry`**: 객체가 회수될 때 콜백 실행. 정리 시점이 보장되지 않으므로 필수 정리 로직에는 쓰지 않음
- 약한 참조는 "**참조 자체가 수명을 결정하는 구조**"를 깨기 위한 도구이지, 상한 없는 캐시의 면죄부가 아님. 일반 캐시는 여전히 LRU·TTL이 정답

<br>

### 7. 정리

- 누수는 "쓰지 않는 객체"가 아니라 **GC 루트에서 도달 가능한 채 남은 객체**이며, Old Space가 단조 증가하는 그래프로 드러남
- 3대 패턴: **상한 없는 전역 컬렉션**, **큰 스코프를 잡은 오래 사는 클로저(타이머)**, **제거되지 않는 이벤트 리스너**. 공통 구조는 "짧게 살 객체가 오래 사는 객체에 매달림"
- `MaxListenersExceededWarning`은 끄지 말고 원인을 찾는 신호로 사용
- 진단 흐름: `heapUsed` 추이 → `--trace-gc`로 GC 후 잔존 확인 → 힙 스냅샷 3-스냅샷 비교 → **Retainers로 루트까지 경로 추적**
- 스냅샷은 **Retained Size**와 **Delta**로 보고, Shallow Size는 참고만 함
- 운영 환경 스냅샷은 이벤트 루프를 멈추므로 트래픽을 뺀 인스턴스에서 촬영함
- 예방: LRU·TTL 캐시, 리스너·타이머 해제 경로, `AbortSignal`, `WeakMap`. 프로세스 재시작(unit08)은 증상 완화일 뿐 근본 해결이 아님
