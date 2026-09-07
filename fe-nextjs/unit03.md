## 데이터 페칭과 폭포수

App Router에서는 서버 컴포넌트가 직접 `async/await`로 데이터를 가져오므로 페칭 코드가 단순해지지만, 그만큼 **요청 폭포수(Waterfall)**가 생기기 쉽다. 폭포수가 왜 생기는지, 병렬 페칭·프리로드·`Suspense` 경계 배치로 어떻게 응답 시간을 줄이는지 이해하면 "페이지가 왜 느린가"라는 질문에 구조적으로 답할 수 있다.

<br>

### 1. App Router의 데이터 페칭 방식

| **방식**                          | **실행 위치**    | **특징**                                                         |
| --------------------------------- | ---------------- | ---------------------------------------------------------------- |
| **서버 컴포넌트 + `fetch`**       | 서버             | 가장 기본. 캐시·재검증 옵션을 `fetch`에 직접 지정(unit04·05 참고) |
| **서버 컴포넌트 + ORM·DB 클라이언트** | 서버         | `fetch`를 거치지 않으므로 데이터 캐시 미적용, `unstable_cache`로 감쌈 |
| **라우트 핸들러(`route.ts`)**     | 서버             | 외부 클라이언트나 클라이언트 컴포넌트가 호출하는 API 엔드포인트  |
| **클라이언트 컴포넌트 + SWR·TanStack Query** | 브라우저 | 실시간성·낙관적 업데이트·폴링이 필요한 경우                      |
| **서버 액션(unit06 참고)**        | 서버             | 데이터 **변경(mutation)** 용도. 조회용으로는 권장하지 않음       |

```tsx
// app/posts/page.tsx — 서버 컴포넌트에서 직접 await
export default async function PostsPage() {
  const res = await fetch("https://api.example.com/posts");
  const posts: { id: string; title: string }[] = await res.json();
  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

> 💡 서버 컴포넌트에서 자기 자신의 라우트 핸들러(`/api/...`)를 `fetch`로 호출하는 것은 **불필요한 네트워크 왕복**이다. 같은 서버 안에 있으므로 DB 접근 함수나 서비스 함수를 직접 import해 호출한다.

<br>

### 2. 폭포수(Waterfall)는 왜 생기는가

폭포수는 **앞 요청이 끝나야 다음 요청이 시작되는 직렬 구조**를 말한다. 각 요청이 200ms라면 3단계 폭포수는 최소 600ms가 걸린다.

**순차 `await`로 인한 폭포수**

```tsx
// ❌ 안티패턴: 서로 독립적인 두 요청을 순서대로 기다림
export default async function Page() {
  const user = await getUser();        // 200ms
  const posts = await getPosts();      // 200ms — user 결과와 무관한데 기다린 뒤 시작
  const banners = await getBanners();  // 200ms
  return <Layout user={user} posts={posts} banners={banners} />; // 총 600ms
}
```

```
시간 →      0ms      200ms     400ms     600ms
getUser     ██████████
getPosts              ██████████
getBanners                      ██████████
```

**컴포넌트 중첩으로 인한 폭포수**

부모 컴포넌트가 `await`를 마쳐야 자식이 렌더링을 시작하므로, **컴포넌트 트리 깊이가 곧 폭포수 단계**가 된다. 부모의 결과가 자식에게 꼭 필요하지 않은데도 부모에서 `await`하면 자식의 요청이 늦게 출발한다.

```tsx
// ❌ 부모가 profile을 다 받아야 <Feed/>가 렌더링을 시작 → 두 요청이 직렬화됨
export default async function Page() {
  const profile = await getProfile();
  return (
    <>
      <ProfileCard profile={profile} />
      <Feed />   {/* 내부에서 getFeed()를 await — profile과 무관 */}
    </>
  );
}
```

<br>

### 3. 병렬 페칭 패턴

### 3-1. `Promise.all`로 독립 요청 묶기

서로 의존하지 않는 요청은 **먼저 모두 시작한 뒤** 한 번에 기다린다.

```tsx
// ✅ 개선: 세 요청을 동시에 출발시킴 → 총 200ms
export default async function Page() {
  const [user, posts, banners] = await Promise.all([
    getUser(),
    getPosts(),
    getBanners(),
  ]);
  return <Layout user={user} posts={posts} banners={banners} />;
}
```

```
시간 →      0ms      200ms
getUser     ██████████
getPosts    ██████████
getBanners  ██████████
```

- 하나라도 실패하면 전체가 거부되는 것이 부담이면 `Promise.allSettled`를 사용한다.
- 의존 관계가 실제로 있는 요청(예: 유저 ID → 유저의 주문 목록)은 병렬화할 수 없다. 이때는 **API를 합치거나(BFF), 자식 컴포넌트로 내려 스트리밍**하는 방식으로 체감 지연을 줄인다.

<br>

### 3-2. 프리로드(Preload) 패턴

자식 컴포넌트가 사용할 데이터를 **부모가 미리 요청만 걸어 두는** 패턴이다. 요청 메모이제이션(unit04 참고) 덕분에 같은 `fetch`가 두 번 호출돼도 실제 네트워크 요청은 한 번만 나간다.

```tsx
// lib/data.ts
import { cache } from "react";

export const getItem = cache(async (id: string) => {
  const res = await fetch(`https://api.example.com/items/${id}`);
  return res.json();
});

export const preloadItem = (id: string) => {
  void getItem(id); // 결과를 기다리지 않고 요청만 시작
};

// app/items/[id]/page.tsx
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  preloadItem(id);                 // ① 요청 출발
  const related = await getRelated(id); // ② 다른 작업을 하는 동안 ①이 진행 중
  return <Item id={id} related={related} />; // ③ Item 내부의 getItem(id)는 메모이제이션 적중
}
```

> ⚠️ `React.cache`는 **한 번의 서버 렌더링 요청 안에서만** 결과를 공유하는 메모이제이션이다. 요청이 끝나면 사라지므로 데이터 캐시(unit04)와 혼동하면 안 된다. `fetch`는 GET 요청에 한해 Next.js가 자동으로 메모이제이션하지만, ORM 호출은 `cache()`로 감싸야 중복 실행을 막을 수 있다.

<br>

### 4. Suspense 경계 배치 — "기다리는 단위"를 쪼갠다

`Promise.all`은 요청을 병렬화하지만 여전히 **가장 느린 요청이 끝날 때까지 화면 전체가 대기**한다. `Suspense`로 경계를 나누면 빠른 부분은 먼저 보여 주고 느린 부분은 스트리밍(unit07 참고)으로 나중에 채운다.

```tsx
import { Suspense } from "react";

export default function Page() {
  return (
    <>
      <Header />                                   {/* 데이터 의존 없음 → 즉시 */}
      <Suspense fallback={<PostsSkeleton />}>
        <Posts />                                  {/* 내부에서 await getPosts() */}
      </Suspense>
      <Suspense fallback={<RecommendSkeleton />}>
        <Recommendations />                        {/* 느린 추천 API — 독립적으로 도착 */}
      </Suspense>
    </>
  );
}
```

```
시간 →        0ms          150ms                 900ms
응답 스트림   [Header + 두 스켈레톤] ─── [Posts 완성 청크] ─── [Recommendations 완성 청크]
사용자 화면   헤더·뼈대 표시            게시글 표시            추천 표시
```

**경계 배치 기준**

- **느린 데이터를 사용하는 컴포넌트 바로 위**에 경계를 둔다. 경계가 너무 위에 있으면 빠른 콘텐츠까지 함께 기다린다.
- **서로 다른 속도의 데이터는 서로 다른 경계**로 분리한다. 같은 경계 안에 넣으면 가장 느린 것에 맞춰진다.
- 경계를 너무 잘게 쪼개면 스켈레톤이 여기저기서 따로 튀어 **레이아웃 시프트(CLS)**가 커진다. 시각적으로 한 덩어리인 영역은 한 경계로 묶는다.
- 라우트 세그먼트 단위의 경계는 `loading.tsx` 파일이 자동으로 만들어 준다(unit08 참고).

> 💡 `Promise.all`은 "요청을 동시에 출발시키는 기법", `Suspense`는 "결과를 따로따로 보여 주는 기법"이다. 둘은 대체 관계가 아니라 **함께 쓰는 관계**다. 한 컴포넌트 안에서 필요한 여러 데이터는 `Promise.all`로, 서로 다른 영역의 데이터는 `Suspense`로 분리한다.

<br>

### 5. 클라이언트로 데이터 넘기기 — `use()`와 Promise 전달

느린 데이터를 서버에서 `await`하지 않고 **Promise 그대로 클라이언트 컴포넌트에 넘기면**, 서버는 즉시 HTML을 보내고 클라이언트가 `use()`로 Promise를 풀어 렌더링한다. 직렬화 제약(unit02 참고)상 Promise는 경계를 넘을 수 있다.

```tsx
// app/page.tsx (서버)
import { Suspense } from "react";
import { Comments } from "./comments";

export default function Page() {
  const commentsPromise = getComments(); // await 하지 않음
  return (
    <Suspense fallback={<p>댓글 불러오는 중...</p>}>
      <Comments promise={commentsPromise} />
    </Suspense>
  );
}

// app/comments.tsx (클라이언트)
"use client";
import { use } from "react";

export function Comments({ promise }: { promise: Promise<{ id: string; body: string }[]> }) {
  const comments = use(promise); // 해결될 때까지 가장 가까운 Suspense fallback 표시
  return <ul>{comments.map((c) => <li key={c.id}>{c.body}</li>)}</ul>;
}
```

- 서버 액션은 조회 용도로 쓰면 **POST로 직렬 실행**되어 병렬화·캐시 이점이 없다. 조회는 서버 컴포넌트나 라우트 핸들러, 변경만 서버 액션으로 나눈다.

<br>

### 6. 정리 — 면접·실무 체크포인트

| **상황**                                         | **선택**                                                   |
| ------------------------------------------------ | ---------------------------------------------------------- |
| **독립적인 요청이 한 컴포넌트에 여러 개**        | `Promise.all`로 병렬화                                     |
| **자식이 쓸 데이터를 부모가 알고 있음**          | 프리로드 패턴 (`cache()` + 요청만 먼저 시작)               |
| **속도가 다른 영역이 한 페이지에 공존**          | 영역별 `Suspense` 경계 + 스트리밍                          |
| **느린 데이터를 클라이언트에서 처리하고 싶음**   | Promise를 props로 전달 + `use()`                           |
| **의존 관계가 있어 병렬화 불가**                 | API 병합(BFF) 또는 하위 컴포넌트로 내려 스트리밍           |
| **실시간·폴링·낙관적 업데이트**                  | 클라이언트 컴포넌트 + SWR·TanStack Query                   |

- 폭포수의 원인은 **순차 `await`**와 **컴포넌트 중첩**, 두 가지다.
- 병렬화(`Promise.all`·프리로드)는 "출발을 앞당기는 것", `Suspense`는 "도착한 순서대로 보여 주는 것"이다.
- 요청 메모이제이션과 데이터 캐시의 차이는 unit04, 스트리밍 SSR의 동작은 unit07을 참고할 것.
