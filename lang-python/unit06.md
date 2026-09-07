## 클래스와 MRO

파이썬 클래스는 **속성을 담은 딕셔너리(`__dict__`)와 부모 클래스 목록**으로 이루어진 런타임 객체이며, 속성 탐색은 **MRO(Method Resolution Order, 메서드 해석 순서)**라는 정해진 순서로 이루어진다. 다중 상속에서 어떤 부모의 메서드가 호출되는지, `super()`가 정확히 무엇을 가리키는지, 그리고 `__len__`·`__eq__` 같은 **매직 메서드(Magic/Dunder Method)**로 객체가 내장 문법과 어떻게 연결되는지는 파이썬 면접에서 반복해서 묻는 주제다.

<br>

### 1. 클래스도 객체다 — 속성 탐색의 기본

```python
class Dog:
    species = "개"                  # 클래스 속성 (모든 인스턴스가 공유)

    def __init__(self, name):
        self.name = name            # 인스턴스 속성 (인스턴스 __dict__에 저장)

    def bark(self):
        return f"{self.name}: 멍"

d = Dog("바둑이")
print(d.__dict__)                   # {'name': '바둑이'}
print(Dog.__dict__["bark"])         # 함수 객체 — 메서드는 클래스에만 있음
print(type(Dog))                    # <class 'type'> — 클래스는 type의 인스턴스
```

`d.bark` 접근 시 인터프리터는 **인스턴스 `__dict__` → 클래스 → MRO 순서의 부모 클래스**를 차례로 뒤진다. 클래스에서 찾은 함수는 **디스크립터 프로토콜**을 통해 `self`가 묶인 **바운드 메서드**로 변환된다.

```
d.bark 탐색 경로
① d.__dict__ 에 'bark'?  ─ 없음
② Dog.__dict__ 에 'bark'? ─ 있음 → 함수.__get__(d, Dog) → 바운드 메서드 반환
③ (없으면) MRO의 다음 클래스로 …  ④ 끝까지 없으면 __getattr__ → AttributeError
```

> 💡 `d.bark()`가 `Dog.bark(d)`와 같은 이유가 이 디스크립터 변환이다. `@staticmethod`·`@classmethod`·`@property`는 모두 `__get__`을 다르게 구현한 디스크립터로, "함수 → 메서드 변환 규칙"을 바꾸는 장치다.

❗️**가변 클래스 속성 공유 함정**: `class A: items = []`처럼 클래스 속성에 리스트를 두고 `self.items.append()`하면 모든 인스턴스가 하나의 리스트를 공유한다. 인스턴스별 상태는 반드시 `__init__`에서 `self.items = []`로 만든다(unit03 참고).

<br>

### 2. 상속과 MRO

### 2-1. C3 선형화

다중 상속에서 클래스가 탐색될 순서를 **C3 선형화(C3 Linearization)** 알고리즘으로 계산해 `__mro__` 튜플에 저장한다. 두 가지 규칙을 반드시 지킨다.

- **자식이 부모보다 먼저** 온다
- **부모 목록에 적은 순서**를 보존한다 (`class D(B, C)`면 B가 C보다 먼저)

```python
class A:
    def hello(self): return "A"
class B(A):
    def hello(self): return "B"
class C(A):
    def hello(self): return "C"
class D(B, C):
    pass

print(D.__mro__)     # (D, B, C, A, object)
print(D().hello())   # "B" — MRO에서 처음 만나는 구현
```

```
        A                다이아몬드 상속
       / \               MRO(D) = D + merge(MRO(B), MRO(C), [B, C])
      B   C                     = D + merge([B, A, obj], [C, A, obj], [B, C])
       \ /                      = D, B, C, A, object
        D                (A는 B·C 모두의 뒤에 있으므로 C 다음에 배치)
```

- 파이썬 2 스타일의 "깊이 우선 왼쪽부터"였다면 `D, B, A, C`가 되어 C가 A보다 늦게 탐색되는 모순이 생김. C3는 이를 해결함
- 규칙을 만족하는 순서가 없으면 클래스 정의 시점에 `TypeError: Cannot create a consistent MRO`가 발생함

<br>

### 2-2. super()는 "부모"가 아니라 "MRO의 다음"

`super()`는 현재 클래스의 부모를 부르는 것이 아니라 **인스턴스의 MRO에서 현재 클래스 다음 위치**에 위임한다. 이 덕분에 다중 상속에서도 각 클래스의 메서드가 **정확히 한 번씩** 호출되는 협력적 호출 체인이 만들어진다.

```python
class A:
    def __init__(self):
        print("A"); super().__init__()
class B(A):
    def __init__(self):
        print("B"); super().__init__()
class C(A):
    def __init__(self):
        print("C"); super().__init__()
class D(B, C):
    def __init__(self):
        print("D"); super().__init__()

D()        # D → B → C → A  (B의 super()가 A가 아닌 C를 가리킴)
```

| **호출 방식**              | **동작**                                    | **문제점**                                              |
| -------------------------- | ------------------------------------------- | ------------------------------------------------------- |
| **`A.__init__(self)` 직접 호출** | 특정 클래스를 고정해서 호출              | 다이아몬드에서 A가 **두 번 호출**되거나 C가 누락됨        |
| **`super().__init__()`**   | MRO의 **다음 클래스**에 위임                | 모든 클래스가 협력해야 하며 시그니처를 맞춰야 함          |

> ⚠️ 협력적 다중 상속에서는 모든 클래스의 메서드가 `super()`를 호출하고 **`*args, **kwargs`를 그대로 넘겨야** 체인이 끊기지 않는다. 믹스인(Mixin) 클래스를 만들 때 `super().__init__(**kwargs)`를 빼먹으면 뒤쪽 클래스의 초기화가 통째로 사라진다.

<br>

### 3. 매직 메서드 — 객체를 언어 문법에 연결하기

`__이름__` 형태의 메서드는 파이썬이 **특정 문법·내장 함수를 만났을 때 자동으로 호출**한다. 직접 호출하지 않고 `len(obj)`, `obj[0]`, `a + b`처럼 사용하는 것이 관례다.

| **분류**           | **매직 메서드**                                    | **트리거 문법**                        |
| ------------------ | -------------------------------------------------- | -------------------------------------- |
| **생성·소멸**      | `__new__`, `__init__`, `__del__`                   | `Cls()` — `__new__`가 생성, `__init__`가 초기화 |
| **표현**           | `__repr__`, `__str__`, `__format__`                | `repr()`, `print()`, f-string          |
| **비교·해시**      | `__eq__`, `__lt__`, `__hash__`                     | `==`, `<`, `dict` 키·`set` 원소        |
| **컨테이너**       | `__len__`, `__getitem__`, `__contains__`, `__iter__` | `len()`, `obj[i]`, `in`, `for`(unit04 참고) |
| **산술**           | `__add__`, `__radd__`, `__iadd__`                  | `a + b`, `b + a`(우측 피연산자), `a += b` |
| **호출·컨텍스트**  | `__call__`, `__enter__`, `__exit__`                | `obj()`, `with`(unit08 참고)            |
| **속성 접근**      | `__getattr__`, `__getattribute__`, `__setattr__`   | 없는 속성 접근, 모든 속성 접근, 속성 대입 |

```python
from functools import total_ordering

@total_ordering                      # __eq__ + __lt__ 하나만 정의하면 나머지 비교 자동 생성
class Money:
    def __init__(self, won):
        self.won = won

    def __repr__(self):              # 개발자용, 재현 가능한 표현 (컨테이너 안에서도 사용됨)
        return f"Money({self.won})"

    def __str__(self):               # 사용자용, print()에서 사용
        return f"{self.won:,}원"

    def __eq__(self, other):
        if not isinstance(other, Money):
            return NotImplemented    # 타입이 다르면 파이썬이 반대쪽 __eq__를 시도
        return self.won == other.won

    def __hash__(self):              # __eq__를 정의하면 __hash__가 None이 되므로 함께 정의
        return hash(self.won)

    def __lt__(self, other):
        return self.won < other.won

    def __add__(self, other):
        return Money(self.won + other.won)

    def __bool__(self):              # 정의하지 않으면 __len__, 그것도 없으면 항상 True
        return self.won != 0

a, b = Money(1000), Money(2000)
print(a + b, a < b, sorted([b, a]))   # 3,000원 True [Money(1000), Money(2000)]
print({a, Money(1000)})               # {Money(1000)} — __eq__/__hash__ 일관성 덕분에 중복 제거
```

> ⚠️ `__eq__`만 재정의하고 `__hash__`를 정의하지 않으면 인스턴스가 **해시 불가**가 되어 `set`·`dict` 키로 쓸 수 없다. 반대로 `__eq__`가 같다고 판단한 두 객체는 반드시 같은 해시를 반환해야 하며, 이 규칙을 어기면 `set`에 같은 값이 두 번 들어간다. `@dataclass`는 `eq=True, frozen=True`일 때만 `__hash__`를 만들어 이 규칙을 자동으로 지킨다.

<br>

### 4. `__new__` vs `__init__`, `__slots__`

| **항목**       | **`__new__`**                                 | **`__init__`**                          |
| -------------- | --------------------------------------------- | --------------------------------------- |
| **역할**       | 인스턴스를 **생성**해 반환 (정적 메서드)       | 이미 생성된 인스턴스를 **초기화**         |
| **첫 인자**    | `cls`                                         | `self`                                  |
| **반환**       | 새 인스턴스 (다른 타입 반환도 가능)            | 반드시 `None`                           |
| **사용 예**    | 싱글턴, 불변 타입(`int`, `tuple`) 서브클래싱   | 대부분의 일반 클래스                     |

```python
class Point:
    __slots__ = ("x", "y")           # 인스턴스 __dict__를 만들지 않고 고정 슬롯에 저장

    def __init__(self, x, y):
        self.x, self.y = x, y

p = Point(1, 2)
p.z = 3                              # AttributeError — 선언되지 않은 속성 금지
```

- `__slots__`는 인스턴스당 메모리를 크게 줄이고(unit02 참고) 속성 오타를 조기에 잡아줌
- 단, 동적 속성 추가·`cached_property`가 불가능하고, 상속 계층 전체가 슬롯을 선언해야 효과가 있음

<br>

### 5. 추상 클래스와 덕 타이핑

파이썬은 **덕 타이핑(Duck Typing)** — "오리처럼 걷고 꽥꽥거리면 오리" — 을 기본으로 하므로 인터페이스 상속 없이도 `__len__`을 구현한 객체는 어디서나 `len()`을 쓸 수 있다. 명시적 계약이 필요하면 `abc` 모듈을 사용한다.

```python
from abc import ABC, abstractmethod

class Storage(ABC):
    @abstractmethod
    def save(self, key, value): ...

    @abstractmethod
    def load(self, key): ...

class MemoryStorage(Storage):
    def __init__(self): self._d = {}
    def save(self, key, value): self._d[key] = value
    def load(self, key): return self._d[key]

Storage()            # TypeError — 추상 메서드가 남아 있어 인스턴스화 불가
MemoryStorage()      # 정상
```

- 상속 없이 "구조"만으로 계약을 검사하려면 `typing.Protocol`을 사용함(unit07 참고)
- `isinstance(obj, collections.abc.Iterable)`처럼 표준 ABC는 `__iter__` 존재만으로 가상 서브클래스로 인정함

<br>

### 6. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                          |
| ------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **MRO란?**                                        | 속성 탐색 순서. C3 선형화로 계산되며 `Cls.__mro__`·`Cls.mro()`로 확인                    |
| **다이아몬드 상속에서 어떤 메서드가 호출되나?**   | MRO에서 먼저 나오는 클래스의 것. `D(B, C)`면 `D, B, C, A, object`                       |
| **`super()`는 부모를 부르나?**                    | 아니다. **MRO의 다음 클래스**에 위임하므로 다중 상속에서 각 클래스가 한 번씩 호출됨       |
| **`__new__`와 `__init__` 차이는?**                | 생성 vs 초기화. 불변 타입 상속·싱글턴은 `__new__`에서 처리                               |
| **`__repr__`와 `__str__` 차이는?**                | 개발자용 재현 표현 vs 사용자용 표현. `__str__`이 없으면 `__repr__`로 대체                |
| **`__eq__` 정의 시 주의점은?**                    | `__hash__`가 사라지므로 함께 정의, 타입이 다르면 `NotImplemented` 반환                    |
| **`__slots__`의 장단점은?**                       | 메모리 절감·오타 방지 vs 동적 속성 불가·상속 계층 전체 필요                              |
| **클래스 속성과 인스턴스 속성의 차이는?**         | 클래스 `__dict__`에 있어 공유 vs 인스턴스 `__dict__`에 있어 개별. 가변 클래스 속성 주의  |
