## 유틸리티 타입과 매핑 타입

**유틸리티 타입(Utility Type)**은 기존 타입을 변형해 새 타입을 만드는 내장 제네릭 타입이며, 그 대부분은 **매핑 타입(Mapped Type)**과 조건부 타입으로 구현되어 있다. 내장 타입을 외우는 것보다 "어떻게 만들어졌는가"를 이해해야 팀 도메인에 맞는 타입 변환을 직접 작성할 수 있다.

<br>

### 1. 매핑 타입의 원리

매핑 타입은 **키 집합을 순회하며 각 키에 대한 프로퍼티를 생성**하는 문법이다. `for...in`을 타입 수준으로 옮긴 것으로 이해하면 된다.

```typescript
type Flags<K extends string> = {
  [P in K]: boolean;   // K의 각 멤버 P마다 boolean 프로퍼티 생성
};
type FeatureFlags = Flags<"darkMode" | "beta">;
// { darkMode: boolean; beta: boolean }

// 기존 객체의 키를 순회: keyof T + 인덱스 접근 T[P] (unit02 참고)
type Copy<T> = {
  [P in keyof T]: T[P];
};
```

```
keyof T ──▶ "id" | "name" | "email"
               │ P in ...
               ├─ P = "id"    ──▶ { id:    T["id"]    }
               ├─ P = "name"  ──▶ { name:  T["name"]  }
               └─ P = "email" ──▶ { email: T["email"] }
                          ⇒ 합쳐서 새 객체 타입
```

**제어자(modifier) 조작** — 매핑 중 `readonly`와 `?`를 붙이거나(`+`) 떼어낼(`-`) 수 있다.

```typescript
type Mutable<T> = { -readonly [P in keyof T]: T[P] };   // readonly 제거
type Concrete<T> = { [P in keyof T]-?: T[P] };           // 선택적(?) 제거 → Required와 동일
```

> 💡 `[P in keyof T]` 형태처럼 **`keyof T`를 그대로 순회하는 매핑 타입을 동형(homomorphic) 매핑 타입**이라 한다. 동형 매핑은 원본의 `readonly`·`?` 제어자를 **자동으로 보존**하고, `T`가 배열·튜플이면 결과도 배열·튜플이 되며, 원시 타입에는 적용되지 않고 그대로 통과된다. 내장 `Partial`·`Readonly`·`Pick`이 이 성질에 의존한다.

<br>

### 2. 객체 변형 유틸리티 — Partial·Required·Readonly·Pick·Omit

| **타입**             | **동작**                                   | **구현 (핵심)**                                         |
| -------------------- | ------------------------------------------ | ------------------------------------------------------- |
| **`Partial<T>`**     | 모든 프로퍼티를 **선택적**으로             | `{ [P in keyof T]?: T[P] }`                             |
| **`Required<T>`**    | 모든 프로퍼티를 **필수**로                 | `{ [P in keyof T]-?: T[P] }`                            |
| **`Readonly<T>`**    | 모든 프로퍼티를 **읽기 전용**으로          | `{ readonly [P in keyof T]: T[P] }`                     |
| **`Pick<T, K>`**     | `K`에 해당하는 프로퍼티만 **선택**         | `{ [P in K]: T[P] }` (`K extends keyof T`)              |
| **`Omit<T, K>`**     | `K`에 해당하는 프로퍼티를 **제외**         | `Pick<T, Exclude<keyof T, K>>` (`K extends keyof any`)  |

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type UserUpdate = Partial<Omit<User, "id">>;          // 수정 요청: id 제외, 나머지 선택적
type PublicUser = Omit<User, "password">;             // 응답: 비밀번호 제거
type UserPreview = Pick<User, "id" | "name">;         // 목록 카드용 최소 필드

function updateUser(id: number, patch: UserUpdate): PublicUser {
  const updated: User = { ...findUser(id), ...patch };
  const { password, ...publicUser } = updated;
  return publicUser;
}
declare function findUser(id: number): User;
```

> ⚠️ `Pick`의 `K`는 `keyof T`로 제약되어 존재하지 않는 키를 넘기면 오류지만, **`Omit`의 `K`는 `keyof any`(`string | number | symbol`)로 느슨하게 제약**되어 있어 오타를 잡아주지 않는다. `Omit<User, "pasword">`는 조용히 통과되어 `password`가 응답에 남는다. 엄격한 검사가 필요하면 `type StrictOmit<T, K extends keyof T> = Omit<T, K>`를 정의해 쓴다.

**`Partial`의 흔한 함정** — `Partial<T>`는 "프로퍼티가 없어도 됨"일 뿐, `undefined`를 명시적으로 넣는 것도 허용된다(`exactOptionalPropertyTypes`가 꺼져 있을 때). `{ ...entity, ...patch }`에서 `patch.name = undefined`이면 기존 값이 `undefined`로 **덮어써진다**.

<br>

### 3. 유니온·함수 유틸리티 — Record·Exclude·Extract·NonNullable·ReturnType

| **타입**                  | **동작**                                          | **예시 결과**                                       |
| ------------------------- | ------------------------------------------------- | --------------------------------------------------- |
| **`Record<K, V>`**        | 키 `K` 전부에 값 `V`를 가진 객체                  | `Record<"a" \| "b", number>` → `{ a: number; b: number }` |
| **`Exclude<T, U>`**       | 유니온 `T`에서 `U`에 해당하는 멤버 **제거**       | `Exclude<"a" \| "b", "a">` → `"b"`                  |
| **`Extract<T, U>`**       | 유니온 `T`에서 `U`에 해당하는 멤버만 **추출**     | `Extract<string \| number, number>` → `number`      |
| **`NonNullable<T>`**      | `null`·`undefined` 제거                           | `NonNullable<string \| null>` → `string`            |
| **`ReturnType<F>`**       | 함수 반환 타입                                    | `ReturnType<() => number>` → `number`               |
| **`Parameters<F>`**       | 매개변수 타입의 튜플                              | `Parameters<(a: string, b: number) => void>` → `[string, number]` |
| **`Awaited<T>`**          | `Promise`를 재귀적으로 벗긴 타입 (4.5+)            | `Awaited<Promise<Promise<string>>>` → `string`       |

```typescript
type Status = "idle" | "loading" | "done" | "error";

// Record로 "모든 상태에 대해 빠짐없이" 매핑 강제 — 키가 빠지면 컴파일 오류
const statusLabel: Record<Status, string> = {
  idle: "대기", loading: "로딩 중", done: "완료", error: "오류",
};

// 함수 타입에서 파생 — 반환 타입을 따로 선언하지 않고 재사용
async function fetchUser(id: number) {
  return { id, name: "김철수", roles: ["admin"] as const };
}
type FetchedUser = Awaited<ReturnType<typeof fetchUser>>;
// { id: number; name: string; roles: readonly ["admin"] }
```

> 💡 `Exclude`·`Extract`·`NonNullable`은 **분산 조건부 타입**(unit02)으로 구현되어 유니온 각 멤버에 개별 적용된다. `Omit`은 `keyof`를 먼저 계산하므로 분산되지 않는다는 점이 중요한 차이다 — `Omit<A | B, K>`는 `A`와 `B`의 **공통 키만** 남긴 뒤 제외하므로 유니온 구조가 사라진다. 유니온 각 멤버에 `Omit`을 적용하려면 `T extends unknown ? Omit<T, K> : never` 형태의 분산 래퍼를 직접 만들어야 한다.

<br>

### 4. 키 재지정(Key Remapping) — `as` 절

TypeScript 4.1부터 매핑 타입의 키 부분에 **`as` 절**을 붙여 키 이름을 바꾸거나 일부 키를 걸러낼 수 있다. **템플릿 리터럴 타입**과 결합하면 `getName`·`onClick` 같은 규칙적인 이름을 타입 수준에서 생성할 수 있다.

<br>

### 4-1. 키 이름 변환

```typescript
type Getters<T> = {
  [P in keyof T as `get${Capitalize<string & P>}`]: () => T[P];
};

interface Person { name: string; age: number }
type PersonGetters = Getters<Person>;
// { getName: () => string; getAge: () => number }
```

- `string & P`: `keyof T`에는 `number`·`symbol` 키가 섞일 수 있으므로 문자열 키만 템플릿에 넣기 위한 교차
- `Capitalize`·`Uncapitalize`·`Uppercase`·`Lowercase`는 문자열 변환용 내장 타입

<br>

### 4-2. 키 필터링 — `as`가 `never`를 반환하면 제거

```typescript
// 값 타입이 함수인 프로퍼티만 남기기
type MethodsOnly<T> = {
  [P in keyof T as T[P] extends Function ? P : never]: T[P];
};

// 특정 키 제외 (Omit의 매핑 타입 버전)
type RemoveKind<T> = {
  [P in keyof T as Exclude<P, "kind">]: T[P];
};

interface Circle { kind: "circle"; radius: number; area(): number }
type CircleMethods = MethodsOnly<Circle>; // { area(): number }
type CircleData = RemoveKind<Circle>;     // { radius: number; area(): number }
```

```
[P in keyof T as 조건 ? P : never]
   ├─ 조건 참  ──▶ 키 P 유지
   └─ 조건 거짓 ──▶ never ──▶ 해당 프로퍼티가 결과에서 사라짐
```

> ⚠️ `[P in keyof T as ...]` 형태는 `as` 절이 있어도 동형 매핑으로 취급되어 원본의 `readonly`·`?` 제어자가 **보존된다**(`Getters<{ readonly name?: string }>`의 `getName`도 읽기 전용·선택적). 반면 `[P in SomeUnion]`처럼 `keyof T`가 아닌 임의의 유니온을 순회하면 제어자 보존이 일어나지 않으므로, 필요하면 `readonly`·`?`를 명시적으로 지정해야 한다.

<br>

### 5. 실무 조합 패턴과 선택 기준

```typescript
// 안티패턴: 같은 구조를 손으로 복제 — User가 바뀌면 동기화 누락
interface CreateUserDto { name: string; email: string; password: string }
interface UserResponse { id: number; name: string; email: string }

// 개선: 원본 User 하나에서 파생 — 필드 추가·변경이 자동 반영
type CreateUserDto2 = Omit<User, "id">;
type UserResponse2 = Omit<User, "password">;
type UserPatch = Partial<Pick<User, "name" | "email">>;
type UserById = Record<User["id"], PublicUser>;   // 인덱스 접근으로 id 타입 재사용
```

| **원하는 것**                          | **도구**                              |
| -------------------------------------- | ------------------------------------- |
| **일부 필드만 선택·제외**              | `Pick` / `Omit` (오타 방지는 `StrictOmit`) |
| **선택적·필수·읽기 전용 전환**         | `Partial` / `Required` / `Readonly`   |
| **키 집합 → 객체 강제 (빠짐없이)**     | `Record<Union, V>`                    |
| **유니온에서 멤버 거르기**             | `Exclude` / `Extract` / `NonNullable` |
| **함수·Promise에서 타입 추출**         | `ReturnType` / `Parameters` / `Awaited` |
| **키 이름 규칙 변환·값 기준 필터링**   | 매핑 타입 + `as` + 템플릿 리터럴      |
| **깊은(중첩) 변환**                    | 내장에 없음 — 재귀 매핑 타입 직접 작성 (`DeepPartial` 등) |

<br>

### 6. 정리 — 면접·실무 체크포인트

- **매핑 타입**은 `[P in K]: V`로 키를 순회해 객체 타입을 생성하며, `+/-readonly`·`+/-?`로 제어자를 조작함. `keyof T`를 그대로 순회하는 **동형 매핑**은 제어자·배열 구조를 보존함
- **`Partial`·`Required`·`Readonly`·`Pick`**은 동형 매핑, **`Omit`**은 `Pick + Exclude` 조합. `Omit`의 `K`는 느슨해 **오타를 잡지 못함**
- **`Record<K, V>`**는 유니온 키 전부를 강제하는 데 유용. **`Exclude`·`Extract`·`NonNullable`**은 분산 조건부 타입이며 `Omit`은 분산되지 않음
- **키 재지정 `as`**(4.1+)로 이름 변환(템플릿 리터럴)과 필터링(`never` 반환)이 가능함
- 원칙: **원본 타입 하나에서 파생**해 DTO·응답·패치 타입을 만들고, 손으로 복제하지 않음
- 매핑·조건부 타입의 기반이 되는 제네릭·`keyof`·조건부 타입은 **unit02**, 인터페이스·타입 별칭 선택은 **unit05**를 참고할 것
