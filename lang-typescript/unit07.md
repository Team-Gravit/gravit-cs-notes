## 컴파일과 런타임 경계

TypeScript의 타입은 **컴파일 타임에만 존재하고 JavaScript로 변환되는 순간 모두 사라진다(타입 소거, Type Erasure)**. 이 경계를 이해하지 못하면 "타입을 붙였으니 안전하다"는 착각으로 API 응답·사용자 입력 같은 **런타임 데이터**를 검증 없이 신뢰하게 되고, 그 지점에서 TypeError가 발생한다.

<br>

### 1. 타입 소거 — 타입은 실행되지 않는다

`tsc`(또는 esbuild·swc·Babel)는 타입 표기·인터페이스·타입 별칭·제네릭·`as` 단언을 **모두 제거**하고 순수 JavaScript만 남긴다. 타입 검사는 변환 전에 한 번 수행될 뿐, 결과 코드에는 검사 로직이 전혀 들어가지 않는다.

```typescript
interface User { id: number; name: string }

function greet(user: User): string {
  return `안녕하세요, ${user.name}님`;
}

const raw = JSON.parse('{"id": 1}') as User;   // 단언 — 검사도, 변환도 일어나지 않음
greet(raw);                                     // 컴파일 통과 → 런타임에 "안녕하세요, undefined님"
```

```javascript
// 위 코드의 컴파일 결과 (tsc --target es2022) — 타입 관련 코드가 전부 사라짐
function greet(user) {
  return `안녕하세요, ${user.name}님`;
}
const raw = JSON.parse('{"id": 1}');
greet(raw);
```

```
 .ts 소스 ──▶ [ 타입 검사 ] ──▶ [ 타입 제거(변환) ] ──▶ .js ──▶ 실행
              컴파일 타임 (정적)                        런타임 (동적)
              · interface / type / 제네릭 / as          · 값만 존재
              · 오류는 여기서만 보고됨                    · 타입 정보 없음 → 검사 불가
```

> 💡 "TypeScript는 런타임에 타입을 검사하는가?"라는 질문의 답은 **"아니다"**이다. TypeScript는 **정적 타입 검사기 + 변환기**이지 런타임이 아니며, 생성된 JS는 타입 없는 JS와 완전히 동일하게 동작한다. 이 한 문장이 이 유닛 전체의 출발점이다.

<br>

### 2. 타입 소거가 만드는 제약

| **하고 싶은 것**                            | **왜 안 되는가**                                     | **대안**                                                     |
| ------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| **`value instanceof SomeInterface`**        | 인터페이스는 런타임에 없음                           | 클래스로 정의하거나 **타입 서술 함수**(unit03)로 구조 검사     |
| **`typeof value === "User"`**               | `typeof`는 JS 원시 타입 문자열만 반환                | 판별자 프로퍼티(`kind`) 비교                                 |
| **제네릭 `T`로 분기 `if (T === string)`**   | `T`는 실행 시 존재하지 않음                          | 값을 인수로 넘겨 런타임에 판단 (`new Array<T>()` → 요소는 검사 안 됨) |
| **`as`로 값을 변환**                        | 단언은 컴파일러에게 하는 말일 뿐 값은 그대로         | `Number(x)`, `String(x)` 등 **실제 변환 함수** 사용            |
| **타입만으로 기본값 생성**                  | 타입에서 값을 만들 수 없음                           | 스키마(값)에서 타입을 **파생**하는 방향으로 설계 (4절)         |

```typescript
// 안티패턴: 타입 단언을 "변환"으로 오해
const input = "42" as unknown as number;
console.log(input + 1); // 컴파일은 number + number로 통과 → 런타임 결과 "421" (문자열 결합)

// 개선: 실제 변환 + 검증
const parsed = Number("42");
if (Number.isNaN(parsed)) throw new Error("숫자가 아닙니다");
console.log(parsed + 1); // 43
```

> ⚠️ `as`는 **컴파일러의 판단을 덮어쓰는 도구**이지 캐스팅이 아니다. `x as unknown as T`처럼 이중 단언을 쓰는 순간 컴파일러는 완전히 손을 떼며, 그 값이 실제로 `T`인지는 오직 개발자의 책임이 된다. 단언이 필요한 지점은 대개 **런타임 검증이 필요한 지점**과 일치한다.

<br>

### 3. 런타임에 흔적을 남기는 문법

모든 TypeScript 문법이 "지워지기만" 하는 것은 아니다. 일부는 **JS 코드를 생성**하며, 이 차이는 번들러·런타임 호환성에서 중요하다.

| **문법**                                   | **런타임 산출물**                         | **비고**                                                         |
| ------------------------------------------ | ----------------------------------------- | ---------------------------------------------------------------- |
| **`interface`·`type`·타입 표기·제네릭·`as`** | 없음 (완전 소거)                          | 타입 전용                                                        |
| **`enum`**                                 | 양방향 매핑 객체(IIFE) 생성                | 값이므로 `import`가 실제 로드를 유발함                            |
| **`const enum`**                            | 사용처에 리터럴 인라인, 선언은 제거        | `isolatedModules` 환경에서 제약 있음                              |
| **`namespace`**                             | IIFE 객체 생성                             | ES 모듈 시대에는 거의 사용하지 않음                               |
| **매개변수 프로퍼티** `constructor(public x)` | `this.x = x` 대입 코드 생성              | 클래스 필드 초기화 순서와 얽혀 주의 필요                          |
| **데코레이터**                              | 호출 코드 생성                             | 5.0의 표준 데코레이터와 레거시(`experimentalDecorators`)가 다름   |

```typescript
enum Role { Admin = "ADMIN", User = "USER" }   // 런타임 객체 생성 → Object.values(Role) 가능
type RoleT = "ADMIN" | "USER";                  // 소거됨 → 런타임에 목록을 얻을 수 없음

// 런타임 목록과 타입을 동시에 얻는 관용구: 값에서 타입을 파생
const ROLES = ["ADMIN", "USER"] as const;
type Role2 = (typeof ROLES)[number];            // "ADMIN" | "USER"
ROLES.includes("GUEST" as Role2);               // 런타임 검사도 가능
```

**`import type`과 `verbatimModuleSyntax`** — 타입만 가져오는 `import`는 소거되어야 하지만, 파일 단위로 변환하는 도구(esbuild·swc·Babel)는 다른 파일을 보지 못해 "이 이름이 타입인지 값인지" 알 수 없다. 그래서 TypeScript 5.0의 **`verbatimModuleSyntax`** 옵션을 켜면 타입 전용 가져오기에 **`import type`**을 강제해, 어떤 도구로 변환해도 결과가 같도록 만든다.

```typescript
import type { User } from "./types";      // 소거 대상임을 명시 — 런타임 로드 없음
import { createUser } from "./service";   // 값 — 런타임에 로드됨
```

> 💡 TypeScript 5.8의 **`erasableSyntaxOnly`** 옵션은 `enum`·`namespace`·매개변수 프로퍼티처럼 **소거만으로는 처리할 수 없는 문법을 아예 금지**한다. Node.js가 타입을 벗겨내기만 하고 TS 파일을 직접 실행하는 기능(타입 스트리핑)을 제공하면서, 이런 "런타임 흔적을 남기는 문법"을 쓰지 않는 코드 스타일이 확산되는 추세다(지원 범위는 Node.js 버전에 따라 다를 수 있음).

<br>

### 4. 런타임 검증이 필요한 지점

컴파일러가 보장하는 것은 **내 코드 안에서 타입이 일관되게 쓰였는가**뿐이다. 코드 바깥에서 들어오는 값은 어떤 타입 표기를 붙여도 실제 모양이 보장되지 않는다.

```
       [ 신뢰 경계(Trust Boundary) ]
외부 ─────────────────┬────────────────── 내부 (타입이 보장되는 영역)
 · HTTP 요청 본문      │  검증(파싱) 통과 후에만
 · API 응답 JSON      │  User / Order 등의 타입으로 취급
 · 환경 변수·설정 파일 │
 · localStorage·URL   │  ← 검증 없이 `as User`로 넘기면 경계가 무의미해짐
 · 사용자 입력        │
```

<br>

### 4-1. 직접 작성하는 검증 — 타입 서술 함수

작은 규모에서는 `unknown`(unit04)으로 받아 **타입 서술 함수**로 좁히는 방식이면 충분하다.

```typescript
interface User { id: number; name: string }

function isUser(v: unknown): v is User {
  return (
    typeof v === "object" && v !== null &&
    typeof (v as Record<string, unknown>).id === "number" &&
    typeof (v as Record<string, unknown>).name === "string"
  );
}

async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  const body: unknown = await res.json();       // any가 아닌 unknown으로 받음
  if (!isUser(body)) throw new Error("응답 형식이 올바르지 않습니다");
  return body;                                   // 여기서부터 User로 보장
}
```

단점은 **타입 정의와 검증 로직이 이중으로 존재**해 필드가 추가될 때 한쪽을 빠뜨리기 쉽다는 점이다(타입 서술 본문은 컴파일러가 검증하지 않음, unit03 참고).

<br>

### 4-2. 스키마 기반 검증 — 값에서 타입을 파생

Zod·Valibot·io-ts 같은 라이브러리는 **런타임 스키마(값)를 먼저 정의하고 거기서 타입을 추론**한다. 검증 로직과 타입이 하나의 원천(Single Source of Truth)에서 나오므로 어긋날 수 없다.

```typescript
import { z } from "zod";

const UserSchema = z.object({
  id: z.number().int().positive(),
  name: z.string().min(1),
  email: z.string().email().optional(),
});
type User = z.infer<typeof UserSchema>;   // 스키마에서 타입 파생 — 별도 interface 불필요

async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  return UserSchema.parse(await res.json());   // 실패 시 상세한 오류와 함께 throw
}
```

| **방식**                    | **장점**                                       | **단점**                                     | **적합한 상황**                        |
| --------------------------- | ---------------------------------------------- | -------------------------------------------- | -------------------------------------- |
| **`as` 단언**               | 코드 없음                                      | **검증 없음** — 경계가 사라짐                | 신뢰할 수 있는 내부 값에 한정           |
| **타입 서술 함수**          | 의존성 없음, 가벼움                            | 타입과 검증 로직 **이중 관리**, 본문 검증 안 됨 | 필드 몇 개의 단순 구조                  |
| **스키마 라이브러리**       | **타입·검증 단일 원천**, 상세 오류, 변환 지원 | 번들 크기·의존성, 학습 비용                   | API 경계·폼 입력·환경 변수 등 대부분의 실무 |
| **런타임 `assert`+JSDoc**   | JS 프로젝트와 공존                             | 타입 시스템과 분리됨                         | 점진적 마이그레이션                     |

> ⚠️ `tsc` 대신 esbuild·swc·Vite 같은 **변환 전용 도구**로만 빌드하면 타입 검사가 **아예 수행되지 않은 채** 배포될 수 있다. 이런 도구는 속도를 위해 타입을 "지우기만" 하므로, CI에서 `tsc --noEmit`을 별도로 실행해 타입 검사를 보장해야 한다. "빌드가 성공했다 = 타입 오류가 없다"가 아니다.

<br>

### 5. 정리 — 면접·실무 체크포인트

| **질문**                                      | **핵심 답변**                                                                                  |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **타입 소거란**                               | 컴파일 시 타입 표기·인터페이스·제네릭·단언이 **모두 제거**되어 JS에 남지 않는 것                 |
| **TypeScript는 런타임 타입 검사를 하는가**    | **하지 않는다.** 정적 검사기 + 변환기일 뿐, 생성된 JS는 일반 JS와 동일                          |
| **`interface`를 `instanceof`로 검사 못 하는 이유** | 런타임에 존재하지 않음 → 타입 서술 함수·판별자·스키마로 대체                                |
| **`as`와 캐스팅의 차이**                      | `as`는 컴파일러 판단만 덮어씀. 값은 변하지 않으므로 **실제 변환 함수**가 필요                    |
| **런타임 흔적을 남기는 문법**                 | `enum`·`namespace`·매개변수 프로퍼티·데코레이터. `erasableSyntaxOnly`(5.8)로 금지 가능           |
| **`import type`이 필요한 이유**               | 파일 단위 변환 도구가 타입/값을 구분 못 함 → `verbatimModuleSyntax`(5.0)로 명시 강제             |
| **런타임 검증은 어디서**                      | **신뢰 경계**(API 응답·요청 본문·환경 변수·저장소·사용자 입력)에서 `unknown` → 검증 → 타입       |
| **검증 방식 선택**                            | 단순 구조는 타입 서술 함수, 실무 경계는 **스키마 라이브러리**로 타입·검증 단일 원천화             |

- `unknown`으로 받고 좁히는 원칙은 **unit04**, 타입 서술·단언 함수의 문법은 **unit03**, 브랜드 타입처럼 타입 전용 표식이 런타임에 사라지는 예는 **unit01**을 참고할 것
