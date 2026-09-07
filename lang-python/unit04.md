## 이터레이터와 제너레이터

**이터레이터(Iterator)**는 `for` 문이 값을 하나씩 꺼내는 표준 규약이고, **제너레이터(Generator)**는 그 규약을 `yield` 한 줄로 구현하게 해 주는 함수다. 값을 미리 다 만들지 않고 필요할 때 하나씩 계산하는 **지연 평가(Lazy Evaluation)** 덕분에 수 GB 파일이나 무한 수열도 상수 메모리로 처리할 수 있으며, 파이썬의 `for`·컴프리헨션·`asyncio`가 모두 이 프로토콜 위에 서 있다.

<br>

### 1. 이터레이션 프로토콜 — for 문은 어떻게 동작하는가

`for x in obj:`는 내부적으로 다음 절차를 수행한다.

```
for x in obj:
    body

  ↓ 인터프리터가 실제로 하는 일

it = iter(obj)            # obj.__iter__() 호출 → 이터레이터 획득
while True:
    try:
        x = next(it)      # it.__next__() 호출 → 다음 값
    except StopIteration: # 더 이상 값이 없으면 종료
        break
    body
```

| **용어**                | **조건**                                              | **예시**                          |
| ----------------------- | ----------------------------------------------------- | --------------------------------- |
| **이터러블(Iterable)**  | `__iter__()`가 이터레이터를 반환 (또는 `__getitem__`) | `list`, `dict`, `str`, `range`, 파일 |
| **이터레이터(Iterator)** | `__next__()`가 있고, `__iter__()`가 **자기 자신** 반환 | `iter([1,2])`, 제너레이터 객체, `map` 객체 |

- 리스트는 이터러블이지만 이터레이터가 아님 — `next([1, 2])`는 `TypeError`
- 이터레이터는 **상태를 가지며 한 번 소진되면 끝**. 다시 순회하려면 `iter()`를 새로 호출해야 함

```python
nums = [1, 2, 3]
it = iter(nums)
print(next(it), next(it), next(it))   # 1 2 3
next(it)                               # StopIteration 발생
```

<br>

### 2. 이터레이터 클래스 직접 구현

```python
class Countdown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):            # 자기 자신을 반환 → 이터레이터
        return self

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

for n in Countdown(3):
    print(n)                       # 3 2 1
```

> 💡 이터러블과 이터레이터를 분리하고 싶다면 `__iter__`가 **새 이터레이터 객체**를 만들어 반환하게 한다. 리스트가 여러 번 순회 가능한 이유가 이것이다. 반대로 위처럼 `self`를 반환하면 한 번만 순회할 수 있다.

<br>

### 3. 제너레이터 — yield로 만드는 이터레이터

함수 본문에 `yield`가 있으면 그 함수는 호출 시 실행되지 않고 **제너레이터 객체**를 반환한다. `next()`가 호출될 때마다 다음 `yield`까지 실행하고 **지역 변수·실행 위치를 그대로 보존한 채 일시 정지**한다.

```python
def countdown(start):
    print("시작")                  # 첫 next()에서야 실행됨
    while start > 0:
        yield start
        start -= 1
    print("끝")                    # 마지막 next()에서 실행 후 StopIteration

gen = countdown(3)
print(type(gen))                   # <class 'generator'>  — 아직 "시작"도 출력 안 됨
print(next(gen))                   # 시작 → 3
print(next(gen))                   # 2
print(list(gen))                   # [1] 그리고 "끝" 출력
```

```
호출자                       제너레이터 프레임
next(gen) ────────────▶ 실행 … yield 3 ──┐ (프레임 보존: start=3, 위치=yield)
          ◀── 3 ─────────────────────────┘
next(gen) ────────────▶ 재개 … yield 2 ──┐ (start=2)
          ◀── 2 ─────────────────────────┘
next(gen) ────────────▶ 재개 … 함수 종료 → StopIteration
```

**제너레이터 표현식**은 리스트 컴프리헨션의 대괄호를 소괄호로 바꾼 형태로, 값을 즉시 만들지 않는다.

```python
squares_list = [x * x for x in range(10**7)]   # 즉시 1천만 개 생성 → 수백 MB
squares_gen  = (x * x for x in range(10**7))   # 제너레이터 객체 하나 → 약 100바이트
print(sum(squares_gen))                        # 하나씩 꺼내 더하므로 메모리 일정
```

<br>

### 4. 지연 평가와 메모리 효율

**대용량 데이터를 상수 메모리로**

```python
def read_lines(path):                  # 안티패턴: f.readlines()는 파일 전체를 리스트로 적재
    with open(path, encoding="utf-8") as f:
        for line in f:                 # 파일 객체 자체가 이터레이터 → 한 줄씩 읽음
            yield line.rstrip("\n")

def parse(lines):
    for line in lines:
        yield line.split(",")

def only_errors(rows):
    for row in rows:
        if row[2] == "ERROR":
            yield row

errors = only_errors(parse(read_lines("app.log")))   # 세 단계가 한 줄씩 흘러감, 메모리 일정
for row in errors:
    print(row)
```

| **항목**          | **리스트 (즉시 평가)**                | **제너레이터 (지연 평가)**                   |
| ----------------- | ------------------------------------- | -------------------------------------------- |
| **메모리**        | 모든 원소를 동시에 보유 O(n)          | 현재 원소 하나 + 프레임 O(1)                 |
| **첫 결과 시간**  | 전체 생성 후                          | **즉시** (첫 `next()`에 첫 값)               |
| **재사용**        | 여러 번 순회·인덱싱·`len()` 가능      | **한 번만** 순회, `len()`·인덱싱 불가         |
| **무한 시퀀스**   | 불가                                  | 가능 (`itertools.count()` 등)                |
| **디버깅**        | 값이 눈에 보임                        | 소비 전까지 값이 없어 추적이 어려움          |

**소진과 재사용의 함정**

```python
gen = (x for x in range(3))
print(list(gen))       # [0, 1, 2]
print(list(gen))       # []  ← 이미 소진됨, 예외 없이 빈 결과
print(3 in gen)        # False — in 연산도 원소를 소비함
```

> ⚠️ 제너레이터를 두 번 순회하거나 `len()`을 호출하는 것은 가장 흔한 버그다. 여러 번 써야 하면 `list()`로 실체화하거나 제너레이터를 만드는 **함수를 다시 호출**한다. `itertools.tee()`로 분기할 수도 있지만 내부 버퍼가 커질 수 있다.

<br>

### 5. 제너레이터 고급 기능

### 5-1. send·throw·close — 양방향 통신

`yield`는 값을 내보낼 뿐 아니라 **표현식으로서 값을 받을 수도** 있다. 이것이 코루틴의 원형이다.

```python
def running_average():
    total, count = 0, 0
    while True:
        value = yield (total / count if count else None)  # send()로 받은 값
        total += value
        count += 1

avg = running_average()
next(avg)              # 첫 yield까지 실행(프라이밍). 프라이밍 전 send(값)는 TypeError
print(avg.send(10))    # 10.0
print(avg.send(20))    # 15.0
avg.close()            # 제너레이터 안에 GeneratorExit 발생 → 정리 후 종료
```

- `gen.throw(exc)`: 일시 정지한 `yield` 지점에서 예외를 발생시킴
- `gen.close()`: `GeneratorExit`를 던져 `finally`·`with` 블록이 실행되게 함. 제너레이터 안에서 파일을 열었다면 여기서 닫힘

<br>

### 5-2. yield from — 제너레이터 위임

중첩 제너레이터의 값을 그대로 전달하고, `send`·`throw`도 안쪽까지 투명하게 전달한다. 반환값(`return`)은 `yield from` 표현식의 결과가 된다.

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)     # for x in flatten(item): yield x 와 동일 + 통신 위임
        else:
            yield item

print(list(flatten([1, [2, [3, 4]], 5])))   # [1, 2, 3, 4, 5]
```

> 💡 `async def`/`await`는 이 제너레이터 메커니즘(프레임 일시 정지·재개) 위에 만들어졌다. `asyncio`의 이벤트 루프는 결국 코루틴을 `send()`로 재개시키는 스케줄러이며, 이 연결 고리를 설명하면 unit01의 asyncio 이해도도 함께 보여줄 수 있다.

<br>

### 6. itertools와 내장 지연 함수

표준 라이브러리는 이터레이터를 조합하는 도구를 풍부하게 제공하며, 대부분 **지연 평가**로 동작한다.

| **함수**                          | **역할**                                    | **예시 결과**                          |
| --------------------------------- | ------------------------------------------- | -------------------------------------- |
| **`map`, `filter`, `zip`**        | 변환·선별·묶기 (3.x에서는 모두 지연)         | `zip("ab", [1, 2])` → `('a',1),('b',2)` |
| **`enumerate`**                   | 인덱스 부여                                 | `enumerate("ab", 1)` → `(1,'a'),(2,'b')` |
| **`itertools.count`, `cycle`**    | 무한 수열·무한 반복                          | `count(10, 2)` → 10, 12, 14, …          |
| **`itertools.islice`**            | 이터레이터 슬라이싱 (무한 수열 자르기)       | `islice(count(), 3)` → 0, 1, 2          |
| **`itertools.chain`**             | 여러 이터러블 이어 붙이기                    | `chain([1], [2, 3])` → 1, 2, 3          |
| **`itertools.groupby`**           | 연속된 같은 키끼리 묶기 (정렬 필요)          | 로그를 날짜별로 묶을 때                 |
| **`itertools.product`, `permutations`, `combinations`** | 완전 탐색용 조합 생성 | 백트래킹·브루트 포스 문제                |

```python
from itertools import islice, count

first_five_even = list(islice((n for n in count() if n % 2 == 0), 5))
print(first_five_even)     # [0, 2, 4, 6, 8] — 무한 수열에서 5개만 안전하게 추출
```

❗️**정렬·역순·`len`은 지연 불가**: `sorted()`, `reversed()`(시퀀스 필요), `len()`은 전체를 알아야 하므로 제너레이터에 쓰면 실체화되거나 오류가 난다. 파이프라인의 마지막 단계에서만 사용한다.

<br>

### 7. 정리

- `for`는 `iter()`로 이터레이터를 얻고 `next()`를 `StopIteration`까지 반복하는 문법 설탕임
- **이터러블**은 `__iter__`, **이터레이터**는 `__iter__` + `__next__`를 구현하며, 이터레이터는 상태를 가지고 **한 번만** 순회됨
- **제너레이터**는 `yield`로 프레임을 일시 정지·재개하며 이터레이터 프로토콜을 자동 구현함
- **지연 평가**는 메모리 O(1)·즉시 첫 결과·무한 시퀀스를 가능하게 하지만, 재사용 불가·`len()` 불가·디버깅 난이도라는 대가가 있음
- `send`/`throw`/`close`와 `yield from`은 코루틴의 기반이며, `asyncio`는 이를 스케줄링하는 구조임(unit01 참고)
- 대용량 파일·스트림·파이프라인 처리에는 리스트 대신 제너레이터 체인을 기본으로 선택할 것
