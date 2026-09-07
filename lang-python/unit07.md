## 타입 힌트

**타입 힌트(Type Hint)**는 함수 인자·반환값·변수에 기대하는 타입을 **주석처럼 표기**하는 문법(PEP 484)이다. 파이썬은 여전히 동적 타이핑 언어라 인터프리터는 힌트를 검사하지 않지만, mypy·pyright 같은 **정적 검사기**와 IDE가 실행 전에 타입 오류를 잡아 주고, 코드가 곧 문서가 된다. 힌트가 "왜" 런타임에 강제되지 않는지와 "어떤 도구로" 검증하는지를 구분해 설명할 수 있어야 한다.

<br>

### 1. 동적 타이핑과 점진적 타이핑

파이썬은 변수가 아니라 **객체가 타입을 가지며**, 같은 변수에 다른 타입을 얼마든지 담을 수 있다. 타입 힌트는 이 자유를 없애는 것이 아니라 **원하는 곳에만 점진적으로(Gradual Typing)** 정적 검사를 추가하는 장치다.

| **구분**            | **정적 타이핑 (Java 등)**       | **동적 타이핑 (Python)**            | **점진적 타이핑 (Python + 힌트)**             |
| ------------------- | ------------------------------- | ----------------------------------- | --------------------------------------------- |
| **타입 확정 시점**  | 컴파일 시                       | 실행 시 (객체가 타입 보유)          | 검사기 실행 시 (힌트 있는 부분만)              |
| **오류 발견**       | 컴파일 오류                     | 실행 중 `TypeError`                 | **실행 전** 검사기 리포트 + 실행 중 `TypeError` |
| **유연성**          | 낮음                            | 높음                                | 필요한 곳만 제약                              |
| **강제 여부**       | 강제                            | 없음                                | **강제 아님** (힌트 없는 코드도 정상 실행)     |

```python
def add(a: int, b: int) -> int:
    return a + b

print(add("안", "녕"))          # "안녕" — 오류 없이 실행됨! 힌트는 검사되지 않는다
print(add.__annotations__)      # {'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
```

<br>

### 2. 런타임에 강제되지 않는 이유

- 힌트는 함수·모듈·클래스의 **`__annotations__` 딕셔너리에 저장될 뿐** 호출 경로에는 관여하지 않음
- 모든 호출마다 `isinstance` 검사를 넣으면 **성능 비용**이 크고, 덕 타이핑(unit06 참고)·제네릭·`Protocol`처럼 런타임 검사가 불가능한 타입이 많음
- 기존 코드와의 **하위 호환** — 힌트를 추가해도 동작이 바뀌지 않아야 점진적 도입이 가능함
- 언어 설계 철학상 힌트는 **도구를 위한 메타데이터**이며, 검증은 검사기·라이브러리의 몫으로 분리함

```
소스 코드 (힌트 포함)
   │
   ├──▶ 인터프리터: 힌트를 __annotations__에 저장하고 무시 → 실행
   │
   └──▶ 정적 검사기(mypy·pyright): 소스를 읽어 타입 그래프 구성 → 오류 리포트
                                    (실행하지 않음, 배포물에 영향 없음)
```

> 💡 "힌트가 강제되지 않는데 왜 쓰나요?"에는 세 가지로 답한다. ① **실행 전** 오류 발견(CI에서 검사기 실행), ② IDE 자동완성·리팩터링 정확도, ③ 함수 시그니처가 곧 문서. 팀 규모가 커질수록 가치가 커진다.

<br>

### 3. 핵심 typing 문법

**기본 타입과 컨테이너**

3.9(PEP 585)부터 `list[int]`처럼 **내장 타입을 직접 제네릭으로** 쓰고, 3.10(PEP 604)부터 `int | None`처럼 `|`로 유니온을 표기한다. `typing.List`·`Optional`은 하위 호환용이다.

```python
from collections.abc import Sequence, Mapping, Callable, Iterable

def mean(values: Sequence[float]) -> float:          # 입력은 추상 타입(넓게)
    return sum(values) / len(values)

def index_by(items: Iterable[dict], key: str) -> dict[str, dict]:   # 반환은 구체 타입(좁게)
    return {item[key]: item for item in items}

def find(name: str) -> str | None:                    # Optional[str]과 동일
    return None

handler: Callable[[int, str], bool]                   # (int, str) -> bool 함수 타입
```

<br>

### 3-1. 제네릭·TypeVar와 3.12 새 문법

```python
from typing import TypeVar

T = TypeVar("T")
def first_old(xs: list[T]) -> T:                     # 3.11 이하 방식
    return xs[0]

def first[T](xs: list[T]) -> T:                      # 3.12+ (PEP 695) — TypeVar 선언 불필요
    return xs[0]

class Stack[T]:                                       # 제네릭 클래스도 동일 문법
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)
    def pop(self) -> T:
        return self._items.pop()

type UserId = int                                     # 3.12+ type 문 — 타입 별칭
type Json = dict[str, "Json"] | list["Json"] | str | int | float | bool | None
```

<br>

### 3-2. 구조적 타이핑 — Protocol과 TypedDict

상속 없이 **"이 메서드가 있으면 이 타입"**으로 인정하는 것이 `Protocol`이다. 덕 타이핑을 정적 검사기가 이해할 수 있게 만든 것으로, `abc`(명목적 타이핑)와 대비된다.

```python
from typing import Protocol, TypedDict, Literal, NotRequired

class SupportsClose(Protocol):
    def close(self) -> None: ...

def shutdown(resource: SupportsClose) -> None:      # 파일·소켓·DB 커넥션 모두 통과
    resource.close()

class UserDict(TypedDict):                           # 딕셔너리 키·값 타입 명세 (JSON 응답 등)
    id: int
    name: str
    email: NotRequired[str]                          # 선택 키

Mode = Literal["r", "w", "a"]                        # 허용 값 집합
def open_file(path: str, mode: Mode = "r") -> None: ...
```

| **도구**            | **검사 방식**     | **런타임 영향**                  | **적합한 상황**                          |
| ------------------- | ----------------- | -------------------------------- | ---------------------------------------- |
| **`abc.ABC`**       | 명목적 (상속)     | 인스턴스화 차단, `isinstance` 가능 | 내부 클래스 계층에 계약 강제              |
| **`Protocol`**      | **구조적** (형태) | 없음 (`runtime_checkable` 시 `isinstance` 가능) | 외부 라이브러리 객체·덕 타이핑 인터페이스 |
| **`TypedDict`**     | 구조적 (키)       | 없음 — 그냥 `dict`               | JSON·API 응답 형태 명세                   |
| **`dataclass`**     | 명목적            | 클래스 생성                      | 자체 데이터 모델                          |

<br>

### 4. 타입 좁히기와 검사기의 흐름 분석

정적 검사기는 `isinstance`·`is None`·`assert`·`match` 같은 조건문을 따라가며 **분기 안에서 타입을 좁힌다(Narrowing)**. 이 원리를 알아야 `str | None` 반환값을 안전하게 다룰 수 있다.

```python
from typing import TypeGuard, TypeIs, assert_never

def get_name(user_id: int) -> str | None: ...

name = get_name(1)
name.upper()                    # 검사기 오류: None에는 upper가 없음
if name is not None:
    name.upper()                # 통과 — 이 블록 안에서 name은 str로 좁혀짐

def is_str_list(xs: list[object]) -> TypeIs[list[str]]:   # 3.13+ (PEP 742), 3.10~은 TypeGuard
    return all(isinstance(x, str) for x in xs)

def handle(mode: Literal["r", "w"]) -> None:
    match mode:
        case "r": ...
        case "w": ...
        case _: assert_never(mode)    # 새 Literal 값이 추가되면 검사기가 누락을 잡아냄
```

> ⚠️ `Any`는 "모든 타입과 호환"이라 검사를 사실상 끄는 탈출구다. 남발하면 힌트의 의미가 사라진다. 타입을 모르면 `object`를 쓰고 사용 시점에 좁히거나, 최소한 `cast()`로 의도를 드러내야 한다. `# type: ignore`도 사유를 함께 적는 것이 관례다.

<br>

### 5. 검증 도구 — 정적 검사와 런타임 검증

| **도구**                | **종류**            | **특징**                                                                  |
| ----------------------- | ------------------- | ------------------------------------------------------------------------- |
| **mypy**                | 정적 검사기         | 가장 오래된 표준 검사기. `--strict`로 엄격도 조절, 플러그인 생태계          |
| **pyright / Pylance**   | 정적 검사기         | VS Code 기본 엔진, 빠르고 타입 추론이 강함. `basic/standard/strict` 모드   |
| **pyre, pytype**        | 정적 검사기         | Meta·Google 개발. pytype은 힌트 없는 코드도 추론                            |
| **pydantic**            | **런타임 검증**     | 힌트를 읽어 입력 데이터를 실제로 검사·변환. FastAPI 요청 검증의 기반        |
| **typeguard, beartype** | 런타임 검증         | 데코레이터로 함수 호출 시 인자 타입을 검사. 테스트·경계 지점에서 사용        |

```bash
pip install mypy pyright
mypy --strict app/            # CI 단계에서 실행, 오류 시 파이프라인 실패
pyright app/
```

```python
from pydantic import BaseModel, ValidationError

class User(BaseModel):                       # 힌트가 곧 검증 규칙이 됨
    id: int
    name: str
    email: str | None = None

print(User(id="42", name="철수"))            # id='42' → 42로 강제 변환(기본 lax 모드)
try:
    User(id="abc", name="철수")
except ValidationError as e:
    print(e.errors()[0]["msg"])              # 런타임에 실제로 검사됨
```

> 💡 pydantic은 기본(lax) 모드에서 `"42"` → `42`처럼 **타입을 강제 변환**하고, `strict=True`를 주면 정확히 일치하는 타입만 허용한다. 정적 검사기는 이런 변환을 전혀 하지 않으므로 "검사(check)"와 "검증·변환(validate)"은 다른 개념임을 구분해 답한다.

- **경계에서만 런타임 검증**: 외부 입력(HTTP 요청, 파일, 환경 변수)은 pydantic 등으로 검증하고, 내부 호출은 정적 검사기에 맡기는 것이 성능·안전의 균형점임
- 검사기는 `.pyi` 스텁 파일과 `py.typed` 마커로 서드파티 라이브러리의 타입 정보를 읽음. 힌트가 없는 라이브러리는 `types-*` 패키지(typeshed)를 설치함

<br>

### 6. 흔한 함정

- **순환 참조·미정의 클래스**: 클래스 안에서 자기 타입을 참조하면 아직 이름이 없어 `NameError`가 남. 문자열 `"Node"`로 쓰거나 `from __future__ import annotations`(PEP 563)로 모든 힌트를 문자열로 지연 평가함. **3.14부터는 PEP 649로 지연 평가가 기본**이 됨
- **가변 기본값 + 힌트**: `items: list[int] = []`도 unit03의 함정 그대로임. 힌트는 이를 막지 못함
- **`list[Dog]`를 `list[Animal]`에 넘기기**: 리스트는 **불변(invariant)**이라 검사기가 거부함. 읽기만 한다면 `Sequence[Animal]`로 받아야 함
- **`Optional` 누락**: 기본값 `None`을 주면서 `x: int = None`으로 쓰면 검사기가 오류를 냄. `int | None`으로 명시함
- **`typing.get_type_hints()`**: 문자열 힌트를 실제 객체로 해석해 주는 함수로, 런타임에 힌트를 읽는 라이브러리를 만들 때 `__annotations__` 대신 사용함

```python
from __future__ import annotations       # 파일 최상단. 힌트를 평가하지 않고 문자열로 저장

class Node:
    def __init__(self, value: int, next: Node | None = None):   # 문자열 없이 자기 참조 가능
        self.value, self.next = value, next
```

<br>

### 7. 정리

- 타입 힌트는 `__annotations__`에 저장되는 **메타데이터**이며 인터프리터는 검사하지 않음 → 성능·덕 타이핑·하위 호환 때문
- 검증은 **정적 검사기(mypy·pyright)**가 실행 전에, **런타임 검증(pydantic 등)**은 외부 입력 경계에서 담당함
- 입력은 넓게(`Sequence`, `Iterable`, `Protocol`) 출력은 좁게(`list`, 구체 클래스) 선언하는 것이 원칙임
- 3.9 `list[int]`, 3.10 `X | Y`, 3.12 `def f[T]`·`type` 문, 3.13 `TypeIs`·`ReadOnly` 등 **버전별 문법 차이**를 프로젝트 최소 버전에 맞춰 선택함
- `Any`·`# type: ignore`는 최소화하고, `assert_never`로 분기 누락을 검사기에 맡길 것
