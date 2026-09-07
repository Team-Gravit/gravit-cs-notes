## 동일 출처 정책과 CORS

브라우저는 **동일 출처 정책(Same-Origin Policy, SOP)**으로 다른 출처의 응답을 스크립트가 읽지 못하게 막고, **CORS(Cross-Origin Resource Sharing)**는 서버가 헤더로 "이 출처는 읽어도 된다"고 허용하는 표준 절차다. 프론트엔드 개발자가 가장 자주 마주치는 에러이므로, 프리플라이트가 언제 발생하는지, `credentials`가 무엇을 요구하는지, 그리고 **우회와 정식 해결을 구분**할 수 있어야 한다.

<br>

### 1. 출처(Origin)와 동일 출처 정책

**출처**는 `스킴(프로토콜) + 호스트 + 포트`의 조합이다. 셋 중 하나라도 다르면 다른 출처다.

| **비교 대상 (기준: `https://app.example.com`)** | **동일 출처?** | **이유**            |
| ----------------------------------------------- | -------------- | ------------------- |
| **`https://app.example.com/users`**             | **동일**       | 경로는 무관         |
| **`http://app.example.com`**                    | 다름           | 스킴이 다름         |
| **`https://api.example.com`**                   | 다름           | 호스트(서브도메인)가 다름 |
| **`https://app.example.com:8443`**              | 다름           | 포트가 다름         |

SOP는 **"다른 출처의 리소스를 가져오는 것"**을 막는 것이 아니라, **"다른 출처의 응답을 스크립트가 읽는 것"**을 막는다. 이 구분이 핵심이다.

- 허용: `<script src>`, 이미지, CSS, `<iframe>` 표시, 폼 제출 — **가져와서 브라우저가 쓰는 것**
- 차단: `fetch`·`XMLHttpRequest`로 받은 **응답 본문 읽기**, 다른 출처 `iframe`의 DOM 접근, 다른 출처의 LocalStorage·쿠키 읽기

> 💡 SOP는 **브라우저**가 강제하는 정책이다. `curl`이나 서버 간 통신에는 CORS가 존재하지 않는다. "서버에서는 되는데 브라우저에서만 안 된다"면 거의 확실히 CORS 문제이며, 서버 로그에 요청이 찍혀 있다면 **요청은 도착했고 브라우저가 응답 읽기를 거부한 것**이다.

<br>

### 2. CORS 동작 원리

CORS는 브라우저가 요청에 `Origin` 헤더를 붙여 보내고, 서버가 응답에 `Access-Control-Allow-Origin`으로 허용 여부를 알려주는 **헤더 기반 협상**이다. 검사는 브라우저가 수행한다.

```
브라우저 (https://app.com)                          서버 (https://api.io)
────────────────────────                          ─────────────────────
GET /users
Origin: https://app.com          ──────────────▶   요청 처리
                                                   200 OK
                                 ◀──────────────   Access-Control-Allow-Origin: https://app.com
검사: 응답의 Allow-Origin에 내 출처가 있는가?
  ○ 있음 → JS에 응답 전달
  ✗ 없음 → 응답 폐기, 콘솔에 CORS 에러 (요청은 이미 서버에서 실행됨!)
```

| **응답 헤더**                           | **의미**                                                   |
| --------------------------------------- | ---------------------------------------------------------- |
| **Access-Control-Allow-Origin**         | 허용할 출처. 특정 출처 하나 또는 `*`                        |
| **Access-Control-Allow-Methods**        | 프리플라이트 응답에서 허용할 메서드 목록                   |
| **Access-Control-Allow-Headers**        | 프리플라이트 응답에서 허용할 요청 헤더 목록                |
| **Access-Control-Allow-Credentials**    | `true`면 쿠키·인증 헤더가 포함된 요청을 허용               |
| **Access-Control-Expose-Headers**       | JS가 읽을 수 있는 응답 헤더 추가 (기본은 `Content-Type` 등 소수만) |
| **Access-Control-Max-Age**              | 프리플라이트 결과 캐시 시간(초). 브라우저별 상한이 있음    |

<br>

### 3. 단순 요청과 프리플라이트

### 3-1. 단순 요청(Simple Request)의 조건

다음 조건을 **모두** 만족하면 브라우저는 프리플라이트 없이 본 요청을 바로 보낸다.

- 메서드가 `GET`, `HEAD`, `POST` 중 하나
- 수동으로 설정한 헤더가 `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` 등 **CORS-안전 목록**에 속함
- `Content-Type`이 `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain` 중 하나

이 조건은 "CORS가 생기기 전부터 HTML 폼으로 보낼 수 있던 요청"과 같다. 즉 단순 요청은 **어차피 막을 수 없었던 요청**이라 프리플라이트가 없다.

> ⚠️ 단순 요청은 **서버에 도달해 실행된다**. `Content-Type: text/plain`으로 보낸 POST는 CORS 헤더가 없어도 서버가 처리하고, 브라우저는 응답만 숨긴다. 따라서 CORS는 **쓰기(부작용) 방어 수단이 아니며**, CSRF 방어는 별도로 해야 한다(unit08).

<br>

### 3-2. 프리플라이트(Preflight) 요청

단순 요청 조건을 벗어나면 브라우저가 본 요청 전에 `OPTIONS` 요청을 먼저 보내 **서버의 허락을 확인**한다. `Content-Type: application/json`, `Authorization` 헤더, `PUT`·`DELETE` 메서드가 대표적인 발생 원인이다.

```
① 프리플라이트
OPTIONS /users/1
Origin: https://app.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: content-type, authorization
                                          ──────────▶
                                          ◀──────────
204 No Content
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 600

② 허락 확인 후 본 요청
PUT /users/1
Content-Type: application/json
Authorization: Bearer ...                  ──────────▶  실제 처리
```

**프리플라이트 실패의 흔한 원인**

- 서버가 `OPTIONS` 메서드에 라우트가 없어 `404`·`405`를 돌려줌 (프리플라이트는 **2xx**여야 함)
- `OPTIONS` 요청이 인증 미들웨어에 걸려 `401`을 돌려줌 (프리플라이트에는 쿠키·인증 헤더가 **붙지 않음**)
- 커스텀 헤더(`X-Request-Id` 등)를 `Allow-Headers`에 빠뜨림
- 프리플라이트 응답이 리다이렉트됨 (허용되지 않음)

> 💡 프리플라이트는 요청마다 왕복(RTT)을 한 번 더 소비한다. `Access-Control-Max-Age`로 결과를 캐시하면 같은 URL·메서드·헤더 조합에 대해 재사용된다. 상한은 브라우저마다 다르므로(Chrome은 2시간 수준) 큰 값을 넣어도 그 이상 캐시되지 않는다.

<br>

### 4. credentials — 쿠키·인증 정보를 포함하는 요청

`fetch`는 기본적으로 **같은 출처에만** 쿠키를 보낸다(`credentials: 'same-origin'`). 다른 출처의 API에 쿠키를 보내려면 클라이언트와 서버 **양쪽 모두** 명시적으로 허용해야 한다.

```typescript
// 클라이언트: 크로스 출처 요청에 쿠키를 포함
const res = await fetch('https://api.io/me', {
  credentials: 'include',            // XHR의 withCredentials = true 에 해당
});
```

```http
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Credentials: true
Vary: Origin
```

**credentials 요청에서 적용되는 추가 규칙**

| **항목**                                | **credentials 없는 요청** | **credentials 있는 요청**                 |
| --------------------------------------- | ------------------------- | ----------------------------------------- |
| **Allow-Origin에 `*` 사용**             | 가능                      | **불가** — 출처를 명시해야 함             |
| **Allow-Headers / Allow-Methods에 `*`** | 가능                      | **불가** — 각 값을 나열해야 함            |
| **Allow-Credentials 헤더**              | 불필요                    | **`true` 필수**                           |
| **쿠키의 SameSite**                     | 무관                      | 사이트가 다르면 **`SameSite=None; Secure`** 필요 (unit06) |

❗️**`*`와 credentials를 동시에 쓰면 무조건 실패한다**: "쿠키가 안 붙어요"의 절반은 `Allow-Origin: *` 때문이고, 나머지 절반은 `SameSite` 때문이다. 여러 출처를 허용해야 한다면 요청의 `Origin` 값을 **허용 목록과 대조한 뒤** 그 값을 그대로 돌려주고 `Vary: Origin`을 붙인다.

<br>

### 5. 우회와 정식 해결의 구분

CORS 에러를 "없애는" 방법은 많지만, 그중 일부는 문제를 숨길 뿐이거나 보안을 무너뜨린다.

| **방법**                                        | **분류**       | **판단**                                                              |
| ----------------------------------------------- | -------------- | --------------------------------------------------------------------- |
| **서버가 정확한 CORS 헤더를 응답**              | **정식 해결**  | 표준 방식. 허용 출처를 화이트리스트로 관리                            |
| **리버스 프록시로 같은 출처에 배치**            | **정식 해결**  | `/api`를 같은 출처에서 서빙하면 CORS 자체가 발생하지 않음             |
| **BFF(Backend for Frontend) 서버 경유**         | **정식 해결**  | 프론트 전용 서버가 외부 API를 호출. 비밀 키를 브라우저에 두지 않아도 됨 |
| **개발 서버의 proxy 설정**                      | 개발용 우회    | 로컬에서만 유효. 배포 환경 해결책은 따로 필요                         |
| **`Allow-Origin`에 요청 Origin을 무조건 반영**  | **위험한 우회** | 모든 출처 허용 + credentials 허용 = 아무 사이트나 사용자 데이터 열람 가능 |
| **공개 CORS 프록시 서비스 경유**                | 위험한 우회    | 요청·응답이 제3자를 거침. 인증 정보 유출                              |
| **브라우저 보안 플래그 해제, 확장 프로그램**    | 우회 아님      | 개발자 PC에서만 동작. 사용자 환경과 무관                              |

```typescript
// 안티패턴: Origin을 검증 없이 반영 → 사실상 SOP 해제
res.setHeader('Access-Control-Allow-Origin', req.headers.origin ?? '*');
res.setHeader('Access-Control-Allow-Credentials', 'true');

// 개선: 화이트리스트와 대조한 뒤에만 반영
const ALLOWED = new Set(['https://app.com', 'https://admin.app.com']);
const origin = req.headers.origin;
if (origin && ALLOWED.has(origin)) {
  res.setHeader('Access-Control-Allow-Origin', origin);
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  res.setHeader('Vary', 'Origin');                 // 캐시가 출처별로 응답을 구분하도록
}
```

```
[리버스 프록시로 같은 출처 만들기]
브라우저 ──▶ https://app.com/          ──▶ 정적 파일 서버
브라우저 ──▶ https://app.com/api/users ──▶ (nginx 등이 내부로 전달) ──▶ http://api-internal:8080/users
             ▲ 브라우저 입장에서는 모두 같은 출처 → CORS 없음
```

> ⚠️ `Vary: Origin`을 빠뜨리면 CDN·브라우저 캐시가 `Allow-Origin: https://a.com`이 박힌 응답을 `https://b.com` 요청에 그대로 돌려줘 **간헐적으로만 실패하는** CORS 에러가 생긴다. 캐시 키와 `Vary`의 관계는 unit09 참고.

<br>

### 6. 정리

| **증상**                                     | **원인**                                            | **해결**                                           |
| -------------------------------------------- | --------------------------------------------------- | -------------------------------------------------- |
| **응답은 200인데 콘솔에 CORS 에러**          | `Allow-Origin` 누락 또는 출처 불일치                | 서버에 정확한 출처 명시                            |
| **OPTIONS 요청이 실패**                      | `OPTIONS` 라우트 없음, 인증 미들웨어가 차단         | 프리플라이트를 인증 전에 2xx로 응답                |
| **쿠키가 전송되지 않음**                     | `credentials: 'include'` 누락, `Allow-Origin: *`, `SameSite` | 양쪽 설정 + `SameSite=None; Secure`       |
| **가끔만 실패**                              | `Vary: Origin` 누락으로 캐시 오염                   | `Vary: Origin` 추가                                |
| **커스텀 응답 헤더를 JS에서 못 읽음**        | `Expose-Headers` 누락                               | 읽어야 할 헤더를 `Expose-Headers`에 나열           |

- 출처는 **스킴 + 호스트 + 포트**이며, SOP는 가져오기가 아니라 **스크립트의 응답 읽기**를 막음
- CORS는 브라우저가 검사하는 **헤더 협상**이고, 서버 간 통신에는 존재하지 않음
- 폼으로 보낼 수 있던 요청은 **단순 요청**으로 바로 전송되며(서버에 도달함), 그 외는 **프리플라이트**를 거침
- credentials 요청은 `Allow-Origin`에 `*`를 쓸 수 없고 `Allow-Credentials: true`와 쿠키 `SameSite` 설정이 필요함
- 정식 해결은 **서버 CORS 헤더·리버스 프록시·BFF**이며, Origin 무조건 반영과 공개 프록시는 보안을 무너뜨리는 우회임
