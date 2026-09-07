## 웹 보안

프론트엔드가 직접 책임지는 브라우저 측 보안을 다룬다. **XSS**(스크립트 주입), **CSRF**(요청 위조)의 원리와 방어, 브라우저가 실행할 리소스를 제한하는 **CSP**, 그리고 unit06·unit07의 저장소·출처 지식을 종합해 **인증 토큰을 어디에 저장할지 판단**하는 기준을 정리한다.

<br>

### 1. 두 공격의 본질 비교

XSS와 CSRF는 자주 혼동되지만 공격 방향이 정반대다.

| **항목**          | **XSS (Cross-Site Scripting)**                     | **CSRF (Cross-Site Request Forgery)**                  |
| ----------------- | -------------------------------------------------- | ------------------------------------------------------ |
| **공격자가 하는 일** | 피해 사이트 안에서 **자기 스크립트를 실행**시킴 | 피해자의 브라우저로 **피해 사이트에 요청을 보내게** 함 |
| **악용하는 것**   | 사이트가 사용자 입력을 **검증 없이 출력**하는 것   | 브라우저가 쿠키를 **자동으로 첨부**하는 것             |
| **공격자가 얻는 것** | 세션·토큰·화면 데이터 **읽기**, 임의 동작       | 피해자 권한으로 **쓰기**(송금·비밀번호 변경). 응답은 못 읽음 |
| **필요 조건**     | 입력이 스크립트로 해석되는 지점                    | 피해자가 로그인 상태 + 쿠키 기반 인증                  |
| **핵심 방어**     | 출력 이스케이프, CSP                               | `SameSite` 쿠키, CSRF 토큰, Origin 검증                |

> 💡 "XSS가 가능하면 CSRF 방어는 모두 무력화된다." XSS로 같은 출처에서 스크립트가 돌면 CSRF 토큰을 읽어 정상 요청을 만들 수 있기 때문이다. 따라서 우선순위는 **XSS 방어가 먼저**다.

<br>

### 2. XSS

**유형과 발생 지점**

| **유형**            | **주입 위치**                          | **예시**                                                   |
| ------------------- | -------------------------------------- | ---------------------------------------------------------- |
| **저장형(Stored)**  | DB에 저장된 뒤 다른 사용자에게 출력    | 게시글·댓글에 `<script>` 삽입 → 모든 열람자에게 실행       |
| **반사형(Reflected)** | 요청 파라미터가 응답에 그대로 반영   | `?q=<script>…</script>` 링크를 피해자가 클릭               |
| **DOM 기반**        | 서버 무관, 클라이언트 JS가 DOM에 삽입  | `location.hash` 값을 `innerHTML`에 넣음                    |

```javascript
// 안티패턴: 사용자 입력을 HTML로 해석시킴
const q = new URLSearchParams(location.search).get('q');
resultTitle.innerHTML = `"${q}" 검색 결과`;      // q 에 onerror 핸들러가 달린 이미지 태그를 넣으면 document.cookie 가 외부로 전송됨

// 개선 ①: 텍스트로만 삽입 — 태그가 문자 그대로 표시됨
resultTitle.textContent = `"${q}" 검색 결과`;

// 개선 ②: HTML이 꼭 필요하면 신뢰할 수 있는 정제(sanitize) 라이브러리를 거침
resultTitle.innerHTML = DOMPurify.sanitize(userHtml);
```

**방어 원칙**

- **출력 시점 이스케이프**: 데이터를 HTML·속성·URL·JS 문자열 중 **어느 문맥에 넣느냐에 따라** 이스케이프 규칙이 다름. 저장 시점이 아니라 **출력 시점**에 문맥별로 처리함
- **위험한 API 회피**: `innerHTML`, `outerHTML`, `document.write`, `eval`, `new Function`, `setTimeout('문자열')`, `javascript:` URL
- **프레임워크의 자동 이스케이프 신뢰**: 대부분의 뷰 라이브러리는 텍스트 바인딩을 이스케이프함. 이를 **우회하는 API**(`dangerouslySetInnerHTML`, `v-html` 등)는 정제된 HTML에만 사용
- **HttpOnly 쿠키**: 스크립트가 실행되더라도 세션 쿠키 **탈취**는 막음 (단, 해당 세션으로 요청을 보내는 것은 막지 못함)
- **CSP**: 인라인 스크립트와 외부 출처를 제한해 주입된 스크립트의 **실행 자체를 차단** (3절)

❗️**URL 문맥은 별도 검증이 필요하다**: `<a href={userUrl}>`에서 이스케이프를 해도 `javascript:alert(1)`은 그대로 실행된다. 스킴이 `http`·`https`인지 화이트리스트로 검사해야 한다.

<br>

### 3. CSP (Content Security Policy)

CSP는 **"이 페이지에서 어떤 출처의 스크립트·스타일·이미지만 실행·로드할 수 있다"**를 서버가 응답 헤더로 선언하는 정책이다. XSS로 스크립트가 주입되더라도 정책에 맞지 않으면 브라우저가 실행을 거부한다.

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-r4nd0m' https://cdn.example.com;
  object-src 'none';
  base-uri 'none';
  report-to csp-endpoint
```

| **지시어**            | **의미**                                                    |
| --------------------- | ----------------------------------------------------------- |
| **default-src**       | 명시하지 않은 리소스 유형의 기본 허용 출처                  |
| **script-src**        | 스크립트 허용 출처. `'unsafe-inline'`을 쓰면 XSS 방어 효과가 거의 사라짐 |
| **'nonce-…'**         | 서버가 요청마다 생성한 난수. 같은 nonce를 가진 `<script nonce>`만 실행 |
| **'strict-dynamic'**  | nonce로 허용된 스크립트가 동적으로 추가한 스크립트도 신뢰   |
| **object-src 'none'** | 플러그인(`<object>`, `<embed>`) 차단                        |
| **base-uri 'none'**   | `<base>` 태그로 상대 URL 기준을 바꾸는 공격 차단            |
| **report-to / report-uri** | 위반 내역을 지정한 엔드포인트로 보고                   |

```html
<!-- nonce가 일치하는 인라인 스크립트만 실행됨. 주입된 <script>는 nonce를 모르므로 차단 -->
<script nonce="r4nd0m">
  window.__CONFIG__ = { apiBase: '/api' };
</script>
```

> ⚠️ CSP를 처음 도입하면 기존 인라인 스크립트·이벤트 핸들러 속성(`onclick="…"`)·서드파티 위젯이 대거 차단된다. `Content-Security-Policy-Report-Only` 헤더로 **차단 없이 위반 보고만 먼저 수집**한 뒤, 정책을 다듬고 나서 강제 모드로 전환하는 것이 정석이다.

<br>

### 4. CSRF

### 4-1. 공격 시나리오

```
① 피해자가 bank.com에 로그인 (세션 쿠키 보유)
② 공격자 사이트 evil.com 방문
   <form action="https://bank.com/transfer" method="POST">
     <input type="hidden" name="to" value="attacker">
     <input type="hidden" name="amount" value="1000000">
   </form>
   <script>document.forms[0].submit()</script>
③ 브라우저가 bank.com 쿠키를 자동 첨부해 POST 전송
④ bank.com은 정상 로그인 사용자의 요청으로 처리
```

공격자는 응답을 읽을 수 없지만(unit07의 SOP) **요청 자체는 도착해 실행**된다. 폼 제출은 CORS 프리플라이트 대상도 아니다.

<br>

### 4-2. 방어 기법

| **기법**                        | **원리**                                                             | **한계·주의**                                       |
| ------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------- |
| **SameSite 쿠키**               | 크로스 사이트 요청에 쿠키를 붙이지 않음 (`Lax` 이상)                 | 같은 사이트 내 서브도메인 XSS에는 무력. 구형 브라우저 |
| **CSRF 토큰(Synchronizer)**     | 서버가 폼·페이지마다 난수를 심고 요청 시 검증. 공격자는 값을 모름    | SPA에서는 토큰 전달 채널 설계 필요                  |
| **Double Submit Cookie**        | 토큰을 쿠키와 요청 본문(헤더)에 동시에 보내 일치 여부 확인            | 서브도메인에서 쿠키를 심을 수 있으면 우회 가능      |
| **Origin / Referer 검증**       | 요청 헤더의 출처가 허용 목록인지 확인                                | `Referer`는 누락될 수 있어 `Origin` 우선            |
| **커스텀 헤더 요구**            | `X-Requested-With` 등 → 단순 요청이 아니게 되어 프리플라이트 발생    | 폼 제출은 막지만 CORS 설정 오류가 있으면 무의미     |
| **상태 변경에 GET 금지**        | 이미지 태그·링크만으로 부작용이 생기지 않게 함                        | 기본 원칙, 단독으로는 불충분                        |

```typescript
// 서버: Origin 검증 + SameSite 쿠키 (이중 방어)
app.use((req, res, next) => {
  if (['POST', 'PUT', 'DELETE', 'PATCH'].includes(req.method)) {
    const origin = req.headers.origin ?? new URL(req.headers.referer ?? '', 'https://x').origin;
    if (origin !== 'https://app.example.com') return res.status(403).end();
  }
  next();
});
res.cookie('sid', sessionId, { httpOnly: true, secure: true, sameSite: 'lax' });
```

> 💡 현대적인 기본 조합은 **`SameSite=Lax` 쿠키 + Origin 헤더 검증**이며, 결제처럼 민감한 동작에는 **CSRF 토큰**을 추가한다. "SameSite만으로 충분한가"라는 질문에는 "대부분 막지만 서브도메인 탈취·구형 브라우저 때문에 서버 측 검증을 병행한다"고 답한다.

<br>

### 5. 토큰 저장 위치 판단

인증 토큰(세션 ID, JWT)을 어디에 두느냐는 **XSS와 CSRF 중 어느 위험에 노출될지**를 고르는 문제다.

```
                       JS에서 읽힘?        자동 첨부?
LocalStorage            ○ (XSS 탈취)        ✗ (CSRF 안전)
HttpOnly 쿠키           ✗ (XSS 탈취 불가)    ○ (CSRF 노출 → SameSite로 방어)
메모리(JS 변수)         △ (실행 중엔 접근 가능, 새로고침 시 소실)  ✗
```

| **저장 위치**                       | **XSS 탈취** | **CSRF**             | **새로고침 유지** | **판단**                                          |
| ----------------------------------- | ------------ | -------------------- | ----------------- | ------------------------------------------------- |
| **LocalStorage**                    | **가능**     | 안전                 | 유지              | 서드파티 스크립트가 하나라도 있으면 비권장         |
| **HttpOnly + Secure + SameSite 쿠키** | 불가       | `SameSite`로 방어    | 유지              | **기본 권장**. 서버 세션·JWT 모두 가능            |
| **메모리 + 리프레시 토큰은 HttpOnly 쿠키** | 액세스 토큰만 짧게 노출 | 리프레시 엔드포인트만 방어 | 재발급으로 복구 | SPA + 별도 API 서버 구성에서 널리 쓰임    |

**판단 순서**

- 프론트와 API가 **같은 사이트**(서브도메인 포함)인가 → `HttpOnly; SameSite=Lax` 쿠키로 충분. 가장 단순하고 안전
- **사이트가 다른** API를 호출해야 하는가 → 쿠키 방식은 `SameSite=None` + CORS credentials 설정(unit07)이 필요. 복잡하다면 메모리 + 리프레시 쿠키 조합
- 모바일 앱·서드파티 클라이언트와 API를 공유하는가 → 헤더 기반 토큰이 필요하지만, 웹에서는 여전히 LocalStorage 대신 메모리에 보관
- 어떤 방식이든 **액세스 토큰 수명은 짧게**(분 단위), 탈취 시 피해 범위를 줄임

❗️**"JWT는 LocalStorage에 넣는 것"이 정답이 아니다**: JWT는 전달 형식일 뿐 저장 위치와 무관하다. JWT를 HttpOnly 쿠키에 담아 전송해도 서버는 서명만 검증하면 되므로 무상태(stateless) 이점은 그대로 유지된다.

<br>

### 6. 정리

| **위협**  | **악용 대상**            | **1차 방어**                         | **2차 방어**                          |
| --------- | ------------------------ | ------------------------------------ | ------------------------------------- |
| **XSS**   | 검증 없는 출력           | 문맥별 이스케이프, `textContent`     | CSP(nonce), HttpOnly 쿠키, 정제 라이브러리 |
| **CSRF**  | 쿠키 자동 첨부           | `SameSite=Lax` 이상                  | Origin 검증, CSRF 토큰                |
| **토큰 탈취** | JS가 읽을 수 있는 저장소 | HttpOnly 쿠키 또는 메모리          | 짧은 수명, 리프레시 토큰 회전         |

- XSS는 **읽기**(탈취), CSRF는 **쓰기**(위조) 공격이며, XSS가 뚫리면 CSRF 방어도 무너지므로 XSS 방어가 우선임
- XSS 방어의 핵심은 **출력 문맥별 이스케이프**와 `innerHTML` 계열 회피이며, CSP는 주입된 스크립트의 실행을 최종 차단함
- CSP는 `Report-Only`로 시작해 위반을 수집한 뒤 강제 모드로 전환함
- CSRF는 `SameSite` 쿠키 + Origin 검증을 기본으로 하고 민감 동작에 CSRF 토큰을 더함
- 토큰 저장은 **XSS 노출(LocalStorage) vs CSRF 노출(쿠키)**의 선택이며, 기본 권장은 `HttpOnly; Secure; SameSite` 쿠키임
- 각 공격의 상세 유형과 서버 측 방어는 web-security 챕터를 참고할 것
