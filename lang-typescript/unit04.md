## any·unknown·never

`any`·`unknown`·`never`는 TypeScript 타입 체계의 **양 끝**에 있는 특수 타입이다. 셋의 차이는 "어떤 값을 담을 수 있는가"와 "담긴 값으로 무엇을 할 수 있는가"의 조합으로 정리되며, 이를 정확히 구분해야 타입 검사를 무력화하지 않으면서 외부 데이터·예외·완전성 검사를 다룰 수 있다.

<br>

### 1. 타입 계층에서의 위치

타입을 "가능한 값의 집합"으로 보면 `unknown`은 **모든 값을 포함하는 전체 집합(최상위 타입, Top Type)**, `never`는 **아무 값도 없는 공집합(최하위 타입, Bottom Type)**이다. `any`는 이 계층에서 벗어나 **검사를 끄는 탈출구**다.

```
                 unknown  (모든 값 — 최상위)
                    ▲
   string   number   boolean   object   null   undefined ...
                    ▲
                  never   (값 없음 — 최하위, 모든 타입에 대입 가능)

   any: 계층 바깥 — 어떤 타입에도 대입되고, 어떤 타입도 받으며, 검사를 생략함
```

| **항목**                   | **`any`**                  | **`unknown`**                          | **`never`**                              |
| -------------------------- | -------------------------- | -------------------------------------- | ---------------------------------------- |
| **어떤 값을 담을 수 있나** | 모든 값                    | **모든 값**                            | **없음** (담을 수 있는 값이 없음)        |
| **다른 타입에 대입**       | 어디든 가능                | `unknown`·`any`에만 가능               | **어디든 가능** (공집합은 모든 집합의 부분집합) |
| **멤버 접근·연산**         | 검사 없이 허용             | **좁히기 전에는 불가**                 | 해당 없음 (도달 불가)                    |
| **도입 시기**              | 초기                       | TypeScript 3.0                         | TypeScript 2.0                           |
| **주 용도**                | 점진적 마이그레이션·긴급 우회 | 외부 입력·예외 등 **모르는 값**의 안전한 표현 | 도달 불가 코드·**완전성 검사**·유니온 필터링 |

<br>

### 2. `any` — 타입 검사를 끄는 스위치

`any`로 선언된 값은 어떤 멤버에 접근하든, 어디에 대입하든 오류가 나지 않는다. 문제는 `any`가 **전염된다**는 점이다. `any` 값에서 파생된 값도 대부분 `any`가 되어 코드베이스 전체의 안전성이 조용히 사라진다.

```typescript
// 안티패턴: any의 전염
function parseConfig(raw: string) {
  const cfg = JSON.parse(raw);       // any (JSON.parse의 반환 타입)
  return cfg.server.port;            // 오타·구조 변경을 전혀 잡지 못함 → 반환도 any
}
const port = parseConfig("{}");      // port: any → 이후 코드도 검사 안 됨
port.toFixed(2);                     // 컴파일 통과, 런타임 TypeError

// 개선: unknown으로 받고 검증해서 좁힘
function parseConfigSafe(raw: string): number {
  const cfg: unknown = JSON.parse(raw);
  if (
    typeof cfg === "object" && cfg !== null &&
    "server" in cfg && typeof (cfg as { server: unknown }).server === "object"
  ) {
    const server = (cfg as { server: { port?: unknown } }).server;
    if (typeof server?.port === "number") return server.port;
  }
  throw new Error("잘못된 설정 형식");
}
```

**`any`가 생기는 경로**

- 명시적 선언: `let x: any`
- 타입 표기 없는 매개변수(`noImplicitAny`가 꺼져 있을 때): `function f(x) {}`
- `JSON.parse`, 일부 오래된 라이브러리 API의 반환 타입
- `as any` 단언

> ⚠️ `tsconfig`의 **`strict: true`**에 포함된 `noImplicitAny`는 암시적 `any`를 오류로 만든다. 신규 프로젝트에서 `strict`를 끄면 `any`가 곳곳에 숨어들어 TypeScript를 쓰는 의미가 크게 줄어든다. 불가피하게 `any`를 써야 한다면 **범위를 한 줄로 최소화**하고 `// eslint-disable-next-line @typescript-eslint/no-explicit-any` 같은 주석으로 의도를 남긴다.

<br>

### 3. `unknown` — "모르는 값"을 안전하게 담는 타입

`unknown`은 `any`처럼 모든 값을 받지만, **좁히기(unit03) 없이는 아무것도 할 수 없다**. "값의 정체는 나중에 확인한다"는 계약을 타입으로 강제하는 셈이다.

```typescript
function stringify(value: unknown): string {
  value.toString();                          // 오류 — unknown에서는 멤버 접근 불가
  if (typeof value === "string") return value;
  if (value instanceof Date) return value.toISOString();
  if (typeof value === "object" && value !== null) return JSON.stringify(value);
  return String(value);
}
```

**`unknown`을 써야 하는 대표 상황**

| **상황**                          | **이유**                                                              |
| --------------------------------- | --------------------------------------------------------------------- |
| **API 응답·`JSON.parse` 결과**    | 런타임에 구조가 보장되지 않음 → 검증 후 사용 (unit07 참고)              |
| **`catch (e)`의 예외 객체**       | JS는 무엇이든 `throw`할 수 있음. `useUnknownInCatchVariables`(`strict`에 포함, 4.4+)로 기본 `unknown` |
| **범용 유틸리티 함수의 입력**     | `isEmpty(value: unknown)`처럼 어떤 값이든 받되 내부에서 안전하게 분기 |
| **제네릭 제약의 기본값**          | `T extends unknown`은 "아무 타입" — `any`보다 의도가 분명              |

```typescript
async function load(url: string) {
  try {
    const res = await fetch(url);
    return await res.json();
  } catch (e) {                     // e: unknown (strict 기본)
    if (e instanceof Error) console.error(e.message);
    else console.error("알 수 없는 오류", e);
    throw e;
  }
}
```

> 💡 "`any`와 `unknown`의 차이"는 TypeScript 면접의 대표 질문이다. 한 줄로 답하면 **"둘 다 무엇이든 담지만, `unknown`은 쓰기 전에 확인을 강제하고 `any`는 확인을 생략한다"**이다. 여기에 "`unknown`은 다른 타입에 대입할 수 없어 전염되지 않는다"를 덧붙이면 충분하다.

<br>

### 4. `never` — 존재할 수 없는 값

`never`는 **어떤 값도 가질 수 없는 타입**이다. 다음 세 가지 상황에서 등장한다.

<br>

### 4-1. 정상적으로 반환하지 않는 함수

항상 예외를 던지거나 무한 루프에 빠지는 함수의 반환 타입은 `never`다. `void`(반환값이 없지만 정상 종료됨)와 구분해야 한다.

```typescript
function fail(message: string): never {
  throw new Error(message);        // 호출부 이후 코드는 도달 불가
}

function getPort(env: Record<string, string | undefined>): number {
  const raw = env.PORT ?? fail("PORT가 설정되지 않았습니다");
  // raw: string — never는 유니온에서 사라지므로 string | never = string
  return Number(raw);
}
```

<br>

### 4-2. 좁히기 끝에 남는 타입 — 완전성 검사

유니온의 모든 멤버를 분기로 처리하고 나면 남은 타입은 `never`가 된다. 이를 `never` 매개변수를 받는 함수에 넘기면, 나중에 멤버가 추가될 때 **컴파일 오류로 누락을 알려 준다**(자세한 예시는 unit03 참고).

```typescript
type Status = "idle" | "loading" | "done";

function label(status: Status): string {
  switch (status) {
    case "idle": return "대기";
    case "loading": return "로딩 중";
    case "done": return "완료";
    default: {
      const exhaustive: never = status; // Status에 "error"가 추가되면 여기서 오류
      return exhaustive;
    }
  }
}
```

<br>

### 4-3. 타입 연산의 결과 — 유니온에서 사라지는 성질

`never`는 유니온에 넣으면 사라지고(`string | never` → `string`), 교차 타입에서는 전체를 삼킨다(`string & number` → `never`). 이 성질 덕분에 조건부 타입에서 **"제외"를 표현하는 값**으로 쓰인다(분산 조건부 타입은 unit02 참고).

```typescript
type Exclude<T, U> = T extends U ? never : T;
type Primitive = Exclude<string | number | (() => void), Function>; // string | number

// 특정 프로퍼티를 "쓸 수 없게" 만드는 데도 사용
type WithoutPassword = { name: string; password?: never };
const u: WithoutPassword = { name: "a", password: "1234" }; // 오류 — never에 값 대입 불가
```

> ⚠️ 빈 배열 리터럴 `[]`은 `strictNullChecks`만 켜고 `noImplicitAny`를 끈 설정에서 `never[]`로 추론되어 `list.push(1)`이 오류가 난다(`strict` 전체를 켜면 나중 사용을 보고 요소 타입을 추론하는 "진화하는 배열"로 처리됨). 이런 경우 `const list: number[] = []`처럼 타입을 명시한다. 그 밖에 `never`가 예상치 못한 곳에 나타난다면 대개 **서로 모순되는 교차 타입**이나 **분기를 모두 소진한 좁히기**가 원인이다.

<br>

### 5. 선택 기준 — 어떤 상황에 무엇을 쓰는가

| **상황**                                          | **선택**       | **이유**                                                 |
| ------------------------------------------------- | -------------- | -------------------------------------------------------- |
| **외부에서 들어온 값(JSON·API·`catch`)**          | **`unknown`**  | 사용 전 검증을 강제, 전염 없음                            |
| **JS 마이그레이션 중 임시 우회**                  | `any` (최소 범위) | 빠르게 컴파일 통과 후 점진적으로 제거                   |
| **어떤 타입이든 받아 그대로 돌려주는 함수**       | 제네릭 `<T>`   | `any`·`unknown` 모두 타입 관계를 잃음 (unit02 참고)       |
| **예외를 던지고 끝나는 함수의 반환**              | **`never`**    | 호출부에서 이후 코드가 도달 불가임을 컴파일러가 인식      |
| **`switch`·`if`의 완전성 보장**                   | **`never`**    | 유니온 멤버 추가 시 컴파일 오류로 누락 감지               |
| **특정 프로퍼티 금지·타입 제외**                  | **`never`**    | 유니온에서 사라지는 성질 활용                             |

```typescript
// any 남용 → 제네릭 + unknown으로 정리한 예
function pickAny(obj: any, key: string): any { return obj[key]; }              // 안티패턴

function pick<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; } // 개선
function isRecord(v: unknown): v is Record<string, unknown> {                  // 외부 값 검증용
  return typeof v === "object" && v !== null;
}
```

<br>

### 6. 정리 — 면접·실무 체크포인트

- **`any`**: 모든 값을 담고 검사를 생략함. **전염**되므로 `strict`(`noImplicitAny`)로 차단하고 범위를 최소화함
- **`unknown`**: 모든 값을 담지만 **좁히기 전에는 사용 불가**. 외부 입력·예외·범용 유틸리티에 사용. 다른 타입에 대입되지 않아 전염되지 않음
- **`never`**: 값이 없는 최하위 타입. **도달 불가 함수·완전성 검사·타입 제외**에 사용하며 유니온에서 사라지고 교차에서 전체를 삼킴
- **`void` vs `never`**: `void`는 정상 종료하되 반환값이 없음, `never`는 **정상 종료 자체가 없음**
- 타입 관계를 보존해야 하면 `any`·`unknown` 대신 **제네릭**(unit02), 외부 데이터 검증은 **런타임 검증**(unit07)과 함께 사용할 것
