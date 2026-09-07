## 라우팅·레이아웃과 미들웨어

App Router는 `app/` 디렉터리의 **폴더 구조가 곧 URL 구조**이며, 폴더 안의 예약된 파일명(`page`·`layout`·`template`·`loading`·`error` 등)이 각 세그먼트의 UI 역할을 결정한다. 여기에 요청이 라우트에 도달하기 **전에** 실행되는 미들웨어가 더해져 리다이렉트·리라이트·헤더 조작을 담당한다. 레이아웃과 템플릿의 차이, 미들웨어의 실행 시점과 제약을 정확히 알아야 "왜 상태가 유지되는가/초기화되는가", "왜 미들웨어에서 DB를 못 쓰는가"에 답할 수 있다.

<br>

### 1. 파일 규칙과 렌더링 중첩 구조

| **파일**            | **역할**                                                | **비고**                                        |
| ------------------- | ------------------------------------------------------- | ----------------------------------------------- |
| **`page.tsx`**      | 해당 세그먼트의 **고유 UI**. 이 파일이 있어야 URL로 접근 가능 | 없으면 폴더는 경로 조직용으로만 쓰임          |
| **`layout.tsx`**    | 하위 세그먼트를 감싸는 **공유 UI**. 내비게이션 시 유지  | 루트 레이아웃은 `<html>`·`<body>` 필수         |
| **`template.tsx`**  | 레이아웃과 같은 위치지만 **내비게이션마다 새로 마운트** | 애니메이션·페이지별 이펙트에 사용               |
| **`loading.tsx`**   | 세그먼트를 자동으로 `Suspense`로 감싸는 로딩 UI         | 스트리밍 단위(unit07 참고)                      |
| **`error.tsx`**     | 세그먼트를 자동으로 Error Boundary로 감쌈               | 클라이언트 컴포넌트여야 함. `reset()` 제공      |
| **`not-found.tsx`** | `notFound()` 호출 시 표시                               | 루트에 두면 전역 404                            |
| **`route.ts`**      | HTTP 핸들러(API). `page.tsx`와 같은 세그먼트에 공존 불가 | GET·POST 등 메서드별 export                    |

```
app/dashboard/settings/page.tsx 렌더링 결과 (바깥 → 안쪽)

<RootLayout>                        app/layout.tsx
  <Template>                        app/template.tsx (있다면)
    <ErrorBoundary>                 app/error.tsx
      <Suspense>                    app/loading.tsx
        <DashboardLayout>           app/dashboard/layout.tsx
          <SettingsPage />          app/dashboard/settings/page.tsx
```

- 레이아웃 → 템플릿 → 에러 경계 → 서스펜스 → 페이지 순으로 **자동 중첩**된다.
- 내비게이션 시 변경된 세그먼트만 서버에 요청하고, 상위 레이아웃은 재렌더링하지 않는다(부분 렌더링). 라우터 캐시와의 관계는 unit04 참고.

<br>

### 2. 라우팅 패턴 모음

| **패턴**              | **폴더 표기**              | **URL 예시**            | **용도**                                           |
| --------------------- | -------------------------- | ----------------------- | -------------------------------------------------- |
| **동적 세그먼트**     | `[id]`                     | `/posts/42`             | `params`로 값 수신 (15부터 `await params`)         |
| **캐치올**            | `[...slug]`                | `/docs/a/b/c`           | 깊이 불문 경로. `[[...slug]]`는 루트도 포함        |
| **라우트 그룹**       | `(marketing)`              | URL에 미포함            | **URL 영향 없이** 레이아웃 공유·폴더 정리          |
| **병렬 라우트**       | `@modal`, `@sidebar`       | 동일 URL                | 한 레이아웃에 독립 슬롯 여러 개 (각자 로딩·에러)   |
| **인터셉팅 라우트**   | `(.)photo/[id]`            | `/photo/1`              | 내비게이션 시 모달로, 새로고침 시 전체 페이지로    |
| **프라이빗 폴더**     | `_components`              | 라우팅 제외             | 컴포넌트·유틸 동거                                 |

```tsx
// app/(shop)/products/[id]/page.tsx
import { notFound } from "next/navigation";

export default async function ProductPage({
  params,
  searchParams,
}: {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ tab?: string }>;
}) {
  const { id } = await params;                 // Next.js 15: Promise. 14: 동기 객체
  const { tab = "info" } = await searchParams; // searchParams 사용 시 동적 렌더링(unit01)
  const product = await getProduct(id);
  if (!product) notFound();                    // 가장 가까운 not-found.tsx 표시
  return <Product data={product} tab={tab} />;
}

export async function generateStaticParams() {
  const ids = await getPopularProductIds();
  return ids.map((id) => ({ id }));            // 빌드 시 미리 생성할 동적 경로(SSG)
}
```

> 💡 라우트 그룹은 "로그인 전/후 레이아웃이 다른데 URL은 같은 깊이여야 하는" 상황의 정답이다. `(auth)/login`과 `(app)/dashboard`처럼 그룹별로 `layout.tsx`를 따로 두면 URL은 `/login`, `/dashboard`로 유지된다.

<br>

### 3. 레이아웃 vs 템플릿

### 3-1. 핵심 차이 — 인스턴스가 유지되는가

| **항목**                          | **`layout.tsx`**                           | **`template.tsx`**                                |
| --------------------------------- | ------------------------------------------ | ------------------------------------------------- |
| **내비게이션 시**                 | **인스턴스 유지**, 리렌더링 없음           | **새 인스턴스 마운트** (언마운트 → 마운트)        |
| **내부 `useState`**               | 유지됨                                     | 초기화됨                                          |
| **`useEffect`**                   | 최초 1회                                   | 페이지 이동마다 재실행                            |
| **DOM 요소**                      | 재사용                                     | 새로 생성                                         |
| **`searchParams`·`pathname` 접근** | 불가 (서버 레이아웃은 props로 못 받음)     | 불가 (동일)                                       |
| **적합한 용도**                   | 내비게이션 바, 사이드바, Provider          | 페이지 진입 애니메이션, 페이지별 로깅, 폼 초기화  |

```tsx
// app/dashboard/template.tsx — 이동할 때마다 진입 애니메이션이 새로 실행됨
"use client";
import { motion } from "framer-motion";

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }}>
      {children}
    </motion.div>
  );
}
```

- 같은 위치에 둘 다 있으면 **레이아웃이 템플릿을 감싼다**. 레이아웃 안의 `{children}` 자리에 템플릿이 들어간다.
- 서버 레이아웃이 현재 경로를 알 수 없는 것은 **부분 렌더링을 위해 의도된 제약**이다. 활성 메뉴 강조는 `usePathname()`을 쓰는 작은 클라이언트 컴포넌트로 분리한다.

> ⚠️ "이동할 때 `useEffect`가 다시 실행되면 좋겠다"고 레이아웃 안에서 `key`를 바꾸거나 상태를 억지로 초기화하는 코드를 넣는 경우가 많다. 그 요구사항이 바로 **템플릿의 존재 이유**이며, 템플릿으로 바꾸면 대부분 해결된다.

<br>

### 3-2. 루트 레이아웃의 특수성

- `app/layout.tsx`는 필수이며 `<html>`과 `<body>`를 직접 렌더링해야 한다.
- 여기서 `cookies()`·`headers()`를 쓰면 **모든 라우트가 동적 렌더링**으로 바뀐다(unit01 참고). 개인화는 하위 컴포넌트로 내린다.
- 폰트·전역 CSS·메타데이터(`export const metadata`)의 기본 위치이다(unit09 참고).

<br>

### 4. 미들웨어 — 실행 시점

**미들웨어(Middleware)**는 프로젝트 루트(또는 `src/`)의 `middleware.ts` 하나로 정의하며, 요청이 **캐시 조회·라우트 매칭보다 먼저** 실행된다.

```
요청 도착
  └─ ① next.config의 headers / redirects / rewrites (beforeFiles)
      └─ ② middleware.ts 실행  ← 여기
          └─ ③ 파일시스템 라우트 (정적 파일, _next/*, app/ 라우트) + 캐시 조회
              └─ ④ next.config의 rewrites (afterFiles) → 동적 라우트 → fallback
```

```tsx
// middleware.ts
import { NextRequest, NextResponse } from "next/server";

export function middleware(req: NextRequest) {
  const token = req.cookies.get("session")?.value;
  const { pathname } = req.nextUrl;

  if (pathname.startsWith("/dashboard") && !token) {
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    url.searchParams.set("next", pathname);
    return NextResponse.redirect(url);            // 302 리다이렉트
  }
  const res = NextResponse.next();                // 계속 진행
  res.headers.set("x-request-id", crypto.randomUUID());
  return res;
}

export const config = {
  // 정적 자산·이미지·파비콘은 제외해 불필요한 실행을 막음
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

- 반환값으로 **`redirect`(URL 변경), `rewrite`(URL 유지·내부 경로 변경), `next`(통과, 헤더·쿠키 수정 가능)** 중 하나를 선택한다.
- `matcher`를 지정하지 않으면 **프리페치·정적 파일·이미지 요청을 포함한 모든 요청**에 실행되어 비용이 커진다.
- 캐시보다 먼저 실행되므로 정적 페이지 요청에도 매번 돈다. 무거운 로직을 두면 SSG의 이점이 사라진다.

> 💡 `rewrite`는 사용자에게 보이는 URL을 바꾸지 않고 내부 경로만 바꾸므로 **다국어(`/about` → `/ko/about`)·A/B 테스트(`/pricing` → `/pricing-b`)·레거시 서버 프록시**에 쓰인다. 반면 `redirect`는 URL이 바뀌고 검색 엔진에도 이동이 알려지므로 영구 이동에는 `redirect`(308)를 쓴다.

<br>

### 5. 미들웨어의 제약과 보안 경계

| **제약**                              | **내용**                                                                                          |
| ------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **런타임**                            | 14/15 기본은 **Edge 런타임**. Node.js API(`fs`, 네이티브 DB 드라이버 등) 사용 불가. Node.js 런타임 선택은 15.2 실험 → 15.5 안정화(버전에 따라 다를 수 있음) |
| **파일 수**                           | 프로젝트당 **하나**. 경로별 분기는 파일 내부에서 처리                                              |
| **응답 본문**                         | HTML을 직접 생성해 응답하는 용도가 아님. 리다이렉트·리라이트·헤더 조작이 본래 역할               |
| **렌더링 컨텍스트**                   | 서버 컴포넌트·`cookies()` 등 렌더링 API 접근 불가. `req.cookies`·`req.headers`만 사용            |
| **실행 비용**                         | 매 요청 실행. 외부 API 호출(세션 검증 등)을 넣으면 모든 페이지의 TTFB에 더해짐                     |

**보안 경계로 삼지 말 것**

- 미들웨어는 "쿠키가 있는지" 같은 **낙관적(optimistic) 검사**로 리다이렉트를 빠르게 처리하는 용도다. 실제 인가는 **데이터에 접근하는 지점**(서버 컴포넌트·서버 액션·라우트 핸들러, unit06 참고)에서 반드시 다시 수행한다.
- 2025년 초 특정 내부 헤더로 미들웨어를 우회할 수 있는 취약점(CVE-2025-29927)이 공개되어 15.2.3 등에서 패치됐다. 미들웨어만으로 보호한 시스템은 이 한 번의 우회로 전부 뚫렸다.
- Next.js 16에서는 이런 오해를 줄이기 위해 `middleware.ts`가 **`proxy.ts`로 이름이 바뀌고 Node.js 런타임으로 고정**됐다. 14/15 프로젝트를 유지보수한다면 마이그레이션 계획이 필요하다.

❗️**Edge 런타임에서의 세션 검증**: JWT 서명 검증처럼 Web Crypto로 가능한 작업은 Edge에서도 되지만, DB 조회가 필요한 세션 검증은 미들웨어에서 하지 말고 `fetch`로 별도 API를 부르거나 서버 컴포넌트로 미룬다.

<br>

### 6. 정리 — 면접·실무 체크포인트

| **질문**                                      | **핵심 답변**                                                                                  |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **레이아웃과 템플릿의 차이는?**               | 레이아웃은 내비게이션 시 **인스턴스·상태 유지**, 템플릿은 **매번 새로 마운트**. 레이아웃이 템플릿을 감쌈 |
| **서버 레이아웃이 현재 경로를 모르는 이유는?** | 부분 렌더링을 위해 레이아웃은 재렌더링하지 않기 때문. 필요하면 `usePathname` 클라이언트 컴포넌트로 분리 |
| **라우트 그룹의 용도는?**                     | URL에 영향 없이 레이아웃을 나누거나 폴더를 정리                                                |
| **미들웨어는 언제 실행되는가?**               | 캐시 조회·라우트 매칭 **이전**, 매 요청마다. `matcher`로 범위 제한 필수                        |
| **미들웨어에서 DB를 못 쓰는 이유는?**         | 기본 Edge 런타임이라 Node.js API 불가. 15.5+ Node 런타임 옵션 또는 16의 `proxy`로 완화         |
| **미들웨어로 인증을 끝내도 되는가?**          | 아니다. 낙관적 리다이렉트 용도이며, 인가는 데이터 접근 지점에서 다시 수행                       |

- 파일 규칙 중첩 순서는 **레이아웃 → 템플릿 → 에러 → 로딩 → 페이지**다.
- 동적 세그먼트의 `params`·`searchParams`는 15부터 **Promise**이며, `searchParams`를 읽으면 동적 렌더링이 된다(unit01 참고).
- 폰트·이미지·메타데이터 등 루트 레이아웃에서 설정하는 최적화는 unit09를 참고할 것.
