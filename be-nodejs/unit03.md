## CPU 바운드 작업 처리

Node.js의 이벤트 루프(unit01)는 I/O 대기에는 강하지만, **CPU를 오래 점유하는 계산**이 끼어들면 모든 요청이 함께 멈춘다. 이 유닛은 이벤트 루프 블로킹을 **진단**하는 방법과, 작업을 분리하는 세 가지 수단인 **워커 스레드(Worker Threads)**·**클러스터(Cluster)**·**자식 프로세스(Child Process)**의 선택 기준을 다룬다.

<br>

### 1. I/O 바운드 vs CPU 바운드

| **구분**         | **I/O 바운드**                               | **CPU 바운드**                                        |
| ---------------- | -------------------------------------------- | ----------------------------------------------------- |
| **병목**         | 디스크·네트워크·DB 응답 대기                  | **연산 자체** (CPU 사이클)                             |
| **예시**         | HTTP 호출, DB 쿼리, 파일 읽기                 | 이미지 리사이징, 암호화, 대용량 JSON 파싱, 정렬·집계, 압축 |
| **Node.js 적합성** | 매우 높음 (논블로킹 I/O)                      | **낮음** — 단일 스레드가 독점됨                        |
| **대응**         | 비동기 API, 커넥션 풀                         | 분할 실행, 워커 스레드, 클러스터, 외부 서비스           |

- 이벤트 루프는 "콜백 하나가 짧다"는 가정 위에서 동작함. CPU 바운드 콜백 하나가 500ms를 쓰면 그 사이 도착한 **모든 요청의 지연이 500ms 증가**함
- 서버가 "가끔 몇 초씩 멈춘다"면 대부분 특정 요청의 CPU 작업이 원인임

<br>

### 2. 이벤트 루프 블로킹 진단

**증상과 측정 지표**

- **이벤트 루프 지연(Event Loop Lag/Delay)**: "타이머를 1ms 뒤에 예약했는데 실제로는 얼마 뒤에 실행됐는가". 이 값이 커지면 루프가 막힌 것
- **이벤트 루프 사용률(Utilization, ELU)**: 루프가 대기하지 않고 바쁘게 일한 시간의 비율. 1.0에 가까우면 포화 상태
- 헬스체크는 응답하지만 p99 지연이 급등, CPU 사용률은 코어 하나만 100%인 패턴이 전형적임

```javascript
const { monitorEventLoopDelay, performance } = require('node:perf_hooks');

const h = monitorEventLoopDelay({ resolution: 20 }); // 20ms 간격으로 샘플링
h.enable();

setInterval(() => {
  const elu = performance.eventLoopUtilization();
  console.log({
    p50ms: (h.percentile(50) / 1e6).toFixed(1),
    p99ms: (h.percentile(99) / 1e6).toFixed(1),
    maxms: (h.max / 1e6).toFixed(1),
    utilization: elu.utilization.toFixed(2),
  });
  h.reset();
}, 10_000);
```

> 💡 운영 환경에서는 위 지표를 Prometheus 등으로 내보내 **p99 지연 > 100ms** 같은 임계값에 알림을 건다. APM(Datadog, New Relic 등)도 "event loop delay" 지표를 기본 제공한다.

**원인 위치 찾기**

- `node --cpu-prof app.js`: 종료 시 `.cpuprofile` 파일 생성 → Chrome DevTools Performance 탭에서 어떤 함수가 CPU를 오래 썼는지 확인
- `node --inspect`로 DevTools를 붙여 실시간 프로파일링
- 루프 지연이 특정 임계값을 넘으면 스택을 기록하는 `blocked-at` 같은 패키지, 종합 진단 도구 `clinic.js`(doctor·flame)
- 의심 구간을 `performance.now()`로 감싸 직접 측정하는 것도 충분히 유효함

> ⚠️ `console.log`로 큰 객체를 찍는 것도 CPU 작업이다. 프로파일링 결과에서 로깅·직렬화(`JSON.stringify`)가 상위에 오는 경우가 의외로 많다.

<br>

### 3. 대응 전략 개요

```
CPU 작업이 이벤트 루프를 막는다
        │
        ├─ 수십 ms 수준, 나눌 수 있음 ──▶ ① 분할 실행 (setImmediate 청크)
        │
        ├─ 수백 ms 이상, 같은 프로세스에서 처리 ──▶ ② worker_threads (스레드 분리)
        │
        ├─ 코어를 다 쓰고 싶음, HTTP 요청 처리량 자체를 늘림 ──▶ ③ cluster / PM2 (프로세스 복제)
        │
        └─ 외부 실행 파일·다른 언어·격리 필요 ──▶ ④ child_process / 별도 서비스(큐)
```

**① 분할 실행**은 가장 가벼운 방법으로, 긴 반복을 조각내고 조각 사이에 `setImmediate`로 I/O에 양보한다 (마이크로태스크로 양보하면 안 되는 이유는 unit02 참고).

```javascript
// 안티패턴: 100만 건을 한 번에 처리 → 처리 시간 동안 모든 요청 정지
function sumAll(items) {
  return items.reduce((acc, v) => acc + heavy(v), 0);
}

// 개선: 1,000건씩 처리하고 setImmediate로 양보
function sumChunked(items, chunk = 1000) {
  return new Promise((resolve) => {
    let i = 0, acc = 0;
    (function next() {
      const end = Math.min(i + chunk, items.length);
      for (; i < end; i++) acc += heavy(items[i]);
      i < items.length ? setImmediate(next) : resolve(acc);
    })();
  });
}
```

- 분할은 **총 처리 시간을 줄이지 않고** 지연을 분산할 뿐이므로, 작업량 자체가 크면 결국 스레드·프로세스 분리가 필요함

<br>

### 4. 워커 스레드(worker_threads)

`worker_threads` 모듈은 **같은 프로세스 안에 별도의 V8 인스턴스와 이벤트 루프**를 가진 스레드를 만든다. 메인 스레드의 이벤트 루프는 계속 요청을 받고, 계산은 워커가 맡는다.

```javascript
// main.js
const { Worker } = require('node:worker_threads');

function fibInWorker(n) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./fib-worker.js', { workerData: n });
    worker.once('message', resolve);
    worker.once('error', reject);
    worker.once('exit', (code) => code !== 0 && reject(new Error(`exit ${code}`)));
  });
}

app.get('/fib/:n', async (req, res) => {
  res.json({ result: await fibInWorker(Number(req.params.n)) }); // 계산 중에도 다른 요청 처리 가능
});
```

```javascript
// fib-worker.js
const { parentPort, workerData } = require('node:worker_threads');
const fib = (n) => (n < 2 ? n : fib(n - 1) + fib(n - 2));
parentPort.postMessage(fib(workerData));
```

**동작 특성**

- 워커는 메모리(힙)를 공유하지 않으며 `postMessage`로 데이터를 **구조적 복제(structured clone)**해 전달함 → 큰 객체를 주고받으면 복사 비용이 큼
- `ArrayBuffer`는 `transferList`로 **소유권 이전**(복사 없음), `SharedArrayBuffer`+`Atomics`로 진짜 공유 메모리 사용 가능
- 워커 생성 비용(수십 ms, 수 MB)이 있으므로 요청마다 새로 만들지 말고 **워커 풀**을 유지함. 직접 구현하기보다 `piscina` 같은 검증된 풀 라이브러리를 사용함

> 💡 워커 개수는 보통 **CPU 코어 수(`os.availableParallelism()`)** 전후로 잡는다. 코어보다 훨씬 많이 만들면 컨텍스트 스위칭만 늘어나고, 메인 스레드가 쓸 CPU도 남겨두어야 한다.

<br>

### 5. 클러스터(cluster)와 프로세스 매니저

`cluster` 모듈은 프로세스를 **코어 수만큼 복제(fork)**해 같은 포트로 들어오는 요청을 나눠 받게 한다. 한 프로세스가 CPU 작업으로 막혀도 나머지 워커 프로세스가 요청을 처리한다.

```javascript
const cluster = require('node:cluster');
const os = require('node:os');

if (cluster.isPrimary) {
  for (let i = 0; i < os.availableParallelism(); i++) cluster.fork();
  cluster.on('exit', (worker) => {
    console.log(`worker ${worker.process.pid} 종료 → 재시작`);
    cluster.fork(); // 크래시 시 자동 복구
  });
} else {
  require('./server'); // 각 워커가 동일한 포트로 listen
}
```

- 프라이머리 프로세스가 연결을 받아 워커에 분배함 (Linux·macOS 기본은 **라운드 로빈**, Windows는 운영체제 위임 — 버전·플랫폼에 따라 다를 수 있음)
- 프로세스 간 메모리가 완전히 분리되므로 **인메모리 세션·캐시·카운터가 워커마다 따로 존재**함 → Redis 등 외부 저장소로 공유해야 함
- 실무에서는 직접 `cluster`를 짜기보다 **PM2 클러스터 모드**나 Kubernetes의 레플리카로 같은 효과를 얻음. 컨테이너 환경에서는 "컨테이너 1개 = 프로세스 1개"로 두고 오케스트레이터가 스케일하는 방식이 일반적임

> ⚠️ 클러스터는 "느린 요청이 다른 요청을 막는 문제"를 **확률적으로만** 완화한다. 워커 8개 중 하나가 3초 막히면 그 워커에 배정된 요청들은 여전히 3초 기다린다. 특정 엔드포인트의 무거운 계산이 원인이라면 클러스터가 아니라 **워커 스레드나 별도 작업 큐**로 옮기는 것이 정답이다.

<br>

### 6. 워커 스레드 vs 클러스터 vs 자식 프로세스

| **항목**           | **worker_threads**                        | **cluster**                                 | **child_process**                           |
| ------------------ | ----------------------------------------- | ------------------------------------------- | ------------------------------------------- |
| **실행 단위**      | 같은 프로세스의 **스레드**                 | 별도 **Node 프로세스** (같은 코드)           | 별도 프로세스 (임의의 명령·스크립트)          |
| **메모리**         | 분리 (SharedArrayBuffer로 공유 가능)       | 완전 분리                                   | 완전 분리                                   |
| **통신 비용**      | 낮음 (구조적 복제·전송)                    | IPC (직렬화)                                | IPC / stdio (직렬화)                         |
| **생성 비용**      | 낮음~중간                                 | 높음 (V8 + 런타임 전체)                      | 높음                                        |
| **주 목적**        | **CPU 작업 오프로딩**                      | **HTTP 처리량 확장**, 멀티코어 활용           | 외부 프로그램 실행, 격리, 다른 언어           |
| **장애 격리**      | 워커 크래시가 메인에 영향 없음 (이벤트로 통지) | 프로세스 단위 격리, 재시작 용이              | 완전 격리                                   |
| **대표 도구**      | piscina, workerpool                       | PM2, k8s 레플리카                            | `spawn`, `exec`, `fork`                     |

<br>

### 7. 선택 기준

- **짧고 나눌 수 있는 작업** → 분할 실행으로 충분. 코드 변경이 가장 적음
- **특정 API의 무거운 계산** (이미지 처리, 해시, 압축, 대용량 파싱) → **워커 스레드 풀**. HTTP 처리 스레드와 계산 스레드를 분리하는 것이 목적
- **전체 처리량이 부족**하고 코어가 놀고 있음 → **클러스터/레플리카**. 상태를 외부 저장소로 빼는 것이 전제
- **수 초~수 분짜리 배치, 재시도·모니터링 필요** → 프로세스 밖 **작업 큐(BullMQ 등) + 워커 서비스**. HTTP 응답은 즉시 "접수됨"으로 돌려주고 결과는 폴링·알림으로 전달
- **외부 바이너리(ffmpeg 등) 호출** → `child_process.spawn`. 출력을 스트림으로 받아 메모리 폭증을 막음 (unit05)
- 두 방법은 배타적이지 않음. 클러스터로 코어를 채우고, 각 프로세스 안에서 워커 풀을 두는 조합이 흔함

<br>

### 8. 정리

- CPU 바운드 콜백 하나가 이벤트 루프를 막으면 **모든 요청의 지연이 함께 증가**함
- 진단은 `monitorEventLoopDelay`(지연)·`eventLoopUtilization`(사용률)로 감지하고, `--cpu-prof`로 원인 함수를 찾음
- 대응 수단: **분할 실행(setImmediate) → 워커 스레드 → 클러스터 → 별도 프로세스·작업 큐** 순으로 무거워짐
- 워커 스레드는 계산 오프로딩, 클러스터는 처리량 확장이 목적이며 클러스터는 느린 요청 문제를 근본적으로 해결하지 못함
- 워커·프로세스 간 데이터는 복사되므로 큰 데이터는 전송(transfer)·공유 메모리·외부 저장소를 고려함
- 워커는 요청마다 생성하지 말고 **풀**로 재사용하며, 개수는 코어 수를 기준으로 잡음
