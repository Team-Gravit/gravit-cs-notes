## 에러 핸들링과 프로세스 안정성

Node.js는 프로세스 하나가 수천 개의 요청을 동시에 처리하므로, 처리되지 않은 예외 하나가 **모든 연결을 함께 끊어 버린다**. 이 유닛은 동기·콜백·Promise 각각의 에러 전파 방식, `uncaughtException`과 `unhandledRejection`의 기본 동작, 크래시를 막는 전략, 그리고 배포·스케일 인 시 요청을 잃지 않는 **우아한 종료(Graceful Shutdown)**를 다룬다.

<br>

### 1. 에러의 두 종류

| **구분**            | **운영 오류 (Operational Error)**                     | **프로그래머 오류 (Programmer Error)**                |
| ------------------- | ----------------------------------------------------- | ----------------------------------------------------- |
| **정의**            | 정상 프로그램이 런타임에 만나는 **예상 가능한** 실패     | 코드의 **버그**                                        |
| **예시**            | DB 연결 실패, 타임아웃, 잘못된 입력, 파일 없음, 404      | `undefined.foo`, 타입 오류, 잘못된 인자 전달            |
| **대응**            | 재시도·대체 응답·사용자에게 4xx/5xx 반환                 | **즉시 실패시키고 고침**. 상태를 신뢰할 수 없음          |
| **프로세스 종료**   | 하지 않음                                              | 종료 후 재시작이 정석                                   |

- 이 구분(Joyent의 에러 처리 가이드)이 모든 전략의 출발점임. 운영 오류는 요청 단위로 처리하고, 프로그래머 오류는 프로세스를 재시작함
- "모든 에러를 잡아서 계속 실행"하면 버그가 발생한 뒤의 **오염된 상태**(반쯤 쓰인 파일, 잠긴 커넥션, 어긋난 카운터)로 계속 서비스하게 됨

<br>

### 2. 비동기 코드에서 에러가 전파되는 방식

```javascript
// ① 동기: try/catch가 동작
try { JSON.parse('{bad'); } catch (e) { /* 잡힘 */ }

// ② 콜백: try/catch가 무력함 — 콜백은 나중에 다른 스택에서 실행됨
try {
  fs.readFile('x', (err, data) => { if (err) throw err; }); // 여기서 throw → uncaughtException
} catch (e) { /* 절대 도달하지 않음 */ }

// ③ Promise / async-await: 거부(rejection)가 체인을 따라 전파, await에 try/catch 가능
async function load() {
  try { return await fs.promises.readFile('x'); }
  catch (e) { throw new AppError('LOAD_FAILED', { cause: e }); } // cause로 원인 보존 (Node 16.9+)
}
```

- 콜백 스타일은 **에러 우선 콜백(error-first callback)** 규약으로 첫 인자에 에러를 넘기며, 콜백 안에서 `throw`하면 잡을 곳이 없음
- Promise는 `.catch()`나 `await`+`try/catch`로만 잡히며, **어디에서도 잡지 않은 거부**가 `unhandledRejection`이 됨
- `EventEmitter`(스트림·소켓 포함)의 `'error'` 이벤트는 리스너가 없으면 **동기적으로 throw**되어 프로세스를 죽임 → 스트림 에러는 unit05의 `pipeline()`으로 처리

> ⚠️ `async` 함수를 `forEach`나 이벤트 핸들러에 넘기면 반환된 Promise를 아무도 기다리지 않아 거부가 **조용히 unhandledRejection**이 된다. `app.get('/x', async (req, res) => {...})`에서 던진 예외도 Express 4는 잡지 못한다(Express 5·NestJS·Fastify는 처리함). 프레임워크가 비동기 핸들러 예외를 어떻게 다루는지 반드시 확인한다.

<br>

### 3. uncaughtException과 unhandledRejection

```
동기 throw (잡히지 않음) ──▶ 'uncaughtException' 이벤트
                                ├─ 리스너 없음 → 스택 출력 후 종료 코드 1
                                └─ 리스너 있음 → 리스너 실행, 프로세스 계속 (위험)

Promise 거부 (잡히지 않음) ─▶ 'unhandledRejection' 이벤트
                                ├─ 리스너 없음 → Node 15+ 기본 throw 모드: uncaughtException으로 승격 → 종료
                                └─ 리스너 있음 → 리스너 실행, 프로세스 계속
```

- **Node 15 이전**에는 미처리 거부가 경고만 출력하고 넘어갔지만, **15부터 기본 모드가 `throw`**로 바뀌어 프로세스가 종료됨. `--unhandled-rejections=warn|strict|none` 플래그로 조정 가능(버전에 따라 다름)
- `uncaughtException` 리스너를 등록하면 종료를 막을 수 있지만, 공식 문서는 "**동기적으로 정리 작업만 하고 종료하라**"고 명시함. 예외가 어디서 났는지 모르므로 상태를 신뢰할 수 없음
- 종료 없이 관찰만 하려면 `'uncaughtExceptionMonitor'` 이벤트(Node 13.7+)를 사용함. 기본 종료 동작을 바꾸지 않음

```javascript
// 안티패턴: 로그만 남기고 계속 실행 → 오염된 상태로 서비스, 메모리·커넥션 누수 누적
process.on('uncaughtException', (err) => logger.error(err));
process.on('unhandledRejection', (err) => logger.error(err));

// 개선: 기록 → 우아한 종료 시도 → 타임아웃 후 강제 종료. 재시작은 프로세스 매니저에 맡김
process.on('unhandledRejection', (reason) => {
  throw reason; // uncaughtException 경로로 통일
});
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'uncaught exception, shutting down');
  shutdown(1); // 4절의 함수
});
```

> 💡 "프로세스를 죽이면 서비스가 끊기지 않나?"라는 질문에는 "**죽이지 않으면 더 큰 장애가 온다**"가 답이다. 클러스터·PM2·Kubernetes가 새 프로세스를 즉시 띄우고, 나머지 프로세스가 트래픽을 받는다. 단일 프로세스로 운영하는 것 자체가 문제다.

<br>

### 4. 크래시 방지 전략

**계층별 방어선**

| **계층**             | **수단**                                                       | **목적**                                     |
| -------------------- | -------------------------------------------------------------- | -------------------------------------------- |
| **입력 경계**        | 검증(파이프·스키마), 크기 제한, 타임아웃                          | 잘못된 입력이 깊은 곳에서 터지지 않게          |
| **요청 단위**        | 프레임워크 에러 핸들러, 예외 필터(unit07)                         | 운영 오류를 4xx/5xx 응답으로 변환             |
| **외부 의존성**      | 재시도(지수 백오프), 서킷 브레이커, 타임아웃, 커넥션 풀 에러 핸들러  | DB·외부 API 장애가 프로세스 장애로 번지지 않게 |
| **프로세스**         | `uncaughtException` 핸들러 + 우아한 종료                         | 상태 오염 없이 빠르게 재시작                  |
| **인프라**           | 클러스터(unit03), PM2, Kubernetes 재시작·헬스체크                 | 프로세스 하나의 죽음을 서비스 장애가 아니게    |

- 프로세스 시작 시 **필수 설정·연결을 검증**하고 실패하면 즉시 종료(fail fast)함. 잘못된 설정으로 "떠 있지만 동작하지 않는" 상태가 가장 진단하기 어려움
- 도메인 에러 클래스를 정의하고(`class NotFoundError extends AppError`), `isOperational` 같은 플래그로 운영 오류와 버그를 구분해 처리함
- `process.exitCode = 1`을 설정한 뒤 자연 종료되게 두면 `process.exit()`의 강제성 없이 종료 코드를 남길 수 있음

> ⚠️ `process.exit()`는 이벤트 루프를 즉시 멈추므로 **아직 플러시되지 않은 로그·응답이 유실**된다. 정리 작업 후 종료하되, 정리가 끝나지 않을 때만 타임아웃으로 강제 종료하는 구조가 필요하다.

<br>

### 5. 우아한 종료(Graceful Shutdown)

배포·오토스케일링 시 오케스트레이터는 프로세스에 **SIGTERM**을 보내고 일정 시간(Kubernetes 기본 30초) 후 SIGKILL로 강제 종료한다. 그 사이에 진행 중인 요청을 끝내고 자원을 정리하는 것이 우아한 종료다.

```
SIGTERM 수신
   │
   ├─ ① 새 연결 수락 중단 (server.close) + 헬스체크 실패 응답 → LB가 트래픽 제외
   ├─ ② 유휴 keep-alive 연결 끊기 (closeIdleConnections) — 안 끊으면 close가 완료되지 않음
   ├─ ③ 진행 중인 요청 완료 대기
   ├─ ④ DB 풀·메시지 큐·캐시 연결 종료, 로그 플러시
   ├─ ⑤ process.exit(0)
   └─ (타임아웃, 예: 10초) 미완료라도 강제 종료
```

```javascript
const server = app.listen(3000);
let shuttingDown = false;

function shutdown(code = 0) {
  if (shuttingDown) return;          // 중복 신호 방지
  shuttingDown = true;

  const force = setTimeout(() => process.exit(code || 1), 10_000).unref(); // unref: 타이머가 종료를 막지 않게

  server.close(async () => {          // ① 새 연결 거부, 기존 요청 완료 후 콜백
    await db.end();                   // ④ 자원 정리
    clearTimeout(force);
    process.exit(code);
  });
  server.closeIdleConnections();      // ② Node 18.2+ : 유휴 keep-alive 연결 정리
}

process.on('SIGTERM', () => shutdown(0));
process.on('SIGINT', () => shutdown(0));
```

- `server.close()`는 **새 연결만** 막고, 열려 있는 keep-alive 연결이 모두 닫힐 때까지 콜백을 호출하지 않음. `closeIdleConnections()`가 없으면 종료가 SIGKILL까지 지연됨
- 헬스체크 엔드포인트가 종료 중에는 실패를 반환하게 하여 로드밸런서가 먼저 트래픽을 빼도록 함. Kubernetes에서는 `preStop` 훅으로 몇 초 대기해 엔드포인트 갱신 시간을 확보하는 방식이 흔함
- NestJS는 `app.enableShutdownHooks()`를 호출하면 SIGTERM 시 `OnModuleDestroy → beforeApplicationShutdown → OnApplicationShutdown` 훅을 순서대로 실행하며, TypeORM·Prisma 등의 연결 종료를 여기서 처리함
- 작업 큐 워커라면 "새 작업 가져오기 중단 → 처리 중인 작업 완료 → 종료" 순서가 동일하게 적용됨

<br>

### 6. 면접·실무 체크포인트

- 에러를 **운영 오류(요청 단위 처리)**와 **프로그래머 오류(프로세스 재시작)**로 구분하는 것이 출발점
- 콜백 안의 `throw`는 `try/catch`로 못 잡고, Promise 거부는 `.catch`/`await`로만 잡힘. 미처리 거부는 **Node 15+에서 프로세스를 종료**함
- `uncaughtException` 핸들러는 **정리 후 종료** 용도이지, 계속 실행하기 위한 것이 아님. 관찰만 하려면 `uncaughtExceptionMonitor`
- `EventEmitter`·스트림의 `'error'` 미처리는 동기 throw로 이어지므로 `pipeline()`과 에러 리스너가 필수
- 프레임워크가 `async` 핸들러의 예외를 잡는지 확인 (Express 4는 미지원, NestJS·Fastify·Express 5는 지원)
- 우아한 종료: **SIGTERM → 수락 중단 → 유휴 연결 정리 → 진행 요청 완료 → 자원 정리 → exit**, 타임아웃으로 강제 종료 보장
- 재시작은 프로세스 매니저·오케스트레이터의 책임이며, 단일 프로세스 운영 자체가 가용성 문제임
- 메모리가 서서히 늘다 OOM으로 죽는 경우는 예외가 아닌 누수 문제이므로 **unit09**로 진단함
