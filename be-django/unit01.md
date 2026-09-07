## 요청·응답 사이클과 미들웨어

Django가 HTTP 요청을 받아 뷰를 실행하고 응답을 돌려주기까지의 **요청·응답 사이클(Request-Response Cycle)** 과, 그 사이에 끼어들어 공통 처리를 담당하는 **미들웨어(Middleware)** 의 동작 원리를 다룬다. 미들웨어의 실행 순서와 각 단계별 훅을 정확히 알아야 인증·로깅·예외 처리가 "왜 이 순서로 동작하는지" 설명할 수 있다.

<br>

### 1. 요청·응답 사이클 전체 흐름

Django 애플리케이션은 WSGI(Web Server Gateway Interface) 또는 ASGI(Asynchronous Server Gateway Interface) 서버 위에서 동작한다. 서버가 요청을 넘기면 Django의 **핸들러(Handler)** 가 `HttpRequest` 객체를 만들고, 미들웨어 체인을 거쳐 URL 라우팅 → 뷰 실행 → 응답 반환 순으로 처리한다.

```
클라이언트
   │  HTTP 요청
   ▼
WSGI/ASGI 서버 (gunicorn, uvicorn 등)
   │  environ / scope → HttpRequest 생성
   ▼
┌─────────────────────────────────────────────┐
│ 미들웨어 체인 (settings.MIDDLEWARE 순서)      │
│  M1 ─▶ M2 ─▶ M3 ─▶ URL 라우팅 ─▶ 뷰(View)    │
│  M1 ◀─ M2 ◀─ M3 ◀───────────── HttpResponse │
└─────────────────────────────────────────────┘
   │  HTTP 응답
   ▼
클라이언트
```

- **요청 단계**: `MIDDLEWARE`에 적힌 순서대로 **위에서 아래로** 통과함
- **응답 단계**: 뷰가 반환한 응답이 **아래에서 위로** 거슬러 올라감
- 뷰는 `HttpResponse`를 반환해야 하며, 템플릿 렌더링·JSON 직렬화(DRF의 `Response`)도 결국 `HttpResponse`의 하위 클래스임

> 💡 요청 객체 `HttpRequest`는 사이클 동안 **하나만 생성되어 공유**된다. 미들웨어가 `request.user`처럼 속성을 붙이면 이후 미들웨어와 뷰가 그대로 사용할 수 있다. `AuthenticationMiddleware`가 `request.user`를 채우는 것이 대표적인 예다.

<br>

### 2. 미들웨어의 구조

### 2-1. 기본 형태

Django 미들웨어는 `get_response`를 인자로 받는 **호출 가능 객체(callable)** 다. 서버 기동 시 한 번 `__init__`이 호출되고, 요청마다 `__call__`이 실행된다.

```python
import time
import logging

logger = logging.getLogger(__name__)


class TimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response  # 다음 미들웨어 또는 뷰

    def __call__(self, request):
        # ── 요청 단계: 뷰 실행 전 ──
        start = time.perf_counter()

        response = self.get_response(request)  # 다음 단계로 위임

        # ── 응답 단계: 뷰 실행 후 ──
        elapsed_ms = (time.perf_counter() - start) * 1000
        response["X-Elapsed-Ms"] = f"{elapsed_ms:.1f}"
        logger.info("%s %s %.1fms", request.method, request.path, elapsed_ms)
        return response
```

- `self.get_response(request)` **호출 전** 코드가 요청 단계, **호출 후** 코드가 응답 단계임
- `get_response`는 다음 미들웨어의 `__call__`이며, 마지막 미들웨어의 `get_response`는 URL 라우팅과 뷰 실행을 담당하는 핸들러 내부 함수임
- `settings.py`의 `MIDDLEWARE` 리스트에 경로 문자열로 등록함

<br>

### 2-2. 양파 구조와 실행 순서

미들웨어는 서로를 감싸는 **양파(onion) 구조**로 조립된다. 기동 시 Django는 `MIDDLEWARE` 리스트를 **역순으로** 순회하며 `M3(뷰)`, `M2(M3)`, `M1(M2)` 순으로 인스턴스를 만들기 때문에, 실행 시에는 리스트 순서대로 바깥에서 안쪽으로 들어간다.

```
MIDDLEWARE = [Security, Session, Auth]   (위 → 아래)

요청:  Security.__call__ ─▶ Session.__call__ ─▶ Auth.__call__ ─▶ 뷰
응답:  Security          ◀─ Session          ◀─ Auth          ◀─ 뷰
```

❗️**순서가 곧 의존성이다**: `AuthenticationMiddleware`는 `request.session`이 필요하므로 반드시 `SessionMiddleware` **아래**에 있어야 한다. 순서를 바꾸면 `AssertionError`나 `AttributeError`가 발생한다.

<br>

### 3. 단계별 훅(Hook)

`__call__` 하나로 대부분 처리할 수 있지만, Django는 더 세밀한 개입 지점을 위해 특수 메서드를 정의해 두었다. 미들웨어 클래스에 이 이름의 메서드가 있으면 핸들러가 자동으로 호출한다.

| **훅**                       | **호출 시점**                              | **시그니처**                                        | **반환값**                                   |
| ---------------------------- | ------------------------------------------ | --------------------------------------------------- | -------------------------------------------- |
| **`process_view`**           | URL 라우팅 후, **뷰 실행 직전**            | `(request, view_func, view_args, view_kwargs)`      | `None`(계속) 또는 `HttpResponse`(뷰 건너뜀)  |
| **`process_exception`**      | 뷰가 **예외를 던졌을 때**                  | `(request, exception)`                              | `None`(계속 전파) 또는 `HttpResponse`        |
| **`process_template_response`** | 뷰가 `TemplateResponse`를 반환했을 때  | `(request, response)`                               | `response` (렌더링 전이므로 컨텍스트 수정 가능) |

- `process_view`는 요청 단계이므로 **위에서 아래** 순서로 호출됨
- `process_exception`과 `process_template_response`는 응답 단계이므로 **아래에서 위** 순서로 호출됨
- `process_view`에서 `HttpResponse`를 반환하면 그 이후의 `process_view`와 뷰는 실행되지 않고, 응답 단계로 바로 넘어감

```python
class BlockBannedIPMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        return self.get_response(request)

    def process_view(self, request, view_func, view_args, view_kwargs):
        # 라우팅이 끝난 뒤라 어떤 뷰가 실행될지 알 수 있음
        if getattr(view_func, "public", False):
            return None  # 공개 뷰는 통과
        if request.META.get("REMOTE_ADDR") in BANNED_IPS:
            return HttpResponseForbidden("차단된 IP입니다.")
        return None

    def process_exception(self, request, exception):
        if isinstance(exception, PermissionDenied):
            return JsonResponse({"detail": "권한 없음"}, status=403)
        return None  # 다른 예외는 다음 미들웨어로 전파
```

> 💡 `process_view`는 **뷰 함수 객체를 인자로 받는다**는 점이 `__call__`과 다르다. CSRF 미들웨어가 `@csrf_exempt` 뷰를 건너뛰는 것도 `process_view` 안에서 `view_func`의 속성을 확인하기 때문이다.

<br>

### 4. 단락(Short-circuit)과 예외 처리 흐름

요청 단계에서 어떤 미들웨어가 `get_response`를 호출하지 않고 바로 응답을 반환하면, **그 안쪽 미들웨어와 뷰는 실행되지 않는다**. 이를 단락(short-circuit)이라 한다.

```
M1 ─▶ M2 (여기서 401 응답 반환, get_response 미호출)
M1 ◀─ M2                       M3·뷰는 실행 안 됨
```

**예외가 발생했을 때의 흐름**

```
뷰에서 예외 발생
   │
   ▼
가장 안쪽 미들웨어의 process_exception 부터 바깥쪽으로 호출
   │  누군가 HttpResponse를 반환하면 → 그 응답으로 응답 단계 진행
   │  아무도 처리하지 않으면
   ▼
핸들러가 예외를 잡아 500(또는 Http404 → 404, PermissionDenied → 403) 응답 생성
   │
   ▼
바깥쪽 미들웨어의 응답 단계는 정상 실행됨
```

- `__call__` 안에서 `self.get_response(request)`를 `try/except`로 감싸도 뷰의 예외를 **직접 잡을 수 없다**. 핸들러가 `process_exception`을 거친 뒤 예외를 응답으로 변환해 돌려주기 때문임
- 따라서 예외를 응답으로 바꾸려면 `process_exception`을 사용해야 함

> ⚠️ 응답 단계 코드는 `get_response`가 **항상 응답 객체를 돌려준다**는 전제로 작성한다. 다만 `Http404`·`PermissionDenied`처럼 Django가 변환한 응답은 `response.status_code`가 404·403이므로, "응답이 왔다 = 성공"으로 가정하면 안 된다.

<br>

### 5. 기본 미들웨어의 역할과 권장 순서

Django 5.x 프로젝트를 `startproject`로 생성하면 아래 순서로 등록된다. 각 미들웨어의 위치에는 이유가 있다.

| **미들웨어**                   | **역할**                                          | **위치 이유**                                              |
| ------------------------------ | ------------------------------------------------- | ---------------------------------------------------------- |
| **SecurityMiddleware**         | HTTPS 리다이렉트, HSTS·보안 헤더                  | **가장 위**: 다른 처리 전에 리다이렉트로 단락시켜야 함     |
| **SessionMiddleware**          | 쿠키에서 세션 로드, 응답에 세션 저장              | Auth·CSRF보다 위: `request.session` 제공                   |
| **CommonMiddleware**           | `APPEND_SLASH`, `DISALLOWED_USER_AGENTS` 등       | 라우팅 전 URL 정규화                                       |
| **CsrfViewMiddleware**         | CSRF 토큰 검증 (`process_view`에서 수행)          | Session 아래, 인증 위(세션 기반 토큰 사용)                 |
| **AuthenticationMiddleware**   | `request.user` 채움 (지연 로딩)                   | Session 아래 필수                                          |
| **MessagesMiddleware**         | 1회성 플래시 메시지                               | Session·Auth 아래                                          |
| **XFrameOptionsMiddleware**    | `X-Frame-Options` 헤더로 클릭재킹 방지            | 응답 헤더만 추가하므로 위치 자유                           |

- Django 5.1부터는 모든 뷰에 로그인을 강제하는 **`LoginRequiredMiddleware`** 가 추가되었으며, `AuthenticationMiddleware` 아래에 두어야 함
- 사용자 정의 미들웨어는 "무엇에 의존하는가"를 기준으로 위치를 정함. `request.user`가 필요하면 Auth 아래, 모든 요청을 차단·리다이렉트해야 하면 위쪽에 둠

<br>

### 6. 동기·비동기 미들웨어

Django 3.1 이후 미들웨어는 동기·비동기 요청 경로를 모두 지원할 수 있다. 클래스 속성 `sync_capable`·`async_capable`로 지원 범위를 선언하며, 경로가 맞지 않으면 Django가 스레드 전환으로 **자동 적응(adapt)** 시킨다.

```python
from asgiref.sync import iscoroutinefunction, markcoroutinefunction


class HybridMiddleware:
    sync_capable = True
    async_capable = True

    def __init__(self, get_response):
        self.get_response = get_response
        if iscoroutinefunction(get_response):
            markcoroutinefunction(self)  # 비동기 경로임을 표시

    def __call__(self, request):
        if iscoroutinefunction(self):
            return self.__acall__(request)
        response = self.get_response(request)
        return response

    async def __acall__(self, request):
        response = await self.get_response(request)
        return response
```

> ⚠️ ASGI 서버에서 **동기 전용 미들웨어**가 하나라도 끼어 있으면 그 지점마다 스레드 전환이 일어나 비동기 뷰의 이점이 줄어든다. 자세한 동기·비동기 혼용 규칙은 unit09를 참고할 것.

<br>

### 7. 면접·실무 체크포인트

| **질문**                                                   | **핵심 답변**                                                                                       |
| ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **미들웨어 실행 순서는?**                                   | 요청 단계는 `MIDDLEWARE` **위→아래**, 응답·예외 단계는 **아래→위** (양파 구조)                       |
| **`__call__`과 `process_view`의 차이는?**                   | `process_view`는 **라우팅 후** 호출되어 `view_func`를 알 수 있고, 응답 반환 시 뷰를 건너뜀           |
| **뷰의 예외를 미들웨어에서 잡으려면?**                       | `__call__`의 try/except가 아니라 **`process_exception`** 사용. 안쪽부터 바깥쪽 순서로 호출됨          |
| **AuthenticationMiddleware가 Session 아래여야 하는 이유는?** | `request.session`에 의존하기 때문. 순서는 곧 **의존성**임                                            |
| **단락(short-circuit)이란?**                                | `get_response`를 호출하지 않고 응답을 반환해 **안쪽 미들웨어와 뷰를 생략**하는 것                     |
| **미들웨어 `__init__`은 언제 호출되나?**                     | 서버 **기동 시 1회**. 요청마다 초기화가 필요한 상태는 `__call__` 안에 두어야 함                       |

- 미들웨어는 "모든 요청에 공통으로 적용되는 횡단 관심사"에만 사용하고, 특정 뷰에만 필요한 로직은 데코레이터나 DRF의 인증·권한 클래스(unit06 참고)로 처리함
- 미들웨어에서 DB를 조회하면 **모든 요청**에 쿼리가 추가되므로, `SimpleLazyObject`처럼 지연 평가하거나 캐시를 활용해 비용을 줄임
