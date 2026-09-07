## 모듈 시스템

Node.js에는 원래부터 있던 **CommonJS(CJS)**와 ECMAScript 표준인 **ESM(ECMAScript Modules)** 두 가지 모듈 시스템이 공존한다. 둘은 로딩 시점·바인딩 방식·파일 판별 규칙이 다르며, 최근 라이브러리가 ESM 전용으로 전환되면서 **상호운용(interop)** 문제가 실무에서 자주 터진다. 이 유닛은 두 시스템의 동작 원리와 차이, 상호운용 규칙, 동적 `import()`를 정리한다.

<br>

### 1. CommonJS의 동작 원리

CommonJS는 `require()`로 모듈을 불러오고 `module.exports`로 내보낸다. Node.js는 각 파일을 **함수로 감싸서(module wrapper)** 실행하기 때문에 파일 최상단 변수가 전역이 되지 않는다.

```javascript
// Node.js가 내부적으로 파일을 감싸는 형태
(function (exports, require, module, __filename, __dirname) {
  // 파일 내용
});
```

```javascript
// math.js
let count = 0;
function increment() { count++; }
module.exports = { count, increment };

// app.js
const math = require('./math');
math.increment();
console.log(math.count); // 0 — 내보낸 시점의 "값"이 복사됨 (라이브 바인딩 아님)
```

**핵심 특성**

- **동기 로딩**: `require()`는 파일을 읽고 실행한 뒤 `module.exports`를 즉시 반환함. 코드 어디서나(조건문 안에서도) 호출 가능
- **캐시**: 한 번 로드한 모듈은 `require.cache`에 저장되어 같은 객체를 반환함 → 모듈 스코프가 사실상 싱글톤이 됨
- **값 복사**: `module.exports`의 프로퍼티는 내보낸 시점의 값이며, 원본 변수가 바뀌어도 반영되지 않음
- `exports`는 `module.exports`의 별칭일 뿐이므로 `exports = {...}`처럼 재할당하면 아무것도 내보내지 않음

> ⚠️ `exports.foo = 1`은 동작하지만 `exports = { foo: 1 }`은 지역 변수만 바꾼다. 객체 전체를 내보낼 때는 반드시 `module.exports = {...}`를 사용해야 한다.

<br>

### 2. ESM의 동작 원리

ESM은 `import`/`export` 문법을 사용하며, Node.js 12+에서 정식 지원된다. 가장 큰 차이는 **정적 구조**와 **비동기 로딩**이다.

```javascript
// math.mjs
export let count = 0;
export function increment() { count++; }

// app.mjs
import { count, increment } from './math.mjs';
increment();
console.log(count); // 1 — 라이브 바인딩: 원본 변수를 그대로 참조
console.log(import.meta.url); // file:///.../app.mjs (__filename 대체)
```

**핵심 특성**

- **정적 분석**: `import`는 파일 최상단에서 문자열 경로로만 선언 가능. 실행 전에 의존 그래프가 확정되어 순환 참조 감지·트리 셰이킹이 가능함
- **3단계 로딩**: 파싱(구문 분석) → 링킹(바인딩 연결) → 평가(실행). 이 과정이 비동기이며 모든 의존성이 링크된 뒤 평가됨
- **라이브 바인딩**: 내보낸 변수의 "참조"를 공유하므로 원본이 바뀌면 가져온 쪽에서도 바뀜 (읽기 전용)
- **최상위 await**: 모듈 최상단에서 `await` 사용 가능 (Node 14.8+)
- **strict mode 기본**, `this`가 `undefined`, `__dirname`·`require` 없음

```javascript
// ESM에서 __dirname 대체
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';
const __dirname = dirname(fileURLToPath(import.meta.url));
// Node 20.11+ / 21.2+ 에서는 import.meta.dirname, import.meta.filename 사용 가능
```

<br>

### 3. 파일이 어느 시스템으로 해석되는가

Node.js는 아래 규칙으로 파일의 모듈 종류를 결정한다.

```
파일 확장자 확인
  ├─ .mjs ──────────────────▶ ESM
  ├─ .cjs ──────────────────▶ CommonJS
  └─ .js ─▶ 가장 가까운 package.json의 "type" 필드
              ├─ "module" ──▶ ESM
              ├─ "commonjs" ─▶ CommonJS
              └─ 없음 ──────▶ CommonJS (기본값)
                              ※ Node 22.7+ 는 구문을 감지해 ESM으로 재시도할 수 있음 (버전에 따라 다름)
```

- 패키지 제작자는 `package.json`의 `"exports"` 필드에 `"import"`·`"require"` 조건을 두어 두 시스템에 각각 다른 진입점을 제공함 (**듀얼 패키지**)
- TypeScript는 `tsconfig`의 `module`·`moduleResolution` 설정이 이 규칙과 맞물려야 함. 소스는 `import` 문법을 써도 컴파일 결과가 CJS일 수 있으니 출력물 기준으로 판단해야 함

<br>

### 4. CommonJS vs ESM 비교

| **항목**           | **CommonJS**                          | **ESM**                                     |
| ------------------ | ------------------------------------- | ------------------------------------------- |
| **문법**           | `require()` / `module.exports`        | `import` / `export`                          |
| **로딩 방식**      | **동기**, 실행 중 호출 가능             | **비동기**, 정적 선언 (동적은 `import()`)     |
| **바인딩**         | 값 복사                                | **라이브 바인딩** (참조)                      |
| **순환 참조**      | 미완성 `exports` 객체를 받음            | 링킹 단계에서 연결, 평가 전 접근 시 TDZ 오류   |
| **최상위 await**   | 불가                                   | 가능                                        |
| **파일 정보**      | `__filename`, `__dirname`             | `import.meta.url` (`.dirname`은 최신 버전)   |
| **strict mode**    | 선택                                   | 항상                                        |
| **트리 셰이킹**    | 어려움                                  | 가능 (정적 분석)                              |
| **기본 확장자**    | `.cjs`, `.js`(기본)                    | `.mjs`, `.js`(`"type": "module"`)            |
| **생태계**         | 기존 패키지 대다수                      | 최신 패키지의 표준, 브라우저와 동일            |

> 💡 면접에서 "왜 ESM이 정적이어야 하는가"를 물으면 **트리 셰이킹과 순환 참조 처리**를 답하면 된다. 실행 전에 의존 그래프를 알 수 있어야 사용하지 않는 export를 제거하고, 순환 구조에서도 바인딩을 미리 연결할 수 있다.

<br>

### 5. 상호운용(Interop)

### 5-1. ESM에서 CommonJS 불러오기

ESM 파일에서 CJS 모듈을 `import`하면 `module.exports` 전체가 **default export**로 들어온다.

```javascript
// legacy.cjs
module.exports = { parse() {}, version: '1.0' };

// app.mjs
import legacy from './legacy.cjs';            // module.exports 객체 전체
import { parse } from './legacy.cjs';         // 정적 분석(cjs-module-lexer)이 성공하면 가능
import * as ns from './legacy.cjs';           // ns.default === module.exports
```

- 명명 가져오기(`{ parse }`)는 Node.js가 CJS 소스를 문법적으로 훑어 `exports.xxx =` 패턴을 찾아낼 때만 동작함. 동적으로 만든 export(`Object.assign(exports, ...)`, 트랜스파일 결과 등)는 감지되지 않아 `SyntaxError: Named export not found`가 발생함 → 이때는 default로 받아 구조 분해함

<br>

### 5-2. CommonJS에서 ESM 불러오기

- 전통적 방법은 **동적 `import()`**: `const mod = await import('esm-only-pkg')` — 비동기이므로 콜백·async 함수 안에서 사용
- **`require(esm)`**: Node **22.12 / 20.19 이상**에서는 플래그 없이 `require()`로 ESM을 동기 로드할 수 있음. 단 해당 모듈 그래프에 **최상위 await가 있으면 `ERR_REQUIRE_ASYNC_MODULE`** 오류가 나므로 그런 경우는 여전히 `import()`를 써야 함
- 그 이하 버전에서는 `ERR_REQUIRE_ESM` 오류가 발생함. "ESM 전용 패키지(chalk 5, node-fetch 3 등)를 CJS 프로젝트에서 require했더니 오류"가 이 상황

```javascript
// CJS 프로젝트에서 ESM 전용 패키지를 사용하는 두 가지 방법
async function main() {
  const { default: chalk } = await import('chalk'); // ① 어느 버전에서나 동작
  console.log(chalk.green('ok'));
}

// ② Node 22.12+ / 20.19+ : 최상위 await가 없는 ESM은 동기 require 가능
// const { default: chalk } = require('chalk');
```

> ⚠️ 상호운용 오류는 대부분 "누가 CJS이고 누가 ESM인지" 파악을 못 해서 생긴다. 오류가 나면 먼저 `package.json`의 `"type"`, 파일 확장자, 패키지의 `"exports"` 조건을 확인하고, 그다음 Node 버전을 본다.

<br>

### 6. 동적 import()

`import()`는 **함수처럼 호출하는 표현식**으로, Promise를 반환하며 CJS·ESM 어디서나 사용할 수 있다.

```javascript
// 조건부·지연 로딩: 무거운 모듈을 필요할 때만 읽어 부팅 시간을 줄임
async function exportPdf(doc) {
  const { PDFDocument } = await import('pdf-lib'); // 처음 호출될 때 한 번만 로드(이후 캐시)
  return PDFDocument.create(doc);
}

// 플러그인 패턴: 경로를 런타임에 결정
const plugin = await import(`./plugins/${name}.mjs`);
plugin.default.register(app);
```

**활용 상황**

- 시작 시 필요 없는 모듈의 **지연 로딩**으로 콜드 스타트 단축 (서버리스에서 특히 유효)
- 런타임 조건(환경, 설정, 플러그인 이름)에 따른 **선택적 로딩**
- CJS 코드베이스에서 **ESM 전용 패키지** 사용
- 번들러(webpack·Vite)는 `import()` 지점을 기준으로 **코드 스플리팅**을 수행함

- 반환값은 모듈 네임스페이스 객체이므로 `default`를 명시적으로 꺼내야 하며, 경로가 동적이면 정적 분석·트리 셰이킹 혜택은 사라짐

<br>

### 7. 정리

- **CommonJS**: 동기 `require`, 값 복사, 캐시 기반 싱글톤, 함수 래퍼. Node.js 기존 생태계의 기본
- **ESM**: 정적 `import/export`, 비동기 3단계 로딩, 라이브 바인딩, 최상위 await, 브라우저와 동일한 표준
- 파일 종류는 **확장자(.mjs/.cjs) → package.json `"type"`** 순으로 결정됨
- ESM → CJS: `module.exports`가 default로 들어오며 명명 가져오기는 정적 감지에 의존함
- CJS → ESM: 동적 `import()`가 기본. Node 22.12/20.19+는 최상위 await 없는 ESM에 한해 `require(esm)` 가능
- 동적 `import()`는 지연·조건부 로딩과 코드 스플리팅에 쓰며 Promise를 반환함
- 모듈 캐시 때문에 모듈 스코프 상태가 프로세스 전체에서 공유되는 점은 메모리 누수(unit09)와 NestJS 싱글톤(unit06) 이해의 기초가 됨
