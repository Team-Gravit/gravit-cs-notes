## 스트림과 백프레셔

**스트림(Stream)**은 데이터를 한 번에 메모리에 올리지 않고 **작은 조각(chunk)** 단위로 흘려보내며 처리하는 추상화다. 수 GB 파일이나 대용량 HTTP 응답을 다룰 때 메모리를 일정하게 유지하는 핵심 도구이며, 생산 속도와 소비 속도가 다를 때 이를 조절하는 **백프레셔(Backpressure)**를 이해하지 못하면 메모리 폭증으로 이어진다.

<br>

### 1. 왜 스트림인가

```javascript
const fs = require('node:fs');

// 안티패턴: 파일 전체를 메모리에 올린 뒤 응답 → 동시 요청 수 × 파일 크기만큼 메모리 사용
app.get('/video', async (req, res) => {
  const buf = await fs.promises.readFile('movie.mp4'); // 2GB 파일이면 2GB 버퍼
  res.end(buf);
});

// 개선: 청크 단위로 읽어 곧바로 전송 → 메모리 사용량은 버퍼 크기(수십 KB) 수준
app.get('/video', (req, res) => {
  fs.createReadStream('movie.mp4').pipe(res);
});
```

- 전체 로딩 방식은 **처리 시작이 파일 끝을 읽은 뒤**이고, 메모리 사용량이 데이터 크기에 비례함
- 스트림은 첫 청크가 도착하는 즉시 처리를 시작하며(**시간 효율**), 메모리 사용량이 버퍼 크기로 고정됨(**공간 효율**)
- HTTP 요청·응답, 소켓, `process.stdin/stdout`, `zlib`, 파일 등 Node.js의 I/O 객체 대부분이 스트림임

<br>

### 2. 스트림의 네 가지 종류

| **종류**        | **역할**                         | **핵심 메서드·이벤트**              | **예시**                                        |
| --------------- | -------------------------------- | ----------------------------------- | ----------------------------------------------- |
| **Readable**    | 데이터를 **읽어오는** 원천        | `on('data')`, `read()`, `pipe()`     | `fs.createReadStream`, HTTP 요청(`req`)          |
| **Writable**    | 데이터를 **써 넣는** 목적지       | `write()`, `end()`, `'drain'`        | `fs.createWriteStream`, HTTP 응답(`res`)         |
| **Duplex**      | 읽기·쓰기가 **독립적으로** 모두 가능 | 양쪽 인터페이스                     | TCP 소켓(`net.Socket`)                           |
| **Transform**   | 입력을 **변환**해 출력하는 Duplex | `_transform(chunk, enc, cb)`         | `zlib.createGzip`, 암호화, CSV 파서              |

- 모든 스트림은 `EventEmitter`를 상속하므로 `'data'`, `'end'`, `'error'`, `'finish'`, `'close'` 이벤트로 상태를 알림
- 기본은 바이너리(`Buffer`)·문자열 단위이며, `objectMode: true`면 임의의 JS 객체를 청크로 흘릴 수 있음

<br>

### 3. Readable의 두 가지 모드와 highWaterMark

```
                 paused 모드                        flowing 모드
        ┌────────────────────────┐        ┌────────────────────────┐
  원천 ─▶│ 내부 버퍼 (≤ highWaterMark)│ ─read()─▶│ 'data' 이벤트로 자동 방출  │─▶ 소비자
        └────────────────────────┘        └────────────────────────┘
   버퍼가 찰 때까지만 읽고 대기          on('data') / pipe() / resume() 호출 시 전환
```

- **paused 모드(기본)**: 소비자가 `read()`를 호출할 때만 데이터를 내어줌. 내부 버퍼가 `highWaterMark`에 도달하면 원천에서 더 읽지 않음
- **flowing 모드**: `'data'` 리스너 등록·`pipe()`·`resume()` 호출 시 전환되며, 데이터가 도착하는 대로 이벤트로 밀어냄. 소비자가 느려도 멈추지 않으므로 **백프레셔를 직접 처리해야 함**
- **highWaterMark**: 내부 버퍼의 목표 상한. 바이트 스트림 기본값은 **Node 20 기준 16KiB, Node 22부터 64KiB**로 상향되었으며(버전에 따라 다름), objectMode는 객체 16개. 하드 리밋이 아니라 "이 이상 쌓이면 읽기를 잠시 멈추라"는 기준선임

> 💡 `highWaterMark`를 키우면 시스템 콜 횟수가 줄어 처리량이 오르지만, 스트림 하나당 메모리 사용량이 그만큼 커진다. 동시 연결이 수천 개인 서버라면 곱셈 효과를 감안해야 한다.

<br>

### 4. 백프레셔: 소비 속도 불균형 다루기

**백프레셔**는 소비자(Writable)가 처리하지 못하는 속도로 생산자(Readable)가 데이터를 밀어 넣을 때, 소비자가 "잠시 멈춰라"는 신호를 되돌려 보내는 메커니즘이다. 디스크 읽기(수백 MB/s)와 네트워크 전송(수 MB/s)처럼 속도 차가 큰 구간에서 반드시 필요하다.

```
       빠른 생산자                   느린 소비자
   fs.createReadStream  ──chunk──▶  res (네트워크)
                                    │ 내부 버퍼가 highWaterMark 초과
                                    ▼
   write() 가 false 반환 ◀────────── "그만 보내"
   → readable.pause()
                                    │ 버퍼를 다 비움
                                    ▼
   'drain' 이벤트 ◀──────────────── "다시 보내도 됨"
   → readable.resume()
```

**규칙**

- `writable.write(chunk)`의 반환값이 `false`이면 내부 버퍼가 `highWaterMark`를 넘었다는 뜻 → 쓰기를 멈추고 `'drain'` 이벤트를 기다림
- 반환값을 무시하고 계속 `write()`해도 오류는 나지 않음. 대신 **버퍼가 무한히 자라** 메모리가 폭증함
- `pipe()`와 `pipeline()`은 이 pause/drain 연동을 **자동으로** 처리함

```javascript
// 안티패턴: write() 반환값 무시 → 소비자가 느리면 메모리에 데이터가 계속 쌓임
for (let i = 0; i < 1e7; i++) {
  writable.write(`line ${i}\n`);
}

// 개선: false를 받으면 'drain'까지 대기
async function writeAll(writable, count) {
  for (let i = 0; i < count; i++) {
    if (!writable.write(`line ${i}\n`)) {
      await new Promise((resolve) => writable.once('drain', resolve));
    }
  }
  writable.end();
}
```

> ⚠️ "스트림을 쓰고 있으니 메모리는 안전하다"는 착각이 흔하다. `on('data')` 안에서 `res.write()`를 호출하고 반환값을 무시하면 스트림을 쓰고도 파일 전체를 메모리에 올린 것과 같아진다. 직접 연결할 때는 항상 `pipeline()`을 쓰거나 `write()` 반환값을 확인해야 한다.

<br>

### 5. pipe vs pipeline

| **항목**             | **`readable.pipe(writable)`**                | **`stream.pipeline(...streams, cb)`** / `stream/promises` |
| -------------------- | -------------------------------------------- | --------------------------------------------------------- |
| **백프레셔**         | 자동 처리                                    | 자동 처리                                                  |
| **에러 전파**        | **전파되지 않음** — 각 스트림에 개별 핸들러 필요 | 어느 스트림에서 나든 **모두 파괴(destroy)하고 콜백에 전달**   |
| **리소스 정리**      | 에러 시 원천 스트림이 열린 채 남을 수 있음(누수) | 파일 디스크립터·소켓 자동 정리                               |
| **Promise 지원**     | 없음                                         | `require('node:stream/promises').pipeline`                 |
| **권장**             | 간단한 데모                                   | **실무 표준**                                               |

```javascript
const { pipeline } = require('node:stream/promises');
const fs = require('node:fs');
const zlib = require('node:zlib');

// 파일 읽기 → gzip 압축 → 파일 쓰기. 어느 단계가 실패해도 전부 정리되고 예외로 전달됨
async function compress(src, dest) {
  await pipeline(
    fs.createReadStream(src),
    zlib.createGzip(),
    fs.createWriteStream(dest),
  );
}
```

- `pipe()` 체인에서 중간 스트림이 에러를 내면 상류 스트림은 계속 열려 있어 **파일 디스크립터 누수**가 생김. Node 10에서 이 문제를 해결하려고 `pipeline()`이 도입됨

<br>

### 6. Transform 스트림과 async iterator

가공 로직은 `Transform`으로 작성하면 파이프라인 중간에 끼워 넣을 수 있다. Node 10+에서는 Readable이 **async iterable**이기도 하므로 `for await`로 소비하는 방식도 널리 쓰인다.

```javascript
const { Transform } = require('node:stream');

// 줄 단위로 대문자 변환하는 Transform
const upperLines = new Transform({
  transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase()); // (에러, 출력 청크)
  },
});

// async iterator로 소비: 백프레셔가 자동 적용됨 (다음 청크는 await가 끝난 뒤 읽음)
async function countLines(path) {
  let lines = 0;
  for await (const chunk of fs.createReadStream(path, { encoding: 'utf8' })) {
    lines += chunk.split('\n').length - 1;
  }
  return lines;
}
```

- `callback`을 호출하기 전까지 다음 청크가 들어오지 않으므로 Transform 자체가 백프레셔에 참여함
- `for await` 안에서 느린 비동기 작업(DB 삽입 등)을 `await`하면 그동안 읽기가 자연스럽게 멈춤 → 별도 처리 없이 소비 속도에 맞춰짐
- `stream.Readable.from(iterable)`로 배열·제너레이터를 스트림으로 바꿀 수 있음

<br>

### 7. 실무 함정 체크리스트

| **함정**                                   | **결과**                                   | **대응**                                                  |
| ------------------------------------------ | ------------------------------------------ | --------------------------------------------------------- |
| `'error'` 리스너 미등록                    | 예외가 `uncaughtException`으로 승격되어 프로세스 종료 (unit08) | `pipeline()` 사용 또는 모든 스트림에 에러 핸들러         |
| `write()` 반환값 무시                      | 메모리 폭증                                 | `drain` 대기 또는 `pipeline()`                            |
| `pipe()` 체인의 에러                       | 상류 스트림·파일 디스크립터 누수            | `pipeline()`으로 교체                                     |
| 청크 경계를 레코드 경계로 착각              | 줄·JSON이 청크 중간에서 잘림                | 줄 단위 파서(`readline`, 전용 파서)로 버퍼링 후 분리        |
| objectMode 스트림에 큰 객체 16개           | 예상보다 많은 메모리                        | `highWaterMark`를 객체 크기에 맞게 조정                    |
| 스트림을 소비하지 않고 방치                 | 원천이 paused로 멈춰 있어 요청이 끝나지 않음  | 사용하지 않는 `req` 본문은 `resume()`·`destroy()` 처리     |

> 💡 HTTP 파일 업로드에서 `req`를 스트림으로 바로 S3 등 저장소에 `pipeline`으로 넘기면 서버 메모리·디스크를 거치지 않는다. 반대로 `multer`처럼 메모리 스토리지에 모으는 방식은 파일 크기 × 동시 업로드 수만큼 메모리를 쓰므로 크기 제한이 필수다.

<br>

### 8. 정리

- 스트림은 데이터를 **청크 단위로 흘려** 메모리 사용량을 고정하고 처리 시작을 앞당김
- 4종류: **Readable · Writable · Duplex · Transform**, 모두 `EventEmitter` 기반
- Readable은 paused/flowing 모드가 있고, `highWaterMark`가 버퍼 기준선 (기본값은 버전에 따라 16~64KiB)
- **백프레셔**: `write()`가 `false`를 반환하면 `'drain'`까지 멈춰야 하며, 무시하면 메모리가 폭증함
- `pipe()`는 에러 전파·정리가 안 되므로 실무에서는 **`pipeline()`**(Promise 버전 포함)이 표준
- `Transform`과 `for await`는 백프레셔에 자연스럽게 참여하므로 가공·소비 로직의 기본 형태로 삼음
- 스트림의 `'error'` 미처리는 프로세스 크래시로 이어지므로 unit08의 에러 처리 전략과 함께 봐야 함
