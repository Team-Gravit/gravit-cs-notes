## 웹 접속 과정과 데이터 흐름

### 1. URL의 구성

**URL**(Uniform Resource Locator)은 인터넷 상의 자원에 접근하기 위한 주소 체계이다.

<br>

**예시**: `https://www.example.com:443/path?query=123#section`

| 구성요소 | 예시 | 설명 |
| --- | --- | --- |
| **프로토콜**(Scheme) | https:// | 통신 방식, HTTPS는 HTTP의 보안 버전 |
| **호스트**(Host/Domain) | [www.example.com](http://www.example.com/) | 서버의 도메인 이름, DNS를 통해 IP로 변환됨 |
| **포트**(Port) | :443 | 논리적 연결 지점, HTTPS는 443, HTTP는 80 |
| **경로**(Path) | /path | 서버 내 자원의 위치 |
| **쿼리**(Query String) | ?query=123 | 서버에 전달할 추가 데이터, 키=값 형태, &로 구분 |
| **프래그먼트**(Fragment) | #section | 문서 내부 특정 위치, 서버로 전송되지 않음 |

<br>

### 2. 주소 입력 및 URL 분석

사용자가 브라우저 주소창에 **URL**을 입력한다.

**예시**: `https://www.example.com/path?query=123`

브라우저는 **URL**을 파싱하여 **프로토콜**, **호스트**, **포트**, **경로**, **쿼리**로 분리한다.

<br>

### 3. DNS 조회

**DNS 서버**에 도메인(`www.example.com`)의 **IP 주소**를 요청하고 응답받는다.

**예시**: `93.184.216.34`

<br>

### 4. TCP 연결 설정

브라우저는 **IP 주소**와 **포트(443)** 정보를 기반으로 서버에 **3-Way Handshake**를 수행하여 연결을 맺는다.

<br>

### 5. HTTP 요청 전송

브라우저는 **TCP** 위에서 **HTTP 요청**을 전송한다.

```
GET /path?query=123 HTTP/1.1
Host: www.example.com
```

이 요청은 서버에 `/path` 리소스를 요청하며, 쿼리 파라미터 `query=123`을 함께 전달한다.

<br>

### 6. 서버 처리 및 HTTP 응답

서버는 요청을 처리하고 해당 자원을 찾아 **HTTP 응답 메시지**로 반환한다.

```
HTTP/1.1 200 OK
Content-Type: text/html
```

응답 본문에는 요청한 웹페이지의 **HTML** 내용이 담겨 있다.

<br>

### 7. 브라우저 렌더링

브라우저는 **HTML → CSS → JavaScript** 순으로 해석하여 **DOM 트리**와 **렌더 트리**(Render Tree)를 생성하고 화면에 웹페이지를 렌더링한다.

<br>

### 8. 연결 종료 또는 재사용

**HTTP/1.1** 이상에서는 `Connection: keep-alive`로 **TCP 연결을 유지**할 수 있다. 모든 데이터 전송이 완료되면 **4-Way Handshake**를 통해 연결이 종료된다.

<br>

### 9. 전체 흐름 요약

1. **URL 입력** 및 파싱
2. **DNS 조회** → IP 주소 확인
3. **TCP 3-Way Handshake** 연결
4. **HTTP 요청** 전송
5. **서버 응답** 수신
6. **브라우저 렌더링**
7. 연결 종료 또는 재사용