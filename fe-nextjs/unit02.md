## 서버·클라이언트 컴포넌트 경계

App Router의 컴포넌트는 기본적으로 **서버 컴포넌트(React Server Component, RSC)**이며, 파일 최상단에 `"use client"` 지시자를 선언한 모듈부터 **클라이언트 컴포넌트**가 된다. 이 경계가 어디까지 전파되는지, 경계를 넘는 데이터에 어떤 직렬화 제약이 있는지 이해해야 "왜 이 훅이 서버 컴포넌트에서 안 되는가", "왜 이 함수를 props로 넘기면 에러가 나는가" 같은 문제를 설계 단계에서 피할 수 있다.

<br>

### 1. 두 종류의 컴포넌트가 하는 일

| **항목**             | **서버 컴포넌트 (기본값)**                          | **클라이언트 컴포넌트 (`"use client"`)**          |
| -------------------- | --------------------------------------------------- | ------------------------------------------------- |
| **실행 위치**        | **서버에서만** 실행 (빌드 시 또는 요청 시)          | 서버에서 **1회 사전 렌더링** + 브라우저에서 실행  |
| **번들 포함 여부**   | 클라이언트 JS 번들에 **포함되지 않음**              | 번들에 포함되어 브라우저로 전송됨                 |
| **가능한 작업**      | DB·파일 시스템 접근, 비밀 키 사용, `async/await`    | `useState`·`useEffect`, 이벤트 핸들러, 브라우저 API |
| **불가능한 작업**    | 상태·이펙트·이벤트 핸들러·브라우저 API              | 서버 전용 리소스 직접 접근                        |
| **출력물**           | HTML + **RSC 페이로드**(직렬화된 트리)              | HTML(사전 렌더링) + 하이드레이션용 JS             |

> 💡 "클라이언트 컴포넌트는 브라우저에서만 실행된다"는 흔한 오해다. 클라이언트 컴포넌트도 **서버에서 한 번 HTML로 사전 렌더링**된 뒤 브라우저에서 하이드레이션(unit07 참고)된다. `window`에 접근하는 코드가 서버에서 터지는 이유가 바로 이것이다.

<br>

### 2. `"use client"` 지시자의 전파 범위

### 2-1. 파일이 아니라 "모듈 그래프의 경계"를 선언한다

`"use client"`는 "이 컴포넌트를 클라이언트에서 실행하라"가 아니라, **"이 모듈부터 아래로 import되는 모든 모듈은 클라이언트 번들에 포함하라"**는 선언이다. 따라서 해당 파일이 import하는 하위 컴포넌트는 지시자가 없어도 전부 클라이언트 컴포넌트가 된다.

```
app/page.tsx (서버)
 └─ <Sidebar/>        sidebar.tsx        "use client"  ← 경계 시작
     ├─ <Menu/>       menu.tsx           (지시자 없음 → 클라이언트로 끌려감)
     │   └─ <Icon/>   icon.tsx           (지시자 없음 → 클라이언트로 끌려감)
     └─ <UserBadge/>  user-badge.tsx     (지시자 없음 → 클라이언트로 끌려감)
```

- 지시자는 **경계에 있는 파일에만** 쓰면 되고, 그 아래 모든 파일에 반복해서 붙일 필요가 없다.
- 반대로 하나의 모듈이 서버 컴포넌트에서도, 클라이언트 컴포넌트에서도 import되면 **양쪽 번들 모두에** 포함된다(공유 컴포넌트).

<br>

### 2-2. 경계는 "import 방향"으로만 전파된다 — children 패턴

클라이언트 컴포넌트 안에 서버 컴포넌트를 두고 싶다면 **import하지 말고 props(`children`)로 전달**한다. import는 경계를 전파하지만, 이미 서버에서 렌더링된 결과를 props로 끼워 넣는 것은 허용된다.

```tsx
// ❌ 안티패턴: 클라이언트 컴포넌트가 서버 전용 컴포넌트를 직접 import
"use client";
import { ServerOnlyList } from "./server-only-list"; // DB 접근 코드가 번들에 포함되려다 빌드 에러

export function Panel() {
  return <div><ServerOnlyList /></div>;
}
```

```tsx
// ✅ 개선: 서버 컴포넌트는 children으로 "구멍"에 끼워 넣는다
"use client";
import { useState } from "react";

export function Panel({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(true);
  return (
    <div>
      <button onClick={() => setOpen((v) => !v)}>토글</button>
      {open && children}
    </div>
  );
}

// app/page.tsx (서버 컴포넌트)
// <Panel><ServerOnlyList /></Panel>  ← ServerOnlyList는 서버에서 렌더링된 뒤 전달됨
```

> ⚠️ 클라이언트 컴포넌트에서 서버 컴포넌트를 import하면 "You're importing a component that needs `xxx`. That only works in a Server Component" 류의 빌드 에러가 나거나, 최악의 경우 **서버 전용 코드가 브라우저 번들로 새어 나간다**. 서버 전용 모듈에는 `import "server-only"`를 선언해 잘못된 import를 빌드 시점에 차단하는 것이 안전하다.

<br>

### 3. 직렬화 제약 — 경계를 넘는 props의 조건

서버 컴포넌트가 클라이언트 컴포넌트에 넘기는 props는 네트워크를 타고 브라우저로 전달되므로 **직렬화(Serialization) 가능한 값**이어야 한다. React는 RSC 페이로드에 값을 담을 때 JSON보다 넓은 범위를 지원하지만, 함수와 클래스 인스턴스는 넘길 수 없다.

| **분류**                     | **예시**                                                 | **경계 통과** |
| ---------------------------- | -------------------------------------------------------- | ------------- |
| **원시값·배열·일반 객체**    | `string`, `number`, `boolean`, `null`, `[]`, `{}`        | **가능**      |
| **Date, Map, Set, BigInt**   | `new Date()`, `new Map()`                                | **가능**      |
| **React 엘리먼트**           | `<ServerComp />`, `children`                             | **가능**      |
| **Promise**                  | `fetch(...)`의 반환값 (클라이언트에서 `use()`로 언랩)    | **가능**      |
| **서버 액션**                | `"use server"` 함수                                      | **가능** (참조로 전달) |
| **일반 함수·이벤트 핸들러**  | `() => console.log()`                                    | **불가**      |
| **클래스 인스턴스·심볼**     | `new UserModel()`, `Symbol()`                            | **불가**      |

```tsx
// ❌ 안티패턴: 서버 컴포넌트가 일반 함수를 props로 전달
// Error: Functions cannot be passed directly to Client Components
export default function Page() {
  return <SortableList onSort={(a, b) => a.localeCompare(b)} />;
}
```

```tsx
// ✅ 개선 1: 정렬 "방식"을 직렬화 가능한 값으로 넘기고, 함수는 클라이언트에서 정의
export default function Page() {
  return <SortableList sortBy="name" />; // 클라이언트 쪽에서 sortBy에 맞는 함수를 고름
}

// ✅ 개선 2: 서버에서 실행돼야 하는 로직이면 서버 액션(unit06 참고)으로 전달
async function saveOrder(ids: string[]) {
  "use server";
  await db.order.updateMany(ids);
}
export function Page2() {
  return <SortableList onSave={saveOrder} />; // 함수 본체가 아니라 "참조 ID"가 직렬화됨
}
```

<br>

### 4. 경계를 어디에 둘 것인가 — 최대한 아래로 내리기

경계를 트리의 위쪽에 두면 그 아래 전체가 클라이언트 번들로 들어간다. **상호작용이 필요한 최소 단위**만 클라이언트 컴포넌트로 분리하는 것이 원칙이다.

```
❌ 경계가 위에 있을 때                ✅ 경계를 잎(leaf)으로 내렸을 때
<Page> "use client"                   <Page>            (서버)
 ├─ <Header>      (클라이언트)         ├─ <Header>      (서버)
 ├─ <ArticleBody> (클라이언트, 큰 텍스트) ├─ <ArticleBody> (서버, 번들 0KB)
 └─ <LikeButton>  (클라이언트)         └─ <LikeButton> "use client" (클라이언트)
번들: 페이지 전체                      번들: 버튼 하나
```

- 데이터 페칭·마크다운 변환·날짜 포맷팅처럼 **상호작용이 없는 로직은 서버에 남긴다**.
- 서드파티 라이브러리가 `useState` 등을 쓰는데 `"use client"`가 없으면, 그 라이브러리를 감싸는 얇은 래퍼 파일을 만들어 지시자를 붙인다.
- Context Provider는 클라이언트 컴포넌트여야 하지만, 루트 레이아웃 자체를 클라이언트로 만들지 말고 **Provider만 분리**해 `children`을 받는 형태로 둔다.

> 💡 "서버 컴포넌트는 클라이언트 컴포넌트를 import할 수 있지만, 클라이언트 컴포넌트는 서버 컴포넌트를 import할 수 없다"는 한 문장으로 규칙을 기억하면 대부분의 상황에 대응할 수 있다.

<br>

### 5. 자주 만나는 에러와 원인

| **에러 메시지 (요약)**                                      | **원인**                                            | **해결**                                          |
| ----------------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------- |
| **`useState only works in Client Components`**              | 서버 컴포넌트에서 훅 사용                           | 상호작용 부분을 분리해 `"use client"` 선언        |
| **`Functions cannot be passed directly to Client Components`** | 일반 함수를 props로 전달                          | 값으로 대체하거나 서버 액션으로 변경              |
| **`window is not defined`**                                 | 클라이언트 컴포넌트가 서버에서 사전 렌더링될 때 접근 | `useEffect` 안에서 접근하거나 `next/dynamic`의 `ssr: false` |
| **`async/await is not yet supported in Client Components`** | 클라이언트 컴포넌트를 `async` 함수로 선언          | 데이터는 서버에서 받아 props로 전달하거나 `use()` 사용 |
| **`server-only` 모듈 import 에러**                          | 서버 전용 모듈을 클라이언트에서 import              | 경계 재설계 (children 패턴)                       |

❗️**환경 변수 노출**: `NEXT_PUBLIC_` 접두사가 없는 환경 변수는 클라이언트 번들에서 `undefined`가 된다. 반대로 접두사를 붙인 값은 **브라우저에 그대로 노출**되므로 비밀 키에 절대 붙이면 안 된다.

<br>

### 6. 정리 — 면접·실무 체크포인트

- 컴포넌트는 **기본이 서버 컴포넌트**이며, `"use client"`는 모듈 그래프에서 **경계를 선언**하는 지시자다. 그 아래 import되는 모듈은 지시자 없이도 전부 클라이언트 번들에 포함된다.
- 경계는 **import 방향**으로만 전파된다. 클라이언트 컴포넌트 안에 서버 컴포넌트를 두려면 import 대신 **`children`/props로 끼워 넣는다**.
- 경계를 넘는 props는 **직렬화 가능**해야 한다. 함수·클래스 인스턴스는 불가, 서버 액션과 Promise·React 엘리먼트는 가능하다.
- 경계는 **가능한 한 잎(leaf) 쪽으로** 내려서 클라이언트 번들을 최소화한다. Provider·서드파티 훅 라이브러리는 얇은 래퍼로 감싼다.
- 클라이언트 컴포넌트도 **서버에서 한 번 사전 렌더링**되므로 `window` 접근은 `useEffect` 안에서 한다(unit07 참고).
- 서버 전용 모듈에는 `import "server-only"`를, 비밀 키에는 `NEXT_PUBLIC_` 접두사를 붙이지 않는 것이 실무 안전장치다.
- 데이터 페칭 위치 결정은 unit03, 서버 액션 전달 방식은 unit06을 참고할 것.
