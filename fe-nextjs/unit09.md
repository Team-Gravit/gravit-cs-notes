## 자산과 번들 최적화

렌더링 전략과 캐시를 아무리 잘 짜도 **이미지 3MB, 폰트 로딩으로 인한 글자 깜빡임, 첫 화면에 필요 없는 300KB JavaScript**가 있으면 사용자는 느리다고 느낀다. Next.js는 `next/image`·`next/font`·`next/dynamic`·`next/script`로 이 문제를 프레임워크 차원에서 해결하며, 각 도구가 **어떤 Core Web Vitals 지표를 왜 개선하는지** 이해해야 도구를 올바른 자리에 쓸 수 있다.

<br>

### 1. 최적화 대상과 지표의 대응 관계

| **자산**              | **방치했을 때의 문제**                          | **영향 지표**            | **Next.js 도구**                       |
| --------------------- | ----------------------------------------------- | ------------------------ | -------------------------------------- |
| **이미지**            | 원본 크기 전송, 크기 미지정으로 레이아웃 밀림   | **LCP**, **CLS**         | `next/image`                           |
| **폰트**              | 외부 요청 왕복, 폰트 교체 시 글자 깜빡임(FOUT)  | **CLS**, FCP             | `next/font`                            |
| **JavaScript 번들**   | 첫 화면에 불필요한 코드 다운로드·실행           | **INP**, TTI, LCP        | 라우트 분할, `next/dynamic`, 서버 컴포넌트 |
| **서드파티 스크립트** | 분석·광고 스크립트가 메인 스레드 점유           | INP, TTI                 | `next/script`                          |

```
페이지 로딩 타임라인 (최적화 전 → 후)

전:  [HTML] [JS 800KB ─────────────────] [폰트 ↔ 외부 CDN] [이미지 3MB ────────]  LCP 4.5s
후:  [HTML] [JS 180KB ────] [폰트(자체 호스팅)] [이미지 120KB WebP]                LCP 1.4s
                └ 나머지 JS는 상호작용 시 지연 로딩
```

> 💡 **LCP(Largest Contentful Paint)**는 대개 히어로 이미지나 큰 제목 폰트가 결정하고, **CLS(Cumulative Layout Shift)**는 크기 미지정 이미지와 폰트 교체가, **INP(Interaction to Next Paint)**는 JS 실행량이 결정한다. "무엇을 최적화하면 어떤 지표가 좋아지는가"를 연결해 답하면 면접에서 설득력이 높다.

<br>

### 2. 이미지 최적화 — `next/image`

**무엇을 자동으로 해 주는가**

- **크기 조정·포맷 변환**: 요청 시 서버(또는 이미지 CDN)가 뷰포트에 맞는 크기로 리사이즈하고 WebP·AVIF로 변환해 전송한다.
- **지연 로딩(lazy loading)**: 뷰포트 밖 이미지는 스크롤이 가까워질 때 로드한다. 기본값이므로 첫 화면 이미지에는 오히려 꺼야 한다.
- **CLS 방지**: `width`·`height`(또는 `fill`)를 강제해 이미지가 로드되기 전에 **자리를 미리 확보**한다.
- **`srcset`·`sizes` 생성**: 기기 해상도별 후보를 만들어 브라우저가 적절한 것을 고르게 한다.

```tsx
// ❌ 안티패턴: 원본을 CSS 배경으로 그대로 사용 → 3MB 전송, srcset·지연 로딩·크기 확보 전부 없음
export function Hero() {
  return <div style={{ backgroundImage: "url(/hero.jpg)" }} role="img" aria-label="메인 배너" />;
}
```

```tsx
// ✅ 개선: 첫 화면 이미지는 priority, 크기·sizes 지정
import Image from "next/image";

export function Hero() {
  return (
    <Image
      src="/hero.jpg"
      alt="메인 배너"
      width={1200}
      height={600}
      priority                       // LCP 후보 → 지연 로딩 해제 + preload 힌트
      sizes="(max-width: 768px) 100vw, 1200px" // 뷰포트별 필요 폭 → 과다 다운로드 방지
    />
  );
}
```

**흔한 실수**

| **실수**                                   | **결과**                                        | **해결**                                           |
| ------------------------------------------ | ----------------------------------------------- | -------------------------------------------------- |
| **첫 화면 이미지에 `priority` 누락**       | LCP 이미지가 지연 로딩되어 LCP 악화             | 뷰포트 안 가장 큰 이미지 1~2개에만 `priority`      |
| **모든 이미지에 `priority`**               | preload 남발로 다른 리소스가 밀림               | LCP 후보에만 사용                                  |
| **`sizes` 미지정 + 반응형**                | 모바일에서도 데스크톱 크기 다운로드             | 실제 렌더링 폭에 맞는 `sizes` 작성                 |
| **외부 도메인 이미지 에러**                | `hostname is not configured` 에러               | `next.config`의 `images.remotePatterns` 등록       |
| **`fill` 사용 시 부모에 `position` 없음**  | 이미지가 화면 전체로 퍼짐                       | 부모에 `position: relative`와 크기 지정            |

> ⚠️ 이미지 최적화는 **요청 시 서버 CPU를 사용**한다. 셀프 호스팅 환경에서 트래픽이 많으면 이미지 변환이 병목이 될 수 있으므로, 이미지 CDN(`loader` 설정)으로 위임하거나 빌드 시 미리 변환하는 방식을 검토한다. 15에서는 `images.localPatterns` 등 설정 항목이 추가됐으며 세부 옵션은 버전에 따라 다를 수 있다.

<br>

### 3. 폰트 최적화 — `next/font`

`next/font`는 빌드 시 폰트 파일을 **프로젝트 안으로 내려받아 자체 호스팅**하고, 폴백 폰트의 크기를 자동 조정(`size-adjust`)해 폰트 교체 시 레이아웃 이동을 없앤다.

```tsx
// app/layout.tsx
import { Noto_Sans_KR } from "next/font/google";
import localFont from "next/font/local";

const notoSans = Noto_Sans_KR({
  subsets: ["latin"],            // 한글 서브셋은 자동 처리, 버전에 따라 옵션이 다를 수 있음
  weight: ["400", "700"],
  display: "swap",
  variable: "--font-sans",       // CSS 변수로 노출 → Tailwind 등과 연동
});

const pretendard = localFont({
  src: "./fonts/PretendardVariable.woff2",
  variable: "--font-pretendard",
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko" className={`${notoSans.variable} ${pretendard.variable}`}>
      <body>{children}</body>
    </html>
  );
}
```

- **외부 요청 0회**: Google Fonts CDN으로의 DNS·연결 비용이 사라지고, 개인정보 관점에서도 유리하다.
- **CLS 0**: 폴백 폰트의 `size-adjust`·`ascent-override`를 계산해 실제 폰트와 같은 공간을 차지하게 만든다.
- 루트 레이아웃(unit08 참고)에서 한 번만 선언하고 CSS 변수로 내려보내는 것이 표준 패턴이다. 여러 파일에서 같은 폰트를 각각 선언하면 중복 인스턴스가 생긴다.

❗️**가변 폰트 우선**: 굵기별로 파일을 여러 개 받는 것보다 가변(Variable) 폰트 하나가 전체 전송량이 작다. 한글 폰트는 특히 용량이 크므로 서브셋과 가변 폰트를 조합한다.

<br>

### 4. 코드 스플리팅 — 라우트 단위와 컴포넌트 단위

### 4-1. 자동 라우트 분할과 서버 컴포넌트

- App Router는 **라우트 세그먼트마다 별도 청크**를 만든다. `/dashboard`를 열 때 `/settings`의 코드는 내려받지 않는다.
- 공유 레이아웃과 공통 라이브러리(React 등)는 별도 공용 청크로 분리되어 라우트 간에 재사용된다.
- **서버 컴포넌트는 애초에 번들에 포함되지 않는다**(unit02 참고). 마크다운 파서·날짜 라이브러리·큰 데이터 변환 로직을 서버 컴포넌트에 두는 것만으로 클라이언트 번들이 줄어든다.

```
클라이언트 번들 구성 (예)
┌───────────────┬──────────────┬───────────────┬─────────────────┐
│ 프레임워크 청크 │ 공용 레이아웃  │ /dashboard 청크 │ /settings 청크   │
│ (React, 라우터) │ (Nav, Provider)│ (첫 방문 시)   │ (이동 시 지연 로딩)│
└───────────────┴──────────────┴───────────────┴─────────────────┘
        + <Link>가 뷰포트에 보이면 대상 라우트 청크를 프리페치
```

<br>

### 4-2. `next/dynamic`으로 컴포넌트 단위 분할

한 라우트 안에서도 **처음에 보이지 않거나 무거운 컴포넌트**는 사용 시점에 로드한다.

```tsx
"use client";
import dynamic from "next/dynamic";
import { useState } from "react";

// 차트 라이브러리(수백 KB)는 버튼을 누를 때만 로드
const HeavyChart = dynamic(() => import("./heavy-chart"), {
  loading: () => <p>차트 로딩 중...</p>,
  ssr: false,   // 브라우저 API에 의존하는 경우만. 기본은 SSR 포함
});

export function Report() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(true)}>차트 보기</button>
      {show && <HeavyChart />}
    </>
  );
}
```

- `dynamic()`은 `React.lazy` + `Suspense`를 감싼 것으로, 모달·에디터·차트·지도처럼 **조건부로 표시되는 무거운 UI**에 적합하다.
- 라이브러리만 지연 로드하려면 컴포넌트가 아니라 **동적 `import()`** 를 이벤트 핸들러 안에서 호출한다(예: `const { default: confetti } = await import("canvas-confetti")`).
- `ssr: false`는 서버 컴포넌트 안에서는 사용할 수 없다(15 기준). 클라이언트 컴포넌트로 감싸서 쓴다.

> 💡 "번들이 큰데 어디서 줄여야 하는가"를 물으면 순서는 **① 서버 컴포넌트로 옮길 수 있는가 → ② 라우트 분할이 제대로 되는가 → ③ `next/dynamic`으로 지연할 수 있는가 → ④ 라이브러리 자체를 가벼운 것으로 바꿀 수 있는가**다. `@next/bundle-analyzer`로 청크 구성을 시각화한 뒤 판단한다.

<br>

### 5. 번들 크기를 줄이는 보조 기법

| **기법**                                    | **효과**                                                          |
| ------------------------------------------- | ----------------------------------------------------------------- |
| **`optimizePackageImports`**                | 배럴 파일(`index.ts` 재export)이 큰 라이브러리에서 사용한 모듈만 포함. 아이콘·UI 킷에 효과적 |
| **`import "server-only"`**                  | 서버 전용 모듈이 실수로 번들에 포함되는 것을 빌드 시 차단(unit02 참고) |
| **`next/script`의 `strategy`**              | `afterInteractive`(기본)·`lazyOnload`·`beforeInteractive`·`worker`로 서드파티 스크립트 로드 시점 제어 |
| **트리 셰이킹 친화적 import**               | `import { debounce } from "lodash-es"`처럼 ES 모듈 경로 사용        |
| **폴리필 최소화**                           | `browserslist`를 현실적인 범위로 설정해 불필요한 변환·폴리필 제거   |
| **메타데이터 API**                          | `export const metadata`·`generateMetadata`로 `<head>`를 서버에서 생성. 클라이언트 `<Head>` 조작 JS 제거 |

```tsx
// app/layout.tsx — 분석 스크립트는 상호작용 이후에, 채팅 위젯은 유휴 시간에
import Script from "next/script";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>
        {children}
        <Script src="https://analytics.example.com/a.js" strategy="afterInteractive" />
        <Script src="https://chat.example.com/widget.js" strategy="lazyOnload" />
      </body>
    </html>
  );
}
```

<br>

### 6. 정리 — 면접·실무 체크포인트

| **질문**                                        | **핵심 답변**                                                                                       |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **`next/image`가 하는 일은?**                   | 리사이즈·포맷 변환·지연 로딩·크기 강제로 **LCP와 CLS** 개선. 첫 화면 이미지에는 `priority`          |
| **`next/font`가 CLS를 없애는 원리는?**          | 폰트를 자체 호스팅하고 폴백 폰트에 **`size-adjust`**를 적용해 교체 시 공간 변화를 제거              |
| **코드 스플리팅은 어떻게 이뤄지는가?**          | 라우트 세그먼트별 자동 분할 + `next/dynamic`으로 컴포넌트 단위 지연 로딩 + 서버 컴포넌트는 번들 제외 |
| **`dynamic`의 `ssr: false`는 언제 쓰는가?**     | 브라우저 API에 의존해 서버 렌더링이 불가능한 경우에만. SEO·첫 화면이 필요하면 쓰지 않음             |
| **서드파티 스크립트는 어떻게 다루는가?**        | `next/script`의 `strategy`로 로드 시점을 뒤로 미룸                                                  |
| **번들 분석 도구는?**                           | `@next/bundle-analyzer`로 청크별 구성을 확인한 뒤 서버 컴포넌트 이동·지연 로딩·라이브러리 교체 순으로 대응 |

- 최적화 우선순위는 **LCP 이미지 → 폰트 → 첫 화면 JS → 서드파티 스크립트** 순으로 잡으면 효과가 크다.
- 하이드레이션 비용(unit07)은 클라이언트 번들 크기에 비례하므로, 번들 최적화는 곧 상호작용 시점(TTI·INP) 최적화다.
- 서버·클라이언트 경계 설계는 unit02, 루트 레이아웃 구성은 unit08을 참고할 것.
