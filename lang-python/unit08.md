## 예외와 컨텍스트 매니저

파이썬은 오류를 **예외(Exception) 객체를 던지고 잡는 방식**으로 처리하며, 파일·락·커넥션처럼 반드시 되돌려줘야 하는 자원은 **컨텍스트 매니저(Context Manager)**와 `with` 문으로 해제를 보장한다. 예외가 어디서 어떻게 전파되는지, `finally`와 `__exit__`가 어떤 순서로 실행되는지를 알아야 자원 누수·예외 삼킴 같은 실무 버그를 막을 수 있다.

<br>

### 1. 예외 처리의 기본 구조

```python
try:
    f = open("data.txt")
    value = int(f.readline())
except FileNotFoundError:
    print("파일이 없습니다")            # 구체적 예외부터 위에
except ValueError as e:
    print(f"숫자가 아님: {e}")          # e는 블록이 끝나면 자동으로 삭제됨
else:
    print(value)                        # 예외가 없을 때만 실행 (try 본문을 최소화)
finally:
    print("항상 실행")                  # 예외 여부·return 여부와 무관하게 실행
```

```
try 본문 실행
 ├─ 예외 없음 ──▶ else 블록 ──┐
 └─ 예외 발생 ──▶ 일치하는 except 탐색 (위에서 아래로)
                  ├─ 있음 ──▶ except 블록 ──┤
                  └─ 없음 ──▶ (상위로 전파) ─┤
                                            ▼
                                       finally 블록 ──▶ 다음 문장 또는 계속 전파
```

- `except` 절은 **위에서 아래로 첫 번째 일치**를 택하므로 하위 클래스 예외를 먼저 적어야 함
- `try` 본문에는 **예외가 날 수 있는 최소한의 코드**만 두고 나머지는 `else`로 옮기면 어떤 줄이 어떤 예외를 내는지 명확해짐

> ⚠️ `finally` 안에서 `return`·`break`를 쓰면 진행 중이던 예외가 **조용히 사라진다**. 3.14부터는 이 패턴에 `SyntaxWarning`이 발생한다(PEP 765). `finally`는 정리 작업만 수행해야 한다.

<br>

### 2. 예외 계층과 잡는 범위

```
BaseException
├── SystemExit, KeyboardInterrupt, GeneratorExit   ← 프로그램 제어용, 잡지 말 것
└── Exception                                       ← 일반 오류의 뿌리
    ├── ArithmeticError ── ZeroDivisionError
    ├── LookupError ── IndexError, KeyError
    ├── OSError ── FileNotFoundError, PermissionError, TimeoutError
    ├── ValueError, TypeError, AttributeError
    └── RuntimeError ── RecursionError, NotImplementedError
```

| **작성 방식**                     | **평가**       | **이유**                                                                  |
| --------------------------------- | -------------- | ------------------------------------------------------------------------- |
| **`except FileNotFoundError:`**   | **권장**       | 처리 방법을 아는 예외만 잡음                                              |
| **`except (KeyError, IndexError):`** | 권장        | 같은 방식으로 처리할 예외를 튜플로 묶음                                   |
| **`except Exception:`**           | 조건부         | 최상위(요청 핸들러·워커 루프)에서 로깅 후 계속할 때만                     |
| **`except:` (bare)**              | **금지**       | `KeyboardInterrupt`·`SystemExit`까지 삼켜 Ctrl+C·종료가 안 됨              |
| **`except Exception: pass`**      | **금지**       | 오류를 숨겨 원인 추적 불가. 최소한 `logging.exception()`으로 기록          |

```python
class AppError(Exception):                      # 프로젝트 공통 뿌리 예외
    """애플리케이션 예외의 기반."""

class NotFoundError(AppError):
    def __init__(self, resource: str, key):
        super().__init__(f"{resource} {key!r}를 찾을 수 없음")
        self.resource, self.key = resource, key  # 핸들러가 활용할 구조화된 정보

try:
    raise NotFoundError("User", 42)
except AppError as e:                            # 뿌리 예외 하나로 도메인 오류 전체를 처리
    print(e, e.resource)
```

> 💡 사용자 정의 예외는 **`Exception`을 상속한 뿌리 클래스 하나**를 두고 그 아래로 계층을 만든다. 호출자는 세부 예외를 몰라도 뿌리 예외로 도메인 오류 전체를 처리할 수 있고, 라이브러리 내부 변경이 호출자 코드를 깨지 않는다.

<br>

### 3. 예외 연쇄와 재전파

```python
def load_config(path):
    try:
        with open(path) as f:
            return parse(f.read())
    except OSError as e:
        raise AppError(f"설정 로드 실패: {path}") from e   # 명시적 연쇄 → __cause__

def retry_wrapper():
    try:
        risky()
    except TimeoutError:
        log.warning("타임아웃, 재전파")
        raise                                   # 원본 트레이스백을 유지한 채 그대로 재전파
```

| **형태**                  | **트레이스백 표시**                                                   | **용도**                                            |
| ------------------------- | --------------------------------------------------------------------- | --------------------------------------------------- |
| **`raise`**               | 원본 그대로                                                           | 로깅·정리 후 같은 예외 재전파                       |
| **`raise New() from e`**  | "The above exception was the direct cause of…" (`__cause__`)          | 저수준 예외를 도메인 예외로 **번역**                |
| **`raise New()` (except 안)** | "During handling…, another exception occurred" (`__context__`)    | 암묵적 연쇄 — 실수로 생기는 경우가 많음             |
| **`raise New() from None`** | 원본 숨김                                                            | 내부 구현을 노출하고 싶지 않을 때 (원인 추적은 어려워짐) |

❗️**`raise e` vs `raise`**: `except ... as e: raise e`는 트레이스백에 현재 줄이 추가되어 원래 발생 지점이 헷갈린다. 같은 예외를 다시 던질 때는 인자 없는 `raise`를 쓴다.

**3.11 예외 그룹**: 동시에 여러 예외가 발생하는 `asyncio.TaskGroup`·병렬 작업을 위해 `ExceptionGroup`과 `except*` 문법이 추가되었다. 그룹 안의 예외를 타입별로 나누어 처리할 수 있으며, `e.add_note()`로 예외에 부가 정보를 덧붙일 수도 있다.

<br>

### 4. EAFP vs LBYL

파이썬은 **EAFP(Easier to Ask Forgiveness than Permission, 일단 시도하고 예외로 처리)** 스타일을 선호한다. 미리 검사하는 **LBYL(Look Before You Leap)**은 검사와 사용 사이에 상태가 바뀔 수 있는(경쟁 조건) 문제가 있다.

```python
if key in cache:                  # LBYL: 검사 후 사용 — 멀티스레드에서 사이에 삭제될 수 있음
    value = cache[key]
else:
    value = compute(key)

try:                              # EAFP: 바로 시도 — 원자적이고 파이썬다움
    value = cache[key]
except KeyError:
    value = compute(key)
```

- 예외가 **드물게** 발생하면 EAFP가 빠르고 (try는 거의 공짜, 실제 발생 시에만 비용), **자주** 발생하면 LBYL이 빠름
- 예외는 흐름 제어가 아니라 **비정상 상황 신호**로 쓰는 것이 원칙이나, `StopIteration`처럼 파이썬 내부는 예외를 제어 흐름에 적극 사용함(unit04 참고)

<br>

### 5. 컨텍스트 매니저와 with 문

### 5-1. 동작 원리 — `__enter__`와 `__exit__`

`with` 문은 블록에 들어갈 때 `__enter__`, 나올 때 **예외 여부와 관계없이** `__exit__`를 호출한다. `try/finally`를 객체에 캡슐화한 것이다.

```python
with open("data.txt") as f:       # f = open(...).__enter__()
    data = f.read()               # 예외가 나도 open(...).__exit__(exc_type, exc, tb)가 호출되어 닫힘

mgr = open("data.txt")            # 위 with 문을 풀어 쓰면 다음과 동일함
f = mgr.__enter__()
try:
    data = f.read()
except BaseException as e:
    if not mgr.__exit__(type(e), e, e.__traceback__):   # True를 반환하면 예외 억제
        raise
else:
    mgr.__exit__(None, None, None)
```

```
with 진입 ──▶ __enter__() ──▶ 블록 실행 ──┬─ 정상 종료 ──▶ __exit__(None, None, None)
                                         └─ 예외 발생 ──▶ __exit__(type, exc, tb)
                                                            ├─ False/None 반환 → 예외 전파
                                                            └─ True 반환      → 예외 억제
```

```python
import threading, time

class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self                                  # as 뒤 변수에 바인딩될 값

    def __exit__(self, exc_type, exc, tb):
        self.elapsed = time.perf_counter() - self.start
        print(f"{self.elapsed:.3f}s (예외: {exc_type.__name__ if exc_type else '없음'})")
        return False                                 # 예외는 억제하지 않고 전파

with Timer() as t:
    sum(range(10**6))

lock = threading.Lock()
with lock:                                           # Lock도 컨텍스트 매니저 — 예외가 나도 반드시 해제
    shared_counter = 1
```

> ⚠️ `__exit__`에서 `True`를 반환하면 **모든 예외가 억제**된다. 특정 예외만 삼키려면 `exc_type`을 검사해 `issubclass(exc_type, ValueError)`처럼 조건부로 `True`를 반환해야 한다. 표준 라이브러리의 `contextlib.suppress(ValueError)`가 이 패턴의 완성형이다.

<br>

### 5-2. contextlib — 제너레이터로 간단히 만들기

클래스를 만들지 않고 `@contextmanager`로 제너레이터 함수를 컨텍스트 매니저로 바꿀 수 있다. `yield` 앞이 `__enter__`, 뒤가 `__exit__`에 해당한다(제너레이터 원리는 unit04 참고).

```python
from contextlib import contextmanager, ExitStack, suppress, closing

@contextmanager
def transaction(conn):
    conn.begin()
    try:
        yield conn                    # 이 값이 as 변수로 전달됨, 블록이 여기서 실행됨
        conn.commit()                 # 블록이 정상 종료했을 때만
    except Exception:
        conn.rollback()               # 블록에서 예외가 나면 yield 지점에서 예외가 발생함
        raise                         # 반드시 재전파 — 삼키면 호출자가 실패를 모름
    finally:
        conn.close()

with transaction(get_conn()) as conn:
    conn.execute("INSERT ...")

with ExitStack() as stack:            # 개수가 동적인 자원을 한꺼번에 관리
    files = [stack.enter_context(open(p)) for p in paths]   # 역순으로 모두 닫힘

with suppress(FileNotFoundError):     # 특정 예외만 무시
    os.remove("tmp.lock")
```

| **도구**                         | **역할**                                                       |
| -------------------------------- | -------------------------------------------------------------- |
| **`@contextmanager`**            | 제너레이터 함수 → 컨텍스트 매니저                              |
| **`closing(obj)`**               | `close()`만 있는 객체를 `with`에서 쓰게 함                      |
| **`suppress(*exc)`**             | 지정한 예외를 조용히 무시                                       |
| **`ExitStack`**                  | 여러 컨텍스트를 동적으로 쌓고 **역순 해제** 보장                 |
| **`AbstractAsyncContextManager`, `asynccontextmanager`** | `async with`용 (`__aenter__`/`__aexit__`)         |

❗️**`@contextmanager` 안의 `yield`는 반드시 `try`로 감쌀 것**: 블록에서 예외가 나면 `yield` 지점에서 예외가 다시 발생하므로, `try/finally` 없이는 `yield` 이후의 정리 코드가 실행되지 않는다.

<br>

### 6. 리소스 해제 보장이 중요한 이유

- **파일**: 닫지 않으면 버퍼가 디스크에 쓰이지 않을 수 있고, OS의 파일 디스크립터 한도(보통 1024)에 도달하면 `Too many open files`
- **락**: 예외로 `release()`를 건너뛰면 다른 스레드가 영원히 대기 → 데드락(unit01 참고)
- **DB 커넥션·소켓**: 커넥션 풀이 고갈되어 서비스 전체가 멈춤
- CPython은 참조 수가 0이 되면 파일을 닫아주지만(unit02 참고), 순환 참조·예외 트레이스백이 객체를 붙잡으면 해제가 **지연**되고, PyPy 등 다른 구현체는 즉시 닫지 않으므로 `with`가 유일하게 이식 가능한 보장임

```python
f = open("log.txt", "a")              # 안티패턴: 예외가 나면 close()에 도달하지 못함
f.write(build_report())
f.close()

with open("log.txt", "a") as f:       # 개선: 어떤 경로로 빠져나가도 닫힘
    f.write(build_report())
```

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                          |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **`try/except/else/finally` 실행 순서는?**            | try → (예외 시 except / 정상 시 else) → finally 항상. `return`이 있어도 finally 실행     |
| **bare `except:`가 왜 위험한가?**                     | `KeyboardInterrupt`·`SystemExit`(BaseException)까지 잡아 종료가 안 됨                    |
| **`raise`와 `raise e`의 차이는?**                     | `raise`는 원본 트레이스백 유지, `raise e`는 현재 줄이 추가됨                             |
| **`raise ... from e`의 의미는?**                      | 명시적 연쇄(`__cause__`). 저수준 예외를 도메인 예외로 번역할 때                           |
| **EAFP란?**                                           | 일단 시도하고 예외로 처리. 검사-사용 사이 경쟁 조건이 없고 예외가 드물면 더 빠름          |
| **`with` 문은 어떻게 동작하나?**                      | `__enter__` 호출 → 블록 → 예외 여부와 무관하게 `__exit__`. `__exit__`가 True면 예외 억제 |
| **`@contextmanager` 주의점은?**                       | `yield`를 `try/finally`로 감싸야 예외 시에도 정리됨, 예외는 삼키지 말고 재전파            |
| **`__del__`로 자원을 닫으면 안 되는 이유는?**          | 호출 시점이 보장되지 않음(unit02 참고). 해제는 항상 컨텍스트 매니저로                     |
