## SOP와 CORS

**동일 출처 정책(SOP)**은 한 출처의 스크립트가 다른 출처의 자원에 함부로 접근하지 못하게 막는 브라우저의 기본 보안 모델이다. **CORS**는 이 제약을 안전하게 완화해 정당한 교차 출처 요청을 허용하는 표준이다. 프론트엔드-백엔드 협업에서 필수 개념이며 면접 단골 주제다.

<br>

### 1. 동일 출처 정책(SOP)

**SOP**(Same-Origin Policy)는 **같은 출처에서 온 자원끼리만** 상호작용을 허용하는 브라우저 정책이다. 다른 출처의 응답을 자바스크립트로 읽는 행위를 기본적으로 차단한다.

<br>

### 2. 출처(Origin)의 정의

출처는 **프로토콜 + 호스트 + 포트** 세 가지가 모두 같아야 동일하다고 판정한다.

```
https://www.example.com:443
└─프로토콜─┘ └───호스트───┘└포트┘
```

<br>

### 2-1. 출처 비교 예시

기준 URL `https://example.com` 과의 비교다.

| **비교 대상 URL**          | **판정**   | **이유**              |
| -------------------------- | ---------- | --------------------- |
| `https://example.com/page` | **같은 출처** | 경로만 다름 (무관)    |
| `http://example.com`       | **다른 출처** | 프로토콜 다름         |
| `https://api.example.com`  | **다른 출처** | 호스트(서브도메인) 다름 |
| `https://example.com:8080` | **다른 출처** | 포트 다름             |

> 💡 경로(path)나 쿼리 스트링은 출처 판정에 **영향을 주지 않는다**. 오직 프로토콜·호스트·포트 세 가지만 비교한다.

<br>

### 3. SOP가 필요한 이유

SOP가 없다면 악성 사이트의 스크립트가 사용자가 로그인한 다른 사이트에 요청을 보내고 그 **응답까지 읽어낼** 수 있다. 이는 CSRF를 넘어선 데이터 탈취로 이어진다. SOP는 이런 교차 출처 정보 유출을 원천 차단한다.

<br>

### 4. CORS의 필요성

현대 웹은 프론트엔드(`www.example.com`)와 API 서버(`api.example.com`)의 **출처가 다른 경우**가 흔하다. 이런 **정당한 교차 출처 요청**까지 SOP가 막으면 서비스가 동작하지 않는다. **CORS**(Cross-Origin Resource Sharing)는 서버가 특정 출처의 요청을 명시적으로 허용하도록 하는 표준 메커니즘이다.

<br>

### 5. 단순 요청 vs 사전 요청(Preflight)

CORS 요청은 조건에 따라 두 가지로 나뉜다.

| **구분**            | **조건**                                                        |
| ------------------- | --------------------------------------------------------------- |
| **단순 요청(Simple)** | GET·POST·HEAD + 표준 헤더 + 제한된 Content-Type                 |
| **사전 요청(Preflight)** | 위 조건을 벗어남 (PUT·DELETE, 커스텀 헤더, JSON 본문 등)      |

사전 요청은 실제 요청 전에 브라우저가 **OPTIONS 메서드**로 서버에 허용 여부를 먼저 물어본다.

<br>

### 5-1. Preflight 흐름

```
[브라우저]                                   [서버]
    │                                          │
    │──── OPTIONS 요청 (사전 확인) ───────────▶│
    │      Origin: https://www.example.com     │
    │      Access-Control-Request-Method: PUT   │
    │                                          │
    │◀─── 응답 (허용 정책) ────────────────────│
    │      Access-Control-Allow-Origin: ...     │
    │      Access-Control-Allow-Methods: PUT    │
    │                                          │
    │  (브라우저가 허용 확인 후)                 │
    │──── 실제 PUT 요청 ──────────────────────▶│
    │◀─── 실제 응답 ───────────────────────────│
```

<br>

### 6. 주요 CORS 응답 헤더

| **헤더**                            | **역할**                                     |
| ----------------------------------- | -------------------------------------------- |
| **Access-Control-Allow-Origin**     | 허용할 출처 지정 (`*` 또는 특정 출처)         |
| **Access-Control-Allow-Methods**    | 허용할 HTTP 메서드                            |
| **Access-Control-Allow-Headers**    | 허용할 요청 헤더                              |
| **Access-Control-Allow-Credentials** | 인증 정보(쿠키) 포함 요청 허용 여부          |
| **Access-Control-Max-Age**          | Preflight 결과 캐싱 시간                      |

<br>

### 7. 서버 설정 예시

서버가 응답에 아래 헤더를 담아 특정 출처를 허용한다.

```http
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
```

<br>

### 8. 자주 하는 오해와 주의점

> ⚠️ CORS 오류는 **서버가 요청을 막는 것이 아니다**. 서버는 응답을 정상적으로 보내지만, 허용 헤더가 없으면 **브라우저가 그 응답을 자바스크립트에 넘겨주지 않고 차단**하는 것이다. 그래서 서버 로그에는 요청이 성공으로 찍히는데 프론트에서만 에러가 나는 상황이 생긴다. 이 원리는 면접 단골 질문이다.

> 💡 **credentials(쿠키 등)를 포함하는 요청**에서는 `Access-Control-Allow-Origin`에 와일드카드 `*`를 쓸 수 없다. 반드시 **구체적인 출처**를 명시해야 하며, `Access-Control-Allow-Credentials: true`도 함께 있어야 한다. 보안상 "모두 허용 + 인증 정보 전달"을 막기 위한 제약이다.

❗️**핵심 요약**: SOP는 브라우저의 기본 방어벽이고, CORS는 서버가 신뢰하는 출처에만 문을 열어주는 열쇠다.
