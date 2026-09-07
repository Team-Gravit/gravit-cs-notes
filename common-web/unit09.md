## HTTP 캐싱

같은 리소스를 매번 서버에서 받아오면 느리고 비싸다. HTTP 캐싱은 **브라우저·CDN이 응답을 저장해 두었다가 재사용하는 규칙**이며, `Cache-Control`로 신선도를 정하고 `ETag`·`Last-Modified`로 조건부 요청을 수행한다. "배포했는데 사용자 화면이 안 바뀐다"는 문제는 대부분 캐시 무효화 전략의 부재에서 나오므로, 캐시 수명과 무효화를 설계하는 기준을 알아야 한다.

<br>

### 1. 캐시가 위치하는 곳과 판정 흐름

캐시는 브라우저 안(메모리·디스크)과 브라우저 밖(CDN·리버스 프록시)에 있다. 브라우저가 리소스를 요청할 때의 판정 흐름은 다음과 같다.

```
요청 발생
  │
  ├─ 캐시에 없음 ─────────────────────────────▶ 서버 요청 (200, 본문 전송)
  │
  └─ 캐시에 있음
       ├─ 신선함(fresh: max-age 안 지남) ────▶ 캐시 사용, 네트워크 요청 없음 (200 from cache)
       │
       └─ 오래됨(stale)
            └─ 조건부 요청 (If-None-Match / If-Modified-Since)
                 ├─ 변하지 않음 ─▶ 304 Not Modified (본문 없음) → 캐시 재사용, 신선도 갱신
                 └─ 변함 ────────▶ 200 + 새 본문 → 캐시 교체
```

| **캐시 위치**            | **공유 범위**             | **제어 헤더**                        |
| ------------------------ | ------------------------- | ------------------------------------ |
| **브라우저 캐시**        | 사용자 한 명 (**private**) | `Cache-Control: max-age`, `private`  |
| **CDN·프록시 캐시**      | 여러 사용자 (**shared**)  | `Cache-Control: s-maxage`, `public`  |
| **서비스 워커 Cache API** | 사용자 한 명, JS로 직접 제어 | 코드로 정의 (HTTP 헤더와 별개)    |

> 💡 캐시의 두 축은 **신선도(freshness)**와 **검증(validation)**이다. 신선한 동안은 서버에 묻지 않고, 오래되면 "바뀌었나요?"만 묻는다. `Cache-Control`이 전자를, `ETag`·`Last-Modified`가 후자를 담당한다.

<br>

### 2. Cache-Control 지시어

| **지시어**                 | **의미**                                                              | **주의**                                        |
| -------------------------- | --------------------------------------------------------------------- | ----------------------------------------------- |
| **max-age=N**              | 응답 후 N초 동안 신선함                                               | `Expires`보다 우선                              |
| **s-maxage=N**             | **공유 캐시(CDN)**에서의 max-age. 브라우저는 무시                     | CDN은 길게, 브라우저는 짧게 줄 때 사용          |
| **no-cache**               | 저장은 하되 **사용 전 반드시 재검증**(조건부 요청)                     | "캐시하지 않음"이 **아님**                      |
| **no-store**               | **아예 저장하지 않음**                                                | 개인정보·결제 응답                              |
| **private**                | 브라우저만 저장, CDN·프록시는 저장 금지                               | 사용자별 응답(마이페이지)                       |
| **public**                 | 인증 헤더가 있어도 공유 캐시 저장 허용                                | 정적 자산                                       |
| **must-revalidate**        | 오래되면 반드시 원 서버에 재검증 (네트워크 불가 시 오래된 응답 금지)  | 금융 데이터                                     |
| **immutable**              | 신선한 동안 재검증 요청 자체를 생략 (새로고침 시에도)                 | 해시 파일명과 함께 사용                         |
| **stale-while-revalidate=N** | 만료 후 N초 동안 **오래된 응답을 먼저 주고 백그라운드로 갱신**      | unit05의 ISR과 같은 원리                        |

```http
Cache-Control: no-cache
```

위 헤더는 "캐시 금지"가 아니라 "매번 확인하고 써라"이다. 실제로 저장을 막으려면 `no-store`를 써야 한다.

> ⚠️ `Cache-Control`이 전혀 없으면 브라우저는 **휴리스틱 캐싱**을 적용한다. 보통 `Last-Modified`로부터 지난 시간의 10% 정도를 신선 기간으로 추정하므로, "헤더를 안 줬으니 캐시 안 되겠지"라는 기대는 틀린다. 캐시하면 안 되는 응답에는 **명시적으로** `no-store`를 준다.

<br>

### 3. 조건부 요청 — ETag와 Last-Modified

신선 기간이 지난 리소스는 버리지 않고 **"내가 가진 버전이 아직 유효한가"**를 묻는다. 유효하면 서버는 본문 없이 `304`만 돌려주므로 전송량이 크게 준다.

<br>

### 3-1. ETag / If-None-Match

`ETag`는 리소스 버전을 나타내는 **식별자**(보통 내용의 해시)다. 응답에 `ETag`가 오면 브라우저는 다음 요청에 `If-None-Match`로 그 값을 보낸다.

```http
GET /api/products/1
If-None-Match: "a1b2c3"
```

```http
HTTP/1.1 304 Not Modified
ETag: "a1b2c3"
Cache-Control: max-age=60
```

- **강한 ETag** `"a1b2c3"`: 바이트 단위로 동일함을 보장. Range 요청에도 사용 가능
- **약한 ETag** `W/"a1b2c3"`: 의미상 동일함만 보장(공백·압축 차이 허용). gzip 적용 시 서버가 자동으로 약한 ETag로 바꾸는 경우가 있음

<br>

### 3-2. Last-Modified / If-Modified-Since

파일의 **마지막 수정 시각**으로 비교한다. 구현이 쉽지만 정밀도가 **1초**라서 1초 안에 두 번 바뀌면 감지하지 못하고, 내용이 같아도 시각이 바뀌면(재배포) 불필요하게 전체를 다시 보낸다. 둘 다 있으면 서버는 `ETag`를 우선 비교한다.

```typescript
// 서버: ETag 생성과 조건부 요청 처리
import { createHash } from 'node:crypto';

app.get('/api/products/:id', async (req, res) => {
  const body = JSON.stringify(await db.findProduct(req.params.id));
  const etag = `"${createHash('sha1').update(body).digest('hex').slice(0, 16)}"`;

  res.setHeader('ETag', etag);
  res.setHeader('Cache-Control', 'private, no-cache');    // 매번 검증하되 304로 절약
  if (req.headers['if-none-match'] === etag) return res.status(304).end();
  res.type('application/json').send(body);
});
```

> 💡 `no-cache` + `ETag` 조합은 "항상 최신을 보장하면서도 전송량은 줄이는" API 응답의 표준 패턴이다. 반면 `max-age`를 주면 그 시간 동안은 검증 요청조차 하지 않으므로, **"얼마나 오래된 데이터를 허용할 수 있는가"**가 곧 `max-age` 값이다.

<br>

### 4. 캐시 무효화 전략

캐시의 어려움은 저장이 아니라 **"바뀌었을 때 어떻게 새 것을 받게 하느냐"**다. 브라우저 캐시는 서버가 강제로 지울 수 없으므로 **URL을 바꾸는 것**이 유일하게 확실한 방법이다.

**해시 파일명 + immutable (정적 자산)**

```
빌드 결과
  app.3f9a2c.js      ← 내용이 바뀌면 해시가 바뀌어 URL 자체가 달라짐
  vendor.7b1e44.js
  index.html         ← 위 파일들을 참조. 항상 최신이어야 함
```

```http
GET /assets/app.3f9a2c.js
Cache-Control: public, max-age=31536000, immutable

GET /index.html
Cache-Control: no-cache
```

- 정적 자산은 **1년 + immutable**: 내용이 바뀌면 파일명이 바뀌므로 오래된 캐시가 재사용될 일이 없음
- HTML은 **no-cache**: 매번 검증해 새 해시 파일명을 가리키는 최신 HTML을 받게 함
- 이 조합이 SPA·정적 사이트 배포의 표준이며, 대부분의 번들러가 해시 파일명을 기본으로 생성함

**그 밖의 방법과 함정**

| **방법**                        | **효과**                                       | **한계**                                             |
| ------------------------------- | ---------------------------------------------- | ---------------------------------------------------- |
| **쿼리 스트링 버전** `?v=2`     | URL이 바뀌어 새로 받음                         | 일부 프록시가 쿼리 있는 URL을 캐시하지 않거나 무시함 |
| **CDN 퍼지(Purge)**             | CDN 캐시를 즉시 삭제                           | **브라우저 캐시는 지우지 못함**                      |
| **짧은 max-age + ETag**         | 갱신 지연을 max-age 이내로 제한                | 요청 수 증가                                         |
| **stale-while-revalidate**      | 오래된 응답을 즉시 주고 백그라운드 갱신        | 첫 요청자는 오래된 데이터를 봄                       |

```
안티패턴: app.js 를 고정 URL로 배포 + max-age=86400
  → 배포 후 최대 하루 동안 사용자는 옛 JS + 새 HTML 조합으로 오류 발생

개선: app.[hash].js + max-age=31536000, immutable / index.html + no-cache
  → HTML만 검증하면 항상 올바른 JS 조합을 받음
```

❗️**HTML에 긴 max-age를 주면 안 된다**: HTML이 하루 동안 캐시되면 새 해시 파일명을 알 방법이 없어 배포가 하루 늦게 반영된다. HTML은 `no-cache`(또는 아주 짧은 `max-age`)가 원칙이다.

<br>

### 5. Vary와 캐시 키

캐시는 기본적으로 **URL을 키**로 저장한다. 같은 URL에 대해 `Accept-Encoding`(gzip 여부), `Accept-Language`, `Origin`(unit07의 CORS)에 따라 응답이 달라진다면 `Vary` 헤더로 **해당 요청 헤더도 키에 포함**시켜야 한다.

```http
Vary: Accept-Encoding, Origin
```

```
Vary 없이 CORS 응답을 캐시했을 때
  ① https://a.com 의 요청 → 응답 Allow-Origin: https://a.com  → CDN에 저장
  ② https://b.com 의 요청 → CDN이 ①의 응답을 그대로 반환 → Allow-Origin 불일치 → CORS 에러
```

> ⚠️ `Vary: *`나 `Vary: Cookie`, `Vary: User-Agent`처럼 값의 조합이 무수히 많은 헤더를 지정하면 캐시 적중률이 0에 가까워진다. 사용자별 응답이라면 `Vary`가 아니라 `Cache-Control: private`이 올바른 선택이다.

<br>

### 6. 정리

| **리소스 유형**                    | **권장 헤더**                                        | **이유**                                   |
| ---------------------------------- | ---------------------------------------------------- | ------------------------------------------ |
| **해시 파일명 정적 자산**          | `public, max-age=31536000, immutable`                | URL이 곧 버전, 재검증 불필요               |
| **HTML 진입점**                    | `no-cache` (+ `ETag`)                                | 항상 최신 자산 목록을 가리켜야 함          |
| **공개 API 응답 (약간의 지연 허용)** | `public, max-age=60, stale-while-revalidate=300`   | 빠른 응답 + 백그라운드 갱신                |
| **사용자별 API 응답**              | `private, no-cache` (+ `ETag`)                       | 공유 캐시 금지, 304로 전송량 절약          |
| **민감 정보 (결제·개인정보)**      | `no-store`                                           | 저장 자체를 금지                           |

- 캐시는 **신선도(`Cache-Control`)**와 **검증(`ETag`·`Last-Modified` → 304)** 두 축으로 동작함
- `no-cache`는 "매번 검증", `no-store`는 "저장 금지"이며, 헤더가 없으면 **휴리스틱 캐싱**이 적용됨
- 브라우저 캐시는 서버가 지울 수 없으므로 무효화는 **URL 변경(해시 파일명)**이 정석이고, HTML은 `no-cache`로 둠
- `s-maxage`·`public`·`private`으로 CDN과 브라우저의 캐시 정책을 분리함
- 응답이 요청 헤더에 따라 달라지면 `Vary`로 캐시 키를 확장하되, 조합이 폭발하는 헤더는 피함
- 캐시 설계는 unit10의 **LCP·TTFB** 개선에 가장 비용 대비 효과가 큰 수단임
