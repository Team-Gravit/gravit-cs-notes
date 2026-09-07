## 제네릭과 제약

**제네릭(Generic)**은 타입을 매개변수로 받아 "여러 타입에서 동작하되 타입 정보는 잃지 않는" 코드를 만드는 도구다. 실무 타입 설계의 핵심은 제네릭 자체보다 **`extends` 제약·`keyof`·조건부 타입**으로 타입 매개변수를 얼마나 정확하게 좁히느냐에 있다.

<br>

### 1. 제네릭이 필요한 이유

`any`를 쓰면 어떤 타입이든 받을 수 있지만 **들어간 타입과 나오는 타입의 관계**가 끊긴다. 제네릭은 그 관계를 유지한다.

```typescript
// 안티패턴: 입력 타입 정보가 사라짐
function firstAny(arr: any[]): any {
  return arr[0];
}
const a = firstAny([1, 2, 3]); // a: any → 이후 오타·잘못된 호출을 못 잡음

// 개선: 타입 매개변수 T가 입력과 출력을 연결
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
const b = first([1, 2, 3]);       // b: number | undefined (T = number로 추론)
const c = first(["a", "b"]);      // c: string | undefined
```

- `T`는 호출 시점에 **인수로부터 추론**되므로 대부분 `first<number>(...)`처럼 명시할 필요가 없음
- 제네릭은 함수뿐 아니라 인터페이스·타입 별칭·클래스에도 적용됨 (`Array<T>`, `Promise<T>`, `Map<K, V>`)

> 💡 "제네릭 대신 `any`나 `unknown`을 쓰면 안 되는 이유"는 단골 질문이다. 핵심은 **입력과 출력 사이의 타입 관계 보존**이다. `unknown`은 안전하지만 반환값을 매번 좁혀야 하고, `any`는 검사를 아예 포기한다(unit04 참고).

<br>

### 2. `extends` 제약(Constraint)

타입 매개변수에 아무 제약이 없으면 컴파일러는 `T`를 **"어떤 타입이든 될 수 있는 값"**으로 취급해 `.length` 같은 멤버 접근을 허용하지 않는다. `T extends X`는 "T는 최소한 X의 구조를 갖는다"는 약속이다.

```typescript
// 오류: T에 length가 있다는 보장이 없음
function logLength<T>(value: T) {
  console.log(value.length); // 'length' 속성이 'T' 형식에 없습니다.
}

// 개선: 구조적 제약 — length: number를 가진 타입만 허용
function logLengthSafe<T extends { length: number }>(value: T): T {
  console.log(value.length);
  return value;
}
logLengthSafe("문자열");      // OK
logLengthSafe([1, 2, 3]);     // OK
logLengthSafe(42);            // 오류 — number에는 length가 없음
```

**제약의 동작 방식**

- 제약은 구조적 타이핑(unit01)에 따라 판단되므로 `{ length: number }`만 만족하면 어떤 타입이든 통과함
- 함수 본문 안에서 `T`는 제약 타입 `X`의 멤버만 사용할 수 있음. 하지만 반환 타입으로 `T`를 돌려주면 호출부에서는 **원래 타입 그대로** 유지됨 (`logLengthSafe("문자열")`의 반환은 `string`)
- 제약을 `X`로 두고 반환도 `X`로 하면 제네릭의 의미가 없어짐 — 그때는 그냥 매개변수 타입을 `X`로 선언하면 됨

| **선언**                       | **의미**                                       | **사용 예**                          |
| ------------------------------ | ---------------------------------------------- | ------------------------------------ |
| **`<T>`**                      | 아무 타입. 본문에서 `T`의 멤버 접근 불가       | 컨테이너·식별 함수                   |
| **`<T extends object>`**       | 원시값 제외, 객체만                            | 객체를 순회·복제하는 유틸            |
| **`<T extends string \| number>`** | 특정 유니온으로 한정                        | 키로 쓸 수 있는 값                   |
| **`<T extends Base>`**         | `Base` 구조를 가진 타입                        | 엔티티 공통 필드(`id`)를 다루는 리포지토리 |
| **`<T = string>`**             | 기본 타입 (제약 아님, 추론 실패 시 사용)        | 옵션형 제네릭 인터페이스             |

<br>

### 3. `keyof`와 인덱스 접근 타입

**`keyof T`**는 객체 타입 `T`의 **프로퍼티 이름들의 유니온**을 만든다. **`T[K]`(인덱스 접근 타입)**는 그 키에 해당하는 프로퍼티의 타입을 꺼낸다. 두 가지를 제약과 조합하면 "객체와 그 객체의 실제 키만 받는" 함수를 만들 수 있다.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}
type UserKey = keyof User; // "id" | "name" | "email"

// K는 반드시 T의 키 중 하나여야 하고, 반환 타입은 그 키의 값 타입
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { id: 1, name: "김철수", email: "kim@example.com" };
const id = getProp(user, "id");     // number
const name = getProp(user, "name"); // string
getProp(user, "age");               // 오류 — "age"는 keyof User에 없음
```

```
keyof User ──▶ "id" | "name" | "email"
                 │
K extends keyof User: K는 위 유니온 중 하나로만 추론
                 │
User[K] ──▶ K = "id"   이면 number
            K = "name" 이면 string
```

> ⚠️ `keyof`의 결과는 `string`만이 아니다. 인덱스 시그니처 `[key: string]: T`를 가진 타입에서 `keyof`는 **`string | number`**가 된다(JS에서 숫자 키는 문자열로 변환되므로). 문자열 키만 필요하면 `keyof T & string`으로 교차시켜야 한다.

<br>

### 4. 조건부 타입 기초

### 4-1. `T extends U ? X : Y`

**조건부 타입(Conditional Type)**은 타입 수준의 삼항 연산자다. `T`가 `U`에 대입 가능하면 `X`, 아니면 `Y`로 평가된다. 제네릭 인수에 따라 반환 타입을 바꿔야 할 때 사용한다.

```typescript
type IsString<T> = T extends string ? "yes" : "no";

type A = IsString<"hello">; // "yes"
type B = IsString<number>;  // "no"

// 실무 예: id를 문자열로 주면 string 결과, 숫자로 주면 number 결과
type IdType<T> = T extends string ? { id: string } : { id: number };
function toEntity<T extends string | number>(id: T): IdType<T> {
  return { id } as IdType<T>;
}
```

<br>

### 4-2. 분산(Distributive) 조건부 타입

검사 대상 `T`가 **단독 타입 매개변수**이고 유니온이 들어오면, 조건부 타입은 유니온의 **각 멤버에 개별 적용**된 뒤 결과가 다시 유니온으로 합쳐진다. 내장 `Exclude`·`Extract`가 이 성질로 구현되어 있다.

```typescript
type ToArray<T> = T extends unknown ? T[] : never;
type R1 = ToArray<string | number>;   // string[] | number[]  (분산됨)

type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;
type R2 = ToArrayNonDist<string | number>; // (string | number)[]  (분산 안 됨)

type MyExclude<T, U> = T extends U ? never : T;
type R3 = MyExclude<"a" | "b" | "c", "a">; // "b" | "c"  — never는 유니온에서 사라짐
```

```
ToArray<string | number>
   ├─ string 에 적용 → string[]
   └─ number 에 적용 → number[]
   ⇒ string[] | number[]

[T] extends [unknown] 처럼 튜플로 감싸면 분산 차단 → (string | number)[]
```

> 💡 분산 조건부 타입이 `never`를 만나면 결과도 `never`가 된다(빈 유니온에 분산하면 빈 유니온). `IsNever<T> = T extends never ? true : false`가 항상 `never`를 돌려주는 유명한 함정이 여기서 나오며, 튜플로 감싸 `[T] extends [never]`로 써야 의도대로 동작한다.

<br>

### 4-3. `infer` — 타입 안에서 타입 꺼내기

조건부 타입의 `extends` 절에서 **`infer R`**을 쓰면 매칭되는 위치의 타입을 변수 `R`로 잡아낼 수 있다. 내장 `ReturnType`·`Parameters`·`Awaited`가 이 방식이다(유틸리티 타입은 unit06 참고).

```typescript
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function fetchUser() {
  return { id: 1, name: "김철수" };
}
type FetchResult = MyReturnType<typeof fetchUser>; // { id: number; name: string }

type ElementOf<T> = T extends (infer E)[] ? E : T;
type E1 = ElementOf<string[]>; // string
type E2 = ElementOf<number>;   // number (배열이 아니면 그대로)
```

<br>

### 5. 제네릭 설계의 함정과 선택 기준

| **상황**                                     | **문제**                                              | **개선**                                                       |
| -------------------------------------------- | ----------------------------------------------------- | -------------------------------------------------------------- |
| **한 번만 쓰이는 타입 매개변수**             | `<T>(x: T): void` — T가 입출력을 연결하지 않음        | 제네릭 제거, `unknown`이나 구체 타입으로 선언                  |
| **제약 없이 멤버 접근**                      | `T`에 `.length` 등 접근 시 오류                       | `T extends { length: number }`처럼 **필요한 최소 구조**만 제약 |
| **제약과 반환이 같은 타입**                  | `<T extends User>(u: T): User` — 제네릭 의미 상실     | 반환을 `T`로 돌려 호출부 타입 보존                             |
| **리터럴 타입을 유지하고 싶을 때**           | `["a", "b"]`가 `string[]`으로 넓혀짐                  | TypeScript 5.0+의 **`const` 타입 매개변수** `<const T>` 사용   |
| **추론 대상이 아닌 매개변수에서 추론이 일어남** | 기본값 인수가 `T` 추론을 오염시킴                  | TypeScript 5.4+의 **`NoInfer<T>`**로 추론 제외                 |

```typescript
// const 타입 매개변수: 인수를 readonly 튜플 리터럴로 추론
function tuple<const T extends readonly unknown[]>(...items: T): T {
  return items;
}
const t = tuple("a", "b"); // readonly ["a", "b"]  (const 없으면 string[])
```

> ⚠️ 제네릭은 **호출부의 타입 정보를 보존할 때만** 가치가 있다. 타입 매개변수가 한 곳에만 등장한다면 그 함수는 제네릭일 이유가 없고, 오히려 오류 메시지만 복잡해진다. "이 `T`가 무엇과 무엇을 연결하는가"를 설명할 수 없다면 제거하는 편이 낫다.

<br>

### 6. 정리 — 면접·실무 체크포인트

| **질문**                                  | **핵심 답변**                                                                                     |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **제네릭 vs `any`**                       | 제네릭은 **입력·출력 타입 관계를 보존**, `any`는 검사 포기                                        |
| **`extends` 제약의 역할**                 | `T`가 최소한 갖춰야 할 **구조를 약속**해 본문에서 멤버 접근을 허용. 호출부 타입은 그대로 유지     |
| **`keyof`와 `T[K]`**                      | 키 유니온과 인덱스 접근 타입. `K extends keyof T`로 **실제 존재하는 키만** 받게 제약               |
| **조건부 타입이란**                       | `T extends U ? X : Y` — 타입 수준 분기. 유니온에 **분산** 적용되며 `[T]`로 감싸면 분산 차단        |
| **`infer`의 용도**                        | 조건부 타입 안에서 **부분 타입을 추출** (`ReturnType`, `Awaited` 등의 구현 원리)                  |
| **불필요한 제네릭 판단 기준**             | 타입 매개변수가 **두 곳 이상을 연결하지 않으면** 제거                                             |

- 조건부·매핑 타입으로 만들어진 내장 유틸리티 타입의 목록과 구현은 **unit06**, 제네릭이 런타임에 어떻게 사라지는지는 **unit07**을 참고할 것
