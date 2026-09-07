## 재검증 전략

**재검증(Revalidation)**은 캐시된 데이터나 페이지를 "이제 오래됐다"고 표시하고 다시 만들게 하는 과정이다. unit04의 캐시 계층은 "무엇을 저장하는가"였다면, 이 유닛은 **언제 그것을 버리는가**를 다룬다. 시간 기반과 태그·경로 기반의 온디맨드(On-demand) 재검증이 각각 어떤 시점에 캐시를 무효화하는지 알아야 신선도와 비용 사이에서 올바른 지점을 고를 수 있다.

<br>

### 1. 재검증이 필요한 이유

- 캐시는 성능을 위해 **오래된 데이터를 의도적으로 보여 주는** 장치다. 문제는 "얼마나 오래"가 허용되는지 서비스마다 다르다는 점이다.
- 재배포로 캐시를 비우는 것은 느리고 비용이 크다. 재검증은 **재배포 없이** 특정 데이터·페이지만 골라 갱신하는 수단이다.
- 재검증 전략은 두 축으로 나뉜다. **시간 기반**(주기적으로 만료)과 **온디맨드**(이벤트가 발생했을 때 즉시 무효화).

| **구분**          | **시간 기반 재검증**                          | **온디맨드 재검증 (태그·경로)**                    |
| ----------------- | --------------------------------------------- | -------------------------------------------------- |
| **트리거**        | 지정한 초가 지난 뒤 **다음 요청**             | `revalidateTag`·`revalidatePath` **호출 즉시**     |
| **신선도**        | 최대 `revalidate`초만큼 지연                  | 변경 직후 반영 (다음 방문부터)                     |
| **구현 난이도**   | 옵션 하나                                     | 변경 지점마다 호출 코드 필요                       |
| **적합한 데이터** | 변경 시점을 알 수 없는 외부 데이터·통계       | 내 시스템에서 변경되는 데이터 (CMS·관리자·유저 액션) |
| **대표 API**      | `fetch(..., { next: { revalidate: N } })`, `export const revalidate = N` | `revalidateTag('tag')`, `revalidatePath('/path')` |

<br>

### 2. 시간 기반 재검증 — stale-while-revalidate

### 2-1. 동작 흐름

시간 기반 재검증은 만료 시각이 지나도 캐시를 즉시 버리지 않는다. **만료 후 첫 요청에는 오래된 값을 그대로 주고, 백그라운드에서 새 값을 만들어 다음 요청부터 반영**한다. 이것이 stale-while-revalidate 방식이며 ISR(unit01 참고)의 기반이다.

```
revalidate = 60 인 경우

시간 →   0s          30s         60s   61s               62s
요청     ①           ②                 ③                 ④
응답     새로 생성    캐시(신선)         캐시(오래됨) 반환   새 캐시 반환
                                       └─ 백그라운드 재생성 → 60s 캐시 교체
```

- 요청 ③은 만료 후 첫 요청이지만 **기다리지 않고 오래된 값을 받는다**. 사용자 지연이 없다는 것이 장점이고, 정확히 "60초 후"가 아니라 "60초 후 첫 방문 이후"에 갱신된다는 것이 함정이다.
- 방문이 없는 페이지는 아무리 시간이 지나도 재생성되지 않는다. 트래픽이 적은 페이지에서는 `revalidate` 값보다 훨씬 오래된 데이터가 보일 수 있다.
- 재생성이 실패하면 기존 캐시를 계속 제공한다.

<br>

### 2-2. 설정 위치 두 가지

```tsx
// ① fetch 단위: 데이터마다 다른 주기
const products = await fetch("https://api.example.com/products", {
  next: { revalidate: 300 },     // 5분
}).then((r) => r.json());

// ② 라우트 세그먼트 단위: 파일 전체의 기본 주기 (page.tsx / layout.tsx / route.ts)
export const revalidate = 60;     // 이 라우트의 전체 라우트 캐시를 60초마다 재생성
```

> ⚠️ 한 라우트 안에 서로 다른 `revalidate` 값이 섞이면 **가장 짧은 값이 라우트 전체에 적용**된다. `revalidate = 3600`인 페이지에 `revalidate: 10`인 `fetch`가 하나 있으면 페이지는 10초마다 재생성된다. `revalidate = 0`은 동적 렌더링(항상 최신), `false`는 무기한 캐시를 뜻한다.

<br>

### 3. 태그 기반 재검증 — `revalidateTag`

데이터에 **태그(라벨)**를 붙여 두고, 변경이 발생했을 때 그 태그가 달린 모든 캐시 항목을 한꺼번에 무효화한다. 어떤 페이지가 그 데이터를 쓰는지 몰라도 되므로 **데이터 중심**으로 관리할 수 있다.

```tsx
// lib/posts.ts — 태그 부여
export async function getPosts() {
  const res = await fetch("https://api.example.com/posts", {
    next: { tags: ["posts"] },          // fetch에 태그
  });
  return res.json();
}

export const getPostById = unstable_cache(
  async (id: string) => db.post.findUnique({ where: { id } }),
  ["post-by-id"],
  { tags: ["posts"] }                   // ORM 호출에도 같은 태그 (unit04 참고)
);
```

```tsx
// app/actions.ts — 변경 후 태그 무효화 (서버 액션, unit06 참고)
"use server";
import { revalidateTag } from "next/cache";

export async function createPost(formData: FormData) {
  await db.post.create({ data: { title: String(formData.get("title")) } });
  revalidateTag("posts");   // "posts" 태그가 달린 데이터 캐시 전부 무효화
}
```

- 무효화된 항목은 **다음 요청 시 다시 페칭**되며, 그 데이터를 쓰는 정적 라우트의 전체 라우트 캐시도 함께 무효화된다.
- 태그는 `["posts", "post-42"]`처럼 여러 개 붙일 수 있다. 목록 전체용 태그와 개별 항목용 태그를 함께 붙이면 "글 하나 수정 → 그 글 상세 + 목록"만 정확히 갱신할 수 있다.

> 💡 `revalidateTag`는 캐시를 "지운다"기보다 **"오래됨(stale)으로 표시"**한다. 표시된 항목은 다음 방문 때 다시 생성되므로, 호출 직후 서버에서 곧바로 재생성이 일어나는 것은 아니다. 호출 시점의 세부 동작(즉시 무효화 vs 만료 표시)은 버전에 따라 다를 수 있다.

<br>

### 4. 경로 기반 재검증 — `revalidatePath`

특정 **URL 경로 단위**로 전체 라우트 캐시와 그 경로가 사용한 데이터 캐시를 무효화한다. 태그를 붙여 두지 않은 데이터나 "이 페이지를 통째로 새로 만들어라"가 목적일 때 사용한다.

```tsx
"use server";
import { revalidatePath } from "next/cache";

export async function updateProfile(formData: FormData) {
  await db.user.update({ /* ... */ });
  revalidatePath("/profile");             // 특정 페이지
  revalidatePath("/blog/[slug]", "page"); // 동적 라우트 패턴 전체
  revalidatePath("/", "layout");          // 루트 레이아웃 아래 모든 경로
}
```

| **호출 형태**                         | **무효화 범위**                                         |
| ------------------------------------- | ------------------------------------------------------- |
| **`revalidatePath('/blog/hello')`**   | 해당 URL 하나                                           |
| **`revalidatePath('/blog/[slug]', 'page')`** | 패턴에 해당하는 모든 페이지                      |
| **`revalidatePath('/blog', 'layout')`** | `/blog` 레이아웃을 공유하는 모든 하위 라우트          |
| **`revalidatePath('/', 'layout')`**   | 사이트 전체 (사실상 캐시 전체 초기화)                   |

❗️**범위를 넓게 잡는 습관의 비용**: `revalidatePath('/', 'layout')`는 편하지만 모든 페이지가 다음 요청에서 재생성되어 서버 부하가 급증한다. 태그로 좁게 무효화할 수 있다면 태그를 우선한다.

<br>

### 5. 외부 이벤트로 무효화하기 — 웹훅 + 라우트 핸들러

CMS나 관리자 시스템처럼 **Next.js 밖에서 데이터가 바뀌는 경우**, 라우트 핸들러를 만들어 외부에서 호출하게 한다.

```tsx
// app/api/revalidate/route.ts
import { NextRequest, NextResponse } from "next/server";
import { revalidateTag } from "next/cache";

export async function POST(req: NextRequest) {
  const secret = req.headers.get("x-revalidate-secret");
  if (secret !== process.env.REVALIDATE_SECRET) {
    return NextResponse.json({ ok: false }, { status: 401 });
  }
  const { tag } = (await req.json()) as { tag: string };
  revalidateTag(tag);
  return NextResponse.json({ ok: true, revalidated: tag, now: Date.now() });
}
```

```
CMS에서 글 발행 ──웹훅(POST /api/revalidate, tag=posts)──→ Next.js 서버
                                                          ├─ 비밀 키 검증
                                                          └─ revalidateTag("posts")
사용자 다음 방문 ─────────────────────────────────────────→ 새 데이터로 재생성
```

- 비밀 키 검증 없이 노출하면 누구나 캐시를 무효화해 **서버를 재생성 폭주로 몰아넣을 수 있다**.
- 라우트 핸들러에서 호출한 `revalidateTag`·`revalidatePath`는 서버 캐시만 갱신한다. 이미 열려 있는 브라우저의 라우터 캐시(unit04 참고)는 다음 내비게이션이나 `router.refresh()` 때 갱신된다.

<br>

### 6. 캐시 무효화 시점 — 어디서 호출하느냐에 따른 차이

| **호출 위치**            | **서버 캐시**          | **브라우저 라우터 캐시**                         | **비고**                                     |
| ------------------------ | ---------------------- | ------------------------------------------------ | -------------------------------------------- |
| **서버 액션 안**         | 즉시 무효화            | **함께 갱신** (액션 응답에 새 RSC 페이로드 포함) | 폼 제출 후 화면이 바로 바뀜 (unit06 참고)    |
| **라우트 핸들러 안**     | 즉시 무효화            | 갱신 안 됨                                       | 웹훅·외부 시스템 연동용                      |
| **렌더링 중(컴포넌트)**  | 호출 불가              | -                                                | 렌더링 중 호출은 에러                        |
| **시간 만료**            | 다음 요청 시 백그라운드 재생성 | 보관 시간 지나면 자동 재요청            | 방문이 없으면 갱신되지 않음                  |

> 💡 "서버 액션에서 `revalidatePath`를 부르면 브라우저 화면이 왜 바로 바뀌는가"는 좋은 면접 질문이다. 서버 액션은 응답에 **갱신된 RSC 페이로드를 함께 실어 보내고**, 클라이언트 라우터가 이를 받아 현재 화면을 갱신하기 때문이다. 라우트 핸들러에는 이 메커니즘이 없다.

<br>

### 7. 정리 — 면접·실무 체크포인트

- **시간 기반**은 `revalidate` 초 단위로 만료하며, 만료 후 첫 요청은 오래된 값을 받고 백그라운드에서 재생성한다(stale-while-revalidate). 한 라우트에서는 **가장 짧은 값**이 이긴다.
- **태그 기반**(`revalidateTag`)은 데이터 중심으로 관련 캐시를 한꺼번에 무효화한다. 목록 태그 + 개별 태그를 함께 붙여 범위를 정밀하게 잡는다.
- **경로 기반**(`revalidatePath`)은 페이지 중심이다. `'layout'` 옵션은 범위가 넓어 서버 부하를 유발하므로 최후 수단으로 쓴다.
- 외부 시스템의 변경은 **비밀 키로 보호된 라우트 핸들러 + 웹훅**으로 받는다.
- 서버 액션에서 호출한 재검증만 **브라우저 라우터 캐시까지 함께 갱신**된다. 라우트 핸들러·웹훅은 서버 캐시만 갱신한다.
- 선택 기준: 변경 시점을 **내가 알면 온디맨드(태그 우선)**, **모르면 시간 기반**, 개인화 데이터는 캐시하지 않고 동적 렌더링(unit01 참고).
