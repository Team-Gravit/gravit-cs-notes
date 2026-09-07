## 데코레이터 동작 원리

**데코레이터(Decorator)**는 함수(또는 클래스)를 인자로 받아 새 함수를 돌려주는 **고차 함수(Higher-Order Function)**이며, `@` 문법은 그 호출 결과를 원래 이름에 다시 할당하는 문법 설탕이다. 로깅·캐싱·인증·재시도처럼 여러 함수에 공통으로 붙는 관심사를 본문과 분리해 재사용하게 해 주며, 원리를 모르면 메타데이터 손실·인자 전달·실행 시점에서 미묘한 버그를 만든다.

<br>

### 1. 전제 — 함수는 일급 객체

파이썬에서 함수는 **변수에 담고, 인자로 넘기고, 반환하고, 속성을 가질 수 있는 객체**다. 데코레이터는 이 성질을 그대로 이용한다.

```python
def greet(name):
    return f"안녕, {name}"

say = greet                     # 변수에 담기
print(say("철수"))
print(greet.__name__)           # 'greet' — 함수 객체가 가진 메타데이터

def apply(func, value):         # 함수를 인자로 받기
    return func(value)

def make_multiplier(n):         # 함수를 반환하기 (클로저)
    def multiply(x):
        return x * n            # 바깥 변수 n을 기억
    return multiply

double = make_multiplier(2)
print(double(5))                # 10
```

- 안쪽 함수가 바깥 함수의 지역 변수를 기억하는 것을 **클로저(Closure)**라 하며, `double.__closure__`에서 확인할 수 있음
- 데코레이터는 "함수를 받아 클로저로 감싸서 돌려주는 함수"에 불과함

<br>

### 2. `@` 문법의 정체

```python
def log_call(func):                       # 데코레이터: 함수를 받아
    def wrapper(*args, **kwargs):         # 새 함수를 정의하고
        print(f"호출: {func.__name__}{args}")
        result = func(*args, **kwargs)    # 원래 함수를 실행한 뒤
        print(f"반환: {result}")
        return result
    return wrapper                        # 새 함수를 돌려줌

@log_call
def add(a, b):
    return a + b

def add(a, b):                            # 위 @ 구문은 아래와 완전히 동일함
    return a + b
add = log_call(add)

add(2, 3)          # 호출: add(2, 3) / 반환: 5
```

```
정의 시점 (모듈 import 시 1회)
  def add … ──▶ log_call(add) ──▶ wrapper 생성 ──▶ 이름 add ──▶ wrapper
                                     │
                                     └── 클로저 셀에 원래 add 보관 (func)
호출 시점 (매번)
  add(2, 3) ──▶ wrapper(2, 3) ──▶ 전처리 ──▶ func(2, 3) ──▶ 후처리 ──▶ 반환
```

> 💡 데코레이터는 **모듈이 import될 때 즉시 실행**된다. 함수를 호출하지 않아도 `@` 아래 함수가 정의되는 순간 데코레이터 본문이 한 번 돈다. 이 시점 차이(정의 시 1회 vs 호출 시 매번)를 정확히 구분해 설명하면 데코레이터를 제대로 이해한 것이다.

<br>

### 3. 메타데이터 보존 — functools.wraps

데코레이터를 씌우면 이름이 `wrapper`로 바뀌므로 `__name__`·`__doc__`·시그니처가 사라진다. 디버깅, 문서화 도구, `pickle`, 테스트 프레임워크가 모두 이 정보를 사용하므로 반드시 복원해야 한다.

```python
import functools

def log_call(func):
    def wrapper(*args, **kwargs):          # 안티패턴: 메타데이터가 wrapper의 것으로 덮임
        return func(*args, **kwargs)
    return wrapper

@log_call
def add(a, b):
    """두 수를 더한다."""
    return a + b

print(add.__name__, add.__doc__)           # wrapper None
```

```python
import functools

def log_call(func):
    @functools.wraps(func)                 # 개선: func의 메타데이터를 wrapper에 복사
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@log_call
def add(a, b):
    """두 수를 더한다."""
    return a + b

print(add.__name__, add.__doc__)           # add 두 수를 더한다.
print(add.__wrapped__)                     # 원본 함수에 접근 가능
```

| **속성**           | **`wraps` 없이**    | **`wraps` 적용 후**       | **영향받는 도구**                      |
| ------------------ | ------------------- | ------------------------- | -------------------------------------- |
| **`__name__`**     | `wrapper`           | **원본 이름**             | 로깅, 트레이스백, 프로파일러           |
| **`__doc__`**      | `None`              | **원본 docstring**        | `help()`, Sphinx                       |
| **`__module__`**   | 데코레이터 모듈     | **원본 모듈**             | `pickle`, `multiprocessing`            |
| **`__wrapped__`**  | 없음                | **원본 함수 참조**        | `inspect.signature`, 테스트에서 원본 호출 |

> ⚠️ `functools.wraps`를 생략한 데코레이터를 `multiprocessing`에 넘기면 pickle이 `__module__.__qualname__`으로 함수를 찾지 못해 실패한다. "데코레이터 씌운 함수가 프로세스 풀에서 안 돌아요"는 대부분 이 문제다.

<br>

### 4. 인자를 받는 데코레이터 — 3중 구조

`@retry(times=3)`처럼 괄호가 붙으면 **먼저 `retry(times=3)`이 호출되어 진짜 데코레이터를 반환**하고, 그 반환값이 함수에 적용된다. 따라서 함수 한 겹이 더 필요하다.

```python
import functools, time

def retry(times=3, delay=0.5):                 # 1단계: 설정을 받아 데코레이터를 반환
    def decorator(func):                       # 2단계: 진짜 데코레이터
        @functools.wraps(func)
        def wrapper(*args, **kwargs):          # 3단계: 실제 실행 래퍼
            last_exc = None
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exc = e
                    print(f"{attempt}회 실패: {e}")
                    time.sleep(delay)
            raise last_exc
        return wrapper
    return decorator

@retry(times=2, delay=0)                      # retry(...) → decorator → decorator(fetch)
def fetch():
    raise ConnectionError("timeout")

fetch()
```

```
@retry(times=2)          ≡   fetch = retry(times=2)(fetch)
                                     └─ decorator ─┘ └─ wrapper
```

❗️**괄호 유무를 통일할 것**: `@retry`와 `@retry()`를 모두 허용하려면 첫 인자가 함수인지 검사하는 분기가 필요하다. 혼란을 줄이려면 인자가 있는 데코레이터는 항상 괄호를 붙이도록 정한다.

<br>

### 5. 데코레이터의 다양한 형태

### 5-1. 클래스 데코레이터와 클래스를 이용한 데코레이터

- **클래스에 적용**: 클래스를 받아 속성을 추가하거나 등록한 뒤 클래스를 돌려줌. `@dataclass`, `@functools.total_ordering`이 대표적
- **클래스로 구현**: `__call__`을 정의한 인스턴스는 함수처럼 호출되므로 상태(호출 횟수 등)를 속성으로 보관하기 편함

```python
import functools

class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)   # 클래스 버전의 wraps
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.func(*args, **kwargs)

@CountCalls
def ping():
    return "pong"

ping(); ping()
print(ping.count)        # 2
```

<br>

### 5-2. 여러 데코레이터의 적용 순서

```python
@a
@b
def f(): ...      # f = a(b(f)) — 아래(안쪽)부터 적용, 호출 시 a의 래퍼가 가장 바깥
```

```
적용 순서: b → a  (함수에 가까운 것부터)
호출 흐름: a.wrapper 전처리 → b.wrapper 전처리 → f → b 후처리 → a 후처리
```

> ⚠️ 클래스의 메서드에 데코레이터를 여러 개 쓸 때 순서가 특히 중요하다. `@staticmethod`·`@classmethod`·`@property`는 **일반 함수를 디스크립터로 바꾸므로** 대부분 가장 위(바깥)에 두어야 하고, 그 아래에 로깅·캐싱 데코레이터를 둔다.

<br>

### 6. 표준 라이브러리의 데코레이터

| **데코레이터**                    | **역할**                                                       | **주의점**                                            |
| --------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------- |
| **`functools.lru_cache`, `cache`** | 인자를 키로 반환값 메모이제이션 (DP·재귀 최적화)               | 인자가 **해시 가능**해야 함. 메서드에 쓰면 `self`가 캐시에 잡혀 누수 위험 |
| **`functools.cached_property`**   | 처음 접근 시 계산해 인스턴스에 저장                            | 인스턴스 `__dict__` 필요 (`__slots__`와 충돌)          |
| **`functools.singledispatch`**    | 첫 인자 타입별로 구현을 분기 (함수 오버로딩 흉내)              | 타입 힌트로 등록 가능(unit07 참고)                    |
| **`contextlib.contextmanager`**   | 제너레이터를 컨텍스트 매니저로 변환                            | unit08 참고                                           |
| **`staticmethod`, `classmethod`, `property`** | 메서드 종류 지정 (디스크립터)                      | unit06 참고                                           |
| **`dataclasses.dataclass`**       | `__init__`·`__repr__`·`__eq__` 자동 생성                       | `frozen=True`로 불변화 가능(unit03 참고)               |
| **`typing.overload`, `warnings.deprecated`(3.13)** | 타입 검사기용 시그니처·폐기 표시            | 런타임 동작은 바꾸지 않음                              |

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(80))                # 캐시 없이는 지수 시간, 캐시로 선형 시간
print(fib.cache_info())       # hits, misses, maxsize, currsize
```

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                   |
| ----------------------------------------------------- | ------------------------------------------------------------------------------- |
| **데코레이터란?**                                     | 함수를 받아 새 함수를 반환하는 고차 함수. `@f`는 `g = f(g)`의 문법 설탕            |
| **언제 실행되나?**                                    | 데코레이터 본문은 **정의 시 1회**, 래퍼는 **호출 시 매번**                        |
| **`functools.wraps`는 왜 필요한가?**                  | `__name__`·`__doc__`·`__module__`·`__wrapped__` 복원. 없으면 디버깅·pickle 문제   |
| **인자 있는 데코레이터 구조는?**                      | 팩토리 → 데코레이터 → 래퍼의 3중 함수. `@d(x)`는 `d(x)(f)`                        |
| **여러 데코레이터 순서는?**                           | 함수에 가까운 것부터 적용, 호출은 바깥부터. `@property` 등은 가장 위에             |
| **클로저와의 관계는?**                                | 래퍼가 원본 함수를 클로저 셀에 보관. 래퍼 안에서 원본을 재할당하려면 `nonlocal` 필요 |
| **`lru_cache`를 메서드에 쓰면?**                      | `self`가 캐시 키에 포함되어 인스턴스가 해제되지 않음. 모듈 함수나 `cached_property` 권장 |
