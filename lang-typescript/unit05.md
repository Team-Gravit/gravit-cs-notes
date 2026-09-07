## 인터페이스와 타입 별칭

객체의 모양을 정의하는 두 문법 **인터페이스(`interface`)**와 **타입 별칭(`type`)**은 대부분의 경우 서로 바꿔 쓸 수 있지만, **선언 병합(Declaration Merging)**·**확장 방식**·**표현 범위**에서 뚜렷한 차이가 있다. "언제 무엇을 쓰는가"를 원리 수준에서 설명할 수 있어야 한다.

<br>

### 1. 공통점 — 객체 타입을 정의하는 두 방법

```typescript
interface UserI {
  id: number;
  name: string;
  greet(): string;
}

type UserT = {
  id: number;
  name: string;
  greet(): string;
};

// 구조가 같으므로 서로 대입 가능 (구조적 타이핑, unit01 참고)
const a: UserI = { id: 1, name: "김철수", greet: () => "안녕" };
const b: UserT = a;
```

- 둘 다 객체 타입·함수 타입·제네릭·`readonly`·선택적 프로퍼티·인덱스 시그니처를 표현할 수 있음
- 클래스는 둘 중 어느 쪽이든 `implements`할 수 있음 (단, 유니온 타입은 `implements` 불가)
- 런타임 코드로 남지 않는다는 점도 같음 (타입 소거, unit07 참고)

<br>

### 2. 차이점 ① — 표현 범위

**타입 별칭은 "이름 붙이기"**이므로 객체 타입뿐 아니라 어떤 타입에도 이름을 줄 수 있다. **인터페이스는 객체(또는 함수·클래스)의 모양만** 선언한다.

```typescript
// 타입 별칭만 가능한 것들
type ID = string | number;                       // 유니온
type Point = [number, number];                   // 튜플
type Handler = (e: Event) => void;               // 함수 (인터페이스로도 가능하나 별칭이 간결)
type Keys = keyof UserT;                         // 타입 연산 결과
type Nullable<T> = T | null;                     // 조건부·매핑 타입의 이름
type EventName = `on${Capitalize<string>}`;      // 템플릿 리터럴 타입

interface Bad = string | number;                 // 문법 오류 — 인터페이스는 유니온 불가
```

| **표현**                          | **`interface`** | **`type`** |
| --------------------------------- | --------------- | ---------- |
| **객체 모양**                     | O               | O          |
| **유니온·교차·튜플·원시 타입 별칭** | **X**          | **O**      |
| **매핑·조건부·템플릿 리터럴 타입** | X              | **O**      |
| **선언 병합**                     | **O**           | **X**      |
| **`extends`로 확장**              | **O**           | X (`&`로 대체) |
| **클래스 `implements` 대상**      | O               | O (객체 타입일 때) |

<br>

### 3. 차이점 ② — 확장 방식: `extends` vs 교차 타입 `&`

### 3-1. 문법과 동작

인터페이스는 **`extends`**로, 타입 별칭은 **교차 타입(Intersection, `&`)**으로 기존 타입을 확장한다. 두 방식은 서로 섞어 쓸 수도 있다(인터페이스가 타입 별칭을 `extends`하거나, 타입 별칭이 인터페이스를 `&`로 결합).

```typescript
interface Animal { name: string }

// 인터페이스 확장
interface Dog extends Animal { bark(): void }

// 타입 별칭 확장 (교차)
type Cat = Animal & { meow(): void };

// 혼합도 가능
interface Puppy extends Dog, Cat {}          // 여러 타입 동시 확장
type Kitten = Cat & { age: number };
```

```
extends (인터페이스)                 & 교차 (타입 별칭)
─────────────────────               ─────────────────────
Animal ──extends──▶ Dog             Animal ─┐
  · 확장 시점에 충돌 검사                     ├─&─▶ Cat
  · 결과가 이름 있는 타입으로 캐시            · 두 타입의 프로퍼티 합집합을 계산
                                             · 충돌 시 오류 대신 프로퍼티가 never
```

<br>

### 3-2. 충돌 처리의 차이 — 가장 중요한 실무 차이

같은 이름의 프로퍼티가 **호환되지 않는 타입**으로 겹칠 때 두 방식의 반응이 다르다.

```typescript
interface Base { value: string }

// extends: 확장 시점에 즉시 오류
interface Ext extends Base {
  value: number; // 오류 — 'value' 형식이 'Base'의 같은 속성과 호환되지 않음
}

// 교차 타입: 오류 없이 통과하지만 value는 string & number = never
type Mixed = Base & { value: number };
const m: Mixed = { value: 1 }; // 여기서야 오류 — number를 never에 대입 불가
```

> ⚠️ 교차 타입의 프로퍼티 충돌은 **선언 시점이 아니라 사용 시점**에 발견되며, 오류 메시지도 "`never`에 대입할 수 없다"는 식이라 원인을 찾기 어렵다. 여러 타입을 합성하는 코드에서 정체불명의 `never`가 나타난다면 교차 타입의 프로퍼티 충돌을 먼저 의심해야 한다. 상속 계층이 명확한 객체 모델은 `extends`가 오류를 더 빨리 드러낸다.

<br>

### 4. 차이점 ③ — 선언 병합(Declaration Merging)

**같은 이름의 인터페이스를 여러 번 선언하면 하나로 합쳐진다.** 타입 별칭은 같은 이름을 두 번 선언하면 "중복 식별자" 오류다.

```typescript
interface Window {
  myAppVersion: string;         // 기존 Window 인터페이스에 프로퍼티 추가
}
window.myAppVersion = "1.0.0";  // OK — 전역 lib.dom.d.ts의 Window와 병합됨

interface Config { host: string }
interface Config { port: number }
const cfg: Config = { host: "localhost", port: 3000 }; // 두 선언이 합쳐짐

type Alias = { a: 1 };
type Alias = { b: 2 };          // 오류 — 'Alias' 식별자가 중복되었습니다.
```

**병합 규칙**

- 프로퍼티는 합집합이 되며, **같은 이름은 타입이 동일해야** 함 (다르면 오류)
- 같은 이름의 메서드(함수 프로퍼티)는 **오버로드로 누적**되며, 나중에 선언된 것이 먼저 매칭됨
- 병합은 같은 스코프(모듈 또는 전역)에서만 일어남. ES 모듈 파일에서 전역 타입을 확장하려면 `declare global { interface Window { ... } }` 블록이 필요함

```typescript
// 라이브러리 타입 확장(모듈 보강, Module Augmentation)의 전형적 형태
import "express";

declare module "express-serve-static-core" {
  interface Request {
    user?: { id: number; role: string }; // 미들웨어가 채워 넣는 필드
  }
}
```

> 💡 선언 병합은 **라이브러리 사용자가 타입을 확장하는 공식 통로**다. Express의 `Request`, Vue의 `ComponentCustomProperties`, styled-components의 `DefaultTheme` 등이 모두 "사용자가 병합으로 채워 넣도록" 인터페이스로 선언되어 있다. 반대로 **내 코드의 타입이 외부에서 몰래 바뀌는 것을 막고 싶다면** 타입 별칭이 안전하다.

<br>

### 5. 그 밖의 실무 차이

| **항목**                 | **`interface`**                                            | **`type`**                                                       |
| ------------------------ | ---------------------------------------------------------- | ---------------------------------------------------------------- |
| **오류 메시지·호버 표시** | 항상 **이름**으로 표시됨                                   | 종종 풀어쓴 구조로 표시되어 길어질 수 있음 (버전에 따라 다름)     |
| **타입 검사 비용**       | `extends` 결과가 **캐시**되어 대규모 코드에서 유리          | 교차 타입은 매번 재계산될 수 있음                                |
| **암시적 인덱스 시그니처** | 없음 — `Record<string, unknown>`에 바로 대입 **불가**      | 있음 — 객체 리터럴 타입 별칭은 인덱스 시그니처 타입에 대입 가능   |
| **재귀 타입**            | 가능                                                       | 가능 (3.7+ 이후 대부분의 재귀 별칭 허용)                          |

```typescript
interface Point { x: number; y: number }
type PointT = { x: number; y: number };

function log(obj: Record<string, unknown>) {}
declare const p: Point;
declare const pt: PointT;
log(pt); // OK — 타입 별칭은 암시적 인덱스 시그니처를 가짐
log(p);  // 오류 — 인터페이스는 선언 병합으로 나중에 바뀔 수 있어 인덱스 시그니처를 암시하지 않음
```

> 💡 인터페이스가 `Record<string, unknown>`에 대입되지 않는 이유는 선언 병합과 연결되어 있다. 인터페이스는 **나중에 다른 파일에서 병합되어 호환되지 않는 프로퍼티가 추가될 수 있으므로** 컴파일러가 "모든 키가 `unknown`"이라고 단정하지 못한다. 타입 별칭은 닫혀 있어 단정할 수 있다.

<br>

### 6. 선택 기준

| **상황**                                                | **권장**        | **이유**                                             |
| ------------------------------------------------------- | --------------- | ---------------------------------------------------- |
| **객체 모양 정의 (특히 공개 API·라이브러리)**            | **`interface`** | 확장·병합 가능, 오류 메시지 명확, 검사 비용 유리       |
| **유니온·튜플·원시 타입·함수 시그니처에 이름 붙이기**    | **`type`**      | 인터페이스로 표현 불가                                |
| **매핑·조건부·템플릿 리터럴 등 타입 연산 결과**          | **`type`**      | 인터페이스로 표현 불가                                |
| **외부 라이브러리 타입 확장**                            | **`interface`** | 선언 병합·모듈 보강이 유일한 수단                     |
| **외부에서 변경되면 안 되는 닫힌 타입**                  | **`type`**      | 병합이 불가능해 의도치 않은 확장 차단                 |
| **상속 계층이 있는 도메인 모델**                         | **`interface`** | `extends`가 충돌을 선언 시점에 잡음                   |

- 팀 컨벤션으로 통일하는 것이 가장 중요하다. TypeScript 공식 핸드북은 "객체 타입에는 우선 `interface`를 쓰고, 그 기능이 필요할 때 `type`을 쓰라"고 안내하지만, "모든 것을 `type`으로 통일"하는 팀도 많으며 둘 다 합리적인 선택이다

<br>

### 7. 정리 — 면접·실무 체크포인트

- **공통점**: 객체 모양 정의, `implements` 가능, 런타임 코드 없음, 구조적 타이핑으로 상호 대입 가능
- **표현 범위**: `type`은 **유니온·튜플·조건부·매핑 타입** 등 어떤 타입에도 이름 부여 가능, `interface`는 객체 모양 전용
- **확장**: `interface extends`는 **충돌을 선언 시점에 오류**로 잡고, `type &`는 충돌 프로퍼티를 **`never`로 만들어 사용 시점에 드러남**
- **선언 병합**: `interface`만 가능. 라이브러리 타입 확장(`declare module`, `declare global`)의 표준 수단이자, 닫힌 타입이 필요할 때는 단점
- **암시적 인덱스 시그니처**: `type`만 가짐 → 인터페이스는 `Record<string, unknown>`에 바로 대입 불가
- 유틸리티·매핑 타입으로 기존 타입을 변형하는 방법은 **unit06**을 참고할 것
