## 타입 좁히기

**타입 좁히기(Narrowing)**는 `string | number`처럼 넓은 타입의 값을 코드 흐름에 따라 더 구체적인 타입으로 줄여 나가는 과정이다. 컴파일러가 제어 흐름을 읽어 자동으로 좁혀 주는 원리를 알면 단언(`as`) 없이도 안전한 코드를 쓸 수 있고, **판별 유니온(Discriminated Union)**과 **`satisfies`**는 그 원리를 설계 단계에서 활용하는 도구다.

<br>

### 1. 제어 흐름 분석(Control Flow Analysis)

TypeScript는 `if`·`return`·`throw`·논리 연산자 등을 따라가며 **각 지점에서 변수가 가질 수 있는 타입**을 계산한다. 값이 흐르는 경로마다 타입이 달라지므로 같은 변수라도 위치에 따라 다른 타입으로 취급된다.

```typescript
function format(value: string | number | null) {
  if (value === null) {
    return "없음";          // 여기서 value: null
  }
  // 이 아래에서 value: string | number (null 제거됨)
  if (typeof value === "string") {
    return value.toUpperCase(); // value: string
  }
  return value.toFixed(2);      // value: number
}
```

```
value: string | number | null
   │
   ├─ value === null ──▶ true  ──▶ return  (value: null)
   │
   ▼ (false 분기)  value: string | number
   ├─ typeof === "string" ──▶ true ──▶ value: string
   │
   ▼ (false 분기)  value: number
```

> 💡 좁히기는 **읽기 전용 참조**에 대해서만 안정적으로 유지된다. `let` 변수는 이후 대입에 의해 넓어질 수 있고, 콜백 함수 안에서는 바깥에서 좁힌 결과가 **초기화되어** 다시 넓은 타입으로 돌아간다(콜백이 나중에 실행될 때 값이 바뀌었을 수 있으므로). `const`에 담아 두면 콜백 안에서도 좁힘이 유지된다.

<br>

### 2. 내장 타입 가드(Type Guard)

| **가드**                     | **좁힐 수 있는 대상**                | **예시**                                   |
| ---------------------------- | ------------------------------------ | ------------------------------------------ |
| **`typeof x === "..."`**     | 원시 타입(`string`·`number`·`boolean`·`bigint`·`symbol`·`undefined`·`function`·`object`) | `typeof v === "number"`   |
| **`x instanceof C`**         | 클래스 인스턴스(프로토타입 체인)     | `err instanceof HttpError`                 |
| **`"key" in x`**             | 특정 프로퍼티 유무로 객체 유니온 분기 | `"swim" in animal`                         |
| **동등 비교(`===`, `!==`)**  | 리터럴·`null`·`undefined`            | `status === "done"`                        |
| **진릿값 검사(`if (x)`)**    | `null`·`undefined`·`""`·`0` 제거     | `if (name) name.trim()`                    |
| **`Array.isArray(x)`**       | 배열 여부                            | `Array.isArray(input)`                     |

```typescript
class HttpError extends Error {
  constructor(public status: number, message: string) {
    super(message);
  }
}

function handle(err: unknown) {
  if (err instanceof HttpError) {
    return `HTTP ${err.status}`;      // err: HttpError
  }
  if (err instanceof Error) {
    return err.message;               // err: Error
  }
  return String(err);                 // err: unknown
}
```

> ⚠️ `typeof null === "object"`이므로 `typeof v === "object"`로 좁혀도 `null`은 남는다. 컴파일러도 이를 알고 `object | null`로 좁히므로 `v !== null` 검사를 추가해야 한다. 진릿값 검사 역시 `0`과 `""`를 함께 걸러내므로, "값이 없음"만 판별하려면 `!= null`을 쓰는 편이 안전하다.

<br>

### 3. 사용자 정의 타입 가드

### 3-1. 타입 서술(Type Predicate) — `x is T`

내장 가드로 표현하기 어려운 조건은 반환 타입을 **`매개변수 is 타입`**으로 선언한 함수로 만든다. 이 함수가 `true`를 반환한 분기에서 컴파일러는 인수를 해당 타입으로 좁힌다.

```typescript
interface Fish { swim(): void }
interface Bird { fly(): void }

function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined;
}

function move(pet: Fish | Bird) {
  if (isFish(pet)) pet.swim(); // pet: Fish
  else pet.fly();              // pet: Bird
}

const pets: (Fish | Bird)[] = [];
const fishes = pets.filter(isFish); // Fish[] — filter 오버로드가 타입 서술을 인식
```

TypeScript 5.5부터는 `(x) => typeof x === "string"`처럼 **본문이 단순한 함수는 타입 서술을 자동 추론**하므로 반환 타입을 직접 쓰지 않아도 되는 경우가 늘었다.

<br>

### 3-2. 단언 함수(Assertion Function) — `asserts x is T`

조건이 틀리면 **예외를 던지는** 검증 함수는 `asserts 매개변수 is 타입`으로 선언한다. 함수 호출 이후의 코드에서 인수가 좁혀진 채로 유지된다.

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError("문자열이 아닙니다");
  }
}

function shout(input: unknown) {
  assertIsString(input);
  return input.toUpperCase(); // input: string — 예외 없이 지나왔다면 문자열
}
```

> ⚠️ 타입 서술·단언 함수의 **본문은 컴파일러가 검증하지 않는다.** `pet is Fish`를 선언해 놓고 본문에서 엉뚱한 검사를 하면 잘못된 타입이 통과된다. 사용자 정의 가드는 "컴파일러에게 내가 책임진다"고 선언하는 것이므로 본문의 검사 로직을 단위 테스트로 보호하는 것이 좋다.

<br>

### 4. 판별 유니온(Discriminated Union)

여러 객체 타입이 **공통 리터럴 프로퍼티(판별자, discriminant)**를 갖도록 설계하면, 그 프로퍼티 비교만으로 유니온 전체가 정확히 좁혀진다. 상태 머신·API 응답·이벤트 등 "종류에 따라 데이터 모양이 달라지는" 모델링의 표준 패턴이다.

```typescript
// 안티패턴: 선택적 프로퍼티로 뭉뚱그림 — 어떤 조합이 유효한지 타입이 말해주지 않음
interface ShapeLoose {
  kind: "circle" | "square";
  radius?: number;
  side?: number;
}

// 개선: 종류별로 타입을 나누고 kind를 판별자로 사용
interface Circle { kind: "circle"; radius: number }
interface Square { kind: "square"; side: number }
type Shape = Circle | Square;

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2; // shape: Circle
    case "square":
      return shape.side ** 2;             // shape: Square
  }
}
```

**설계 규칙**

- 판별자는 **리터럴 타입**(문자열·숫자 리터럴, `true`/`false`)이어야 함. `string`으로 선언하면 좁히기가 일어나지 않음
- 판별자 이름은 유니온의 모든 멤버에서 **같아야** 함 (`kind`, `type`, `status` 등)
- 객체를 구조 분해한 뒤에도 좁힘이 유지되려면 `const { kind, radius } = shape`처럼 **`const` 구조 분해**여야 함 (TypeScript 4.6+)

<br>

### 4-1. 완전성 검사(Exhaustiveness Check)

`never`를 이용하면 모든 분기를 처리했는지 컴파일 타임에 강제할 수 있다. 나중에 유니온에 멤버가 추가되었을 때 **처리하지 않은 분기가 컴파일 오류로 드러난다**(`never`의 의미는 unit04 참고).

```typescript
function assertNever(x: never): never {
  throw new Error(`처리되지 않은 케이스: ${JSON.stringify(x)}`);
}

function describe(shape: Shape): string {
  switch (shape.kind) {
    case "circle": return "원";
    case "square": return "정사각형";
    default:
      return assertNever(shape); // Shape에 Triangle이 추가되면 여기서 오류 발생
  }
}
```

```
Shape = Circle | Square (| Triangle 추가 시)
 switch(kind)
   ├─ "circle" 처리 → 남은 타입: Square (| Triangle)
   ├─ "square" 처리 → 남은 타입: never   (| Triangle)
   └─ default: assertNever(shape)
        └─ 남은 타입이 never가 아니면 컴파일 오류 → 누락된 분기 발견
```

<br>

### 5. `satisfies` — 검사는 하되 추론은 유지

TypeScript 4.9에서 추가된 **`satisfies`** 연산자는 "이 값이 타입 `T`를 만족하는지 검사하되, 변수의 타입은 **더 좁게 추론된 원래 타입**으로 유지"한다. 타입 표기(`: T`)는 검사와 동시에 변수 타입을 `T`로 **넓혀 버리는** 반면, `satisfies`는 좁힘 정보를 잃지 않는다.

```typescript
type Theme = Record<string, string | [number, number, number]>;

// 타입 표기: 검사는 되지만 각 값이 string | [number, number, number]로 넓어짐
const theme1: Theme = { primary: "#0af", accent: [255, 0, 0] };
theme1.primary.toUpperCase(); // 오류 — 튜플일 수도 있다고 판단

// satisfies: 검사도 되고, 각 프로퍼티는 리터럴 그대로 추론됨
const theme2 = { primary: "#0af", accent: [255, 0, 0] } satisfies Theme;
theme2.primary.toUpperCase(); // OK — primary: string
theme2.accent[0];             // OK — accent: [number, number, number]
theme2.secondary;             // 오류 — 존재하지 않는 키 (초과 프로퍼티 검사도 적용)
```

| **구문**              | **타입 검사** | **변수의 최종 타입**           | **적합한 상황**                                  |
| --------------------- | ------------- | ------------------------------ | ------------------------------------------------ |
| **`const x: T = v`**  | O             | **`T`** (넓어짐)               | 외부에 `T`로 노출해야 할 때                       |
| **`const x = v as T`** | 최소한만      | `T` (단언 — 안전성 약함)       | 컴파일러보다 개발자가 더 잘 아는 특수 상황       |
| **`const x = v satisfies T`** | O      | **`v`의 추론 타입** (좁게 유지) | 설정 객체·상수 맵처럼 키와 리터럴을 살려야 할 때 |

> 💡 `satisfies`는 `as const`와 자주 함께 쓰인다. `{ ... } as const satisfies Theme`처럼 쓰면 값은 읽기 전용 리터럴로 고정하면서도 `Theme` 구조를 만족하는지 검사할 수 있어, 라우트 테이블·i18n 키 맵 같은 상수 정의에 특히 유용하다.

<br>

### 6. 정리 — 면접·실무 체크포인트

| **질문**                                     | **핵심 답변**                                                                                     |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **타입 좁히기는 어떻게 동작하는가**          | **제어 흐름 분석** — 분기·반환·예외를 따라 각 지점의 가능한 타입을 계산                           |
| **내장 가드 종류**                           | `typeof`·`instanceof`·`in`·동등 비교·진릿값 검사. `typeof null`은 `"object"`임에 주의            |
| **사용자 정의 가드 두 가지**                 | `x is T`(불리언 반환)와 `asserts x is T`(예외 던짐). **본문은 검증되지 않으므로** 테스트 필요     |
| **판별 유니온이란**                          | 공통 **리터럴 판별자**로 분기하는 유니온. 선택적 프로퍼티 남발 대신 종류별 타입 분리              |
| **완전성 검사 방법**                         | `default` 분기에서 `never` 매개변수 함수 호출 → 멤버 추가 시 **컴파일 오류로 누락 발견**           |
| **`satisfies` vs 타입 표기**                 | 둘 다 검사하지만 `satisfies`는 **추론된 좁은 타입을 유지**. TypeScript 4.9+                        |

- `unknown`을 좁혀서 쓰는 원칙과 `never`의 의미는 **unit04**, 런타임 데이터(JSON·API 응답)를 검증해 좁히는 방법은 **unit07**을 참고할 것
