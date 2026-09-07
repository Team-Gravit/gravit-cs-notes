## 서버 액션

**서버 액션(Server Action)**은 `"use server"` 지시자를 붙인 비동기 함수로, 클라이언트에서 호출하면 Next.js가 자동으로 POST 요청을 만들어 **서버에서 실행**해 주는 데이터 변경(mutation) 수단이다. API 라우트를 따로 만들지 않고도 폼 제출·삭제·좋아요 같은 변경을 처리하고 곧바로 캐시를 재검증할 수 있지만, "그냥 함수처럼 보이는 공개 HTTP 엔드포인트"라는 본질을 잊으면 심각한 보안 문제로 이어진다.

<br>

### 1. 서버 액션의 동작 원리

```
브라우저                                           서버
<form action={createPost}>                         ┌─ createPost() 실행 (DB 변경)
  └─ 제출 ──POST /현재경로 (Next-Action: <id>)──→   ├─ revalidateTag / redirect
                                                   └─ 응답: 반환값 + 갱신된 RSC 페이로드
  ← 화면 갱신 (라우터 캐시 교체) ──────────────────┘
```

- 빌드 시 각 액션에 **고유 ID**가 부여되고, 클라이언트 번들에는 함수 본체 대신 이 ID만 남는다. 호출 시 ID와 인자를 담은 POST 요청이 나간다(unit02 직렬화 제약 참고).
- 서버는 ID로 함수를 찾아 실행하고, 결과와 함께 **갱신된 RSC 페이로드**를 응답한다. 덕분에 액션 안에서 `revalidatePath`를 부르면 별도 재요청 없이 화면이 바뀐다(unit05 참고).
- 서버 컴포넌트 안에서 인라인으로 정의하거나, `"use server"`를 파일 최상단에 쓴 별도 파일(`actions.ts`)에서 export해 클라이언트 컴포넌트로 import할 수 있다.

> 💡 `"use client"`는 "경계"를 선언하지만, `"use server"`는 "이 함수를 **서버에서 실행되는 원격 호출 대상**으로 노출하라"는 뜻이다. 두 지시자는 대칭이 아니며, `"use server"`를 컴포넌트 파일에 붙인다고 서버 컴포넌트가 되는 것도 아니다.

<br>

### 2. 폼 처리 기본 패턴

### 2-1. `<form action>`과 점진적 향상

```tsx
// app/posts/actions.ts
"use server";
import { revalidateTag } from "next/cache";
import { redirect } from "next/navigation";

export async function createPost(formData: FormData) {
  const title = String(formData.get("title") ?? "").trim();
  if (!title) throw new Error("제목은 필수입니다");
  const post = await db.post.create({ data: { title } });
  revalidateTag("posts");          // 목록 캐시 무효화
  redirect(`/posts/${post.id}`);   // 상세 페이지로 이동 (throw 방식이므로 마지막에 호출)
}

// app/posts/new/page.tsx (서버 컴포넌트)
import { createPost } from "../actions";

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <button type="submit">작성</button>
    </form>
  );
}
```

- `<form action={서버액션}>`은 JavaScript가 로드되기 전에 제출해도 **일반 HTML 폼 POST로 동작**한다(점진적 향상, Progressive Enhancement). 하이드레이션(unit07 참고)이 끝나면 페이지 전환 없는 fetch 방식으로 바뀐다.
- `redirect()`는 예외를 던지는 방식으로 구현되어 있으므로 `try/catch` 안에서 호출하면 catch에 잡혀 동작하지 않는다.

<br>

### 2-2. 상태·로딩·낙관적 업데이트 훅

| **훅**                | **역할**                                          | **비고**                                                  |
| --------------------- | ------------------------------------------------- | --------------------------------------------------------- |
| **`useActionState`**  | 액션의 반환값(에러 메시지 등)과 pending 상태 관리 | React 19 / Next.js 15 기준. 14에서는 `useFormState`       |
| **`useFormStatus`**   | 부모 `<form>`의 제출 중 여부를 자식에서 읽음      | 제출 버튼 비활성화용. 반드시 `<form>` **자식** 컴포넌트에서 사용 |
| **`useOptimistic`**   | 서버 응답 전에 UI를 먼저 갱신                     | 좋아요·댓글처럼 실패 확률이 낮은 변경에 적합              |

```tsx
"use client";
import { useActionState } from "react";
import { useFormStatus } from "react-dom";
import { createPost } from "./actions";

type State = { error?: string };

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? "저장 중..." : "작성"}</button>;
}

export function PostForm() {
  const [state, formAction] = useActionState<State, FormData>(
    async (_prev, formData) => {
      try { await createPost(formData); return {}; }
      catch (e) { return { error: (e as Error).message }; }
    },
    {}
  );
  return (
    <form action={formAction}>
      <input name="title" />
      {state.error && <p role="alert">{state.error}</p>}
      <SubmitButton />
    </form>
  );
}
```

> ⚠️ 서버 액션에서 `throw`한 에러의 메시지는 **프로덕션에서는 클라이언트에 그대로 전달되지 않고 일반 메시지로 대체**된다(내부 정보 노출 방지). 사용자에게 보여 줄 검증 오류는 예외가 아니라 **반환값**으로 돌려주는 것이 정석이다.

<br>

### 3. 변경 후 재검증과 화면 갱신

액션이 데이터를 바꾼 뒤에는 관련 캐시(unit04·05 참고)를 갱신해야 화면에 반영된다.

| **상황**                                | **호출**                           | **효과**                                               |
| --------------------------------------- | ---------------------------------- | ------------------------------------------------------ |
| **목록·상세 등 관련 데이터가 흩어짐**   | `revalidateTag('posts')`           | 태그가 달린 데이터 캐시 + 이를 쓰는 라우트 캐시 무효화 |
| **현재 페이지만 새로 그리면 됨**        | `revalidatePath('/posts')`         | 해당 경로의 전체 라우트 캐시 무효화                    |
| **다른 페이지로 이동**                  | `redirect('/posts/1')`             | 이동 + 대상 페이지 최신 렌더링                         |
| **쿠키 변경(로그인·설정)**              | `(await cookies()).set(...)`       | 액션 안에서만 쿠키 쓰기 가능. 응답과 함께 화면 갱신    |

- 서버 액션 안에서 호출한 재검증은 **응답에 새 RSC 페이로드를 실어** 브라우저의 라우터 캐시까지 갱신한다. 라우트 핸들러에서 호출했을 때와의 차이는 unit05 참고.
- 서버 액션은 **직렬로 실행**된다. 같은 사용자가 액션을 연달아 호출하면 큐에 쌓여 순서대로 처리되므로, 조회 용도로 남용하면 병렬 페칭(unit03 참고)의 이점을 잃는다.

<br>

### 4. 보안 고려사항 — "함수"가 아니라 "공개 엔드포인트"

**반드시 액션 내부에서 인증·인가·검증**

서버 액션은 **URL을 아는 누구나 호출할 수 있는 POST 엔드포인트**다. 버튼을 숨겼다고, 페이지에 접근 못 하게 했다고 액션이 보호되는 것이 아니다.

```tsx
// ❌ 안티패턴: 페이지에서 권한을 확인했으니 액션은 믿고 실행
"use server";
export async function deletePost(id: string) {
  await db.post.delete({ where: { id } });   // 누구나 임의 id로 호출 가능
}
```

```tsx
// ✅ 개선: 액션 안에서 세션 확인 → 소유권 확인 → 입력 검증
"use server";
import { z } from "zod";
import { revalidateTag } from "next/cache";

const schema = z.object({ id: z.string().uuid() });

export async function deletePost(input: unknown) {
  const session = await getSession();                 // ① 인증
  if (!session) return { error: "로그인이 필요합니다" };

  const parsed = schema.safeParse(input);             // ② 입력 검증
  if (!parsed.success) return { error: "잘못된 요청" };

  const post = await db.post.findUnique({ where: { id: parsed.data.id } });
  if (post?.authorId !== session.userId) return { error: "권한 없음" }; // ③ 인가

  await db.post.delete({ where: { id: parsed.data.id } });
  revalidateTag("posts");
  return { ok: true };
}
```

**프레임워크가 제공하는 보호와 그 한계**

| **항목**                    | **내용**                                                                                             |
| --------------------------- | ---------------------------------------------------------------------------------------------------- |
| **CSRF 방어**               | 서버 액션은 POST만 허용하고 `Origin`과 `Host` 헤더를 비교해 불일치 시 거부함. 리버스 프록시 뒤라면 `serverActions.allowedOrigins` 설정 필요 |
| **클로저 변수 암호화**      | 인라인 액션이 캡처한 변수는 암호화되어 클라이언트를 왕복함. 다만 **복호화 후 서버에서 사용**되므로 값 변조보다 "노출"에 주의 |
| **액션 ID 보안 (15)**       | Next.js 15부터 액션 ID를 **추측 불가능한 값**으로 생성하고, 사용되지 않는 액션은 번들에서 제거(데드 코드 제거)함 |
| **에러 메시지 마스킹**      | 프로덕션에서 서버 에러 상세는 클라이언트에 전달되지 않음                                              |
| **인자 신뢰 불가**          | 클라이언트가 보낸 인자는 **항상 조작 가능**. 숨김 필드·disabled 필드 값도 신뢰하면 안 됨              |

❗️**export된 액션은 모두 엔드포인트**: `"use server"` 파일에서 export한 함수는 어디서도 import하지 않아도(15의 데드 코드 제거 이전 버전에서는 특히) 호출 가능한 엔드포인트가 될 수 있다. 헬퍼 함수는 액션 파일에 두지 않거나 export하지 않는다.

> 💡 면접에서 "서버 액션은 API 라우트보다 안전한가"라고 물으면, **전송 계층의 CSRF 방어와 ID 난독화는 프레임워크가 해 주지만, 인증·인가·검증은 개발자 책임이며 API 라우트와 동일하게 취급해야 한다**고 답하는 것이 정확하다.

<br>

### 5. 서버 액션 vs 라우트 핸들러 선택 기준

| **기준**                         | **서버 액션**                              | **라우트 핸들러(`route.ts`)**              |
| -------------------------------- | ------------------------------------------ | ------------------------------------------ |
| **주 용도**                      | **UI에서 시작되는 변경** (폼·버튼)         | 외부 시스템·모바일 앱·웹훅용 **공개 API**  |
| **HTTP 메서드**                  | POST 고정                                  | GET·POST·PUT·DELETE 등 자유                |
| **캐시 재검증 후 화면 갱신**     | 자동 (RSC 페이로드 응답)                   | 수동 (`router.refresh()` 등)               |
| **점진적 향상**                  | 지원 (JS 없이도 폼 동작)                   | 별도 구현 필요                             |
| **응답 형식 제어**               | 직렬화 가능한 값만 반환                    | 상태 코드·헤더·스트리밍 등 완전 제어       |
| **데이터 조회**                  | 비권장 (직렬 POST)                         | 적합 (GET, 캐시 가능)                      |

<br>

### 6. 정리 — 면접·실무 체크포인트

- 서버 액션은 `"use server"` 함수를 **ID 기반 POST 엔드포인트**로 노출한 것이며, 응답에 갱신된 RSC 페이로드를 실어 **재검증과 화면 갱신을 한 번에** 처리한다.
- 폼은 `<form action>`으로 점진적 향상을 얻고, 상태는 `useActionState`(14는 `useFormState`), 로딩은 `useFormStatus`, 낙관적 UI는 `useOptimistic`으로 처리한다.
- 변경 후에는 **`revalidateTag`(데이터 중심) 또는 `revalidatePath`(페이지 중심)**를 호출하고, 이동이 필요하면 `redirect`를 마지막에 호출한다.
- 보안의 핵심은 **액션 내부에서 인증 → 검증 → 인가**를 매번 수행하는 것이다. 프레임워크는 CSRF와 ID 난독화만 도와준다.
- 조회는 서버 컴포넌트·라우트 핸들러, 변경은 서버 액션으로 역할을 나눈다.
- 캐시 계층은 unit04, 재검증 호출 위치별 차이는 unit05, 직렬화 제약은 unit02를 참고할 것.
