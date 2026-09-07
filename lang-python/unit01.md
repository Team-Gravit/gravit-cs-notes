## GIL과 동시성 모델

**GIL(Global Interpreter Lock, 전역 인터프리터 락)**은 CPython에서 한 번에 하나의 스레드만 파이썬 바이트코드를 실행하도록 강제하는 뮤텍스다. 파이썬에서 멀티스레드가 CPU 작업을 빠르게 하지 못하는 이유이자, 멀티프로세싱·asyncio 중 무엇을 선택할지 결정하는 기준이 되므로 반드시 원리를 이해해야 한다.

<br>

### 1. GIL이 존재하는 이유

CPython은 객체마다 **레퍼런스 카운트(참조 횟수)**를 저장해 메모리를 관리한다(unit02 참고). 여러 스레드가 동시에 같은 객체의 카운트를 증감하면 경쟁 조건(Race Condition)이 생겨 객체가 너무 일찍 해제되거나 영원히 남을 수 있다.

- 객체마다 락을 두면 락 획득·해제 비용이 커지고 데드락 위험이 생김
- 대신 **인터프리터 전체에 락 하나**를 두면 단일 스레드 성능이 좋고 C 확장 모듈 작성이 단순해짐
- 결과적으로 단일 스레드 성능과 구현 단순성을 얻는 대신 **멀티코어 병렬성을 포기**한 설계임

> 💡 GIL은 파이썬 언어 명세가 아니라 **CPython 구현체의 선택**이다. Jython·IronPython에는 GIL이 없고, PyPy에는 있다. 면접에서 "파이썬은 GIL 때문에…"라고 답할 때 "CPython 기준"이라고 한정하면 정확하다.

<br>

### 2. GIL의 동작 방식

**스레드 전환 타임라인**

CPython 3.2 이후 스레드는 **일정 시간(기본 5ms, `sys.setswitchinterval`)**마다 GIL 반납을 요청받는다. 또한 파일 읽기·소켓 대기·`time.sleep` 같은 **블로킹 I/O 호출 직전에 GIL을 스스로 반납**한다.

```
시간 →
스레드 A: [GIL 보유: 바이트코드 실행]──반납──────대기──────[GIL 보유]──
스레드 B: ──────대기──────[GIL 보유: 바이트코드 실행]──반납──────대기──
                       ↑ switch interval(5ms) 경과 또는 I/O 진입 시 전환

CPU 코어 4개가 있어도 바이트코드를 실행하는 스레드는 항상 1개
```

**CPU 바운드 vs I/O 바운드**

| **작업 유형**    | **특징**                          | **GIL의 영향**                                   | **멀티스레드 효과** |
| ---------------- | --------------------------------- | ------------------------------------------------ | ------------------- |
| **CPU 바운드**   | 연산이 대부분 (수치 계산, 파싱)   | 스레드가 GIL을 두고 경쟁 → 사실상 직렬 실행      | **없음, 오히려 느려질 수 있음** |
| **I/O 바운드**   | 네트워크·디스크 대기가 대부분     | I/O 대기 중 GIL 반납 → 다른 스레드가 실행        | **있음**            |

```python
import threading, time

def cpu_task(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

N = 10_000_000

start = time.perf_counter()
cpu_task(N); cpu_task(N)
print(f"순차 실행: {time.perf_counter() - start:.2f}s")

start = time.perf_counter()
ts = [threading.Thread(target=cpu_task, args=(N,)) for _ in range(2)]
for t in ts: t.start()
for t in ts: t.join()
print(f"스레드 2개: {time.perf_counter() - start:.2f}s")  # 순차와 거의 같거나 더 느림
```

> ⚠️ CPU 바운드 작업을 스레드로 나누면 GIL 경합과 컨텍스트 스위칭 비용이 더해져 **단일 스레드보다 느려지는 경우**가 흔하다. "스레드를 늘렸는데 왜 느려졌나"는 GIL을 이해했는지 묻는 대표 질문이다.

<br>

### 3. GIL이 있어도 스레드 안전하지 않은 이유

GIL은 **바이트코드 한 줄 단위**로 원자성을 보장할 뿐, 파이썬 코드 한 줄을 원자적으로 만들지 않는다. `count += 1`은 LOAD → ADD → STORE 여러 바이트코드로 나뉘므로 중간에 스레드가 전환될 수 있다.

```python
import threading

count = 0
lock = threading.Lock()

def unsafe():
    global count
    for _ in range(100_000):
        count += 1          # 읽기-수정-쓰기 사이에 전환 가능 → 값 유실

def safe():
    global count
    for _ in range(100_000):
        with lock:          # 임계 구역을 명시적으로 보호
            count += 1
```

- `list.append`, `dict[key] = value` 같은 **단일 C 함수 호출**은 GIL 덕분에 원자적으로 동작함
- 그러나 "읽고 판단해서 쓰는" 복합 연산은 반드시 `Lock`·`RLock`·`queue.Queue` 등으로 보호해야 함

<br>

### 4. 동시성 도구 세 가지

### 4-1. threading — I/O 대기를 겹치기

- 스레드 생성 비용이 낮고 메모리를 공유하므로 데이터 전달이 쉬움
- I/O 대기 중 GIL을 반납하므로 **네트워크 요청 수십 개 병행** 같은 상황에 적합함
- 고수준 API인 `concurrent.futures.ThreadPoolExecutor`를 사용하면 풀 관리가 간단함

<br>

### 4-2. multiprocessing — GIL을 우회해 진짜 병렬 실행

프로세스마다 **독립된 인터프리터와 GIL**을 가지므로 CPU 코어를 모두 활용할 수 있다.

```python
from concurrent.futures import ProcessPoolExecutor

def cpu_task(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":            # 자식 프로세스가 모듈을 재import하므로 필수
    with ProcessPoolExecutor() as pool:
        results = list(pool.map(cpu_task, [10_000_000] * 4))
    print(results)
```

- 인자와 반환값은 **pickle로 직렬화**되어 프로세스 간에 복사되므로 큰 데이터를 자주 주고받으면 오히려 느려짐
- 프로세스 생성·메모리 비용이 스레드보다 훨씬 큼

> ⚠️ 시작 방식(start method)은 OS마다 다르다. Windows와 macOS는 `spawn`(새 인터프리터 실행)이 기본이고, Linux는 전통적으로 `fork`였으나 **3.14부터는 기본값이 `forkserver`로 바뀌었다**(3.12부터 멀티스레드 상태에서 fork 시 경고 발생). `if __name__ == "__main__":` 가드를 빠뜨리면 자식이 무한히 생성되는 사고가 난다.

<br>

### 4-3. asyncio — 단일 스레드 협력적 동시성

이벤트 루프 하나가 **코루틴을 번갈아 실행**하며, `await` 지점에서만 제어권을 넘긴다. 스레드가 하나이므로 GIL 경합·락이 거의 필요 없고, 수천 개의 동시 연결을 적은 메모리로 처리할 수 있다.

```python
import asyncio

async def fetch(i):
    await asyncio.sleep(1)          # 실제로는 aiohttp 등 비동기 I/O
    return i

async def main():
    results = await asyncio.gather(*(fetch(i) for i in range(1000)))
    print(len(results))             # 약 1초 만에 1000개 완료

asyncio.run(main())
```

❗️**블로킹 호출 금지**: 코루틴 안에서 `time.sleep`, `requests.get` 같은 동기 함수를 호출하면 이벤트 루프 전체가 멈춘다. 불가피하면 `await asyncio.to_thread(func)`로 스레드에 위임한다.

<br>

### 5. 선택 기준 비교

| **항목**          | **threading**            | **multiprocessing**       | **asyncio**                     |
| ----------------- | ------------------------ | ------------------------- | ------------------------------- |
| **적합한 작업**   | **I/O 바운드**(소규모)   | **CPU 바운드**            | **I/O 바운드**(대규모 연결)     |
| **병렬성**        | 없음 (GIL)               | **있음** (코어 수만큼)    | 없음 (단일 스레드)              |
| **메모리 공유**   | 공유 (락 필요)           | 독립 (IPC·pickle 필요)    | 공유 (락 거의 불필요)           |
| **전환 방식**     | 선점형 (인터프리터 결정) | OS 스케줄링               | **협력형** (`await` 지점)       |
| **라이브러리 요구** | 기존 동기 코드 그대로  | 함수가 pickle 가능해야 함 | **비동기 지원 라이브러리 필요** |
| **비용**          | 낮음                     | 높음                      | 매우 낮음                       |

```
작업이 CPU 바운드인가?
 ├─ 예 → multiprocessing (또는 NumPy·C 확장처럼 GIL을 놓는 라이브러리)
 └─ 아니오 (I/O 바운드)
      ├─ 동시 연결이 많고 비동기 라이브러리가 있는가? → asyncio
      └─ 동기 라이브러리만 있거나 규모가 작은가?     → threading
```

> 💡 NumPy·pandas 등 C로 작성된 라이브러리는 무거운 연산 중 **GIL을 직접 해제**하므로, 이런 연산은 스레드로도 병렬 효과를 얻는다. "파이썬은 무조건 멀티프로세싱"이 아니라 라이브러리 특성까지 고려해야 한다.

**GIL의 미래 — 버전별 변화**

| **버전**          | **변화**                                                                                     |
| ----------------- | -------------------------------------------------------------------------------------------- |
| **3.12**          | PEP 684 — **서브인터프리터별 GIL**. C API 수준에서 인터프리터마다 독립 GIL 보유 가능           |
| **3.13**          | PEP 703 — **free-threaded 빌드(실험적)**. `python3.13t`로 GIL 없이 실행, `sys._is_gil_enabled()`로 확인 |
| **3.14**          | PEP 734 — `concurrent.interpreters` 모듈로 서브인터프리터를 파이썬에서 직접 사용. free-threaded 빌드는 정식 지원 단계로 승격 |

- free-threaded 빌드는 **별도로 컴파일된 인터프리터**이며 기본 배포판에서는 GIL이 여전히 켜져 있음
- GIL을 없애는 대신 레퍼런스 카운팅을 스레드 안전하게 바꿔야 하므로 **단일 스레드 성능이 일부 저하**되고, 많은 C 확장 모듈이 아직 호환되지 않음
- 버전별 세부 동작과 성능 수치는 릴리스마다 바뀌므로 실무 도입 전 해당 버전 문서를 확인해야 함

<br>

### 6. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                 |
| ----------------------------------------------------- | ----------------------------------------------------------------------------- |
| **GIL이 뭔가요?**                                     | CPython에서 한 번에 한 스레드만 바이트코드를 실행하게 하는 락. 레퍼런스 카운팅 보호 목적 |
| **멀티스레드로 CPU 작업이 빨라지지 않는 이유는?**     | 스레드가 GIL을 두고 경쟁해 사실상 직렬 실행, 전환 비용까지 추가됨               |
| **그럼 스레드는 언제 쓰나요?**                        | I/O 대기 중 GIL을 반납하므로 **I/O 바운드** 작업에 유효                        |
| **GIL이 있으면 락이 필요 없나요?**                    | 아니다. `+=` 같은 복합 연산은 바이트코드 여러 개라 중간 전환이 가능함          |
| **multiprocessing 주의점은?**                         | pickle 직렬화 비용, `__main__` 가드, OS별 시작 방식 차이                        |
| **asyncio와 스레드의 차이는?**                        | 협력형 vs 선점형. asyncio는 `await`에서만 전환되며 블로킹 호출 시 루프 정지    |
| **3.13 free-threaded 빌드란?**                        | GIL을 제거한 실험적 별도 빌드. 기본 빌드는 여전히 GIL 사용                     |

- 스레드 안전성과 락의 일반 원리는 운영체제 과목(동기화·교착상태 유닛)을 함께 참고할 것
